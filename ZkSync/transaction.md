# Transaction提交流程
此处以zargo中的call为例，因为这个操作比较全面。
## 数据获取
所有的数据都需要从合约编译好的文件中获取。其中“input.json”，”private_key“两个文件需要填入对应的函数参数和私钥。

* 必须获取的有msg信息和method方法及其参数。例如：
```
"msg": {
    "sender": "",
    "recipient"： ""，
    "token_address": ""
    "amount": ""
},
"arguments": {
    "get_fee": {},
    "new": {
        "_fee": "0"
    },
    "exchange": {
        "withdraw_token": "0x0"
    },
    "deposit": {}
}
```
* private_key需要是已经在zkSync的L2上激活的账户私钥。[激活方法](https://zinc.zksync.io/07-smart-contracts/04-troubleshooting.html)
## 计算流程
* 通过private_key可以以及后续调用时输入的网络，计算出钱包凭证。
```
pub struct WalletCredentials<S: EthereumSigner> {
    pub(crate) eth_signer: Option<S>,
    pub(crate) eth_address: Address,
    pub(crate) zksync_private_key: Privatekey,
}
```
* 根据钱包凭证从zkSync的指定网络上获取临时钱包： `wallet`, 该钱包可以用于后续签名。
* 结合读取的msg和wallet得到初始的Transaction(此处计算时fee一项为None)，用于从网络上获取该交易需要的基础费用fee。
```
let transaction = crate::transaction::try_into_zksync(msg.clone(), &wallet, None).await>;
// 此处address为合约地址，method和arguments是input.json中读取的。
let response = http_client
.fee(
    zinc_types::FeeRequestQuery::new(address, method.clone()),
    zinc_types::FeeRequestBody::new(arguments.clone(), transaction),
)
.await?
```
* 再根据msg, wallet, fee计算出实际的transaction。注意，此处传入的只是基础费用，在计算时还会加上tx fee，该值也是从网络获取的。
* 利用http服务进行call。
```
let response = http_client
.call(
    zinc_types::CallRequestQuery::new(address, method),
    zinc_types::CallRequestBody::new(arguments, transaction),
)
.await?
```
## 结构分析
主要来看一下Transaction的结构
```
pub struct Transaction {
    pub tx: ZkSyncTx,
    pub ethereum_signature: Option<EthereumSignature>,
}
```
其中，ZkSyncTx是定义在zkSync中的实际Transaction结构
```
pub enum ZkSyncTx {
    Transfer(Box<Transfer>),
    Withdraw(Box<Withdraw>),
    Close(Box<Close>),
    ChangePubKey(Box<ChangePubKey>),
    ForcedExit(Box<ForcedExit>)
}
// 此处只以Transfer为例，其它类似
pub struct Transfer {
    pub account_id: AccountId,
    pub from: Address,
    pub to: Address,
    pub token: TokenId,
    pub amount: BigUint,
    pub fee: BigUint,
    pub nonce: Nonce,
    pub signature: TxSignature,
    cached_signer: VerifiedSignatureCache,
}
```
可见transaction结构和以太坊非常类似。signature就是通过私钥对交易信息的签名结果。

再看EthereumSignature结构,显然这是一个以太坊签名结果。
```
pub struct EthereumSignature {
    /// The default signature type.
    pub r#type: String,
    /// The signature as a hex string.
    pub signature: PackedEthSignature,
}

pub struct PackedEthSignature(ETHSignature);

pub struct Signature([u8; 65]);
```
总的来说，对一个transaction进行的签名同时包含了zkSync签名的结果和以太坊签名结果。这些组合起来上传到了zkSync服务器中进行处理。

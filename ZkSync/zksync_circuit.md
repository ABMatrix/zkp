#Zksync Circuit Overview

主要是从几个方面对zksync实现的电路进行梳理（暂不包含Zksync中与聚合证明 *AggregatedProof* 相关的电路），以确保在更改协议之后可以有效地对电路实现的约束进行修改。

### Circuit -> R1CS Implementation
Zksync的电路是基于Bellman的电路进行实现，因此对是通过实现 *Circuit* 下的 *synthesize* 函数来满足电路的约束编写
```
fn synthesize<CS: ConstraintSystem<E>>(self, cs: &mut CS) -> Result<(), SynthesisError>
```

而在电路中实现约束系统，基础类型是Variable。在Zksync之中，它通过对Variable进行一层包装以 *AllocatedNum* 进行表示，而 *CircuitElement* 则是 *AllocatedNum* 的封装，包含了更多的信息。
```
    pub struct AllocatedNum<E: Engine> {
        value: Option<E::Fr>,
        variable: Variable
    }

    pub struct CircuitElement<E: Engine> {
        number: AllocatedNum<E>,
        bits_le: Vec<Boolean>,
        length: usize,
    }   
```

首先，需要明确电路是一个或者多个约束的集合，即在 *synthesize* 函数中需要保证约束成立并将约束转换为R1CS的表达形式，再去产生statement的相关证明。根据Bellman定义的 *Constraint System*，对于需要被确保约束，会通过调用 *enforce* 这个函数来构建R1CS。通过举例说明会更加清楚
> 约束 a+b=c
>```
>cs.enforce(
>    || "a*b=c",
>    |lc| lc + a + b,
>    |lc| lc + CS::one(),
>    |lc| lc + c
>);
>```
> 约束 a*b=c
>```
>cs.enforce(
>    || "a*b=c",
>    |lc| lc + a,
>    |lc| lc + b,
>    |lc| lc + c
>);
>```

### Logic of Zksync Circuit
确认电路需要的参数：
```
pub struct ZkSyncCircuit<'a, E: RescueEngine + JubjubEngine> {
    pub rescue_params: &'a <E as RescueEngine>::Params,
    pub jubjub_params: &'a <E as JubjubEngine>::Params,
    /// The old root of the tree
    pub old_root: Option<E::Fr>,
    pub initial_used_subtree_root: Option<E::Fr>,

    pub block_number: Option<E::Fr>,
    pub validator_address: Option<E::Fr>,
    pub block_timestamp: Option<E::Fr>,

    pub pub_data_commitment: Option<E::Fr>,
    pub operations: Vec<Operation<E>>,

    pub validator_balances: Vec<Option<E::Fr>>,
    pub validator_audit_path: Vec<Option<E::Fr>>,
    pub validator_account: AccountWitness<E>,
}
```
唯一公开输入：pub_data_commitment

需要保证的约束（即通过调用`cs.enforce()`生成R1CS）：
    
1. 账户所有权验证：包含所有账户的最小树根 *initial_used_subtree_root* 确实是树根 *old_root* 的子树
```
let old_root = AllocatedNum::alloc(cs.namespace(|| "old_root"), || self.old_root.grab())?;
let mut rolling_root = {
    ...
    cs.enforce(
        || "old_root contains initial_used_subtree_root",
        |lc| lc + old_root_from_subroot.get_variable(),
        |lc| lc + CS::one(),
        |lc| lc + old_root.get_variable(),
    );
    ...
}
```

2. 交易数据有效性验证：验证每次交易前账户数据对应的 *state_root* 与先前根据交易数据计算得到的 *rolling_state* 是否一致以及对于一个token而言其交易是否成功执行
```
// Main cycle that processes operations:
for (i, operation) in self.operations.iter().enumerate() {
    ...
    self.select_op(
        cs.namespace(|| "execute op"),
        ...
    );
    ...   
}
```

3. 交易完整性验证：验证计算后的 *chunk_data* 是否为当前交易的最后一个chunk，即说明最后一个operation是完整的
> · 一次交易会分割成多个operation
> · 每次operation对应的数据 *pub_data* 会被分成多个 *chunk*
```
cs.enforce(
    || "ensure last chunk of the block is a last chunk of corresponding transaction",
    |_| {
        global_variables
            .chunk_data
            .is_chunk_last
            .lc(CS::one(), E::Fr::one())
    },
    |lc| lc + CS::one(),
    |lc| lc + CS::one(),
);
```
    
4. Validator交易验证：验证支付Validator交易费用后的新的状态根的一致性
    
5. 公开输入的Commitment验证：多次哈希迭代验证
![Public data Commitment Verification](https://mmbiz.qpic.cn/mmbiz_png/IXicdJl0t7b7zzVGN5Y9bcql4TB9NgbAkMMouRB3mayJAribia261ZhZgbvk4sGYeakHkdyzUVI4A97mJaVq5icRaw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### Circuit of Operation
<font size=2>**NFT相关：**</font>:考虑到nft结构修改，主要是在相关operation上会影响到电路修改，即涉及上一章的第2块逻辑约束。

每个operation的复原操作都由*execute_op*这个函数进行调用

· cur: 代表存款账户，其中包含存款账户信息
```
cur: &mut AllocatedOperationBranch<E>
```
· op_data: 代表存款内容以及交易手续费
```
op_data: &AllocatedOperationData<E>
```

对于每一次的execute_op，需要保证至少有一种operation是正确的，而每个operation的具体操作正确与否需要结合一些条件进行判断
```
let op_valid = multi_or(cs.namespace(|| "op_valid"), &op_flags)?;

Boolean::enforce_equal(
    cs.namespace(|| "op_valid is true"),
    &op_valid,
    &Boolean::constant(true),
)?;
```

##### Deposit
需要验证的逻辑：
1. 对于交易信息的pubdata的还原并验证pubdata是否完全还原或者已经更新到一个新的chunk
```
let pubdata_properly_copied = boolean_or(
    cs.namespace(|| "first chunk or pubdata is copied properly"),
    &is_first_chunk,
    &is_equal_pubdata,
)?;
```
2. 验证deposit的手续费是否为0，关于元素0则需要注意其使用方式
```
&global_variables.explicit_zero.get_number(),
```
3. 验证还原后的pubdata_chunk与chunk_number是否与预期的chunk_number相同
4. 验证交易的操作码*tx_code*确实是deposit
5. 验证交易地址是否与交易账户地址相同或是交易账户地址为空
> (op_data.eth_address == cur.account.address) ^ is_accout_empty
6. 验证交易数额是否正确
```
//verify correct amounts
let is_a_correct = CircuitElement::equals(
    cs.namespace(|| "a == amount"),
    &op_data.full_amount,
    &op_data.a,
)?;
```

<font color="#dd0000">总结：</font>1-6条为所有约束，如果全为真则代表交易有效
7. 如果在交易有效的前提下且chunk为first_chunk则计算当前账户更新后的balance，并更新账户地址
```
let is_first_chunk = Boolean::from(Expression::equals(
    cs.namespace(|| "is_first_chunk"),
    &global_variables.chunk_data.chunk_number,
    Expression::constant::<CS>(E::Fr::zero()),
)?);
```

<font size=2>**NFT相关**</font>: 主要是第7步中更新balance需要修改，因为需要确定nft类型而不是单单获取数字

##### Transfer
需要对于lhs, rhs有所了解，这代表transfer的两个账户，而cur对应的账户是二者之一，具体是根据chunk_number进行决定。在对约束进行判断之前需要还原交易的公开数据以及以两种方式序列化交易比特，具体差别在是否存在有效期：
```
serialized_tx_bits.extend(op_data.valid_from.get_bits_be());
serialized_tx_bits.extend(op_data.valid_until.get_bits_be());
```

需要验证的逻辑：
1. 确认还原后的chunk_number与预期的chunk_number相同
2. 验证交易的操作码*tx_code*确实是transfer
3. timestamp是否正确
4. 对于还原后交易信息的pubdata验证是否完全还原或者已经更新到一个新的chunk
分别为lhs和rhs两个账户的一些逻辑进行验证:
*lhs*:
    1. 是否为first_chunk（根据定义，交易是first_chunk时，cur为lhs）
        ```
        let mut current_branch = self.select_branch(
            cs.namespace(|| "select appropriate branch"),
            &lhs,
            &rhs,
            operation,
            &global_variables.chunk_data,
        )?;
        ```
    2. 验证当前lhs与cur账户余额相同
    3. 交易的费用与计算费用是否相同
        ```
        let sum_amount_fee = Expression::from(&op_data.amount_unpacked.get_number())
            + Expression::from(&op_data.fee.get_number());

        let is_b_correct = Boolean::from(Expression::equals(
            cs.namespace(|| "is_b_correct"),
            &op_data.b.get_number(),
            sum_amount_fee.clone(),
        )?);
        ```
    4. 验证当前账户的余额确实大于交易费用+手续费
    5. 确认签名的正确性以及当前账户的nonce并未溢出
    6. 确认序列化后的交易格式的正确性

&emsp;&emsp;*rhs*:
    &emsp;&emsp;&emsp;1. 确认当前chunk_number是第二个chunk
    &emsp;&emsp;&emsp;2. 确认账户不为空

<font color="#dd0000">总结：</font>不管对于lhs账户还是rhs账户，1-4条都必须保持正确。对lhs账户而言，其1-6条必须保持正确才可以更新其对应账户余额和账户nonce；对rhs账户而言，其1-2条必须保持正确才可以更新其对应账户余额。
```
...
// calculate new lhs balance value
let updated_balance = Expression::from(&cur.balance.get_number()) - sum_amount_fee;
...
// calculate new rhs balance value
let updated_balance = Expression::from(&cur.balance.get_number())
    + Expression::from(&op_data.amount_unpacked.get_number());
...
```
<font size=2>**NFT相关：**</font>同样，肯定需要修改更新账户余额的步骤；nft交易过程中不仅仅涉及到nft的交易，还有手续费的交易，二者显然无法通过这一个*updated_balance*进行表示，因此需要分开处理流程。

##### Transfer_to_new
针对lhs账户的逻辑与Transfer一致
针对rhs账户，在Transfer的逻辑上新增了修改当前账户地址的选项：
```
cur.account.address = CircuitElement::conditionally_select(
    cs.namespace(|| "mutated_pubkey"),
    &op_data.eth_address,
    &cur.account.address,
    &rhs_valid,
)?;
```
但是除了lhs与rhs账户，新增了一个ohs的选项，针对于chunk不为1或者2的对象
```
...
ohs_valid_flags.push(is_first_chunk.not());
ohs_valid_flags.push(is_second_chunk.not());
...
```
<font color="#dd0000">总结：</font>逻辑与Transfer类似，ohs的含义代表了数据正确但chunk尚未传输完毕。
<font size=2>**NFT相关：**</font>与Transfer类似，只有在涉及lhs与rhs才存在与balance相关的逻辑修改，因此仍需要处理这部分逻辑。

##### Withdraw
需要验证的逻辑：
1. 对于还原后交易信息的pubdata验证是否完全还原或者已经更新到一个新的chunk
2. 确认还原后的chunk_number与预期的chunk_number相同
3. 验证交易的操作码*tx_code*确实是withdraw
4. timestamp是否正确
5. 确认签名有效或并非first_chunk(判断是执行ohs还是lhs部分逻辑)
```
let is_sig_correct = multi_or(
    cs.namespace(|| "sig is valid or not first chunk"),
    &[is_signed_correctly, is_first_chunk.not()],
)?;
```
6. 确认序列化是否正确
分别为lhs和ohs两个部分进行逻辑验证:
*lhs*:
    1. 是否为first_chunk（根据定义，交易是first_chunk时，cur为lhs）
    2. 验证当前lhs与cur账户余额相同
    3. 交易的费用与计算费用是否相同
    5. 验证当前账户的余额确实大于交易费用+手续费
    6. 确认当前账户的nonce并未溢出

&emsp;&emsp;*ohs*:ohs在确保共同前提的基础上验证是否不为first_chunk
```
ohs_valid_flags.push(is_first_chunk.not());
```
<font color="#dd0000">总结：</font>逻辑与deposit类似，仍需要注意只有lhs部分逻辑才会更新balance和nonce；不管对于lhs还是ohs逻辑，共同的1-6条都是需要保证正确的，而lhs因为存在更新账户状态所以需要验证更多逻辑；只有在lhs或是ohs其中一个逻辑成立的情况下整个操作才能称之为有效。
```
let tx_valid = multi_or(
    cs.namespace(|| "tx_valid"),
    &[lhs_valid.clone(), is_ohs_valid],
)?;
```
<font size=2>**NFT相关：**</font>与Deposit类似，只有在涉及lhs才存在与balance相关的逻辑修改，因此仍需要处理这部分逻辑。

### Plonk Proof
Plonk算法的证明是在 *PlonkStepByStepProver* 下的方法调用实现（不考虑聚合证明）
```
 fn create_single_block_proof(
    &self,
    witness: zksync_circuit::circuit::ZkSyncCircuit<'_, Engine>,
    block_size: usize,
) -> anyhow::Result<SingleProof>
```
其中 *block_size* 是表示电路大小的参数，用于确定Plonk算法的初始参数和 *verifying key* 大小。

在证明生成阶段，存在一个预处理过程，会根据电路大小从setup的crs中读取合适的预计算数据供后续证明使用，并在此过程中将对应于R1CS的电路通过函数转换为Plonk所支持的电路类型。
```
pub fn prepare_setup_for_step_by_step_prover<C: Circuit<Engine> + Clone>(
    circuit: C,
    download_setup_file: bool,
) -> Result<Self, anyhow::Error> {
    let hints = transpile(circuit.clone())?;
    let setup_polynomials = setup(circuit, &hints)?;
    ...
}
```
在转换之后，会获得Plonk所需的电路的门系数多项式和sigma函数（置换函数）。

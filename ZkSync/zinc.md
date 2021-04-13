## 编译环境安装配置
* 下载[链接](https://github.com/matter-labs/zinc/releases/download/0.2.3/zinc-0.2.3-linux.tar.gz)
* 解压
`tar -xzvf zinc-0.2.3-linux.tar.gz`
* 将路径添加到环境变量中
`vim ~/.bashrc`
* 添加路径："export PATH=(文件路径)/zinc-0.2.3-linux:$PATH"到.bashrc最后一行。
* source环境
`source ~/.bashrc`

## zargo
* zargo类似于cargo，是zinc的编译工具，包含各种编译，上传，下载功能。注意，几乎所有的上传，下载等都需要指明特定网络。

* 这里特别吐槽一下目前的依赖功能。

    * 第一，dependencies中只能写已经上传到特定网络(如rinkeby)的库。
    * 第二，如果创建的是library，那么无法依赖contract。
    * 第三，如果要知道有那些合约可以依赖，需要自行下载查看且不区分library和contract，如：
    `zargo download --network rinkeby --list > dependencies.txt`
    * 第四，没有一个好的浏览工具查看各种可依赖库，没有document系统来查阅可用函数等。

## 文档链接
[语法文档](https://zinc.zksync.io/index.html)

## 语法等注意点

*整型支持u8到u248, i8到i248之间以8位为间隔单位的任意长度。如：u40，i48等。(这一点沿用Solidity)
特殊的类型field，长度位254位，类似于整型，可以相互转化，与椭圆曲线运算相关。
field类型不支持除法，取余，位运算。

*目前string类型只有在dbg！宏和require()函数中用到，作为错误返回值。

*目前match语法只支持简单类型如enum， integer等，不支持array，tuple，struct。

*enum类型沿用了c语言的定义, 每一项需要编号，如：
```
enum Order {
    FIRST = 0,
    SECOND = 1,
}
```
因此enum可以作为整数使用，可以与integer相互转化。
如：
```
let a = 1;
let b = a as Order;
let c: u8 = Order::FIRST;
```

*MTreeMap类型只能在contract中使用。

*除法使用欧几里得除法运算。特别在负数运算时注意：如-45 % 7 = 4。此处有：-7 * 7 = -49，则-45 - （-49） = 4。
因为很少用到负数对正数取余，所以这一点在平常用别的语言时也很少注意到。

*逻辑运算中除了||, &&, !之外还有个^^, ^^代表异或，类似位运算中的^。

*可以定义复杂的静态类型：
```
type ComplexType = [(u8, [bool;8], field);16]
fn example(data: ComplexType) {}
```

*整数可以用“伪小数”表示法表示。比如WEI单位和比特币：1.0_E18, 0.001_E8, 42_E6。

*for-while语法要点，先看这段代码： 
```
fn main() {
    let x = 7;
    for i in 0..10 while i % x != 2 {
        dbg!("{}", i);
    }
}
```
输出结果为 0， 1
要点：
1. 循环范围只能用常数表示，不能用变量。如for i in 0..a 这种写法就不行。
2. for循环无法提前返回，因此需要while作为抑制器，来提前结束循环。(zinc语言是图灵不完备的)

## contract

### 隐式存储域
每一个contract默认存在的存储域，对于一个空的contract Empty{},其实际代码是这样的：
```
contract Empty {
    pub address：u160
    pub balances: std::collections::MTreeMap<u160, u248>
}
```
address：zkSync token address，
balances：address->balances

### 显式存储
和普通的定义方法一样，分为public和private, 具体看文档。

### 测试合约
代码为文档中的基本代码[link](https://zinc.zksync.io/07-smart-contracts/02-minimal-example.html)

部署之前，需要先激活zkSync钱包，具体流程[link](https://zinc.zksync.io/07-smart-contracts/04-troubleshooting.html)
部署合约
`cargo build`
在./data/input.json中部分代码设为如下：
```
...
    "new": {
        "_fee": "30"
    }
...
```
设置手续费为30/10000即0.3%
在private_key文件中，将自己的以太坊账户private_key写入。接着:
`zargo publish --network rinkeby --instance default`
返回结果：
* Address 0xc251936d118b3f7b59f1782df5fde3ed0d719ab5
* Account ID 126842

接着根据文档依次输入参数，调用函数实现调用和查询功能。
(该合约只能进行deposit和exchange，deposit进去的只能用另一种币换出来，上当了......)

## 总结
该语言语法十分类似Rust，在结构设计上又参考了Solidity，熟悉这两种语言的可以学的很快。整体上来说，基本功能都差不多有，但是还
需要大量的库做支持才能发展好。std库的功能感觉都不够多。

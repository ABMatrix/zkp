# Withdraw方法

## 普通情况下的withdraw方法
方法：withdraw

    发起层：L2
    
    作用：将L2上的钱转到L1上，需要zksync正常运行。


方法：ForcedExit

    发起层： L2
    
    作用：为某些无法设置singing key的账户（如合约账户）从L2上取回钱，费用由调用者承担，因此需要一个可用账户来执行。
    
## 特殊情况下的withdraw方法(如zksync宕机)
方法：FullExit

    发起层：L1
    
    作用：发起退款要求，作用于L1上的合约，生成一个PriorityRequest并插入到PriorityRequest列表中，拥有优先执行权。
    
当zksync宕机后，若PriorityRequest列表中存在超过期限未执行的request，则可以调用合约特定方法进入exodus mode。

进入该模式后，所有用户利用自己的proof来进入exodus状态，然后调用exit再调用WithdrawETH和WithdrawErc20取回自己的所有钱。

具体工具和方法可查看[参考文档](https://github.com/ABMatrix/zksync/tree/develop/infrastructure/exit-tool)

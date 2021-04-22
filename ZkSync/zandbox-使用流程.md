# Zinc接口


## 合约相关

一笔Swap的Call交易，MSG中的`msg.token_address`将被转成

`tokenname`与`arguments`中第一个元素组成`pair`
 
<br />
 
比如:
`let pair_name = format!("{}/{}",msg.token_address.to_tokenname(),arguments[0]);`

按这个形式，假设"ETH"和"SFF"两种币分别填在

`arguments`第一个参数 和 msg里的`token_address`位置
 <br />
 
将会得到pair: `"ETH/SFF" ` 
 
 <br />
交易数量取的是 `msg.amount`

<br />
 
<br />
  
##  外部接口相关
 <br />
 
### 1 .查询pair组合
1.1. `http://8.210.23.214:4001/api/v1/swap/query`方法外加对应参数

 <br />
 
### 2. 添加新的 tokenname 和 address 配对
2.1.首先用上面的`/query`方法向zandbox查询相关pair组合

2.2.将查询到的pair每种的token，例如`"ETH"`字符串和他对应地址
address 例如`"0x12345645464"`建立映射

比如调用`http://8.210.23.214:4001/api/v1/swap/add_token_name?tokenname=ETH&address=0x1265464454`

来添加 地址和币名的 映射


 <br />
 
 eg `http://8.210.23.214:4001/api/v1/swap/add_token_name?tokenname=PPP&address=0xafsfasf1199999afdfsdfafd`
 
 return: `200ok`
 
  <br />
  
### 3.查询24h信息
输入 pair name 比如"ETH/BTC"

`http://8.210.23.214:4001/api/v1/swap/get_data?name=ETH/BTC`

返回

```
{
    "amount": 151550174,
    "count": 15,
    "fee": 1583
}
```

参数意义: 

amount - 24h成交额  

count - 24h成交笔数

fee - 24h手续费

 <br />
 
 可用测试案例:

假设pairname 是 123

`http://8.210.23.214:4001/api/v1/swap/get_data?name=123`

返回

```
{
    "amount": 50300000000,
    "count": 1,
    "fee": 17
}
```

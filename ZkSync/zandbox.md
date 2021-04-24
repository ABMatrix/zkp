# Zandbox

服务端口： 4001

## 启动服务
zinc代码中的.run.sh没法编译通过，问题未解决，
所以，先cargo build --release 编译出zandbox。

启动时需指定数据库，最好是分库，然而这里为了简便，直接使用plasma数据库与zksync共用。（后续操作在zandbox目录中）

修改.env，将DATABASE_URL指定到plasma数据库：
DATABASE_URL=postgres://postgres@localhost/plasma

创建表：
sqlx migrate run

启动：
./target/release/zandbox -vvv --network localhost --postgresql postgres://postgres@localhost/plasma

# IoT-Device-Rental-Platform (IDRP)

A DeFi-based IoT device rental platform, with hierarchical authentication and access control features

`IDRP`是一个基于去中心化金融（DeFi）的物联网租赁平台，计划实现分级身份认证和访问控制。

## 0. 实验环境、编译和部署

### 0.1. 服务端环境配置

`IDRP`使用的`solidity`的版本是`v0.5.17+commit.d19bba13`；使用`truffle`变异和部署；使用`Ganache`提供以太坊测试网。

启动`Ganache`，请遵循下面的步骤配置：

1. 新建`WORKSPACE`，把`truffle-config.js`添加到`TRUFFLE PROJECTS`；
2. 在`SERVER`，你可以自行配置，**如果你想在容器运行客户端，那么请把`HOSTNAME`配置成`0.0.0.0`**；
3. 在`HARDFORK`，请把`HARDFORK`设置成`Petersburg`，关于`GAS`，本项目使用缺省配置。

在本仓库目录下执行下列命令，把合约部署到以太坊测试网：

```bash
$ rm -rf build # 删除过时的编译产物
$ truffle migrate # 把合约部署到以太坊测试网
```

### 0.2. 客户端环境配置

在`client`目录下，我们提供了环境配置文件模版`environment.yaml.template`，请从它出发完成客户端环境配置：

```bash
$ mv environment.yaml.template environment.yaml
```

请遵循注释修改`environment.yaml`。

## 1. 租金池

### 1.0. 租金池合约调试

我们的目标是正常地执行对租金池合约`ReceiveETH`和`PayETH`函数的外部调用。

我容器化了测试文件`ReceiveETH_test.js`和`PayETH_test.js`，你可以借助Docker运行它们。

我们可以这样测试`ReceiveETH`函数：

```bash
$ docker build -f Dockerfile.ReceiveETH.test -t idrp-receive-eth:testing .
$ docker run -p 0.0.0.0:7545:7545 idrp-receive-eth:testing
```

我们可以这样测试`PayETH`函数：

```bash
$ docker build -f Dockerfile.PayETH.test -t idrp-pay-eth:testing .
$ docker run -p 0.0.0.0:7545:7545 idrp-pay-eth:testing
```

### 1.1. 租金池合约的目标

`IDRP`仅支持存取以太币（ETH）。租金池合约的目标如下：

1. 用户操作简单，按需存款，活期取款；
2. 减少非必要开销，压缩合约对外流水，节省gas开销；
3. *(TBD)* 用户有利可图，合约通过提供基础金融服务盈利。

### 1.2. 租金池数据结构

`IDRP`租金池线性地记录用户的可取存款，即未被租赁合同锁定的存款，这些存款可以活期取出；同一 (承租人, 出租人) 仅允许一笔待处理租赁订单，后续订单会被自动取消。

`deposit`记录了全体用户的可取存款，数据类型`mapping(address => uint256)`；`rent`记录了任意 (承租人, 出租人) 的订单情况，数据类型`mapping(address => mapping(address => uint256))`，`0`代表无待处理订单，否则代表承租人预支付的租金（可能不满足出租人的要求导致租赁失败，订单取消）。

# 2. Gas 消耗

对设备的各种操作在 `Operations.sol` 中有实现，每个操作的 Gas 消耗可以通过运行测试（测试脚本在 `/test` 目录下）来得到

### 2.1. 运行所有测试用例
```bash
$ npm run test
```
或者
```bash
或直接
$ truffle test
```
### 2.2. 单独运行某个测试用例

指令规则 `npm run test [path to test script]` 示例如下

```bash
$ npm run test test/Delegate.js
```
或者
```bash
$ truffle test test/Delegate.js
```

### 2.3. 实验结果

实验一：本地 `test` 网络中测得**各个操作正常执行功能**的 gas 消耗如下（仅供参考）

| method                     | gas      |
| -------------------------- | -------- |
| Register                   |  128551  |
| Deposit                    |  44016   |
| Withdraw                   |  21523   |
| Transfer                   |  67759   |
| Delegate (type is "owner") |  1581660 |
| Delegate (type is "user")  |  1608802 |
| GenerateSessionID          |  24884   |
| Revoke                     |  41054   |

实验二：随着 IoTDevices 或 accTab 存储内容的增长，Delegate，generateSessionId，Revoke 三个函数在搜索遍历时的 gas 消耗

**accTab is array**

| accTabLength        | GenerateSessionID | Delegate | Revoke  |
| ------------------- | ----------------- | -------- | ------- |
|0                    |     38993         | 1614121  | 44560   | 
|2                    |     44788         | 1619917  | 47457   | 
|22                   |     102750        | 1677889  | 76438   | 
|100                  |     328940        | 1904121  | 259075  | 
|222                  |     683187        | 2258431  | 613323  | 
|400                  |     1201028       | 2776372  | 1131181 | 
|600                  |     1784292       | 3359732  | 1714457 |
|800                  |     2369053       | 3944597  | 2299217 |
|1000                 |     2955295       | 4530958  | 2885484 |
|1200                 |     3543047       | 5118817  | 3473248 |
|1400                 |     4132296       | 5708151  | 4062509 |
|1600                 |     4723041       | 6299001  | 4653244 |

**accTab is mapping**

| accTabLength        | GenerateSessionID | Delegate | Revoke |
| ------------------- | ----------------- | -------- | ------ |
|10                   |     30766         | 1539894  | 32620  | 
|100                  |     30766         | 1539894  | 32620  | 
|200                  |     30766         | 1539894  | 32620  | 
|300                  |     30766         | 1539894  | 32620  | 
|400                  |     30766         | 1539894  | 32620  |
|500                  |     30766         | 1539894  | 32620  |
|600                  |     30766         | 1539894  | 32620  |
|700                  |     30766         | 1539894  | 32620  |
|800                  |     30766         | 1539894  | 32620  |
|900                  |     30766         | 1539894  | 32620  |
|1000                 |      30766        | 1539894  | 32620  |
|1200                 |      30766        | 1539894  | 32620  |
|1400                 |      30766        | 1539894  | 32620  |
|1600                 |      30766        | 1539894  | 32620  |
|1800                 |      30766        | 1539894  | 32620  |
|2000                 |      30766        | 1539894  | 32620  |
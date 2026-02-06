### 1. 智能合约基础（Solidity）

#### 1.1. 状态存储与 EVM 存储模型

**踩坑记录**： `storage` 和 `memory` 的区别。

```solidity
// storage: 链上永久存储，每个变量都有固定的 storage slot
// memory: 函数执行时的临时内存，函数结束即销毁
// calldata: 函数参数存储位置，只读

contract StorageDemo {
    uint256 public storedData; // storage slot 0
    uint256[] public array;    // storage slot 1（存储长度），数据存在 keccak256(slot)+offset
    
    function process(uint256[] calldata data) external {
        uint256[] memory tempArray = data; // 从 calldata 拷贝到 memory
        // 不能直接修改 calldata
        tempArray[0] = 1; // ✓ 合法
        // data[0] = 1;   // ✗ 报错：calldata is read-only
    }
}
```

**存储布局规则**（来源：Solidity 官方文档 v0.8.26）：
- 静态变量（uint256, address）按声明顺序占 slot 0, 1, 2...
- 动态数组（T[]）只在 slot N 存长度，数据存在 `keccak256(N) + i`
- mapping（mapping(K=>V)）不占用连续 slot，key 存在 `keccak256(key . N)`


#### 1.2. 函数可见性修饰符

刚开始总搞混 `external` 和 `public`，后来发现 gas 消耗差这么多：

| 修饰符 | 调用方式 | Gas 成本 | 典型场景 |
|--------|----------|----------|----------|
| `public` | 内部 + 外部 | 较高 | 需要被合约内部调用 |
| `external` | 仅外部 | 较低（calldata 直接传递） | 纯外部接口 |
| `private` | 仅本合约内 | 最低 | 内部辅助函数 |
| `internal` | 本合约 + 继承合约 | 较低 | 继承场景 |

```solidity
contract VisibilityDemo {
    function externalFunc() external pure returns (uint) {
        // this.externalFunc(); // ✗ 内部不能调用 external
        return 1;
    }
    
    function publicFunc() public pure returns (uint) {
        this.externalFunc(); // ✓ public 可以通过 this 调用 external
        return 2;
    }
}
```

**来源**：OpenZeppelin Solidity 安全指南 2024

### 2. DeFi 核心协议解析

#### 2.1. 自动化做市商（AMM）与无常损失

这个概念我花了将近一周才理解透彻，特别是无常损失（IL）的数学原理。简单来说：

**AMM 定价公式**（来源：Uniswap V2 白皮书）：
- `x * y = k`（恒定乘积）
- 价格 `P = y / x`
- 交易时保持 `k` 不变，所以 `Δy = (y * Δx) / (x + Δx)`

**无常损失推导**：
假设你提供流动性时 ETH 价格为 $1000，存入 1 ETH + 1000 DAI。
当 ETH 价格涨到 $2000 时，池子自动套利，你实际持有的资产价值为：
- ETH: 0.707 ETH
- DAI: 1414 DAI
总价值 = 0.707 * 2000 + 1414 = $2828

如果你一直持有不进入池子：
- 价值 = 1 * 2000 + 1000 = $3000

**IL = (3000 - 2828) / 3000 = 5.73%**

这个计算在 Ethernaut 第 22 关（Dex）会考察价格操纵，我当时用 Excel 算了半天才验证出池子的滑点公式。

**踩坑记录**：在复现 DeFiHackLabs 的某个闪电贷攻击时，我以为 IL 是损失来源，结果实际漏洞是价格预言机滞后，白白走了弯路。

#### 2.2. 价格预言机（Oracle）

预言机是 DeFi 的命脉，也是攻击重灾区。目前主要分为：

| 类型 | 代表项目 | 优点 | 缺点 | 典型攻击案例 |
|------|----------|------|------|--------------|
| TWAP | Uniswap V2 | 抗瞬时操纵 | 滞后、需维护历史 | Warp Finance $77M |
| Chainlink | Aave, Compound | 多节点、去中心化 | 延迟、成本高 | Compound 清算事件 |
| Pyth | Solana 生态 | 低延迟、高频率 | 数据源集中 | Mango Markets |
| Self-reserve | Reserve Protocol | 抵押品支撑 | 流动性差 | - |

**Chainlink 工作原理**（来源：Chainlink 官方文档）：
1. 多个独立节点从交易所 API 获取价格
2. 节点将价格 + 签名提交到链上
3. 聚合合约剔除异常值，取中位数
4. 消费者合约调用 `latestRoundData()` 获取价格

**攻击面分析**：
- **延迟攻击**：预言机更新间隔（通常 1 小时）与闪电贷瞬时操纵的时间差
- **数据质量**：小交易所流动性不足，易被操纵
- **聚合算法**：剔除逻辑是否有漏洞

Ethernaut 第 23 关（Pool）就是典型的预言机操纵，攻击者通过 skew 池子 ratio 再触发清算，我当时模拟了 10 多次才找到最优攻击路径。

### 3. 常见漏洞类型详解

#### 3.1. 重入攻击（Reentrancy）

**经典案例**：TheDAO 攻击（来源：以太坊官方安全公告）

**底层原理**：
```solidity
contract VulnerableBank {
    mapping(address => uint) public balances;
    
    function withdraw() public {
        uint amount = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: amount}(""); // 外部调用
        require(success);
        balances[msg.sender] = 0; // 状态更新在外部调用之后！
    }
}

contract Attacker {
    VulnerableBank public bank;
    
    fallback() external payable {
        if (address(bank).balance >= 1 ether) {
            bank.withdraw(); // 重入！
        }
    }
}
```

**Gas 限制绕过**：
重入不一定发生在 fallback，攻击合约可以在 `receive()` 或任何 `payable` 函数中递归调用。

**防护方案对比**（来源：Consensys Diligence）：
- **Checks-Effects-Interactions模式**：状态更新在前，外部调用在后（推荐）
- **ReentrancyGuard**：OpenZeppelin 的 `nonReentrant` 修饰符（有Gas overhead）
- **Mutex锁**：自定义状态锁（需确保无绕过）

Ethernaut 第 1 关（Fallback）和第 10 关（Re-entrancy）都是这个原理，我最初以为 `nonReentrant` 万能，结果在 DeFiHackLabs 看到一个 case 是跨合约重入绕过了 guard。

#### 3.2. 整数溢出/下溢

**Solidity 0.8.0 之后**：内置溢出检查，但 `unchecked` 块仍需注意。

```solidity
contract OverflowDemo {
    function unsafeSubtract(uint a, uint b) public pure returns (uint) {
        unchecked {
            return a - b; // 若 a < b，下溢返回极大值
        }
    }
}
```

**实际攻击场景**（来源：Trail of Bits Blog）：
- 代币余额突然变成 2^256-1
- 价格计算 `price = total / supply` 中 supply 被操纵为 0

#### 3.3. 访问控制漏洞

**典型案例**：Compound 治理合约漏洞（来源：OpenZeppelin 安全公告）

```solidity
contract Governance {
    address public owner;
    
    // 遗漏修饰器！
    function changeOwner(address newOwner) public { // 应为 onlyOwner
        owner = newOwner;
    }
}
```

**常见错误模式**：
- 构造函数拼写错误（`constructor` 写成 `constructor()`）
- `onlyOwner` 修饰器逻辑错误
- `tx.origin` 误用（应为 `msg.sender`）
---
creation date: 2025-12-09 11:20
modification date: 2025-12-09 11:20
---
# 算法原理

## 1. 核心原理
基于**大整数分解难题**。
*   给定两个素数 $p, q$，求 $n = pq$ 很简单（单向容易）。
*   给定 $n$，分解出 $p, q$ 极难（逆向困难）。

注意，$\varphi(n)$ 只在生成密钥时使用，在加解密时使用 $n$。
## 2. 算法流程 (KeyGen, Enc, Dec)

### Step 1: 密钥生成 (KeyGen)
1.  **选择素数**：选取两个大素数 $p, q$。
2.  **计算模数**：$n = p \times q$。
3.  **计算欧拉函数**：$\varphi(n) = (p-1)(q-1)$。
4.  **选择公钥指数 $e$**：
    *   $1 < e < \varphi(n)$
    *   $\gcd(e, \varphi(n)) = 1$
5.  **计算私钥指数 $d$**：
    *   $ed \equiv 1 \pmod{\varphi(n)}$
    *   即 $d$ 是 $e$ 模 $\varphi(n)$ 的逆元。

| 类型                   | 包含内容                   | 可见性   |
| :------------------- | :--------------------- | :---- |
| **公钥 (Public Key)**  | $\{e, n\}$             | 🌍 公开 |
| **私钥 (Private Key)** | $\{d, n\}$ (以及 $p, q$) | 🔒 保密 |

### Step 2: 加密 (Encryption)
发送方使用接收方的**公钥**。
$$C \equiv M^e \pmod n$$

### Step 3: 解密 (Decryption)
接收方使用自己的**私钥**。
$$M \equiv C^d \pmod n$$

## 3. 安全性根基
攻击者若想破解 $d$，必须解方程 $ed \equiv 1 \pmod{\varphi(n)}$。
要解此方程，必须知道 $\varphi(n) = (p-1)(q-1)$。
要知道 $p$ 和 $q$，必须对 $n$ 进行**质因数分解**。
$\implies$ **RSA 的安全性依赖于大整数分解的困难性。**

## 4. 常用小结论 (做题用)
*   若题目给出 $p, q, e$，求 $d$：使用**扩展欧几里得算法**。
*   $M$ 必须小于 $n$。
*   $e$ 常取 65537 ($2^{16}+1$)，因为它也是素数且二进制只有两个1，加密运算快。
# 证明
## 1. 证明目标
证明 RSA 加解密的可逆性，即：
$$D(E(m)) = m \implies c^d \equiv m \pmod n$$
代入 $c \equiv m^e$，等价于证明：
$$m^{ed} \equiv m \pmod n$$

## 2. 关键已知条件
> [!info] 核心公式
> 1. $n = p \cdot q$ ($p, q$ 为互异素数)
> 2. $ed \equiv 1 \pmod{\varphi(n)}$
> 3. 展开指数：$ed = k \cdot \varphi(n) + 1 = k(p-1)(q-1) + 1$

## 3. 证明步骤 (分情况讨论)

由于欧拉定理要求底数与模数互质，故需分两种情况。

### 情况 1：$m$ 与 $n$ 互质 ($\gcd(m, n) = 1$)
> [!tip] 思路
> 直接使用欧拉定理 $m^{\varphi(n)} \equiv 1 \pmod n$。

$$
\begin{aligned}
m^{ed} &= m^{k\varphi(n)+1} \\
       &= (m^{\varphi(n)})^k \cdot m \\
       &\equiv 1^k \cdot m \pmod n \\
       &\equiv m \pmod n \quad \checkmark
\end{aligned}
$$

### 情况 2：$m$ 与 $n$ 不互质 ($\gcd(m, n) \neq 1$)
> [!tip] 思路
> 此时欧拉定理失效。
> 策略：利用 $n=pq$，分别证明在 $\pmod p$ 和 $\pmod q$ 下成立，再通过 CRT 合并。
> 假设 $m$ 是 $p$ 的倍数，即 $m = tp$。

**Step 1: 证明模 $p$ 成立**
因为 $m$ 是 $p$ 的倍数，显而易见：
$$m \equiv 0 \pmod p \implies m^{ed} \equiv 0 \equiv m \pmod p \quad (1)$$

**Step 2: 证明模 $q$ 成立**
因为 $m < n=pq$ 且 $m$ 是 $p$ 的倍数，故 $m$ 一定不是 $q$ 的倍数。
$\therefore \gcd(m, q) = 1$。
根据**费马小定理**：$m^{q-1} \equiv 1 \pmod q$。

$$
\begin{aligned}
m^{ed} &= m^{k(p-1)(q-1)+1} \quad \text{(代入指数展开式)} \\
       &= (m^{q-1})^{k(p-1)} \cdot m \\
       &\equiv 1^{k(p-1)} \cdot m \pmod q \\
       &\equiv m \pmod q \quad (2)
\end{aligned}
$$

**Step 3: 合并结论**
结合 (1) 和 (2)：
$$
\begin{cases}
m^{ed} \equiv m \pmod p \\
m^{ed} \equiv m \pmod q
\end{cases}
$$
因为 $\gcd(p, q) = 1$，由模运算性质（或中国剩余定理）：
$$m^{ed} \equiv m \pmod{pq}$$
即：
$$m^{ed} \equiv m \pmod n \quad \checkmark$$

**证毕。**
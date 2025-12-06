## 延展性 (Malleability)
- **定义**：如果敌手能将密文 $c$ 修改为 $c'$，使得解密后的明文 $m'$ 与 $m$ 存在某种关联（如 $m' = m \oplus 1$），则称该方案具有延展性。
- **影响**：所有 CPA 安全的流密码（如 CTR）都是可延展的（翻转密文位=翻转明文位）。

## Padding Oracle 攻击
- **场景**：针对使用 PKCS#5 填充的 CBC 模式。
- **原理**：
  - 敌手修改密文，发送给服务器解密。
  - 服务器返回“解密失败”或“填充错误”。
  - 利用这个布尔反馈 (Oracle)，敌手可以逐字节推断出中间状态，进而恢复明文，完全不需要密钥。
- **防御**：验证 MAC 失败后应立即拒绝，不进行解密；或者确保填充错误和MAC错误的响应时间一致。

## 认证加密 (Authenticated Encryption, AE)
- **目标**：同时实现 CCA 安全 (机密性) 和 认证性 (完整性)。
- **组合方式分析**：
  1. **Encrypt-and-MAC** (SSH)：$c \leftarrow Enc(m), t \leftarrow Mac(m)$。不推荐，因为 $t$ 可能泄露 $m$ 的信息。
  2. **MAC-then-Encrypt** (TLS 1.0)：$c \leftarrow Enc(m || t)$。曾经被认为安全，但容易遭受 Padding Oracle 攻击。
  3. **Encrypt-then-MAC** (IPsec)：$c \leftarrow Enc(m), t \leftarrow Mac(c)$。**最佳实践**。
     - **证明**：如果 Enc 是 CPA 安全且 MAC 是强不可伪造的，则该组合是 **CCA 安全** 的。
     - 理由：先验证 $t$，如果伪造直接丢弃，密文永远不会进入解密函数，从而阻断了 Padding Oracle 等 CCA 攻击。

## 安全会话 (Secure Sessions)
- 仅有加密是不够的，还需防止：
  - **重放 (Replay)**：重发旧消息。
  - **重排序 (Re-order)**：改变消息顺序。
  - **反射 (Reflection)**：将消息发回给发送者。
- **解决方案**：使用序列号 (Sequence Numbers) 或 计数器 (Counters) 并包含在 MAC 的计算中。
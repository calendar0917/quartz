---
creation date: 2025-12-12 20:53
modification date: 2025-12-12 20:53
---
点估计是用样本统计量 $\hat{\theta}(X_1, \dots, X_n)$ 的某个具体数值来代表总体未知参数 $\theta$。

## 1. 矩估计法 (Moment Estimation)
**核心思想**：用“样本矩”替换“总体矩”。
$$A_k \xrightarrow{\text{替换}} \mu_k$$
即：$\frac{1}{n}\sum X_i^k = E(X^k)$。

**解题步骤**：
1. **单参数 $\theta$**：
   - 计算总体期望 $E(X)$（它是 $\theta$ 的函数）。
   - 令 $E(X) = \bar{X}$，解出 $\theta$。
2. **双参数 $\theta_1, \theta_2$**（如 $U(a,b)$ 或 $N(\mu, \sigma^2)$）：
   - 列方程组：
     $$
     \begin{cases} E(X) = \bar{X} \\ E(X^2) = \frac{1}{n}\sum X_i^2 (= A_2) \end{cases}
     $$
     或者用方差方程：$D(X) = E(X^2) - [E(X)]^2 = \frac{1}{n}\sum X_i^2 - \bar{X}^2 = B_2$（样本二阶中心矩）。
   - 解方程组得 $\hat{\theta}_1, \hat{\theta}_2$。

## 2. 极大似然估计法 (MLE)
**核心思想**：取让“样本出现概率最大”的那个参数值作为估计值。

**解题步骤 (标准四步走)**：
1. **写似然函数 $L(\theta)$**：
   $$L(\theta) = \prod_{i=1}^n f(x_i; \theta)$$
   > *离散型*：$L(\theta) = \prod P(X=x_i)$。
   > *技巧*：对于示性函数（如均匀分布），要把范围写在连乘后面，如 $\mathbf{I}_{\{a \le x_{(1)} \le x_{(n)} \le b\}}$。
2. **取对数**：
   $$\ln L(\theta) = \sum_{i=1}^n \ln f(x_i; \theta)$$
   > 这一步是为了把乘积变求和，方便求导。
3. **求导令为0 (似然方程)**：
   $$\frac{d}{d\theta} \ln L(\theta) = 0$$
4. **求解**：解出的驻点即为 $\hat{\theta}$。

> **注意**：对于均匀分布 $U(a, b)$，似然函数是单调的，不能求导。
> - $\hat{a}_{MLE} = \min(X_i) = X_{(1)}$
> - $\hat{b}_{MLE} = \max(X_i) = X_{(n)}$
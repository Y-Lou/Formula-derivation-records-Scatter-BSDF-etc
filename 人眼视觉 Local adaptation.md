# 人眼视觉 Local adaptation

## 方法1

输入：L(x) 每个像素的物理亮度  $cd/m^2​$

输出：V(x) 归一化人眼显示值[0,1]

### 确定适应亮度$L_\alpha$

$L_\alpha = Local Adaptation Luminance​$

### 亮度归一化

$$
r(x) = \frac{L(x)} {L_\alpha}
$$

r=1	该像素亮度等于 眼睛当前适应亮度
r >> 1    高亮
r << 1    暗部

### 用人眼响应曲线f(r)做非线性压缩

经典视觉响应Naka–Rushton：
$$
f(r) = \frac{r^n}{r^n + 1}
$$
$n表示对比强弱。n\in[0.7, 1.3]。n越大，适应点附近对比越强、亮度压缩更明显​$

当$L = L_\alpha时 r = 1, V = f(1) = 0.5$

### 白点/黑点设定与归一化

f(r)有三个重要特征：

1. 有界但不一定刚好用满[0,1]

   $f(r) \in (0,1)​$ 但是最暗点可能是0.02，最亮点可能是0.85。没用满显示动态范围

2. 极端亮点/暗点可能会异常高亮/偏暗

3. 显示设备是有限范围的，必须是[0, 1]或[0, 255]

所以需要把亮度范围拉伸到显示域[0, 1]

#### 选定显示黑点、白点

* 黑点$r_b​$：最暗亮度
* 白点$r_w$：最亮亮度

如果直接用：$r_b = min(r), r_w = max(r)​$

可能：一个像素点特别亮，导致$r_w​$特别亮，其它的像素被压缩到很小的区间

导致：画面发暗，对比很差

避免极少数异常亮点/暗点把整体拉坏，在相对亮度域r(x)上取分位数：
$$
r_{black} = Q_\alpha(r)
\\
r_{white} = Q_\beta(r)
$$
$\alpha = [0.001,0.01], \beta = [0.99,0.999]​$

$Q_p(r)：r的第p分位数，在所有r值中，有 p×100\% 的数不大于它。​$

```
eg:
r = [0.02, 0.05, 0.1, 0.2, 0.5, 1, 2, 5, 20, 100]
$Q_0.5(r)$：[0.02, 0.05, 0.1, 0.2, 0.5 | 1, 2, 5, 20, 100]
```

$r_b：只有 \alpha×100\%的像素比它更暗​$

$r_w：只有 (1 - \beta) ×100\% 的像素比它更亮 ​$

#### 从$r_b, r_w$到$v_b, v_w$

选好了要显示的亮度范围，现在把它映射到响应值：
$$
v(x) = f(r(x))
$$

黑白点对应值：

$$
v_b = f(r_b), v_w = f(r_w)
\\
v_b: 被映射到显示0
\\
v_w：被映射到显示1
$$

#### 归一化

$$
V(x) = clamp(\frac{v(x) - v_b}{v_w - v_b}, 0, 1)
\\
v(x)：当前像素的人眼响应值
\\
v_b：黑点响应值 → 映射到 0
\\
v_w：白点响应值 → 映射到 1
\\
clamp：防止数值溢出
$$

--------

## 方法2

### 输入

亮度图L(x) ，cd/m2

### 计算局部适应亮度场单尺度高斯

**方式1：**单尺度高斯
$$
L_\alpha(x) = (G_\sigma * L)(x)
$$
$G_\sigma$：2D高斯核  $\sigma$：感受野尺度（像素）

**方式2：**多尺度融合
$$
L_\alpha(x) = \sum_{k = 1}^{K}{w_k (G_{\sigma_k} * L)(x)}
$$
$k​$：尺度数（3-6）

$\sigma_k​$：不同尺度						$\sigma_k = \{2, 8,32,128/64 \}​$像素

$G_{\sigma_k}​$：不同尺度高斯核

$w_k$：权重，满足$\sum w_k = 1, w_k \geq 0$	     $w_k = \{0.4, 0.3, 0.2, 0.1\}$



#### **高斯核运算：**$(G_{\sigma_k} * L)(x)$

$\sigma_k$是高斯的标准差（单位：像素）

由$\sigma_k​$确定核半径$R_k：R_k = 3\sigma_k​$

原因：在$r = 3\sigma​$处：$e^{-\frac{9}{2}} \approx 0.011​$，权重已经可以忽略

核大小：$(2R_k + 1) \times (2R_k + 1)​$

#### 构造高斯核$G_{\sigma_k}(u,v)$

偏移(u,v)，其中$u,v \in [-R_k, R_k]​$

高斯核权重：

$\widetilde{G_{\sigma_k}(u,v)} = exp({-\frac{u^2 + v^2}{2\sigma_k^2}})$

归一化：
$$
S_k = \sum_{u = -R_k}^{R_k}  \sum_{v = -R_k}^{R_k}{\widetilde{G_{\sigma_k}(u,v)} }
$$

$$
G_{\sigma_k}(u,v) = \frac{\widetilde{G_{\sigma_k}(u,v)} }{S_k}
\\
且\sum{G_{\sigma_k}(u,v)} = 1
$$

#### 高斯核卷积亮度图

对于每个像素x = (i, j)
$$
(G_{\sigma_k} * L)(i, j) = \sum_{u = -R_k}^{R_k}  \sum_{v = -R_k}^{R_k}{L(i+u, j+v)G_{\sigma_k}(u,v) }
\\
= \sum_{u = -R_k}^{R_k}  \sum_{v = -R_k}^{R_k} \frac{exp({-\frac{u^2 + v^2}{2\sigma_k^2}})}{S_k}
$$

### 把亮度变成相对亮度

$$
r(x) = \frac{L(x)}{L_\alpha(x)}
$$

同样 100 cd/m²，在暗环境$L_\alpha$小，r大（更亮）；在亮环境$L_\alpha$大，r小（更暗）

### Luminance选项 ？？

决定视觉响应曲线的工作状态

$L_{ref}​$ = Luminance 单位cd/m2
$$
r_{50}(L_{ref}) = a ( \frac{L_{ref}}{L_0})^b
$$
$r_{50}$：当$r = r_{50}$时响应到一半

$L_0​$：参考亮度（1或10）

a,b需要标定，a = 1，b = 0

### 视觉非线性响应

Naka–Rushton：
$$
v(x) = \frac{r(x)^n}{r(x)^n + r_{50}(L_{ref})^n}
$$
$v(x) \in (0, 1), n \in [0.8, 1.2]$

### 黑点/白点

**方式1：**黑点取最小值，白点取最大值
$$
r_b = r_{min}
\\
r_w = r_{max}
$$

**方式2：**对r(x)做分位数截断
$$
r_b = Q_{p_b}(r)
\\
r_w = Q_{p_w}(r)
$$
$Q_p$：分位数

$p_b \in [0.01, 0.05]，p_w \in [0.95, 0.99] $	

$Q_{0.01}r​$：有1%的r小于$Q_{0.01}r​$

$Q_{0.99}r$：有99%的r小于$Q_{0.99}r​$



对应到响应域：

$$
v_b = \frac{r_b^n}{r_b^n + r_{50}(L_{ref})^n}
\\
v_w = \frac{r_w^n}{r_w^n + r_{50}(L_{ref})^n}
$$

### 归一化到显示域

感知显示域归一化
$$
V(x) = \frac{v(x) - v_b}{v_w - v_b}  \in (0,1)
$$

### 映射到亮度图

**方式1：**映射到一个显示白点亮度$L_{white}$
$$
L_{out}(x) = V(x) * L_{white}
$$
$L_{while}$：希望输出的最大亮度

**方式2：**保持与局部适应一致
$$
L_{out}(x) = V(x) * L_\alpha(x)
$$
输出会随局部背景变化



-----

### 完整公式链：

**1. 计算高斯卷积：**
$$
L_\alpha(x) =  \sum_{k = 1}^{K}{w_k (G_{\sigma_k} * L)(x)}
$$

$\sigma_k = \{2, 8,32,128/64 \}​$像素
$w_k = \{0.4, 0.3, 0.2, 0.1\}​$
$$
R_k = 3\sigma_k
\\
\widetilde{G_{\sigma_k}(u,v)} =exp({-\frac{u^2 + v^2}{2\sigma_k^2}})
\\
S_k = \sum_{u = -R_k}^{R_k}  \sum_{v = -R_k}^{R_k}{\widetilde{G_{\sigma_k}(u,v)} }
\\
G_{\sigma_k}(u,v) = \frac{\widetilde{G_{\sigma_k}(u,v)} }{S_k},
且\sum{G_{\sigma_k}(u,v)} = 1
\\
$$
第k个高斯核对亮度图进行卷积：
$$
L_k(x) = (G_{\sigma_k} * L)(x)
\\
= \sum_{u = -R_k}^{R_k}  \sum_{v = -R_k}^{R_k}{G_{\sigma_k}(u,v)L(u,v)}
$$
加权融合所有尺度：
$$
L_\alpha(x) = \sum_{k = 1}^{K}{w_k L_k(x)}
$$
**2. 计算相对亮度：**
$$
r(x) = \frac{L(x)}{L_\alpha(x)}
\\
r_{50}(L_{ref}) = a ( \frac{L_{ref}}{L_0})^b,{\quad} L_{ref} =Luminance
\\
$$
**3. 视觉非线性响应：**
$$
v(x) = \frac{r(x)^n}{r(x)^n + r_{50}(L_{ref})^n}, {\quad}{\quad}{\quad} n = 1
$$
**4. 计算黑点/白点：**
$$
r_b = r_{min}, {\quad}{\quad}
r_w = r_{max}
\\or\\
r_b = Q_{p_b}(r), {\quad}{\quad}
r_w = Q_{p_w}(r)
\\ 
p_b = 0.02, p_w = 0.98
$$
**5. 计算黑点/白点的视觉非线性响应：**
$$
v_b = \frac{r_b^n}{r_b^n + r_{50}(L_{ref})^n},{\quad}
v_w = \frac{r_w^n}{r_w^n + r_{50}(L_{ref})^n}
$$
**6. 归一化到显示域：**
$$
V(x) = \frac{v(x) - v_b}{v_w - v_b}  \in (0,1)
$$
**7. 映射到亮度图：**
$$
L_{out}(x) = V(x) * L_{white}
\\or\\
L_{out}(x) = V(x) * L_\alpha(x)
$$










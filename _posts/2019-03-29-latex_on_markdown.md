---
layout: post
title: markdown中Latex语法梳理
datetime: 2019-03-29 22:21:32
description: markdown中Latex语法梳理
comments: true
tags:
 - Markdown
categories:
 - Markdown
---



## 插入公式的方法

- 行内公式：`$...$` 和 `\(...\)`
- 块内公式：`$$...$$` 和  `\[...\]`



## Latex运算符

| 运算符号 | Latex公式 | 运算符号 | Latex公式 |
| :--------: | :----------: | ---------- | ---------- |
|   $\geq$   |   `$\geq$`   |   $\coprod$   |   `$\coprod$`   |
|   $\leq$   |   `$\leq$`   |   $\sum$   |   `$\sum$`   |
|    $\sim$    |    `$\sim$`    | $\prod$ | `$\prod$` |
|  $\backsim$  |  `$\backsim$`  | $\not\supset$ | `$\not\supset$` |
|    $\bot$    |    `$\bot$`    | $\subset$ | `$\subset$` |
|     $\pm$    |   `$\pm$`    | $\supset$ | `$\supset$` |
|     $\cdot$     |   `$\cdot$`    | $\in$ | `$\in$` |
|   $\times$     |  `$\times$`  |  $\notin$  |  `$\notin$`  |
| $\ast$ | `$\ast$` | $\subseteq$ | `$\subseteq$` |
|    $\div$     |   `$\div$`   |   $\supseteq$   |   `$\supseteq$`   |
| $\not=$ | `$\not=$` |   $\bigcap$   |   `$\bigcap$`   |
| $\approx$ | `$\approx$` | $\bigcup$ | `$\bigcup$` |
| $\not<$| `$\not<$` | $\bigvee$ | `$\bigvee$` |
|    $\mid$    |    `$\mid$`    | $\bigvee$ | `$\bigvee$` |
| $\log$ | `$\log$` | $\bigwedge$ | `$\bigwedge$` |
| $\log_2{18}$ | `$\log_2{18}$` | $\hat{y}$ | `$\hat{y}$` |
| $\ln$ | `$\ln$` | $\check{y}$ | `$\check{y}$` |
|    $\lg$     |    `$\lg$`     | $\breve{y}$ | `$\breve{y}$` |
| $\angle$ | `$\angle$` | $\ll$ | `$\ll$` |
| $30^\circ$ |`$30^\circ$` |$\gg$ |`$\gg$` |
| $\sin$ | `$\sin$` | $\lim$ | `$\lim$` |
| $\cos$ | `$\cos$` | $\infty$ | `$\infty$` |
| $\tan$ | `$\tan$` | $\nabla$ | `$\nabla$` |
| $\cot$ | `$\cot$` | $\oint$ | `$\oint$` |
| $\csc$ | `$\csc$` | $\prime$ | `$\prime$` |
| $\sec$ | `$\sec$` | $\bigodot$ | `$\bigodot$` |
| $\bigotimes$ | `$\bigotimes$` | $\bigoplus$ | `$\bigoplus$` |



## Latex数学表达式



| 名称 | 符号 | 数学表达式 | Latex公式 |
| :----: | :--: | :----------: | :------------: |
| 上标 | `^`  | $a^b$ | `$a^b$` |
| 下标 | `_` | $a_b$ | `$a_b$` |
| 分数 | `\frac` | $\frac{1 + a}{b + c}$ | `$\frac{1 + a}{b + c}$` |
| 求和 | `\sum` | $$\sum{2x^n}$$ | `$\sum{2x^n}$` |
| 带范围求和 | `\sum_{ }^{ }` | $\sum_{n=1}^N$ | `$\sum_{n=1}^N$` |
| 累乘 | `\prod_{ }^{ }` | $\prod_{n=1}^{N}{2x^n}$ | `$\prod_{n=1}^{N}{2x^n}$` |
| 开方 | `\sqrt[ ]{ }` | $\sqrt[2]{100}$ | `$\sqrt[2]{100}$` |
| 积分 | `\int_{ }^{ }` | $\int^5_1{f(x)}{\rm d}x$ | `$\int^5_1{f(x)}{\rm d}x$` |
| 二重积分 | `\iint_{ }^{ }` | $\iint^5_1{f(x)}{\rm d}x$ | `$\iint^5_1{f(x)}{\rm d}x$` |
| 三重积分 | `\iiint_{ }^{ }` | $\iiint^5_1{f(x)}{\rm d}x$ | `$\iiint^5_1{f(x)}{\rm d}x$` |
| 正无穷 | `$\infty$` | $+\infty$ | `$+\infty$` |
| 正无穷 | `$\infty$` | $-\infty$ | `$-\infty$` |
| 极限 |      | $\lim_{n\rightarrow+\infty} n$ | `$\lim_{n\rightarrow+\infty} n$` |



## 箭头


| 箭头符号       | Latex公式                        |
| :--------------: | :-----------------------------------------: |
| $\uparrow$     | `$\uparrow$`                               |
| $\downarrow$   | `$\downarrow$`                             |
| $\Uparrow$     | `$\Uparrow$`                               |
| $\Downarrow$   | `$\Downarrow$`                             |
|$\rightarrow$            | `$\rightarrow$`                            |
|$\leftarrow$             | `$\leftarrow$`                             |
| $\Rightarrow$             | `$\Rightarrow$`                            |
| $\Leftarrow$             | `$\Leftarrow$`                             |
| $\longrightarrow$             | `$\longrightarrow$`                        |
| $\longleftarrow$             | `$\longleftarrow$`                         |
| $\Longrightarrow$             | `$\Longrightarrow$`                        |
| $\Longleftarrow$             | `$\Longleftarrow$`                         |
|$f: {\mathbf x_t} \mapsto {\mathbf y_t}$ | `$f: {\mathbf x_t} \mapsto {\mathbf y_t}$` |
| $\Longleftrightarrow$          | `\Longleftrightarrow`                      |



## 括号和分隔符

> `()`、`[]`和`|`表示符号本身，使用 `\{\}` 来表示 `{}`。当要显示大号的括号或分隔符时，要用 `\left` 和 `\right` 命令。

| Latex公式             | 显示             |
| :-----------------------: | :----------------: |
| `$$\langle...\rangle$$` | $$\langle...\rangle$$ |
| `$$\lceil...\rceil$$`   | $$\lceil...\rceil$$ |
| `$$\lfloor...\rfloor$$` | $$\lfloor...\rfloor$$ |
| `$$\lbrace...\rbrace$$` | $$\lbrace...\rbrace$$ |

**实例1：**  `$$ f(x,y,z) = 3y^2z \left( 3+\frac{7x+5}{1+y^2} \right) $$`
$$
f(x,y,z) = 3y^2z \left( 3+\frac{7x+5}{1+y^2} \right)
$$



**实例2：**大括号和行标的使用

```
$$
f\left(
   \left[ 
     \frac{
       1+\left\{x,y\right\}
     }{
       \left(
          \frac{x}{y}+\frac{y}{x}
       \right)
       \left(u+1\right)
     }+a
   \right]^{3/2}
\right)
\tag{行标}
$$
```


$$
f\left(
\left[ 
\frac{
1+\left\{x,y\right\}
}{
\left(
\frac{x}{y}+\frac{y}{x}
\right)
\left(u+1\right)
}+a
\right]^{3/2}
\right)
\tag{1}
$$


## 省略号

> 数学公式中常见的省略号有两种
>
> - \ldots 表示与文本底线对齐的省略号
> - \cdots 表示与文本中线对齐的省略号



实例：`$$f(x_1,x_2,\underbrace{\ldots}_{\rm ldots} ,x_n) = x_1^2 + x_2^2 + \underbrace{\cdots}_{\rm cdots} + x_n^2$$`



$$
f(x_1,x_2,\underbrace{\ldots}_{\rm ldots} ,x_n) = x_1^2 + x_2^2 + \underbrace{\cdots}_{\rm cdots} + x_n^2
$$

## 矢量输入

> 使用 `\vec{矢量}`可以产生一个矢量。也可以使用 `\overrightarrow`命令自定义字母上方的符号。

**实例1：**`$$\vec{a} \cdot \vec{b}=0$$`
$$
\vec{a} \cdot \vec{b}=0
$$



实例2：**`$$\overleftarrow{xy} \quad and \quad \overleftrightarrow{xy} \quad and \quad \overrightarrow{xy}$$`

$$
\overleftarrow{xy} \quad and \quad \overleftrightarrow{xy} \quad and \quad \overrightarrow{xy}
$$



## 字体转换

使用 `{\字体 {需转换的部分字符}}` 命令可以转换字符字体。其中 `\字体` 部分可以参照下表选择合适的字体。`公式默认为意大利体.`

| Latex公式   | 说明       | 显示实例 |
| :-------------: | :----------: | :--------: |
| `\rm`         | 罗马体     | $\rm{A}$     |
| `\cal`        | 花体       | $\cal{B}$       |
| `\it`         | 意大利体   | $\it{C}$      |
| `\Bbb`        | 黑板粗体   | $\Bbb{D}$       |
| `\bf`         | 粗体       | $\bf{E}$      |
| `\mit`        | 数学斜体   | $\mit{F}$      |
| `\sf`         | 等线体     | $\sf{G}$       |
| `\scr`        | 手写体     | $\scr{H}$       |
| `\tt`         | 打字机体   | $\tt{M}$      |
| `\frak`       | 旧德式字体 | $\frak{N}$       |
| `\boldsymbol` | 黑体       | $\boldsymbol{X, x}$   |
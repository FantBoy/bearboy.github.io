---

layout: post
title: 布隆过滤器的实现原理
datetime: 2019-02-11 22:21:32
description: 布隆过滤器的实现原理
comments: true
tags:
 - 布隆过滤器
categories:
 - 布隆过滤器
---


在大数据集场景，如爬虫系统中，往往需要判断某一个元素是否存在在某一个集合中，或者查询某一个URL是否已经被访问过。在日志处理系统、业务缓存和垃圾邮件过滤等业务中也会存在类似的查询业务。

> 这些场景存在一个共同点：需要判断某元素是否存在在某集合中，且数据量巨大、查询频率较高



## 常规算法

常规的算法包括：

- 数组  `二分查询提高查询性能`

- 字典树

- map（红黑树）

- HashMap

  

他们有一个共同点，就是在数据量较小时能满足业务需求，但是在数据量较大时，虽然在查询效率上能够满足（数组可以通过二分查询提高查询性能），但是需要消耗大量的存储空间。比如1亿条URL，将给存储带来性能瓶颈。

## 哈希表的做法

哈希表主要依赖哈希函数来实现的。哈希函数将一个元素（如URL）Hash映射成固定字节（8bytes）的指纹。且哈希函数为了降低冲突，哈希表的存储效率通常低于50%，因此可以推算：



一亿条URL共消耗内存：$$8 bytes * 2 * 1 亿 = 1.6GB $$

这将会给操作系统带来不小的内存消耗的压力。在这种情况下，布隆过滤器（Bloom Filter）产生了。



## 布隆过滤器的设计基础

布隆过滤器实际上是由一串很长的二进制向量（位数组）和一系列的随机映射函数（哈希函数）组成的。

### 哈希函数

哈希函数的主要工作是将任意长度的数据（整数、字符串等）转换成特定大小的数据（哈希值`hash code`）的函数。

![](/images/posts/bloom_filter/hash_function.png)

>  哈希函数将原始数据映射位固定大小的哈希编码，让数据得到有效的压缩，这也是布隆过滤器得以实现的基础

### 布隆过滤器的操作流程

假设位数组大小为m，哈希函数个数为k。那具体的操作流程：

1. 将位数组初始化，都设置为0
2. 添加元素：
   - 将元素进行k次哈希，得到k个哈希值
   - 将位数组中与k个哈希值相对应的k个位置全部置1
3. 查询元素
   - 将要查询的元素进行k次哈希，得到k个哈希值
   - 取位数组中与k个哈希值对应的k个位置
   - 若k个位置对应的值全部位1，则视为命中；反之，不存在

> 命中则可以判断该次查询的元素可能存在在集合中。该方法存在一定的误判率，可能将原本不存在的元素判为存在，但不会将存在的元素误判为不存在。



### 布隆过滤器的优势

在可以承受一定的误判率的前提下，布隆过滤器拥有很大的空间消耗的优势。

- 以bit位来存储每一个元素的hash结果。大大节省了内存开销
- 对于一个有1%误报率和最优k值的布隆过滤器来说，一个元素只需要9.6bits来存储，且每降低到原误判率的10%，则需要增加4.8bits `后面会验证`
- 哈希表实际上为`k=1`的布隆过滤器。若要保证1%的误判率，则bit数组只能存储`m/100`个元素，浪费大量空间



## 布隆过滤器的设计原理

### 内存消耗

假设需要存储的元素个数为n，位数组大小为m。若哈希表半满 `n/m = 50%` ，即在保证一定效率的情况下，哈希表的存储效率不超过50%。传统方法消耗内存 1.6GB。

$$
\frac{10^8 * 8 bytes}{50\%} = 1.6GB
$$

若使用布隆过滤器，假设哈希函数的个数`k=8`，每一个元素占用8bits的存储空间，且总的空间利用率取50%，则一亿条URL共占用空间：200MB。

$$
\frac{8 bits * 10^8}{50\%} = 1.6 * 10^9 bits = 200 MB
$$

> 布隆过滤器所占用空间是传统方式的1/8，且误判率在万分之一一下

### 误判率的计算

假设布隆过滤器的所有Hash Function都满足Simple uniform hashing假设：每个元素都等概率的hash到m个slot中的任何一个，且与其他元素被hash到那个slot无关。

若bit数位为m，则对某一特定的bit位，在某一特定的元素通过Hash Funciton插入时，该bit位没有被置1的概率为：



$$
1-\frac{1}{m}
$$



则该元素经过k个Hash Funciton映射之后，该bit位没有被置1的概率为：



$$
(1-\frac{1}{m})^k
$$



当插入n个元素之后，该bit位还未被置1的概率为：



$$
(1-\frac{1}{m})^{kn}
$$



则，整个过程中，该bit位被置1的概率为：



$$
1-(1-\frac{1}{m})^{kn}
$$



那么在元素查询的时候，查询的流程类似于插入的流程。若对应该查询的元素的所有k个bit位都为1，则可以判定该元素可能在集合中。则误判率为：



$$
(1-(1-\frac{1}{m})^{kn})^k
$$



由于 x趋于0时，$(1+x)^{\frac{1}{x}}$ 趋于 e。则



$$
(1-(1-\frac{1}{m})^{kn})^k = (1-(1-\frac{1}{m})^{\frac{(-m)(-kn)}{m}})^k
$$


当m足够大时， $(1-\frac{1}{m})^{-m}$ 趋于 e。则误判率 $P(error)$为：



$$
p(error)=(1-e^{\frac{-kn}{m}})^k
$$



> 由此可以看到，当m增加或者n减少时，可以降低误判率，这个实际情况也是吻合的。



### 最优误判率

参考 $$P(error)$$和m、k、n的关系，选取最优布隆过滤器参数。



$$
f(k)=(1-e^{\frac{-kn}{m}})^k
$$



设$$b=e^{\frac{n}{m}}$$，



$$
f(k)=(1-b^{-k})^k
$$



取对数：



$$
\ln{f(k)} = \ln{(1-b^{-k})^k} = k*\ln{(1-b^{-k})}
$$



两边对k取导：



$$
\frac{1}{f(k)}*f^\prime(k)=\ln(1-b^{-k})+k*\frac{1}{1-b^{-k}}(-b^{-k})*\ln{b}*(-1)=\ln{(1-b^{-k}})+k*\frac{b^{-k}*\ln{b}}{1-b^{-k}}
$$



取最值：



$$
\ln{(1-b^{-k}})+k*\frac{b^{-k}*\ln{b}}{1-b^{-k}}=0
$$

$$
(1-b^{-k})*\ln{(1-b^{-k}}) = -k*b^{-k}*ln{b} = b^{-k}*\ln{b^{-k}}
$$



得到：$$1-b^{-k}= b^{-k} $$ ，$$b^{-k} = \frac{1}{2}$$， $$e^{\frac{-kn}{m}} = \frac{1}{2}$$，得到：



$$
\frac{kn}{m}=\ln{2} \quad,\quad k=\ln{2}*\frac{m}{n}=0.7*\frac{m}{n}
$$



因此，可以得出，当 $$k=0.7\frac{m}{n}$$ 时，可以得到最低误判率：



$$
p(error) = (1-\frac{1}{2})^k = 2^{-k} = 2^{-\ln{2}*\frac{m}{n}}\approx0.6185\frac{m}{n}
$$



> 由上可以得出，若要 $$p(error)<1/2$$，则 $$k\geq1$$。即：
>
> 
>
> $$
> \frac{m}{n}\geq\frac{1}{ln{2}}
> $$
>
> 
>
> 若要保持$$p(error)$$不变，则需要保持$$\frac{m}{n}$$不变，即：布隆过滤器的bit数m与添加的元素集合大小n需要保持线性同步变化。



## 布隆过滤器的设计流程

根据布隆过滤器的设计基础，设计布隆过滤器：

用户输入：

- 需要添加的元素总数：n
- 期望的误判率：p

### 根据用户输入计算最优的m



$$
p(error) = (1-\frac{1}{2})^k
$$

$$
ln{p} = ln{2}*(-ln{2})*\frac{m}{n}
$$



得到布隆过滤器中bit位数量m：


$$
m = -\frac{n*ln{p}}{(ln{2})^2}
$$


### 计算最优k值

由m，n计算布隆过滤器中hash function数量k：


$$
k = ln{2}*\frac{m}{n} = 0.7*\frac{m}{n}
$$


> 由 $$k=0.7*\frac{m}{n}$$ 使得 $$p(error)$$ 最小。$$p(error)=(1-e^{(-\frac{nk}{m})})^k$$ 中的 $$e^{-\frac{nk}{m}}=\frac{1}{2}$$
>
> 其为某bit位在插入了n个元素之后，未被置位的概率



> 若要保持错误率低，布隆过滤器的空间使用率需低于50%



当k取最优时，


$$
p(error) = 2^{-k}
$$


$$
log_2{p} = -k
$$


$$
k = log_2{\frac{1}{p}}
$$


$$
ln2*\frac{m}{n} = log_2{\frac{1}{p}}
$$



得到：$$\frac{m}{n}=ln2*log_2{\frac{1}{p}} = 1.44*log_2{\frac{1}{p}}$$

验证，若取$$p=1\%$$时，存储每个元素需要 $9.6bits$，


$$
\frac{m}{n} = 1.44*log_2{\frac{1}{0.01}} = 9.6bits
$$


>这9.6bits包含了被置为1的bit数，也包含了没有被置1的bits数
>
>
>$$
>k = 0.7*\frac{m}{n}=0.7*9.6bits=6.72bits{（被置1的bit位数）}
>$$
>
>每当将误判率降低为原来的1/10，则存储每个元素需要多消耗4.8bits
>
>$$
>\frac{m1}{n1} - \frac{m2}{n2} = 1.44*log_2{a} - 1.44*log_2{\frac{a}{10}}=1.44*lof_2{10}=4.8bits
>$$

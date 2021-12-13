---
title: 【学习笔记】Web安全 - 哈希
tags:
  - security
  - web
date: 2021-12-13 18:30:55
---


### 哈希与加密

从下图中，我们可以看到哈希与加密的不同：

1. 哈希是单向的，而加密是可逆的。
2. 两者所生成结果的信息量不同：哈希算法通常用于数据摘要，生成相同长度的文本；而加密算法生成的密文长度与明文长度有关，加密生成的密文需要能够被解密恢复成明文。

![](https://pic2.zhimg.com/737fa75c8409117d6202fd3b0c636d21_r.jpg)

### 哈希算法

一个优秀的哈希算法，需要满足以下几个特性：

1. 正向快速：给定原文和 Hash 算法，在有限时间和有限资源内能计算得到 Hash 值；
2. 逆向困难：给定（若干）Hash 值，在有限时间内无法（基本不可能）逆推出原文；
3. 输入敏感：原始输入信息发生任何改变，新产生的 Hash 值都应该发生很大变化；
4. 碰撞避免：很难找到两段内容不同的明文，使得它们的 Hash 值一致（即发生碰撞）。
   - 弱抗碰撞性：给定原文前提下，无法找到与之碰撞的其它原文
   - 强抗碰撞性：无法找到任意两个可碰撞的原文

常见的算法有：

1. MD：主要包括 MD4 和 MD5 两个算法。MD4（RFC 1320）是 MIT 的 Ronald L. Rivest 在 1990 年设计的，其输出为 128 位。MD4 已证明不够安全。MD5（RFC 1321）是 Rivest 于 1991 年对 MD4 的改进版本。它对输入仍以 512 位进行分组，其输出是 128 位。MD5 比 MD4 更加安全，但过程更加复杂，计算速度要慢一点。MD5 已于 2004 年被成功碰撞，其安全性已不足应用于商业场景。
2. SHA：由美国国家标准与技术院（National Institute of Standards and Technology，NIST）征集制定。首个实现 SHA-0 算法于 1993 年问世，1998 年即遭破解。随后的修订版本 SHA-1 算法在 1995 年面世，它的输出为长度 160 位的 Hash 值，安全性更好。SHA-1 设计采用了 MD4 算法类似原理。SHA-1 已于 2005 年被成功碰撞，意味着无法满足商用需求。为了提高安全性，NIST 后来制定出更安全的 SHA-224、SHA-256、SHA-384 和 SHA-512 算法（统称为 SHA-2 算法）。新一代的 SHA-3 相关算法也正在研究中。

### 数字摘要

由于哈希算法碰撞避免的特性，它通常被用于进行数字摘要。通过生成的数字摘要，可以进行数据的校验。如：网络下载的资源是否遭受过篡改、鉴权协议（[挑战应答方式](https://baike.baidu.com/item/%E6%8C%91%E6%88%98%E5%BA%94%E7%AD%94%E6%96%B9%E5%BC%8F/191313)）。下面我们将以用户密码校验这个应用场景为例看下哈希算法的用途。

#### 用户密码校验

为了保障用户密码的安全性，在数据传输与存储的过程中直接使用用户密码实际都是不安全的。通过哈希生成数字签名，在传输和校验的过程中，只匹配使用密码生成的数字签名而不是密码，可以保障密码的安全性。在以下的场景中，我们会假设：1. 数据的传输是不安全的，数据可能会被中间人截获、重放；2. 数据的存储也是不安全的，攻击者可能会[拖库](https://www.zhihu.com/question/40059755/answer/392404577)、撞库。

**使用明文或者加密**

通过明文传输显然是不安全的。而即便使用加密算法，一旦私钥泄露、数据库泄露，用户密码同样也会被泄露。

**使用哈希**

单纯使用哈希和明文传输实际上并没有本质区别。

1. 从数据传输角度来看：攻击者可以进行中间人重放攻击，截取用户登录请求，使用同样的哈希值即可伪造用户进行登录。
2. 从数据存储角度来看：攻击者获取数据库后，可以使用rainbow table，从哈希值中反向计算处可能的密码明文。

**使用哈希+动态salt**

所谓salt就是一段随机的字符串，通过在原文上加上这段字符串，可以增加攻击的成本。常用的salt就是验证码，由于验证码是动态生成的，这保障了每次能够成功登录的数字签名是不同的。它的优势体现在：

1. 从数据传输角度来看：由于数字签名是动态的，无法再实施中间人重放攻击。
2. 从数据存储角度来看：由于每个rainbow table对应的是一套哈希算法，因此对每个验证码都需要生成一个独立的rainbow table，这极大增大了暴力破解的成本。

**使用bcrypt, scrypt**

随着内存大小的提升、和显卡并行能力的支持，使用哈希+动态salt的方式也不再那么安全。而[bscrypt](https://en.wikipedia.org/wiki/Bcrypt)、scrypt这类算法都有一个特点，即：算法中都有个**因子**，用于指明计算密码摘要所需要的资源和时间，也就是**计算强度**。计算强度越大，攻击者建立rainbow table越困难，以至于不可继续。由于计算强度因子的存在，随着算力的提升，通过调整因子，可以保障密码仍然不轻易被攻破。同时，算法的设计也确保了已有用户仍然能够正常登录。

以bscrypt为例，一个bscrypt的hash字符串的格式如下。由于cost字段的存在，即便后来调整了计算强度因子，老的用户仍然可以通过原有的低cost哈希值进行登录。

```
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
\__/\/ \____________________/\_____________________________/
Alg Cost      Salt                        Hash
```

bcrypt算法的结果就是使用[Blowfish](https://en.wikipedia.org/wiki/Blowfish_(cipher))算法对文本"OrpheanBeholderScryDoubt"进行64次加密的结果。它利用了blowfish进行key change[操作慢的特性](https://en.wikipedia.org/wiki/Blowfish_(cipher)#:~:text=Each%20new%20key%20requires%20the%20pre%2Dprocessing%20equivalent%20of%20encrypting%20about%204%20kilobytes%20of%20text%2C%20which%20is%20very%20slow%20compared%20to%20other%20block%20ciphers.)，在key初始化的时候调用`2^cost`次ExpandKey函数。

### 彩虹表

​	使用哈希来替代明文存储数据也并非安全的。仅管哈希无法通过直接逆向计算得到密码明文，攻击者仍然可以通过暴力计算得到明文和哈希的映射关系。以此间接地获得密码明文。

​	[rainbow table](https://en.wikipedia.org/wiki/Rainbow_table)就是一个预先计算好的存储哈希函数输入输出映射关系的表。通过空间换时间，攻击者可以通过表查询的方式快速获得密码明文。值得一提的是，rainbow table使用[哈希链](https://en.wikipedia.org/wiki/Rainbow_table#:~:text=to the right.-,precomputed hash chains)的方式来减小表的大小。在rainbow table中只存储哈希链的**开始值**和**结束值**，在查询表时：

1. 计算输入哈希值的哈希链
2. 每次计算后，尝试匹配表中哈希链的**结束值**
3. 匹配到结束值后，即可查询到包含该输入的哈希链的**开始值**
4. 最后，通过开始值即可还原整条哈希链，在哈希链中我们就可以获取输入哈希值所对应的明文

以下图为例：

1. 计算哈希链，尝试与rainbow table（左侧）中**结束值**（黄色）进行匹配
2. 当执行两次R函数时，匹配到了项（passwd - linux23）
3. 通过**开始值**（绿色）恢复哈希链，获取密码 - `culture`

![Simple rainbow search.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/7/72/Simple_rainbow_search.svg/650px-Simple_rainbow_search.svg.png)

可以看到，在图中，计算哈希链使用了多个R函数。这是一种减少哈希碰撞的优化的算法：只要不是在不同哈希链的同个序号发生碰撞，哈希链就不会出现合并的问题（因为R函数不同，同样的哈希值可以计算出不同结果）。仅管无法避免原型链的重复，但这可以减小总体碰撞的数量，增大指定表大小下获得正确结果的可能性。但同时它也将查询单个项的时间复杂度由O(N)增加到了O(N^2)。（但这是可以接受的，因为如果使用普通的算法，rainbow table在表项增加时会由于哈希链合并问题而变得低效；而通常会增加表的数量，在每个表中查询来解决这个问题，但这同样也会增加查询的次数，而且还增加了表大小。）

> 哈希链：在哈希链中，有两个关键的函数：1. H: hash function 哈希函数；2. R: reduction function 规约函数（能够将任意哈希值映射成特定字符的纯文本值，并非哈希函数的反函数）。这样，通过反复执行R和H，我们就可以得到一条哈希链（如下图所示）。由于H和R已知，对于任意输入的哈希值，通过计算**结束值**（即：kiebgt），我们可以判断它是否在数据库存储的某条哈希链之上；而通过**开始值**（即：aaaaaa）我们可以完整复原整条哈希链。

![{\mathbf  {aaaaaa}}\,{\xrightarrow[ {\;H\;}]{}}\,{\mathrm  {281DAF40}}\,{\xrightarrow[ {\;R\;}]{}}\,{\mathrm  {sgfnyd}}\,{\xrightarrow[ {\;H\;}]{}}\,{\mathrm  {920ECF10}}\,{\xrightarrow[ {\;R\;}]{}}\,{\mathbf  {kiebgt}}](https://wikimedia.org/api/rest_v1/media/math/render/svg/bbacb4ee3811ce261fa6023c6de90718e22c7b49)

## References

- [Download Security for Web Developers PDF](https://oiipdf.com/security-for-web-developers) （动物书）
- [security-guide-for-developers](https://github.com/FallibleInc/security-guide-for-developers) (已长年未更新，但提供了web安全所涉及知识的提纲)
- [ security-guide-for-developers](https://github.com/TransformCore/security-guide-for-developers)
- 知乎
  - [Web *前端*密码*加密*是否有意义？](https://www.zhihu.com/question/25539382/answer/547509246)
  - [当我们在谈论*前端加密*时，我们在谈些什么](https://zhuanlan.zhihu.com/p/22289839)
  - [互联网网站应该如何存储密码？ - 韩竹的回答 - 知乎](https://www.zhihu.com/question/20479856/answer/15243887)
- [Hash 算法与数字摘要](https://yeasy.gitbook.io/blockchain_guide/05_crypto/hash#chang-jian-suan-fa)

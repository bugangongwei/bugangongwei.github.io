

# Hash 加密算法

这完全取决于您的要求和期望。

以下是这些哈希函数算法之间的简要区别：

## CRC（CRC-8 / 16/32/64）
是不加密的哈希算法（它使用基于循环冗余校验的线性函数）
可以产生9、17、33或65位
不能用于加密目的，因为不做任何加密保证，
不适合用于数字签名，因为它很容易反转2006，
不应用于加密目的，
不同的字符串会产生碰撞，
于1961年发明，并用于以太网和许多其他标准，

## MD5
是一种密码哈希算法，
产生一个128位（16字节）的哈希值（32位十六进制数字）
它是一个加密哈希，但是如果您担心安全性，则认为它已弃用，
已知的字符串具有相同的MD5哈希值
可以用于加密目的，

## SHA-1
是一种密码哈希算法，

产生一个160位（20字节）的哈希值，称为消息摘要

它是一个加密哈希，自2005年以来，它不再被认为是安全的，

可以用于加密目的，

发现sha1碰撞的一个例子

最早于1993年（以SHA-0的形式）发布，然后在1995年以SHA-1的形式发布，

系列：SHA-0，SHA-1，SHA-2，SHA-3，

综上所述，使用SHA-1不再被认为可以抵抗资金充裕的对手，因为2005年，密码分析家发现对SHA-1的攻击，这表明它对于持续使用Schneier可能不够安全。美国国家标准与技术研究院（NIST）建议，联邦机构应停止将SHA1-1用于要求抗碰撞的应用，并且必须在2010 NIST之后使用SHA-2 。

因此，如果您正在寻找一种简单，快速的解决方案来检查文件的完整性（针对损坏），或者出于性能方面的考虑，可以考虑使用CRC-32，对于哈希，您可以考虑使用MD5，但是，如果您正在开发专业应用程序（应该是安全且一致的），请避免使用任何碰撞概率-请使用SHA-2及更高版本（例如SHA-3）。
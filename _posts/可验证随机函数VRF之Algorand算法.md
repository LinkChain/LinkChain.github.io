原文链接：https://zhuanlan.zhihu.com/p/29429006

DFINITY的阈值接力结构与可验证随机函数（VRF）密切相关，VRF算法作为一种基于密码学的新型共识模型被提出，最大的优势是快速共识、抗攻击能力、极低算力需求（较高的经济性），业界已问世的解决方案有图灵奖得主[Micali](http://link.zhihu.com/?target=https%3A//arxiv.org/find/cs/1/au%3A%2BMicali_S/0/1/0/all/0/1)提出的Algorand算法和DFINITY中基于BLS的算法。这一篇将对Algorand算法做一个解析。

作者：丛宏雷

[Algorand论文地址](https://arxiv.org/abs/1607.01341)

## **Algorand设计考虑**

算法假设：

1\. 网络中诚实节点的数目始终占优。

2\. 节点可以自由地随时加入网络，而不需要申请。

网络中每个节点通过一个公钥地址（同时也是钱包地址）表示，对于新加入的节点地址，只有被网络中其他节点转账成功（即钱包余额大于0）后，才可以参与到网络中的区块共识。

3\. 攻击者也是动态变化的（诚实节点随时可能变为攻击者）。

网络假设：

网络消息传播时间上限：固定时间内完成对固定比例的用户的网络传播。

比如，比特币：1KB消息，在1秒钟内完成全网95%的传播，而1MB消息需要1.5分钟完成全网95%的传播。

Algorand的性质

l 可控制的分叉风险，（足够小概率的分叉可能性，）

l 需要很少的计算量

l 链上生成一个块的延迟近似于在网络中完成区块广播的延迟

l 不要求网络节点7x24在线 （Lazy Honesty）

## **Algorand算法**

## leader的选择

可能的攻击：

1\. 尽管可以通过对上一个区块的哈希计算来确定构建下一个区块的leader节点和verifier set节点，但是由于哈希函数自身的性质，攻击节点只需要在区块中添加一些微小的改动就可以很大影响下一个区块的leader节点的选择，从而破坏leader／verifier的随机性。为保证完全随机，在区块中引入block quantity，Qr（r为第r个块）一个区块的Qr值只有在当前区块的leader在整个网络中被揭晓时才能最终确认，从而使攻击者无法事先攻击。

2\. 即使leader／verifier的选择是完全随机的，攻击者也有可能在leader／verifier被揭晓的同时，马上攻击这些节点，从而控制leader/verifier。为解决这个问题，采用的方案是设计多个潜在的leader，并且每个潜在leader都独立完成区块的构建，然后每个潜在leader都将自己的认证信息，构建的区块一起发送到网络中，通过共识算法选定真正的leader。由于在真正leader的身份在被揭晓之前，网络已经完成了区块数据的广播，即使攻击者攻陷了真正的leader也无法改变区块的数据。

3\. 算法中，区块生成都需要经过若干步骤，如果在算法执行过程中verifier节点被攻击，比如网络被断开，可能造成算法无法持续执行下去，从而造成整个区块无法确认，整个网络被停滞。而且，也无法要求每个节点都7x24在线，始终为整个网络进行服务。因此设计算法支持player-replaceable，从而使任何节点都可以随时被其他节点接管。

具体算法：

输入参数：（r, Qr-1）：其中r为第r轮（第r个区块），Qr-1为上一个区块的Qr值

判断节点是potential leader的条件：

H（Sig（r, 1, Qr-1）） <= 1 / size(PKr-k)

size(PKr-k)为第r-k轮中网络中参与区块共识的公钥个数（也就是钱包的数目）

所有potential leader中哈希值最小的为真正的leader

## verifier的选择

定义回看参数k，概率p

输入参数：（r, s, Qr-1）: 其中r为第r轮，s为第s步，Qr-1为上一轮的Q值

判断方法：对于i，如果 H（Sig(r, s, Qr-1)）< p，则i为verifier

## Qr的计算

如果leader_r存在而且在给定的时间内发布了对应的credential：

Qr = H(Sigleader_r(Qr-1), r-1)，即首先用leader对于Qr-1的签名，再对签名和r-1计算哈希

否则: Qr = H(Qr-1, r-1)，即对Qr-1和r-1计算哈希

## 一次性密钥

节点密钥只用于以下用途：

1\. 网络支付

2\. 生成本节点的认证证书（credential）

3\. 认证一次性密钥

对于节点在网络中发送的每个消息，采用一次性密钥进行签名。因此攻击者无法盗取消息密钥对网络消息进行改写。

## m次Binary BA

m次Binary BA算法所有节点，执行m步下列消息传播，在每一步：

1\. 每个节点发送比特b和签名Sigi(r, s)给网络中所有节点（包括自身）

2\. 每个节点收集网络中广播的消息

3\. 判定：

a) 如果＃s(0) >= 2t + 1，则设定 b = 0

b) 如果＃s(1) >= 2t + 1，则设定 b = 1

c) 否则设定b为所有接收到消息的节点的消息签名的最小哈希值lsb

b = lsb(min H(Sigi(s)))

通过以上算法可以证明，拜占庭环境（n，t）配置下，如果 n>= 3t + 1，达成共识的概率 >= m

也就是说，在每一步开始达成共识的概率至少为

## **Algorand实现**

算法参数选择：

n t : 每个验证组中存在的攻击节点最大数目，即BFT算法中的f

n k：回看参数，在r－k轮共识时有效的网络节点才能参与到此轮共识

n p：每个验证组中节点占共识算法所有参与节点的比重，控制验证节点组的大小

n r：网络传播时间界限内消息在网络中的reachability, r(n – t)
> 2t + 1

n A：区块数据的传播延迟

n ƛ：哈希数据的传播延迟

整个共识算法分为3+m步，每一步都定义严格的时间界限。

第一步：为潜在leader向网络传播备选区块，所需时间依赖于区块数据大小，定义时间界限为A

后续步骤都只包含共识区块的哈希值，定义每一步的时间界限为ƛ

每一轮共识算法所需要的严格时间界限为：T = A + (m+2) ƛ

## **1\. 潜在leader生成备选区块**

如果一个节点判断自己为潜在leader，执行以下操作

1\. 搜集待执行的事务，构建一个尽可能大的区块，组成PAYi,
r

2\. 构造备选区块Bi, r = (r,
PAYi, r, Sig_i(Qr-1), H(Br-1))

3\. 生成一个一次性密钥对，对H(Bi,r)进行签名

4\. 销毁密钥对的私钥

5\. 向网络广播消息mi,(r, 1) : 第i个节点，在第r轮的第1个step中发出的消息

a) 节点的leader credential ＝ Sigleader_r(r, 1, Qr-1)

b) 一次性公钥

c) 添加一次性公钥后的COMi(r)

d) 备选区块Bi,r

e) 一次性密钥对备选区块的签名

## **2\. 第二验证组验证备选区块**

如果一个节点判断自己为(r, 2)的验证组，执行以下操作：

1\. 等待给定时间（系统配置时间A）完成对备选区块的收集

2\. 验证消息的正确性

3\. 计算所有备选leader的credential的哈希，选择最小的哈希值为实际leader

4\. 验证实际leader发送过来的区块内容的有效性，否则设置区块为空块，得到实际区块B

5\. 生成一个一次性密钥对，对H(B)进行签名

6\. 销毁密钥对的私钥

7\. 向网络广播消息mi,(r, 2)

a) 当前验证节点的verifier credential =
SIGi(r, s, Qr-1)

b) 一次性公钥

c) 添加一次性公钥后的commitment COMi(r)

d) 实际区块的哈希值

e) 一次性密钥对实际区块哈希值的签名

f) 实际leader所发消息的（a, b, c, e）项内容

## **3\. 第三验证组收集验证结果**

如果一个节点判断自己为(r, 3)验证组，执行以下操作

1\. 等待系统给定时间（A+ƛ），收集第二验证组的验证结果

2\. 如果存在>=2t+1个验证节点对某个区块的完成验证，则执行如下操作

a) 生成一个一次性密钥对，对对应区块的哈希进行签名

b) 销毁密钥对的私钥

c) 向网络广播消息mi,(r, 3)

i. 当前验证节点的verifier credential

ii. 一次性公钥

iii. 添加最新一次性公钥后的commitment，COMi(r)

iv. 一次性密钥对实际区块哈希值的签名

## **4\. 第四验证组进行0/1投票**

如果一个节点判断自己为(r, 4)验证组，执行以下操作

1\. 等待系统给定时间，收集之前验证组的验证结果

2\. 计算分级共识

a) 如果存在 >= 2t+1个验证节点对某个区块完成验证，v = x, g = 2

b) 如果存在 >= t+1个验证节点对某个区块完成验证，v = x, g = 1

c) 否则 g ＝
0

3\. 如果 g = 2，设定 b = 0，否则设定 b = 1

4\. 生成一次性密钥对，对b进行签名，然后销毁私钥

5\. 向网络发送消息 mi,(r, 4)

a) 当前验证节点的verifier credential

b) 一次性公钥

c) 添加最新一次性公钥后的commitment，COMi(r)

d) 一次性密钥对b的签名

## **5\. 后续(5 - 2+m)验证组持续投票**

此步骤为循环执行，直至达成共识。如果共识，执行步骤6.

如果一个节点判断自己为(r, s)验证组，执行以下操作

1\. 等待系统给定时间，收集之前验证组的投票结果

2\. 计算分级共识

a) 如果存在 >= 2t+1个节点投票为0，b = 0

b) 如果存在 >= 2t+1个节点投票为1，b = 1

c) 否则设定b为
lsb(min H(Sig(s))

3\. 生成一次性密钥对，对b进行签名，然后销毁私钥

4\. 向网络发送消息 mi,(r, s)

a) 当前验证节点的verifier credential

b) 一次性公钥

c) 添加最新一次性公钥后的commitment，COMi(r)

d) 一次性密钥对b的签名

## **6\. 完成区块签名**

1\. 等待系统给定时间，收集投票结果

2\. 如果超过2t+1个节点投票为0，选定B为实际leader所提供的区块，否则设定B为空块

3\. 生成一次性密钥对，对H(B)进行签名，并销毁私钥

4\. 向网络发送消息mi,(r, s)

a) 当前验证节点的verifier credential

b) 一次性公钥

c) 添加最新一次性公钥后的commitment，COMi(r)

d) 一次性密钥对H(B)的签名

至此完成共识：

Br = B

CERTr = { mr, 3+m }

原文链接：https://mp.weixin.qq.com/s/FD_zkmNcLHv440Y2oqpoag

## Algorand背景介绍 (Background)
Algorand是MIT机械工程与计算机科学系SilvioMicali教授与合作者于2016年提出的一个区块链协议，主要是为了解决比特币区块链采用的pow共识协议存在的算力浪费，扩展性弱、易分叉、确认时间长等不足。因此SilvioMicali教授在algorand区块链协议中提出了一种新的共识协议BA，其目标是：
1.能耗低，不管系统中有多用户，大约每1500名用户中只有1名会被系统挑中执行长达几秒钟的计算。
2.民主化，不会出现类似比特币区块链系统的“矿工”群体。
3.出现分叉的概率低于一兆分之一（即10-18）。假设Algorand中平均每分钟产生一个区块，这个概率意味着平均每190万年出现一次分叉。
4.可拓展性好。

##Algorand概述 (Algorand, in aNutshell)
(一)    假想环境 (Setting)
1.在无需准入(permissionless)和需要准入(permissioned)环境下都能正常工作，当然在需要准入的环境下能表现得更好。
2.敌手能力很强(VeryAdversary Environments)
1) 可以立刻腐蚀(corrupt)任何他想要的用户(user)，前提条件是：在无需准入的环境中，需要2/3以上的金额(money)属于诚实(honest)用户；而在需要准入且一人一票的环境中，需要2/3以上的诚实用户
2) 完全控制和完美协调已腐蚀的用户
3)调度已腐蚀用户所发送的所有信息，前提条件是，诚实用户发送的消息需要在一定的时间内发送到95%以上的其他诚实用户，而其延迟只和消息大小有关。


(二)    主要特点 (Main Properties)

1.  计算量为最优，不论系统中存在多少用户，每1500个用户的计算量之和最多仅为几秒钟
2. 一个新的区块在10分钟内被生成，并且永远不会因分叉问题而被主链抛弃，事实上Algorand发生分叉的概率微乎其微
3. 没有矿工，所有有投票权的用户都有机会参与新块的产生过程

(三)    Algorand采用的技术 (Algorand’s Techniques)

1.  一种新的拜占庭共识(BA: Byzantine Agreement)协议，即BA*，这也是后文将重点介绍的协议
2. 采用密码学抽签：BA*协议中每一轮参与投票的用户都可以证明确实是随机选取的
3. 种子参数：选取完全无法预测的种子参数，从而保证不被敌手所影响，上一轮的种子参数会参与下一轮投票用户的生成
4. 秘密抽签和秘密资格：所有参与共识投票的用户都是秘密地得知他们的身份，投票后他们的身份被暴露，虽然敌手可以马上腐蚀他们，但是他们发送的消息已经无法被撤回，另外在消息生成后，用于签名的一次性临时秘钥(后文会提到)会立刻被丢弃，使得敌手在该轮无法再次生成任何合法消息
5. 用户可替换(player-replacable)：在拜占庭协议中，每个参与共识者需要投票多轮以达成共识，而在BA*中这并不可行，因为一旦投票后自己就暴露了，会被敌手腐蚀。配合密码学秘密抽签，用户会秘密知道自己有且只有参与某特定时刻的投票的资格，只要在该时刻参与投票，因为接下来投票权会转移给别人，这就使敌手的腐蚀失去了意义。
另外，诚实的用户可以是懒惰的(Lazy Honesty)。一个用户不需要时刻在线，可以根据适当的条件适当在线并参与共识即可。

## Algorand的前置条件 (Preliminaries) 
(一)    密码学前置条件 (Cryptographic Primitives)

1.理想(ideal)的hash函数
给定一个hash函数H，可以生成任意长度字符串s的hash值H(s)。要求在给定H(s)的长度后，能做到分布是完全均匀和独立于s的(H as a *random oracle*)。在本文中，H(s)的长度固定为256位比特(256-bit long)，事实上这个长度已经能够抵抗碰撞(*collision-resilient*)，即给定两个字符串x，ｙ，其H(x)=H(y)的概率足够小(根据生日悖论，发生碰撞需要的尝试次数约为2<sup>256/2</sup>=2<sup>128</sup>)。

2.数字签名(Digital Signing)
数字签名体系(*digital digning scheme*)包括三个高效函数：基于概率的公私钥生成(*key generator*)函数G，签名算法(signing algorithm)S，以及验证算法(verification algorithm)V。
定义用户i对消息m的签名如下：
![image](http://upload-images.jianshu.io/upload_images/3959874-5a5722e466f462d9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中pk<sub>i</sub>和sk<sub>i</sub>为用户i的公钥和私钥。这里使用H(m)是利用了其能够抵抗碰撞的特性。

通过V可以验证这个签名：

若s = sig<sub>i</sub>(m)，则V(pk<sub>i</sub>,m, s) = YES

并且数字签名很难被伪造，即很难找到另外一个ｍ使得V(pk<sub>i</sub>, m, s) = YES成立。作为用户i要做的，就是保管好自己的私钥sk<sub>i</sub>，并公开其对应的公钥pk<sub>i</sub>，使得其他人能够验证签名。

一般情况下是不能从签名sig<sub>i</sub>(m)中还原ｍ的，同时需要用户信息而找到其对应的公钥，所以一般情况下的“签名”，包括用户身份信息i，消息原文ｍ，以及对应的签名，定义如下：

![image](http://upload-images.jianshu.io/upload_images/3959874-95d43ec18280c421?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数字签名应该是唯一的，即通过数字签名体系(G, S, V)，很难找到两个不同的签名s ≠ s’，使得V(pk, m, s) =V(pk, m, s’) = 1。

在Algorand里，用户使用数字签名来：

为交易生成签名

生成凭证(credential)，用以证明自己有投票权

为消息ｍ生成签名，这里需要使用一次性临时秘钥

(二) 理想的公开账本 (The Idealized Public Ledger)

在Algorand中，我们假设账本的模型如下：

1.拥有一个初始状态(*Initial Status*)，假设每个公钥pk<sub>1</sub>，… ，pk<sub>j</sub>以及其对应的金额为a<sub>1</sub>，…，a<sub>j</sub>，表示如下：

S<sup>0</sup>= (pk<sub>1</sub>, a<sub>1</sub>), …, (pk<sub>j</sub>, a<sub>j</sub>)

并且这个状态作为系统公共常识(common knowledge of the system)而存在。

2.交易(Payments)，我们定义交易如下：

℘= SIG<sub>pk</sub>(pk, pk’, a’, I, H(I’))

pk和pk’分别为支付方和接收方的公钥，在这里公开的信息为I，敏感信息I’被H混淆。

3.    魔法账本(The Magic Ledger)，这是一个无法篡改的列表，定义为：

L= PAY<sup>1</sup>, PAY<sup>2</sup>, …

每个PAY<sup>r+1</sup>都包括了该块中所有的交易，并且只在PAY<sup>r</sup>之后加入，理想状态下，一个新的块会在固定或有限(fixed or finite)的时间内加入账本。

这个账本模型比比特币的账本模型更加一般化，每个公钥相当于一个钱包，其金额可以通过查询当前的状态S<sup>r</sup>得到，而S<sup>r</sup>是通过S<sup>0</sup>经过L = PAY<sup>1</sup>, …, PAY<sup>r</sup>线性变换后计算得到。

(三) 基本概念和符号 (Basic Notions and Notations)**

1. 公钥，用户和拥有者(Keys,Users, and Owners)，一般情况下，公钥和用户其实是等价的，而拥有者一般是表示拥有这个钱包的使用权，即拥有这个公钥对应的私钥，当一个公钥j付款给另一个公钥i后，可以理解为用户i加入了系统。

2. 无需准入和需要准入的系统(Permissionlessand Permissioned Systems)，如果一个公钥可以随意加入系统，一个用户可以拥有多个公钥，则这是一个无需准入的系统，否则便是一个需要准入的系统。

3. 同速时钟(Same-SpeedClocks)，用户之间的时间可以不同，但是时钟的走速必须相同。

4. 轮次(Round)，Algorand的轮次对应于区块链的块高度的概念，我们统一使用上标表示轮次。在轮次r>0开始前，所有公钥的集合为PKr，而其系统状态为：

![image](http://upload-images.jianshu.io/upload_images/3959874-317735077088d07d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意其中![image](http://upload-images.jianshu.io/upload_images/3959874-9f6b3275f7ce5467?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

是公钥i对应的金额，在这里因为a是一个数字，我们采用了(r)以区别于a的r次方。注意到PK<sup>r</sup>是能够从S<sup>r</sup>中获得的，而S<sup>r</sup>中也能包括除了公钥和金额外的其他状态。可以理解，在轮次0，PK<sup>0</sup>和S<sup>0</sup>作为初始公钥集(*initial public keys*)和初始状态(*initial status*)，是系统的公共常识。而在轮次r，已经变为PK<sup>1</sup>，… ，PK<sup>r</sup>和S<sup>1</sup>，… ，S<sup>r</sup>。在轮次r，系统从S<sup>1</sup>迁移到S<sup>r+1</sup>，即：

![image](http://upload-images.jianshu.io/upload_images/3959874-2e6f29aae4ab1740?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 正式交易集(OfficialPaysets)，第r轮的交易集即PAY<sup>r</sup>是由该轮次的所有交易所组成。而通过Algorand算法选出的交易集(可能为空集)即为正式交易集。用户只有在某轮次通过正式交易集中接收了一定金额之后，才被认为加入了系统。事实上正式交易集PAY<sup>r</sup>是状态S<sup>r</sup>到状态S<sup>r+1</sup>的映射：

![image](http://upload-images.jianshu.io/upload_images/3959874-bb142163cd018f9c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不难理解，一个理想的系统中，可以从S<sup>0</sup>和PAY<sup>0</sup>，… ，PAY<sup>r</sup>获取S<sup>r+1</sup>。

(四)   块和证明块 (Blocks and Proven Blocks)

一个块包括轮次r，该轮的正式交易集PAY<sup>r</sup>，一个随机种子Q<sup>r</sup>，以及上一个块的hash，也就是说从B<sup>0</sup>之后，一个传统的区块链大致如下所示：

![image](http://upload-images.jianshu.io/upload_images/3959874-be1d4bb915aaabc0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当一个区块B<sup>r</sup>附带上凭证(certificate)，标记为CERT<sup>r</sup>，之后得到已经证明的块(proven block)![image](http://upload-images.jianshu.io/upload_images/3959874-18c578c9b68cfc5f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

。如此以来，魔法账本其实是一串已证明的块：

![image](http://upload-images.jianshu.io/upload_images/3959874-ea223f92bd6947bb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(五)  敌手模型 (The Adversarial Model)

1.      诚实和恶意用户(Honest and Malicious Users)，诚实用户的行为完全符合预定规则，如执行相应逻辑，收发信息等等，而这里的恶意(malicious)是指任何违反预定规则的行为，即拜占庭错误

2.      敌手(The Adversary)，敌手是一个有效的算法，即可以在多项式时间内，可以在任何时间，使任何用户变为恶意，即腐化(corrupt)用户，并且可以完全控制和协调所有恶意用户，可以以用户名字做出任何违反规则的行为，或是简单地选择不收发任何消息。在用户做出任何恶意行为前，没人能知道他已经被腐化了，而特定的行为能够暴露其已经被腐化的事实。但是这个敌手：

被约束在算力和密码学的范围内，即基本不能伪造诚实用户的数字签名
无法干扰诚实节点之间的消息传送。

3.     好人掌钱(HMM: Honesty Majority of Money)，我们假定一个连续的好人掌钱模型，对于一个非负整数k和一个实数h > 1/2，我们认为好人在第r-k轮掌握的钱的比例是大于h的。Algorand采用“向前看”的策略，即在第r轮参与投票的候选人，是从r-k轮选出来的，所以即使整个网络在r-k被腐化了，其真正掌权也需要等到地r轮。

(六)    通信模型（The Communication Model）

同比特币一样，Algorand采用点对点绯闻通信协议来完成消息的传播。

如果把消息的到达率(reachability)表示为ρ。模型认为一条消息被诚实用户发出到超过ρ的人接收到的时间只和消息长度μ ∈ Z<sup>+</sup>有关，定义该函数为λ<sub>ρ,μ</sub>。即从时间t发出的一条大小为μ的消息，一定能在t+λ<sub>ρ,μ</sub>之前到达比例为ρ的诚实用户。

Algorand面临的是多个用户同时发送各自消息，并且还要帮助其他用户传递合法消息的模型。所以函数λ被认为和三个变量相关，到达率ρ，消息长度μ ∈ Z<sup>+</sup>以及用户数n。模型修改为对所有的诚实用户n，在t时刻同时发送了各自的长度为μ ∈ Z<sup>+</sup>的消息m<sub>1</sub>，… ，m<sub>n</sub>，则所有这些消息一定能在t+λ<sub>n,ρ,μ</sub>之前到达比例为ρ的诚实用户。为了简化，后文一般情况下会使ρ＝１，并且不再提及到达率的问题。另外，敌手是可以控制任何用户，以加速某些消息的到达时间。

Algorand实例化 (Embodiments of Algorand)
Algorand能够在同步和异步网络中工作。为了方便理解，可以将其网络环境假想为一个同步完全网络 (SC networks: synchronous complete networks)。在这样的网络环境下，假设存在一个全局时钟，每次计时都是在整数点上r=1，２，…。在每个偶数点r，网络中的用户i独立并发地发送一个消息!](http://upload-images.jianshu.io/upload_images/3959874-1fe79fd6208bc00d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(消息可以为空)，而在r+1的时刻，j就可以收到这个消息，j知道消息是从i处发送的。敌手可以在任何奇数时间点上发动攻击，腐化诚实用户，联合恶意用户，控制其逻辑和消息发送等行为。

## **(一)    目标 (Objectives)**

在理想情况下，共识协议的最终目的为：

1.  完全正确性(*Perfect Correctness*)，即所有的诚实用户都共识在B<sup>r</sup>上。

2.  完整性为1(Completeness 1)：以１的概率，B<sup>r</sup>包含的交易集PAY<sup>r</sup>，是最大化的(maximal)，也就是包含尽可能多的有效交易。

而Algorand的目标更加现实，因为有恶意用户的存在，如果假设诚实用户的占比h>2/3，非正式地描述，Algorand的目标是：

*   保证以压倒性的概率做到完全正确，并且完整性接近h。

牺牲完整性带来的坏处是降低交易效率，但正确性得以保证，排除了分叉的风险。

(二)    出块者和验证者(Leader and Verifiers)**

Algorand采用了抽签的方式来获知自己的身份。

对于出块者而言，其出块权的计算公式为：

.H(SIG<sub>i</sub>(r,1, Q<sup>r-1</sup>)) ≤ p

这里Q<sup>r-1</sup>是随机表示种子，后文会介绍其公式。.H函数表示把H函数的输出字符串均匀地匹配到[0,1]区间之中去。这样以来，可以直接和概率作比较计算。在出块时，如果诚实用户发现自己有出块权(.H≤p)，会选择按照最大化的原则出块，同时可能有多个用户满足.H≤p。

验证者具有投票权，其计算公式为：

 .H(SIG<sub>i</sub>(r, s, Q<sup>r-1</sup>)) ≤ p’

其中s既是步骤(step)，从第二步开始，所有的验证者都采用这个公式。且每一步的p’都是一样的。如果诚实的用户发现自己有验证权(.H≤p’)，会在对应的步骤投票。

(三)    块的生成 (Glock Generation)**

轮次r的出块者l<sup>r</sup>不一定是诚实的，若为诚实，则出块形式应为：

![image](http://upload-images.jianshu.io/upload_images/3959874-a3e827b51fa5e5f7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

若出块者为恶意，其出块形式会是以下两种之一：

![image](http://upload-images.jianshu.io/upload_images/3959874-9e3b664d561ee008?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要注意，若PAY<sup>r</sup>确实为空集，其![image](http://upload-images.jianshu.io/upload_images/3959874-b165ed40e17b4357?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

，要注意这和空集是不同的，前者表示一切正常，但是后者表示出错了，因此输出了默认值。

不论那种表示形式，其随机数种子保存在第三项中，供用户计算凭证所用。

(四)    随机数种子Qr和向后看参数k (The Seed Q r and the Look-Back Parameter k)**

Algorand的随机数种子可以从上一轮的块的第三项中获取，计算公式为：

![image](http://upload-images.jianshu.io/upload_images/3959874-344025a0baab1209?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另外为了防止敌手控制，Algorand引入了向后看参数k，第r轮的共识参与者，需要在第r-k轮加入系统。

随机数种子和向后看参数的引入都是为了增加敌手根据当前的状态预测将来的难度，从而无法准确选择腐蚀目标。

(五)    概念(Notations)

r ≥ 0：当前轮次

s ≥ 1：当前步骤，其中第一步出块，后面步全部是投票

B<sup>r</sup>：轮次r的出块

PK<sup>r</sup>：在第r-1轮结束以及第r轮开始时的公钥集合

S<sup>r</sup>：在第r-1轮结束以及第r轮开始时的系统状态

PAY<sup>r</sup>：B<sup>r</sup>中的交易集

 l<sup>r</sup>：轮次r的出块者(leader)，出块者可以选择轮次r的交易集PAY<sup>r</sup>

SV<sup>r,s</sup>：轮次r，步骤s的验证者集

SV<sup>r</sup>：轮次r的验证者集，SV<sup>r</sup> = ∪<sub>s≥1</sub>SV<sup>r,s</sup>

MSV<sup>r,s</sup>和MSV<sup>r,s</sup>：诚实用户和恶意用户集合，易知MSV<sup>r,s</sup>∪HSV<sup>r,s</sup>= SV<sup>r,s</sup> 以及 MSV<sup>r,s</sup>∩HSV<sup>r,s</sup>=∅

n<sub>1 </sub>∈ Z<sup>＋</sup>和n ∈ Z<sup>+</sup>：出块者集和验证者集的成员个数的期望值，注意到n<sub>1</sub><<n，因为只要在概率上保证出块者集中至少有一个诚实用户，而验证者集中诚实用户要占多数

h ∈ (0, 1)：一个大于2/3的常数，代表诚实用户在系统中的比例，表示诚实用户或者在诚实用户手中的金额在每个PK<sup>r</sup>中的比例至少为h

H：密码学随机函数

⊥：一个特殊字符串，表示默认值，其长度等于H的输出

F ∈ (0, 1)：一个参数，用来表示允许的错误概率，当≤F时可以认为不可能发生，概率≥1-F可以认为几乎必然发生

p<sub>h </sub>∈ (0, 1)：出块者是诚实的概率，理想情况下等于h，在有敌手的情况下，需要分情况讨论分析

k∈ Z<sup>＋</sup>：向后看参数，第r轮的出块者和验证者在第r-k轮产生(严格说来，应为max{0, r-k}轮产生)，即SV<sup>r</sup>⊆ PK<sup>r−k</sup>

p<sub>1 </sub>∈ (0, 1)：在轮次r的第一轮，在r-k轮的用户被选入出块者集SV<sup>r,1</sup>的概率![image](http://upload-images.jianshu.io/upload_images/3959874-922a81af8c2f6dc5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

p∈ (0, 1)：在轮次r的第s轮，在r-k轮的用户被选入出块者集SV<sup>r,s</sup>的概率![image](http://upload-images.jianshu.io/upload_images/3959874-2724145845f2d428?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CERT<sup>r</sup>：B<sup>r</sup>的凭证集合，集合的凭证(即对H(B<sup>r</sup>)的签名)数应该大于t<sub>H</sub>。

 ![image](http://upload-images.jianshu.io/upload_images/3959874-77d0cdce2f81df2d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    ：被证明的块

![image](http://upload-images.jianshu.io/upload_images/3959874-d60fb23d372172d0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    ：用户i知道B<sup>r</sup>的本地时间，Algorand允许每个用户有自己的独立时间，只要其时钟走速相同即可

![image](http://upload-images.jianshu.io/upload_images/3959874-41e1ee956cf7fa82?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    ：每个用户i在第r轮第s步的开始和结束时间

  Λ和λ：表示执行第一步和其他步的时间上限

(六)    参数 (Parameters)**

1.     参数间的关系

 在轮次r，投票者和候选的出块者从集合PK<sup>r-k</sup>中选出。选择k使得敌手无法在轮次r-k-1以高于F的概率预测Q<sup>r-1</sup>，否则的话敌手可以在r-k轮引入恶意用户，他们可能成为第r轮的出块者或候选者

在每轮的第一步，选取n<sub>1</sub>使得SV<sup>r,1</sup>不为空集

2.     重要参数的选取

H的输出为256比特长

h=80%，n<sub>1</sub>=35

 Λ=1分钟，λ=10秒钟

3.      协议初始化，协议在时间0，轮次0开始，因为没有B<sup>-1</sup>或CERT<sup>-1</sup>，规定B<sup>-1</sup>是一个公开的参数，在其中包括Q<sup>-1</sup>，所有的用户都在时间0知道B<sup>-1</sup>


 BA*共识**

Algorand的拜占庭共识协议BA*，按顺序包括出块者(Leader)选举，分级共识(GC:Graded Consensus)协议和二元拜占庭(BBA*)协议几个步骤。这里涉及到额外几个概念和参数。

(一)    概念 (Notations)**

1.      m∈ Z<sup>＋</sup>：BBA*中的步数，是3的倍数

2.      L<sup>r </sup>≤ m/3：一个随机变量，是伯努利试验能看到１的尝试次数，每次尝试的概率为P<sub>h</sub>/2，最多尝试m/3次，所有尝试都失败时，L<sup>r</sup> = m/3，L<sup>r</sup>决定了需要生成B<sup>r</sup>的时间上限

3.      t<sub>H</sub> = 2n/3 + 1：协议需要的签名个数

4.      CERT<sup>r</sup>：B<sup>r</sup>的凭证集，包含t<sub>H</sub>个第r轮的验证者对于H(B<sup>r</sup>)的合法签名

(二)    参数 (Parameters)**

1.  参数间的关系

*   对轮次r的每一步s > 1，选择n，使得以下条件必然成立

|HSV<sup>r,s</sup>| > 2|MSV<sup>r,s</sup> | 以及 |HSV<sup>r,s</sup> | +4|MSV<sup>r,s</sup> | < 2n

      h越接近1，n可以越小

*   选取m，使得L<sup>r</sup> < m/3必然成立

2.      重要参数的选取

*   F = 10<sup>-12</sup>

*   n ≈ 1500，k = 40，m = 180

(三)    临时秘钥的生成 (Implementing Ephemeral Keys)**

权威机构可以为用户U生成秘钥，将主公钥(PMK: *public master key*)公开，将主私钥私(SMK: *secret master key*)下交给用户。假定共识上线为180步，用户需要参与共识100万轮共识，则用户可以利用主私钥生成10<sup>6</sup>*180个临时私钥，并将主私钥销毁。而其他人通过主公钥以及轮次和步数信息，可以生成对应的公钥，以验证签名。

当然另外也可以通过公钥默克尔树的方式来管理秘钥，用户提前将树根公开，每次发消息时将公钥和对应路径同时发出，其他人可以确认公钥的真实性，再用公钥验证签名。

(四)    算法实质 (Actual Protocol)**

*（因公式较多，以下以图片展示，可点击图片查阅）*

![image](http://upload-images.jianshu.io/upload_images/3959874-cbb7ef5dbb1dd9b4?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](http://upload-images.jianshu.io/upload_images/3959874-f1c97f6749ecb67a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[图片上传中...(image-89950c-1521609358967-1)]

![image](http://upload-images.jianshu.io/upload_images/3959874-3541720b5af6189f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(五)    协议分析 (Analysis of Algorand)**

1.  分级共识协议(GC: Graded Consensus Protocol)

其GC协议的定义如下：

若协议P中的所有用户为公共常识，每个用户i各自都知道任意的初始值v’<sub>i</sub>。

我们称P是一个(n-t)分级共识协议，若n个用户的每次行动中，至多只有t个恶意用户，其中n ≥ 3t + 1每个诚实用户i停机输出一个（值-级）对(a value-gradepair)(v<sub>i</sub>, g<sub>i</sub>)，其中g<sub>i</sub> ∈ {0, 1, 2}，他们满足如下条件：

*   对于所有的诚实用户i和j，|g<sub>i</sub> –g<sub>j</sub>| ≤ 1

*   对于所有的诚实用户i和i，g<sub>i</sub>,g<sub>j</sub>> 0 ⇒ v<sub>i</sub> = v<sub>j</sub>

*   若对某一v，所有的输入v’<sub>1</sub>= … = v’<sub>n</sub> = v，则对所有诚实用户i，都有v<sub>i</sub> = v以及g<sub>i</sub> = 2

GC协议的二阶段对应于BA*的Step2和Step3，以及拜占庭协议的prepare和commit部分,而输出(v<sub>i</sub>, g<sub>i</sub>)可以理解为commit票。若存在用户i，使得g<sub>i</sub> = 2，则其必然收到了2t + 1个对特定v的prepare票，即使其中有t个恶意用户，也就是说其他诚实用户，是少收到了对这个特定v的t + 1张prepare票，因此条件1成立。另外，在出票时间内，如果发现上一轮的用户给出不一致的票，则该用户的所有投票都会被视为作废，如此以来，每个用户只能给出一次有效票，如果两个v<sub>i</sub>和v<sub>j</sub>都有g<sub>i</sub>,g<sub>j</sub> > 0，那在一定commit轮发出了t+1张v<sub>i</sub>和t + 1张v<sub>j</sub>的票，则commit用户中有t + 1个用户收到了2t + 1张v<sub>1</sub>的prepare票，另有t + 1个收到了2t + 1张v<sub>2</sub>的prepare票，因为最多存在t个恶意用户，这需要每个恶意用户投两张prepare票，而如前述，恶意用户不能这么做，否则自己所有的票都会被视为作废，所以2成立。3显然成立，事实上，这相当于出块者为诚实的情况，由拜占庭协议可以保证所有诚实节点对v达成共识，而诚实节点的额数量为2t + 1，则对于任何用户i，g<sub>i</sub> = 2。

2.  二元拜占庭共识协议 (The Binary BA Protocol BBA*)

所谓二元既是共识结果为{0,1}，在BA*协议中，是用户i的证明消息的第一个参数ESIG<sub>i</sub>(b<sub>i</sub>)。当BBA*结束时，它是一个可靠的(n-t)-拜占庭共识协议，其中n ≥ 3t + 1。事实上应该理解为没有共识的情况下，协议会永远循环下去，虽然存在概率，但是要是永远不发生共识，在现实中是不可能发生的事情。

而对应于BA*在共识过程，恶意节点会不断捣乱，阻止诚实节点在第s步达成共识(5 ≤ s ≤ m + 2)，但是每次在s – 2 ≡ 2 mod 3阶段，会有一次机会通过抽签的方式有概率，让诚实用户i对b<sub>i</sub>达成共识。而在最后的m+3阶段，会选择强制出空块的方式结束此轮。

3.  BA

BA*协议就是将出块流程，GC协议和BBA*协议串联在一起，最后完成出块流程。它是一个可靠的(n-t)-拜占庭共识协议，其中n ≥ 3t + 1。当每一轮结束时，都满足：

 一致性(Consisteny)：所有诚实用户i都在同一个v上达成共识，即vi = v

共识性(Agreement)：所有诚实用户i要不都发生共识，要不都不发生共识

4.  定理 (The Theorem)

算法作者通过归纳法证明了如下结论：

在任何轮次r > 0，如下属性必然成立：

 所有的诚实用户共识与同一个块B<sup>r</sup>

出块者为诚实的，则Br包含了交易的最大集，其达成共识的时间上限为T<sup>r</sup>+1 ≤ T<sup>r</sup>+ 8λ + Λ。

出块者为恶意的，其对Br达成共识的时间上限为T<sup>r</sup>+1 ≤ T<sup>r</sup> + (6L<sup>r</sup> + 10)λ + Λ

对于L<sup>r</sup>，p<sub>h</sub> = h<sup>2</sup>(1 + h – h<sup>2</sup>)，出块者为诚实的概率至少为 p<sub>h</sub>

事实上，绝大多数情况下，BA*协议都可以快速达成共识。敌手需要掌握恶意出块者，还要不断调整每一轮投票者的行为，最后需要不错的运气，才能使该轮以最长的时间结束出块。可以算得，其达到共识的期望时间为12.7λ + Λ

 ##结论 (Conclusion)

Algorand基本解决了pow遇到的很多问题。根据论文给出的数据，其交易效率也很高，并且不会因为用户数的增加和区块的增大而增加共识时间，事实上，其最消耗时间的地方是在区块的传播上。








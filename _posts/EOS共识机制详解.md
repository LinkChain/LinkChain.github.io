##传统 DPoS
DPoS（拜占庭容错的委托股权证明），对于PoS机制的加密货币，每个节点都可以创建区块，并按照个人的持股比例获得“利息”。DPoS是由被社区选举的可信帐户来创建区块。为了成为正式受托人，用户要去社区拉票，获得足够多用户的信任。用户根据自己持有的加密货币数量占总量的百分比来投票。DPoS机制类似于股份制公司，普通股民进不了董事会，要投票选举代表（受托人）代他们做决策。
DPOS使用随机的见证人出块顺序，出块速度为 3 秒，21个节点需要14个见证人，2/3 以上的见证人确认的交易，就是不可逆的交易了，所以交易不可逆需要45秒。
![图片1.png](https://upload-images.jianshu.io/upload_images/3959874-1201ef70e9b59f44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
见证人出块顺序，出块速度为 3 秒，每个出块者生产一个区块。 2/3 以上的见证人确认的交易， 21个节点需要14个见证人，需要15*3s=45s，此时交易就不可逆了。
##拜占庭容错（BFT）
借鉴 PBFT（Practical Byzantine Fault Tolerance，拜占庭容错算法）的机制。在传统 DPoS 共识机制中，我们让每个见证人在出块时向全网广播这个区块，但即使其他见证人收到了目前的新区块，也无法对新区块进行确认，需要等待轮到自己出块时，才能通过生产区块来确认之前的区块。
见证人出块时向全网广播，其他见证人收到新区块后，立即对此区块进行验证，并将验证签名完成的区块立即返回出块见证人，不需等待其他见证人自己出块时再确认。
![图片2.png](https://upload-images.jianshu.io/upload_images/3959874-0cc0fe8e073d1386.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
出块见证人生产了一个区块，并全网广播，然后陆续收到了其他见证人对此区块的确认，在收到 2/3 见证人确认的瞬间，区块（包括其中的交易）就不可逆了。交易确认时间大大缩短，从 45 秒缩短至 3 秒左右（主要为等待生产区块的时间）。
##BFT-DPoS共识
Daniel Larimer 在上述基础上又进行了修改。将出块速度由 3 秒 缩短至 0.5 秒，理论上这样可以极大提升系统性能，但带来了网络延迟问题：0.5 秒的确认时间会导致下一个出块者还没有收到上一个出块者的区块，就该生产下一个区块了，那么下一个出块者会忽略上一个区块，导致区块链分叉（相同区块高度有两个区块）。
比如：中国见证人后面可能就是美国见证人，中美网络延迟有时高达 300ms，很有可能到时美国见证人没有收到中国见证人的区块时，就该出块了，那么中国见证人的区块就会被略过。
![图片3.png](https://upload-images.jianshu.io/upload_images/3959874-cb354ad7fe9284fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
为解决这个问题，Daniel Larimer 将原先的随机出块顺序改为由见证人商议后确定的出块顺序，这样网络连接延迟较低的见证人之间就可以相邻出块。每个见证人连续生产 6 个区块，也就是每个见证人还是负责 3 秒的区块生产，但是由最初的只生产 1 个变成生产 6 个。最恶劣的情况下，6 个区块中，最后一个或两个有可能因为网络延迟或其他意外被下一个见证人略过，但 6 个区块中的前几个会有足够的时间传递给下一个见证人。
交易确认时间问题：每个区块生产后立即进行全网广播，区块生产者一边等待 0.5 秒生产下一个区块，同时会接收其他见证人对于上一个区块的确认结果。新区块的生产和旧区块确认的接收同时进行。大部分的情况下，交易会在 1 秒之内确认（不可逆）。这其中包括了 0.5 秒的区块生产，和要求其他见证人确认的时间。
分叉问题：所有节点都不会自动转移到分叉链上，因为分叉链上没有区块生产者可以满足上面所说的15/21法则。即使多数见证人想分叉区块链，也只能以相同的速度（0.5秒）与主链竞争，就算主链只剩下一个见证人，分叉链也永远不会追上主链，保证了系统的稳定。
##BFT-DPOS共识机制总结
21个超级节点（主力见证人节点） + 100个备选见证人节点；
0.5秒出块时间 + 1秒全网确认；
每个主力见证人节点通过协商方式确定各自出块顺序，并且每轮产生6个区块以减少网络延时的影响，见证人间按顺序处理交易，可尽量减少地理影响；
当21个主力见证人的15个确认交易后，交易即不可逆转；
当达到不可逆转状态后，就无法分叉。 
##BFT-DPOS共识机制缺点
1.不是完全去中心化，可能会有多个中心之间共同串通而损害整个社区利益的行为。
2.依赖于投票机制
投票制度其实有以下问题，首先有可能最后投票的参与度会很低，影响投票结果。其次也会可能有这种情况，例如用户把币都存在了交易所，交易所有可能会代替他们去投票，但是用户并不是很在意到底交易所会把票投向何处。也就是说有时候代币持有者的兴趣点和用户的是可能不完全一样的。

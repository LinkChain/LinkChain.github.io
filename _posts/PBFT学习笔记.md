##一、	相关概念
1.	Byzantine算法用于异步分布式系统，可保证在某些节点出错的情况下系统仍能运行。
2.	总节点数n与出错的节点数f满足关系：f <= ⌊(n-1)/3⌋。


##二、算法概要

![image.png](https://upload-images.jianshu.io/upload_images/3959874-c48de8445502009f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1.	客户端发送请求到primary节点，primary由公式p = V mod R决定，p是节点的编号，V是视图号，R是整个系统中节点的个数。请求的消息格式为<REQUEST,o,t,c>，其中o是请求的操作，t是客户端的本地时间，c是客户端；
2.	Primary广播请求消息到每一个backup节点；
3.	所有节点（primary和backups）执行请求，把结果返回到客户端，返回的消息格式为<REPLY,v,t,c,i,r>，v是节点维持的当前的视图号，t是与请求消息中一样的时间，i是节点的编号，r是执行请求的结果；
4.	如果客户端收到f+1个来自不同节点且t相同、r也相同的应答，则接受此应答。如果客户端一直没有收到应答，则客户端向每一个节点都发送此请求消息，如果此请求已经被执行，则直接把结果返回给客户端（每个节点保存上一个发送给客户端的应答）；如果此请求还没执行，那么非primary节点把此请求转发给primary。如果primary一直不广播请求，backups收不到请求，则怀疑primary节点出错，导致view change（详见第四部分）。

##三、算法详细步骤
本算法分为三个阶段：pre-prepare、prepare 和 commit。Pre-prepare和Prepare保证同一个view中的请求有序，Prepare和Committed保证不同view中的请求被有序地执行。

1.pre-prepare阶段
当primary节点收到请求时，则开始pre-prepare阶段：primary对收到的请求标记序号n（n是按序增长的），把pre-prepare消息进行广播，并把此消息记入自己的log中。消息格式为<PRE-PREPARE,v,n,m>，v是primary节点当前的视图号，n是这个请求的序号，m是请求。
Backup节点收到<PRE-PREPARE,v,n,m>消息时进行检查，如果满足：
（1）	v与自己的视图号相同；
（2）	n在[h,H]之间（h，H的计算详见第三章第4节）。
则接受此<PRE-PREPARE,v,n,m>消息。

2.prepare阶段
一个backup（i）节点接受<PRE-PREPARE,v,n,m>消息后，则进入prepare阶段：i广播一个prepare消息到其他所有节点，消息格式为<PREPARE,v,n,d,i>，其中d是m的digest，i把<PRE-PREPARE,v,n,m>消息和<PREPARE,v,n,d,i>消息都记入自己的log中。
如果一个节点收到2f个来自不同节点且与PRE-PREPARE匹配的<PREPARE,v,n,d,i>消息时（匹配的条件是要有相同的v和相同的n），我们把这2f个<PREPARE,v,n,d,i>消息作为prepared的证明。

3、commit阶段
如果一个节点（i）得到了prepared的证明，则广播commit消息到每一节点，并把此commit消息记入log中。消息格式为<COMMIT,v,n,d,i>。如果一个节点收到2f+1个来自不同节点且有相同的n、相同的v、相同的d的<COMMIT,v,n,d,i>消息，我们把这2f+1个commit消息作为committed的证明。

当一个请求被提交到一个节点，这个节点执行请求，并把结果返回给客户端。节点保存的有上一个返回给客户端的应答（last reply），从中可以得到last reply的时间戳，节点不接受时间戳小于last reply的时间戳的请求。

##四、垃圾回收
节点的日志不能无限制地增长，当到达某一时刻，节点要删除一些老旧的log。节点不能执行过一个请求后就把相关的log立刻删除，这样是不安全的。删除log的过程如下：

1.当一个节点收到一个request的序号为n，而n能被一个整数K（例如100）整除时，我们把此时的这个状态叫做checkpoint。当一个节点（i）产生checkpoint时，它广播一个checkpoint消息到其他所有节点，消息格式为<CHECKPOINT,v,n,d,i>，n是上一个产生stable checkpoint（定义见下文）的请求的序号，v是n对应的视图号，d是n对应的请求的digest。       
每个节点都保存有3种state：（1）上一个stable checkpoint；（2）0个或多个不是stable的checkpoint；（3）当前的state。

2.如果一个节点收到f+1个来自不同节点（包括它自己的）的<CHECKPOINT,v,n,d,i>消息且有相同的n、相同的d，那么，这f+1个消息就是checkpoint的证据，我们把一个拥有证据的checkpoint叫做stable checkpoint。

3.当一个节点产生了stable checkpoint时，就把日志中小于n的所有消息都删除，并删除之前的checkpoint消息。	

4.h等于上一个stable checkpoint对应的n的序号，H=h+k，k足够大，使不妨碍节点产生checkpoint。例如，每100个请求产生一个checkpoint，那么h=200。


##五、View change
![image.png](https://upload-images.jianshu.io/upload_images/3959874-0b7a94040c8f7b22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

View change确保即使当前的primary节点出错，整个系统也能继续运行。View change防止backup节点无限地等待请求的执行。
当一个backup节点收到一个请求（此请求由于客户端一直没有收到应答而把请求广播到所有节点）时，启动一个timer；如果此节点不需要等待请求，则timer停止；如果此节点需要执行其他请求，则timer重启。如果timer超时（超市时间根据实际情况指定），则触发view change。view change过程如下：

1.当一个backup节点（i）的timer超时，i停止接收消息（但仍然接收checkpoint、view change和new view（定义见下文）消息），并广播一个view change消息，消息格式为<VIEW-CHANGE,v+1,n,s,C,P,i>，n是节点i中上一个stable checkpoint（s） 对应的序号，C是s的证据，P是i中每一个序号大于n的请求对应的prepared的证据。

2.由于视图号变为v+1，所以产生了一个新的primary节点（原因是p=V mod R）。如果新的primary节点收到视图号为v+1的2f+1个来自不同节点（可能包含自己）的<VIEW-CHANGE,v+1,n,s,C,P,i>消息，我们把这个2f+1个消息作为new view的证据。

3.如果primary得到new view的证据，那么它广播一个new view消息到其他节点。消息格式为<NEW-VIEW,v+1,V,O,N>，V是new view的证据，O与N是某些pre-prepare消息的集合，O与N的计算过程如下：

（1）primary指定h的值为V中上一个stable checkpoint对应的序号，指定H的值为V中的prepared证据中的最大的序号；

（2）primary对每一个序号为n（h<n<=H）的请求创建一个新的且视图号为v+1的pre-prepare消息。此时有两种情况：（一）在V中，存在序号为n的prepared的证据；（二）不存在这样的prepared证据。对于（一），primary创建一个<PRE-PREPARE,v+1,n,m>消息，并把此消息记入O，m是（一）中序号为n的请求；对于（二），primary创建一个<PRE-PREPARE,v+1,n,null>消息，null是“null”请求（此请求不做任何操作）的digest，并把此消息记入N。

Primary把O与N记入log。如果h大于上一个stable checkpoint的序号，那么primary把序号为h的checkpoint的证据记入log，并删除第三章第3节中描述的log。如果h大于primary当前的state，那么primary更新当前的state为h。
此时，primary节点进入view为v+1阶段，并可以接收视图号为v+1的消息。

4.一个backup节点若能接收一个<NEW-VIEW,v+1,V,O,N>消息，需满足以下两个条件：
	
（1）此节点已经含有视图号为v+1的new-view的证据；
（2）此节点验证O与N是正确的，验证方法与primary创建O与N的方法一样。

之后，此节点把O与N记入log。并且对O与N中的每一个消息都创建一个对应的prepare消息，并广播这些prepare消息到每一个节点，并把这些消息记入log，进入v+1阶段。
每一个节点对序号为n，n在h与H之间的每个消息重做上述协议。










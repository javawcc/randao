RANDAO: A DAO working as RNG of Ethereum

[![Join the chat at https://gitter.im/randao/randao](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/randao/randao?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)


###随机数的重要性

在一个确定性系统中生成随机数是一个非常困难的事情

###矿工不可信任!

###解决方案
构建一个人人可以参与的DAO，由所有参与者共同决定一个随机数。

生成一个随机数的基本流程如下：
共分为三个阶段：

#####第一个阶段：收集有效sha3(s)
所有希望参与随机数生成的个体，在指定的时间窗口期内（例如6个出块周期，大约72s），向合约C发送m个ether的保证金，同时附上各自挑选的任意数字的sha3(s)。

#####第二个阶段：收集有效s
之前成功提交sha3(s)的所有个体，在第一阶段结束后，在第二阶段指定的时间窗口期内，向合约C发送各自在第一阶段选中的数字s。合约C检查数字s是否合格的，如果合格，保存该s到最终随机数生成函数的seeds中。

#####第三个阶段：计算随机数，发放保证金及奖励
1. 全部的s收集完成后，以f(s1,s2,...,sn)作为最终的随机数，把随机数写入到C的存储空间中，向所有请求该随机数的其他合约返回结果。
2. 把参与者在第一阶段发送的保证金退还，同时把本期随机数生成过程中，其他合约支付给C的手续费作为奖励发送给本期的所有参与者


为了保证随机数结果不被操纵，兼顾安全和效率，合约C (DAO) 有如下这些额外规则：

1. 第一阶段中，如果先后有两个同样的sha3(s)被提交，只接受第一个。
2. 第一阶段中，有一个最小参与人数的设定，如果在窗口期时间内未能达到预设人数，则本期随机数构造失败。
3. 如果提交的sha3(s)结果被合约C接受，则在第二阶段必须提交s。

  3.1 如果在第二阶段的窗口期中某个参与者没能提交s的话，则在第一阶段中发送的m个ether会被没收，不再返还。
  3.2 如果在第二阶段未能收集到全部的s，则本期随机数生成失败，退还其他合约支付的手续费后，将本期收集的保证金分发给在第二阶段成功发送s的其他参与者。

#### 经济激励
随机数生成的整个频次非常高，以1小时20次循环为例，如果每次收益率达到 0.001%，每个月的收益率可达 `0.00001 * 20 * 24 * 30 = 0.144`。
以14.4%月收益率为目标，如果每次随机数生成平均有n个人参与，消耗成本为 `n * 3 * 500 * gasPrice + Ccost` . (Ccost是合约C内部消耗的gas价值，包括运算以及存储等）
假设每期随机数平均有r个调用，每次调用价格是p ETH, 则收益为`rp`。则当期每人收益为 `(rp - 1500 * n * gasPrice - Ccost) / n`。
当前的gasPrice是10 szabo，估计C会消耗1500n gas，则净收益预估为 `rp/n - 0.03` ETH。
假设每期随机数生成有10人参与，每期保证金为1000ETH,  则在保证收益率大于0.001%的情况下，需要的最小收入是0.4。这样如果随机数只有1次调用，价格为0.4，如果是10次调用，每次价格只需0.04。

这个随机数DAO在Ethereum的系统中是作为一个基础设施存在，供其他Contract调用。不同目的的Contract对随机数的要求不同：有些需要高安全性，例如大乐透的开奖结果；有些需要稳定及时，每次调用都立刻有返回，例如不涉及利益的普通Contract；还有些需要callback, 他希望某期随机数生成后，随机数生成合约能够主动推送信息。

显然不可能通过一个合约满足各种场景下的不同需求，所以会创建很多个不同初始参数的DAO Contract, 但基本规则不变。

比如对于高安全性的需求，会大幅提高第一阶段保证金的数量。这样，试图通过不提交s而导致随机数生成失败的成本大幅提高。而对于低要求的随机数合约，则可以对最小参与人数，保证金放低要求。

我们以一个赌奇偶的应用来说明，如何通过调整不同参数，达到期望的高安全性：即使得违约成本高于预期收益。
假设该赌奇偶的应用的赌本是1000ETH，调用随机数合约C1，如果C1当期随机数生成失败，则等待C1的下一个随机数，一直到有随机数产生为止。
现在我们来构造随机数合约C1，C1要求的保证金是2000ETH。如果参与奇偶赌局的赌徒G同时参与了C1，则当他发现自己处于不利局面的时候，选择不提交s，从而使得随机数生成失败。此时他在C1上损失了2000ETH，仅仅获得了1000ETH的期望收益，是非常不划算的行为。但G可以通过一些手段降低在C1上的损失，比如他同时用2个帐号参与了C1，发送了2个sha3(s), 如果局面不利的话，则只让一个帐号不提交s，如果当期只有1个除G之外的其他参与者参与了C1, 则G的损失只有1000 ETH, 而在奇偶赌局的期望收益是1000ETH, 是一次值得尝试的攻击。

一个应对方案是直接罚没，罚没的金额不作为奖励返回给参与者，这样只需要一个保证金为1000ETH的随机数合约就可以满足该奇偶赌局的要求。

除了罚没之外，另外一个方案通过引入额外的制度来杜绝这种攻击：RANDAO会员。
要成为会员必须先缴纳会费，任何人只要缴纳了会费都可以成为会员。根据缴纳会费多少的不同，会有不同的会员等级。会员不属于某一个合约，只是有资格参与一些随机数合约的身份证明。如果该会员在任何一个合约中违约，则缴纳的会员费会被没收。
现在我们给合约C1增加一个额外的约定，只接受一定等级之上的会员（会员费超过1000ETH）提交的随机数。这样就可以确保任何人都没有动机进行攻击。


QA:

Q: 为什么不让矿工参与随机数生成？为什么不用tx hash, nonce等区块链数据？ 

A: 矿工有能力操纵这些区块链数据，从而间接的影响随机数。引入任何区块链数据，都给了矿工足够的能力构造有利于自己的随机数。

Q: 如果矿工通过拒绝将某些不利于自己的随机数纳入到集合中，该怎么办？

A: 这就是时间窗口期的作用。一个合理的窗口期应该大于6个块，我们相信攻击者没有能力连续制造6个块。所以只要是一个诚实的参与者，只要在每个窗口期开启的时候立即提交数字，完全不需要担心被排除在外。

Q: 为什么要用所有参与者的数字，而不是其中的子集？

A: 选用子集的规则只可能是确定性的，这样参与者可以通过各种办法影响自己提交的数字在整个集合中的位置，从而提前知晓根据某个子集生成出来的随机数是什么。如果选用子集是基于随机数的，不就可以..., 呃，这个选用子集的随机数从哪里得到？

Q: 没收的保证金和会员费去哪了？

A: 可以作为善款捐出去，也可以作为RANDAO的开发维持资金。





附注1：sha3是一个单向函数，一个数字经过sha3操作后的得到的结果，是无法从这个结果反推出数字s
附注2: f(s1,s2,...,sn)是一个多元函数，举例 r = s1 xor s2 xor s3 ... xor sn 。 又或者 sha3(sn+sha3(sn-1+...(sha3(s2+s1))))

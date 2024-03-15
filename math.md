具体做法就是，假设参与者$i$ 的权重是$w_i$，所有参与者的权重总和$W=\sum$$i$ $w_i$,对于具体的抽签目的，我们设置一个期望值$t$,表示在所有的权重当中，希望抽出$t$份权重。这样，我们基于“伯努利试验”的抽签方式，按$p=\frac{t}{w}$的概率，让每个参与者 $i$ 依据自己的权重$w_i$,做$w_i$

 次抽签，这样所有参与者就总共做了$W$次抽签，抽签结果将是$\color{red}{符合二项分布}$的。

 接下来，需要做的就是构造这个“伯努利试验”，这就用到了VRF。首先，要求每个参与者都拥有一对公私钥，$(pk_i,sk_i)$然后，为了满足前面提到的要求，还需要定义一个种子seed，以及标识抽签阶段的一些数据，比如round。其中，需要尽可能让参与者在开始某一轮抽签的时候，才知道所用到的seed，这个根据具体的应用场景来定。

 这时，就可以基于VRF构建我们的“伯努利试验”了。

 设 $\color{red}{x}$ 是由seed、round等组成的抽签参数，则参与者先计算 x 的VRF哈希及证明：$hash,pi = VRF_{sk_i}(x)$这时得到的哈希的长度是固定的，比如32字节，由VRF的安全特性，我们知道hash是在区间 内均匀分布的，将该哈希值变为一个小数，即$ d=\frac{hash}{2^{256}}$，这时 $d$ 就在区间 $0,1$ 之间均匀分布。

 另外，已知以概率 p 做 n 次伯努利试验，实际成功次数为 k 次的概率，计算公式如下：

$$\begin{pmatrix}n\k \end{pmatrix}p^k(1-p)^{n-k}$$

将 k=0...j 所对应的概率加起来，假设用 Sum(j) 表示，我们找到一个j值，使其满足式子$Sum(j)\leq{d}\leq{Sum(j+1)}$这时$j$就是参与者的抽签结果了。$j=0$时表示没抽中，$j>0$时表示抽中了$j$份权重。

由于用户的权重越大，即w越大，其抽签次数就越多，从而基于相同的概率p，得到 $j>0$ 的概率也就会越大，因此，用户被选中的概率是跟他们的权重是成比例的。另外，当 $j>0$，我们可以想象成用户有 $j$ 个子用户被抽中了，如果是投票，就可以投$j$票。这样一来，w个权重为1的用户，有j个被抽中的概率，跟1个权重为w的用户，有j个子用户被抽中的概率，是一样的。也就是说，这种抽签是防女巫攻击的。
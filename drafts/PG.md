# 深入浅出策略梯度(Policy Gradient)

强化学习(Reinforcement Learning)是一类用于复杂场景的学习算法，被广泛应用在机器控制任务。近几年来，随着神经网络的重新兴起，强化学习也被逐渐应用在一些新的领域，比如自动驾驶，计算机视觉等。Alpha GO 战胜人类棋手标志着机器学习特别是强化学习正在成为逐渐成熟并能够处理更加复杂的问题，成为研究的热点，也被认为能够在未来在人工智能领域取得突破的方向之一。本文旨在介绍强化学习中梯度策略（PG）的基本原理，相关概念，并着重介绍作者在学习PG过程中遇到的一些难点如理解目标函数和实现技术。


# 相关概念

## 强化学习
Alpha GO战胜人类让大众惊叹于人工智能的突飞猛进的同时，也让人不禁好奇机器学习到底是如何实现对人类的超越，一方面监督式学习(supervised learning)的效果受限于训练数据， 另一方面采用简单穷举的方式会遇到硬件的限制（围棋的状态组合空间$10^{360}$达到了宇宙粒子的数量级）。其实Alpha GO的一个秘密武器就是今天的主角-强化学习，强化学习是机器学习中的一类方法。~~简单的说，这类方法通过和环境的交互获得反馈，再根据反馈进行优化。~~强化学习诞生于上世纪80年代，开始应用于制造业，特别是工业机器人的自动控制，近年来随着其他机器方法的流行开始应用于更加“智能”的场景，除了大名鼎鼎的Alpha GO，google deepmind团队还应用强化学习实现了计算机自主学习玩Atari系列电子游戏并超越了人类万家的水平，**。越来越多成功的应用表明强化学习 ---
### Markov Decision Process (MDP)
不同于监督式学习通过标签数据进行学习，强化学习通过和环境交互来学习。环境一般被形式化的描述为MDP，由以下几部分组成：
-   **$S$** : 状态集，Set of states. At each time step the state of the environment is an element  $s \in  S$.
-   **$A$**: 动作集 Set of actions. At each time step the agent choses an action  $a \in  A$ to perform.
-   **$p(s_{t+1} | s_t, a_t)$** : State transition model that describes how the environment state changes when the user performs an action  `a`depending on the action  `a`and the current state st.
-   **$p(r_{t+1} | s_t, a_t)$** : Reward model that describes the real-valued reward value that the agent recieves from the environment after performing an action. In MDP the the reward value depends on the current state and the action performed.
-  **𝛾** : 折扣系数，用于调整未来对当前的影响

强化学习的过程是一个通过和环境交互获得反馈，再根据返回调整以期使总奖励最大化的过程，这个是一个多步的交互的过程，每一步交互都会影响其后的所有步骤。强化学习中的一次交互是指Agent 对环境施加一个动作，这会导致导致环境的状态发生改变并且由环境回馈给Agent一个奖励（奖励既可以是正向的也可以是负向的），强化学习的目标就是寻找一个最优的策略使得整个学习过程获得的奖励最大化。


The way by which the agent choses which action to perform is named the agent  `policy`  which is a function that takes the current environment state to return an action. The policy is often denoted by the symbol 𝛑.
![](https://media.shellypalmer.com/wp-content/images/2016/03/alphago.jpg)
![](https://atariage.com/2600/hacks/screenshots/s_SpaceInvaders_RK_Hack_2.png)
强化学习它包括如下图所示的几个部分：
![](https://cdn-images-1.medium.com/max/1600/1*c3pEt4pFk0Mx684DDVsW-w.png)
1.  主体(Agent) 指能够通过动作与环境交互的**，在RL的环境中主体通常是运行中的算法，比如在Atari游戏中的主体是用于控制飞船的算法
2.  环境(Environment) 指主体动作作用的对象， 比如Atari游戏本身。
3.  动作 (Action): 指所有可能的作用于环境上的操作，比如Atari游戏中算法控制飞船进行移动或射击。
4.  状态 (State): 指可被主体感知的关于环境的信息，比如Atari游戏中屏幕显示的所有物体的位置以及移动方向和速度信息
5.  奖励 (Reward): 指由环境回馈给主体的描述上一个动作效果的信息，比如Atari游戏中飞船动作导致的得分变化。


同监督式学习类似，强化学习也是一个通过多个轮次逐渐优化算法参数来增强学习效果的过程，每个轮次包含两部分：前向传递和反向传递。处于初始状态的Agent根据算法的当前参数生成动作作用于环境，环境返回给Agent新的状态和对动作的奖励，在轮次结束后算法通过汇总所有在本轮收集到的反馈调整算法的参数开始下一轮的学习，直到学习的效果不再增长。常见的强化学习有两类：基于值的方法和基于策略的方法。
## 基于值的方法

基于值的方法的基本思想是求一个函数Q满足bellman方程，使得在状态$s$下使用动作$a$可能得到最大的奖励
$$
Q(s,a) = r + \gamma max_{a'} Q(s', a')
$$
其中，$s'$表示在当前状态$s$下使用动作$a$得到的下一个状态，$\gamma$表示对未来事件的折扣率，bellman方程描述了一个递归的计算方法，即$s$状态在使用$a$动作得到的总奖励等于当前的直接奖励$r$和下一步动作$a'$产生的总奖励的和。从这个方程可知，为了计算$s$状态下的最大奖励，需要求处其后所有状态的总奖励，~~因此由于需要保存所有状态的奖励，这种方法要求问题的状态空间不能太大。~~
### 基于策略的方法

robot control, Boston Dynamics




# 策略梯度（PG）
## PG的基本原理
DQN可以直接计算奖励从而可以得到最优解，这听上去很不错，但这种确定性(deterministic)的方法恰恰无法处理一些博弈问题，比如玩100把石头剪刀布的游戏，最好的解法是随机的使用石头、剪刀和布并尽量保证这三种手势出现的概率一样，因为任何一种手势的概率高于其他手势都会被对手注意到并使用相应的手势赢得游戏。 再比如下图这个例子：


采用DQN的方法在确定的状态下将得到确定的reward，因此在使用DQN方法决定灰色方格的下一步（左或右）将必然会导致bad result。事实上很多实际的都有类似的特征，即

~~由此看来，我们需要一种非确定性的方法来~~
另外，有限的状态数量也是使用DQN的一个大问题，因为基于值的方法需要保存状态表（所有状态-动作的对应关系），因此很多现实问题（例如机器人控制和自动驾驶都是连续动作空间），都因为巨量的状态而无法计算。
那么如何解决上面的两个问题呢？有没有一种方法能在确定的状态下得到不同的动作呢？又有什么方法能避免维护庞大的状态表呢？
简单来说，PG则采取了随机（stochastic）的方式解决了上述2个问题。首先随机能提供非确定的结果，但这种非确定的结果并不是完全的随机而是服从某种概率分布的随机，PG不计算reward而是以直接使用策略选择action，这样就避免了因为计算奖励而维护状态表。 
那么PG的学习到底是怎样的呢？在解释这个过程之前先介绍几个概念：
**对象系统**：就是PG的学习对象，这个对象即可以是一个系统，比如汽车或一个游戏，也可以是一个对手，比如势头剪刀布的游戏对手或者一个职业的围棋手。
~~MDP:Reward function~~
**Policy策略** $\pi_\theta(a|s)$ 表示在状态$s$和参数$\theta$条件下发生动作$a$的概率
**Episode轮次**: 表示从起始状态开始使用某种策略产生动作与对象系统交互，直到某个终结状态结束。比如在围棋游戏中的一个轮次就是从棋盘中的第一个落子开始直到对弈分出胜负，或者自动驾驶的轮次指从汽车启动一直到顺利抵达指定的目的地，当然撞车或者开进水塘也是种不理想的终结状态。
**Trajectory轨迹** $\tau$ 表示在PG一个轮次的学习中状态$s$，动作$a$和奖励$r$的顺序排列
$$
\tau = (s_0, a_0, r_0, s_1, a_1, r_1, ... , s_t, a_t, r_t)
$$
由于策略产生的是非确定的动作，同一个策略在多个轮次可以产生多个不同的轨迹。~~因此在实现中对每个策略会求多个轮次的平均值~~

>如果Agent的某个行为策略导致环境正的奖赏(强化信号)，那么Agent以后产生这个行为策略的趋势便会加强。Agent的目标是在每个离散状态发现最优策略以使期望的折扣奖赏和最大。
强化学习把学习看作试探评价过程，Agent选择一个动作用于环境，环境接受该动作后状态发生变化，同时产生一个强化信号(奖或惩)反馈给Agent，Agent根据强化信号和环境当前状态再选择下一个动作，选择的原则是使受到正强化(奖)的概率增大。选择的动作不仅影响立即强化值，而且影响环境下一时刻的状态及最终的强化值。
强化学习不同于连接主义学习中的监督学习，主要表现在教师信号上，强化学习中由环境提供的强化信号是Agent对所产生动作的好坏作一种评价(通常为标量信号)，而不是告诉Agent如何去产生正确的动作。由于外部环境提供了很少的信息，Agent必须靠自身的经历进行学习。通过这种方式，Agent在行动一一评价的环境中获得知识，改进行动方案以适应环境。
强化学习系统学习的目标是动态地调整参数，以达到强化信号最大。若已知r/A梯度信息，则可直接可以使用监督学习算法。因为强化信号r与Agent产生的动作A没有明确的函数形式描述，所以梯度信息r/A无法得到。因此，在[强化学习](https://baike.baidu.com/item/%E5%BC%BA%E5%8C%96%E5%AD%A6%E4%B9%A0)系统中，需要某种随机单元，使用这种随机单元，Agent在可能动作空间中进行搜索并发现正确的动作。


PG的学习是一个策略的优化过程，最开始随机的生成一个策略，当然这个策略对对象系统一无所知，所以用这个策略产生的动作会从对象系统那里很可能会得到一个负面奖励，这个过程就好像我们的自动驾驶策略在面对笔直的路面而产生右转的动作导致汽车撞上路边的行人这样的严重后果。为了更好的驾驶汽车PG需要不断的改变策略从而获得更高的轮次奖励（安全快速的到达目的地），PG在一轮的学习中使用同一个策略直到该轮结束，通过梯度上升改变策略并开始下一轮学习，如此往复直到轮次累计奖励不再增长停止。   ~~一个使得action的选择服从一定的概率分布，通过使用这个策略完成所有交互，这就把一个复杂的实际问题转化成了概率优化问题~~
![enter image description here](https://github.com/eric2323223/ML/blob/dev/drafts/PG1.PNG?raw=true)
## PG的目标函数
根据上述PG的基本原理，我们可以把PG的目标形式化的描述为以下表达式

$$
J(\theta) = argmax_\theta \mathbb E[r_0+r_1+r_2+...+r_t|\pi_\theta]  
$$
其中$\mathbb E[r_0+r_1+r_2+...+r_t|\pi_\theta]$表示在策略$\pi_\theta$条件下一轮交互（$0$到$t$步）中的累计奖励的期望值，这里是期望而不是确定值是因为每一步的奖励是根据策略得到的期望值而不是确定值。由于$r_i$可以由$r(\tau_{i-1})$计算得到，因此可以把累计奖励的期望值写成如下：

$$
J(\theta) = \mathbb E_{\tau\sim \pi_\theta(\tau)}[\sum_t r(\tau)] \approx \frac {1}{N}\sum_i\sum_t r(s_{i,t}, a_{i,t})
$$
我们把单个轮次的累计奖励作为PG的目标函数$J(\theta)$，则PG的目标就是确定构成策略的参数$\theta$使得$J(\theta)$取得最大值

$$
\theta ^* = argmax J(\theta)
$$
图片Intuition trajectories.

现在PG的学习就变成了一个对$J(\theta)$求最大值的问题，和监督式学习中使用的梯度下降(gradient descent)求损失函数(loss function)的最小值类似，PG中使用梯度上升(gradient ascent)来更新$\theta$。
根据期望值的数学定义，

$$
J(\theta) = \mathbb E_{r\sim \pi_\theta(\tau)} [\sum r_\tau]= \int_\tau r(\tau)\pi_\theta(\tau) d\tau
$$
对这个积分表达式求导数，
$$
\nabla_\theta J(\theta) =  \nabla_\theta \int_\tau r(\tau)\pi_\theta(\tau) d\tau= \int_\tau r(\tau)\nabla_\theta \pi_\theta(\tau) d\tau
$$
由于$\pi_\theta(\tau)$本身依赖于$\theta$，我们无法直接求导，因此要使用一个小技巧，根据$\nabla log f(x)=\frac {\nabla f(x)}{f(x)}$可得下式
$$
\nabla_\theta\pi_\theta(\tau) = \pi_\theta(\tau)\frac{\nabla_\theta\pi_\theta(\tau)}{\pi_\theta(\tau)}=\pi_\theta(\tau)\nabla_\theta log\pi_\theta(\tau)
$$
因此
$$
\nabla_\theta J(\theta) =\int r(\tau)\nabla_\theta\pi_\theta(\tau)d\tau = \int r(\tau)\pi_\theta(\tau)\nabla_\theta log\pi_\theta(\tau)d\tau
$$
再根据期望值的定义，
$$
\nabla_\theta J(\theta) =\int r(\tau)\pi_\theta(\tau)\nabla_\theta log\pi_\theta(\tau)d\tau=\mathbb E_{\tau\sim\pi_\theta(\tau)}[\nabla_\theta log\pi_\theta(\tau)r(\tau)]
$$
由于，
$$
log\pi_\theta(\tau) = logp(s_1) + \sum_{t=1}^Tlog\pi_\theta(a_t|s_t)+logp(s_{t+1}|s_t, a_t)
$$
$$
r(\tau)=\sum_{t=1}^Tr(s_t,a_t)
$$
可得
$$
\nabla_\theta J(\theta) =\mathbb E_{\tau\sim\pi_\theta(\tau)}[(\sum_{t=1}^Tlog\pi_\theta(a_t|s_t))(\sum_{t=1}^Tr(s_t,a_t))]
$$
还需要指出的是上述表达式描述了当前策略$\pi_\theta$通过一轮获得的导数，前面我们已经提到过由于策略产生的是非确定的动作，因此相同策略在多轮次中会产生不同的轨迹，为了避免个体的偏差，我们需要多次取样并去均值来提高准确性，所以，
$$
\nabla_\theta J(\theta) \approx \frac{1}{N}\sum_{i=1}^N[(\sum_{t=1}^Tlog\pi_\theta(a_{i,t}|s_{i,t}))(\sum_{t=1}^Tr(s_{i,t},a_{i,t}))]
$$
至此，我们就得到了可计算的目标函数的导数$\nabla J(\theta)$，在轮次的反向传递(back propagation)中使用学习率$\alpha$与$\nabla J(\theta)$的乘积作为差值$\delta$更新$\theta$
$$
\theta = \theta + \alpha \nabla J(\theta)
$$
以上为了推导用于反向传递的可计算的$\nabla J(\theta)$列出了很多表达式，目的是帮助读者理解PG算法实现，因为在代码实现中会直接使用~~表达式x~~计算$\nabla J(\theta)$，如果直接看代码而不了解$\nabla J(\theta)$的变形的话恐怕会觉得费解。不过从$\nabla_\theta J(\theta)$和$\pi_\theta(\tau)$$r(\tau)$的基本关系还是能够作出这样的直观解释：如果奖励($r(\tau)$)比较高时，策略($\pi_\theta(\tau)$)会倾向于增加相应的动作的概率，如果奖励比较低时，策略会倾向于降低相应动作的概率。从机器学习的原理的角度来看，PG和传统的监督式学习的学习过程还是比较相似的，每轮次都由前向传递和反向传递构成，前向传递负责计算目标函数，反向传递负责更新算法的参数，依此进行多轮次的学习指导学习效果稳定收敛。唯一不同的是，监督式学习的目标函数相对直接，即目标值和真实值的差，这个值一次前向传递就能得到；而PG的目标函数源自轮次内所有得到的奖励，并且需要进行一定的数学转换才能计算，另外由于用取样模拟期望，也需要对同一套参数进行多次前向传递来增加模拟的准确性。
figure: intuition

从实现角度看，PG的学习过程可以分为三个阶段
1. **取样**	对当前策略取多个轨迹用以准确计算目标函数，取样的过程就是用当前策略进行多次前向传递并保存轨迹
2. **计算$\nabla J(\theta)$**
3. **改进策略**	使用2计算出的$\nabla J(\theta)$更新$\theta$

伪实现
>function REINFORCE()
Initialize $\theta$ arbitrarily
for all episodes $(s_1, a_1, r_2, ... , s_{T-1}, a_{T-1})\sim \pi_\theta$ do
for t = 1 to T - 1 do
$\theta = \theta + \alpha\nabla log\pi_\theta(a_t, s_t)r(a_t, s_t)$
end for
end for
return $\theta$
end function

## PG的改进

虽然PG理论上能处理基于值的方法无法处理的复杂问题，但由于PG依赖样本来优化策略，导致这种方法受样本个体差异影响有比较大的方差，学习的效果不容易持续增强和收敛。一个基本的改进思路是通过减少无效的元素来降低方差，由于当前的动作不会对过去的奖励产生影响，因此可以将$\nabla_\theta J(\theta)$改写为
$$
\nabla_\theta J(\theta) \approx\frac{1}{N}\sum_{i=1}^N[(\sum_{t=1}^Tlog\pi_\theta(a_{i,t}|s_{i,t}))(\sum_{t'=t}^Tr(a_{i,t'}, s_{i, t'}))]
$$
我们还可以借鉴MDP折扣系数的思想降低未来的影响
$$
\nabla_\theta J(\theta) \approx\frac{1}{N}\sum_{i=1}^N[(\sum_{t=1}^Tlog\pi_\theta(a_{i,t}|s_{i,t}))(\sum_{t'=t}^T\gamma^{t'-t} r(a_{i,t'}, s_{i, t'}))]
$$

另外一个思路是通过引入基准(baseline)$b$减小方差，这是因为实际计算中产生的总奖励并不能准确代表这个策略的好坏程度，比如当前已经得到了一个较好的策略，而在下一轮的学习中用一个不太好的样本也能得到一个正向的总奖励$\sum r$，PG算法依然会由于这个正向的总奖励调整原来的策略而导致学习效果下降。如果有一个基准值$b$使得PG不会向低于基准值的方向移动（$\sum r - b >0$），就可以有效的避免这种问题。
我们还可以证明基准值的引入不会对目标函数造成影响，即$\frac{1}{N}\sum_{i=1}^N[(\sum_{t=1}^Tlog\pi_\theta(a_{i,t}|s_{i,t}))(\sum_{t'=t}^Tr(a_{i,t'}, s_{i, t'})-b)]$是对$\nabla_\theta J(\theta)$的无偏估计(unbiased estimator)， 因为
$$
\mathbb E[\nabla_\theta log\pi(\tau)b]=\int \pi_\theta(\tau)\nabla_\ log _\theta(\tau)bd\tau=\int \pi_\theta(\tau)\nabla_\theta(\tau)bd\tau = b\nabla_\theta\int \pi_\theta(\tau)d\tau
$$
由于$\pi()$是概率密度函数$\int \pi_\theta(\tau)d\tau=1$，因此
$$
\mathbb E[\nabla_\theta log\pi(\tau)b] = b\nabla_\theta1=0
$$
所以
$$
\nabla_\theta J(\theta) \approx\frac{1}{N}\sum_{i=1}^N[(\sum_{t=1}^Tlog\pi_\theta(a_{i,t}|s_{i,t}))(\sum_{t'=t}^Tr(a_{i,t'}, s_{i, t'})-b)]
$$
常见的基准值是均值
$$
b=\frac{1}{N}\sum_{i=1}^N r(a_i, s_i)
$$
目前也有对基准值进行改进的方法，比如Actor-Critic算法，其基本思想是使用DQN进行基准值的优化，由于篇幅所限制在这里不展开介绍。

# 总结

个人体会PG可以总结一个关键词：抽样，通过抽样模拟目标函数，避免了遍历，由于抽样导致较大的方查

----------





# Markdown extensions

StackEdit extends the standard Markdown syntax by adding extra **Markdown extensions**, providing you with some nice features.

> **ProTip:** You can disable ny **Markdown extension** in the **File properties** dialog.


## SmartyPants

SmartyPants converts ASCII punctuation characters into "smart" typographic punctuation HTML entities. For example:

|                |ASCII                          |HTML                         |
|----------------|-------------------------------|-----------------------------|
|Single backticks|`'Isn't this fun?'`            |'Isn't this fun?'            |
|Quotes          |`"Isn't this fun?"`            |"Isn't this fun?"            |
|Dashes          |`-- is en-dash, --- is em-dash`|-- is en-dash, --- is em-dash|


## KaTeX

You can render LaTeX mathematical expressions using [KaTeX](https://khan.github.io/KaTeX/):

The *Gamma function* satisfying $\Gamma(n) = (n-1)!\quad\forall n\in\mathbb N$ is via the Euler integral

$$
\Gamma(z) = \int_0^\infty t^{z-1}e^{-t}dt
$$

> You can find more information about **LaTeX** mathematical expressions [here](http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference).


## UML diagrams

You can render UML diagrams using [Mermaid](https://mermaidjs.github.io/). For example, this will produce a sequence diagram:

```mermaid
sequenceDiagram
Alice ->> Bob: Hello Bob, how are you?
Bob-->>John: How about you John?
Bob--x Alice: I am good thanks!
Bob-x John: I am good thanks!
Note right of John: Bob thinks a long<br/>long time, so long<br/>that the text does<br/>not fit on a row.

Bob-->Alice: Checking with John...
Alice->John: Yes... John, how are you?
```

And this will produce a flow chart:

```mermaid
graph LR
A[Square Rect] -- Link text --> B((Circle))
A --> C(Round Rect)
B --> D{Rhombus}
C --> D
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU5Njk3NTQyOSwxNjU3NjIxNzM5XX0=
-->
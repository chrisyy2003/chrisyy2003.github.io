---
layout: post
title: uniswap-v3
date: 2022-12-30 09:20:49
hide: true
categories:
    - blockchain
---

公式：https://atiselsts.github.io/pdfs/uniswap-v3-liquidity-math.pdf

首选阅读官方白皮书：https://uniswap.org/whitepaper-v3.pdf

中文白皮书：

https://www.jinse.com/news/blockchain/1057182.html

原理解析：

https://learnblockchain.cn/article/3129

https://learnblockchain.cn/article/2357

https://learnblockchain.cn/article/3112

https://learnblockchain.cn/article/1494

https://zhuanlan.zhihu.com/p/448382469

源码解析：

https://learnblockchain.cn/article/2371

做市方案：

https://zhuanlan.zhihu.com/p/390751130

视频教程：

https://www.youtube.com/watch?v=7dD405ncoGg&list=PLO5VPQH6OWdX-Rh7RonjZhOd9pb9zOnHW&index=56



https://www.bilibili.com/video/BV1Cv411V71o/?spm_id_from=333.337.search-card.all.click&vd_source=7087823edd54a070fb303adf701d23cb







# v2 to v3

在Uni v2中，流动性沿着xy=k的不变曲线均匀分散，

这些不同的代币对，对于流动性池的配置有不同的要求。最主要的区别在于，锚定代币对需要非常高的流动性来降低大额交易对其价格的影响。USDC与USDT的价格必须保持在1附近，无论我要买卖多大数目的代币。由于 Uniswap V2 的通用 AMM 算法对于稳定币交易并没有很好的适配，所以在稳定币的交易中其他的 AMM（主要是[Curve](https://curve.fi/)）更加流行。

但由于大多数交易活动都是在某个价格段内发生的，xy=k曲线的其他部分的流动性资金没有得到利用，即资本效率低下。

## 资金利用率

在协议设计上最核心的改变就是引入了**集中流动性**，从而解决了v2版本中资金利用率不高的问题。

集中流动性的定义：流动性提供者可以将其提供的流动性”限制”在**任意的价格区间**内来提供流动性。

这个机制通过将更多的流动性提供在一个相对狭窄的价格区间，来大大提高资产利用效率。

简单地来说，一个 Uniswap V3 的交易对由许多个 Uniswap V2 的交易对构成。V2 与 V3 的区别是，在 V3 中，一个交易对有许多的**价格区间**，而每个价格区间内都有**一定数量的资产**。从零到正无穷的整个价格区间被划分成了许多个小的价格区间，每一个区间中都有一定数量的流动性。而更关键的点在于，在每个小的价格区间中，**工作机制与 Uniswap V2** 完全一样。这也是为什么我们说一个 Uniswap V3 的池子就是许多个 V2 的池子。

## 集中流动性

我们通过图像来演示这一问题，这是一个v2的交易曲线，在没有手续费的情况下，始终符合曲线xy=k的，图中过原点的直线与曲线的交点的**斜率**其含义是曲线上那一点的**价格**，那一点和原点围成的面积也就是当前所拥有的流动性

图2

可以发现，我们的点实际上可以遍历整个0，正无穷的区间，但是实际上两种的代币的价格只会在某一个范围上浮动，例如eth/usdt

并且在交换过程中，只需要图中的x，y的数量即可，但是实际上我们却提供了整个x，和y的流动性，所以可以算得我们实际被用到的资金利用率为x‘/x，

这里为了便于查看，夸大了价格的变动区间，**实际交易过程中，普通用户的交易造成价格的变动是很小的，**，所以橙色区域是极小的。

所以我们需要将这一部分没有用的流动性移除，替换为虚拟流动性，对于x代币，实际的数量是x + x real，y代币，数量则为y + y _real，此时包含的范围则是x v 右边和 y v 上边部分的曲线，同样在这个区间内是符合v2曲线，这与v2中的恒定乘积公式是相同的，所以两个数量相乘同样是k
$$
\left(x_{\text {real }}+x_{\text {virtual }}\right) \times\left(y_{\text {real }}+y_{\text {virtual }}\right)=k
$$
那么如上图所示，用户在这个区间内，用户实际的流动性提供为$x_{real}$和$y_{real}$，此外很显然此公式只限于$[a, b]$这个区间中间，当价格滑出$[a, b]$这个区间外时，流动性完全由一项资产组成，因为另一项资产的储备必须已经完全耗尽，即全部为$x_{real}$或者$y_{real}$，并且此区间的流动性将会冻结为当前状态，价格在区间外时，此区间不参与手续费计算。但如果价格再次进入该范围时，流动性就会**再次激活**。

价格区间$[P_a,P_b]$之间提供流动性，在这个区间里只需要保证有足够的资产X来覆盖价格变动到上限$P_b$，有足够的资产Y来覆盖价格变动到下限$P_a$即可，例如在价格$P_c$时*，* *，*那么在$[a,b]$区间所有的点都满足有如下方程式：



公式1中的virtual的值具体是多少，我们还继续推导一下这个公式

![image-20221230092913778](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221230092913778.png)

在流动性不改变的情况下，由于p_a和p_b是固定的，所以x_virtual和y_virtual也是固定值，

根据上面推导出来的方程，虚拟储备金和实际储备金曲线，带入到曲线方程中进行一些推导来获得新的曲线方程

实际上函数中只是加上了一些常数，我们甚至可以利用初中的数学函数平移规律，左加右减上加下减来理解新的函数图像





![image-20221230092955208](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221230092955208.png)

在v3中每一个用户都可以设置自己认为合理的价格区间，这个区间在v3中被称为头寸 position，每个头寸都有自己的价格范围。通过这种方式，有限合伙人可以在价格空间上近似任何所需的流动性分布（参见图 3 中的几个示例）。此外，这是一种让市场决定流动性应该分配到哪里的机制。 Rational LP 可以通过将其流动性集中在当前价格附近的窄带内，并随着价格变动而添加或移除代币以保持其流动性活跃，从而降低其资本成本。

那么多个用户添加自己的头寸之后，函数图像应该显示是这样，每一个正比例函数之间则是一个价格区间，区间内有用户提供的不同的流动性大小

![image-20230111002907079](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20230111002907079.png)

既然公式被替换了由l和p表示的公式，那么不如我们换一个视角，换为一个由l和p表示的坐标轴，

我们需要值在一笔交易没有手续费的情况下，交易池的流动性不会变化，变化只有池子中的价格，所以

图一张表示了V2的流动性是“普适”的，在所有价格点上流动性相同。

图二表示了有一个价格区间的情况，在ab之间存在流动性，但是在ab之外是没有的

图三则表示了V3的流动性是由一系列不同区间上的流动性组成。是当很多个流动性叠加后，池内流动性的分布，也就是说因为每笔注入的流动性价格区间不同，他们会有重叠的区域，于是在重叠的区域上，流动性是会叠加的，也就形成了右侧这种不同价格区间内，堆叠高度不同的柱状分布。

![image-20230111002334698](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20230111002334698.png)

所以在现在v3的一个代币界面，就和图三是类似的

# v3核心公式







# 添加流动性


$$
\begin{aligned}
& \Delta Y= \begin{cases}0 & i_c<i_l \\
\Delta L \cdot\left(\sqrt{P}-\sqrt{p\left(i_l\right)}\right) & i_l \leq i_c<i_u \\
\Delta L \cdot\left(\sqrt{p\left(i_u\right)}-\sqrt{p\left(i_l\right)}\right) & i_c \geq i_u\end{cases} \\
& \Delta X= \begin{cases}\Delta L \cdot\left(\frac{1}{\sqrt{p\left(i_l\right)}}-\frac{1}{\sqrt{p\left(i_u\right)}}\right) & i_c<i_l \\
\Delta L \cdot\left(\frac{1}{\sqrt{P}}-\frac{1}{\sqrt{p\left(i_u\right)}}\right) & i_l \leq i_c<i_u \\
0 & i_c \geq i_u\end{cases} \\
&
\end{aligned}
$$


# 交易

现在我们已经有了流动性，我们可以进行我们的第一笔交易了！

## 区间内交换

当在同一区间内进行交易时，V3 的表现和 V2 一致，在决定了我们希望投入的资金量之后，我们需要计算我们会获得多少 token。

**交易会导致价格变动**（回忆一下，交易使得现价沿着曲线移动），通过token数量变化能够计算出最终的目标价格(target price)，再通过目标价格就可以计算出投入 token 的数量和输出 token 的数量。

token_in —— price_next ——token_out



`SwapState` 维护了当前 swap 的状态。

-   `amoutSpecifiedRemaining` 跟踪了还需要从池子中获取的 token 数量：当这个数量为 0 时，这笔订单就被填满了。
-   `amountCalculated` 是由合约计算出的输出数量。
-   `sqrtPriceX96` **是交易结束后的价格**
-   `tick` 是交易结束后的价格 `tick`

`StepState` 维护了当前交易“一步”的状态。这个结构体跟踪“填满订单”过程中**一个循环**的状态。

-   `sqrtPriceStartX96` **跟踪循环开始时的价格**。
-   `nextTick` 是能够为交易提供流动性的下一个已初始化的`tick`，
-   `sqrtPriceNextX96` **是下一个 `tick` 的价格**。
-   `amountIn` 和 `amountOut` 是当前循环中流动性能够提供的数量。



这是整个swap的核心逻辑所在。这个函数计算了一个价格区间内部的交易数量以及对应的流动性。它的返回值是：新的现价、输入 token 数量、输出 token 数量。尽管输入 token 数量是由用户提供的，我们仍然需要进行计算在对于 `computeSwapStep` 的一次调用中可以处理多少用户提供的 token。

### swapMath

接下来，让我们更深入研究一下 `SwapMath.computeSwapStep`：

这个函数计算了一个价格区间内部的交易数量以及对应的流动性。

它的返回值是：新的现价、输入 token 数量、输出 token 数量。



## 多区间交换

在这个milestone中，我们会：

1.  更新 `mint` 函数，使得能够在不同的价格区间提供流动性；
2.  更新 `swap` 函数，使得在当前价格区间流动性不足时能够跨价格区间交易；
3.  学习如何在智能合约中计算流动性；
4.  在 `mint` 和 `swap` 函数中实现滑点控制；
5.  更新前端用户界面，使得能够在不同价格区间添加流动性；
6.  增加对于定点数运算的一些了解。
7.  

为了改进上面函数，我们需要考虑以下几个场景：

1.  当现价和下一个 tick 之间的流动性足够填满 `amoutRemaining`；
2.  当这个区间不能填满 `amoutRemaining`。

在第一种情况中，交易会在当前区间中全部完成——这是我们已经实现的部分。在第二个场景中，我们会消耗掉当前区间所有流动性，并且**移动到下一个区间**（如果存在的话）。考虑到这点，我们来重新实现 `computeSwapStep`：

# 费率

https://foresightnews.pro/article/detail/13985

什么时候计算

交易费用仅仅当一个价格区间在使用中的时候才会被收集到这个区间中。因此，我们需要跟踪穿过价格区间边界的时间。我们希望在下列的时间开始对一个价格区间的费率收集：

1.  当价格上升，tick 穿过价格区间的下界；
2.  当价格下降，tick 穿过价格区间的上界；

而在下列的时刻一个价格区间被停用：

1.  当价格上升，tick 穿过价格区间的上界；
2.  当价格下降，tick 穿过价格区间的下界；

怎么计算

一句话概括的话就是，V3的费用计算是通过跟踪**一个单位的流动性产生的总费用**，通过总费用减去价格区间之外累计的费用，来计算一个价格区间的费率的。

这里有单位流动性、总费用、价格区间外好几个概念，不过不用担心，以下会依次解答。



## 区间外费率

而在一个价格区间之外累积的费用是当一个 tick 被穿过时追踪的（当交易移动价格时，tick 被穿过；费用在交易中累计）。用这种方法，我们不需要去在每一笔交易中更新每一个位置累计的费用——这会节省大量的 gas，





# 预言机

# 源码解析

## 参考链接

https://hackmd.io/@adshao

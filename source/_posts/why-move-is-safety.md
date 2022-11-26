---
layout: post
title: Move语言为什么是一个安全的智能合约语言
date: 2022-11-15 16:08:24
categories:
    - blockchain
---

> 在Move开发者大会上徐萌教授主讲了「用 Move Prover对智能合约进行快速、可靠的形式化验证」主题学术报告，从用户视角解读 Move 语言及 Move Prover，内容深入浅出。
>
> 本文仅是对Movebit文章进行了精简和整理，有兴趣请参考Movebit原文：https://zhuanlan.zhihu.com/p/574171956

# 概述

区块链是一个很重要的系统，大量资产存放在了区块链上，而且区块链上的这些交易，一旦执行了它是不可以撤销的。所以**智能合约就需要有一些规范来规定它是否真的能提供了它所宣传提供的功能，而不是一些意料之外的功能，这就需要我们有一个语言和一套工具来保证这一点。** 在这种背景下， Move 语言以及 Move Prover这个工具就应运而生。

Move 是一个面向区块链设计的编程语言，它提供了更好的安全性。简单来说 Move 语言的安全性能是由两部分组成的，一是Move的类型系统，二是Move Prover形式化验证的工具。

并且这两个系统分别提供了不同程度的安全保护，Move中的类型系统提供了：

- 类型安全

- 资源安全

- 引用安全。

形式化验证工具 Move Prover，它主要提供了更多更高级的表述方法，就是更高级的去表达你想做什么的规范。其中:

- 结构体不变量struct invariants：可以表达一个类型，一个结构体应该有什么样的状态
- 函数规范unit specification (per single function ) ：每一个程序每一个函数要遵循什么样的规范
- 状态机规范state-machine specification：表达在区块链上整个全局的状态要遵循怎么样的一个状态规范。

这三个东西都可以由 Move prover 形式化验证的工具来规范。

Move 最早是为 Libra 项目所设计，后来Libra 改名为 Deim，Diem 后来卖给了Silvergate。而 Move 现在变成一个很多开发者参与，由社区来维护的开源的语言。

Move 的核心设计思想其实包括两点，第一点是它遵循了所谓的borrow semantic，一个借用的思想，这个就很像Rust的 borrow semantic，如果大家用过Rust语言的话，你会发现很多思想其实是借鉴于Rust来的。

另外一点就是 Move 遵循了所谓的 Linear Types，就像如果大家习惯于用这种所谓 smart pointer 的话，那么在c++里面有一个东西叫 unique pointer，它其实就是规范了一个对象的所有权，然后这个东西也被 Move 所吸收进来。

同时Move中是不存在动态调用。也就是说在编译一个 Move 智能合约的时候，我们就知道所有可能的执行路径，不会像 solidity 这样会有一个所谓的call back(动态调用的回调)，然后调用到`fallback`函数。这种事情在 Move 里面是不存在的。

最后Move中有一个完整的并且很强大的一个规范语言MSL。除了 Move 语言本身，开发者还可以对 Move 的程做出一些规范（spec），这个规范包括常见的pre/post condition，然后还有全局的全局状态不变量（global state invariant），这些我们都可以通过规范语言来实现。MSL规范语言其实是比 Move 语言本身要更强大的。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-09.jpg)

# Move 类型系统

Move 类型系统其实是 Move 的第一个安全特性，它主要保证了三个问题，一个是类型安全，一个是资源安全，一个是引用安全。

## 类型安全

首先类型安全，简单来说 Move 它是一个强类型系统，并且它严格保证了在任何情况下都不可以做类型转换，你永远不可能把一个类型转化为另一个类型，这是 Move 这个类型系统所提供的一个保证。

如果更详细的说的话，Move 类型安全表达的是：

- 每一个变量或者表达式，在Move中都有且只有一个类型

- 这个类型是在编译的时候就知道的

- 并且这个变量类型永远不可能被改变


以上就是Move的类型系统所要保证的。

熟悉 Move 会觉得这个说法的不太对，比如Move中具有一个freeze这样的操作，它可以把一个mutable reference改成一个immutable，或者有那种as语法可以把一个uint8变成另一个uint64。但实际你却发现这个操作它并没有对变量做改变，它做的其实是重新新建了一个变量，并且把新的变量给它一个固定的类型，所以这些操作并没有违反 Move 的类型安全保证。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-18.jpg)

例如如果试图编译如下的代码的话，可以发现如果你写了一个类型不安全的程序， Move compiler根本就不会让你通过，会直接报错，说你在这里试图把一个`integer`转换成一个`Coin`，当然这样是不合理的，所以这就是 Move 类型安全所要保证的。

```move
module 0x1::Account {
    struct Coin has key {
        balance: u64
    }

    public fun wrong1(acc: signer) {
        let val = 1000;
        move_to<Coin>(&acc, val);
    }

    public fun wrong2(acc: signer) {
        let val = 1000;
        let coin: Coin = val;
        move_to<Coin>(&acc, coin);
    }
 }
```



## 资源安全

在常见的这种编程语言里没有资源安全这个东西，Move中的资源安全允许你去做一些更细粒度的控制。对于比如说一个object它可不可以被复制，可不可以被丢弃，或者这个资源能存在于什么地方，以及这个资源的谁能生成，这个是对整个这种type的资源的一个控制。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-22.jpg)

我们如果举一个具体的例子来说的话。比如区块链上具有一种资源`Coin`，显然你如果试图复制一个 `coin`这种资源，相当于你从一个 `Coin` 变成两个`Coin`，那显然你钱就变多了，这个是不能接受的，同时 `Coin` 也不可能凭空消失，你必须把它花出去。简单来说就是一个东西它不能凭空出现，也不能凭空消失。这样的稀缺性就是由Move中的ablity，copy和drop来实现的。

Move资源具有[四种能力](https://movebook.chrisyy.top/abilities.html)：分别是copy、drop、key、store。

copy和drop这两个ability，它保证说一个资源它是不是能被复制，或者是不是能被扔到一边不管，它是由这两个ability决定。

还有两个ability (key 和 store )决定这个资源它可以出现在哪里，它是否可以出现在你的function里面，并且随着你的智能合约结束就消失了，还是它可以写到区块链上，这个也是这两个ability决定的。

另一个例子是[Hot Potato](https://move-patterns.chrisyy.top/hot-potato.html)，这是Move设计模式的一种，它通过Ability优雅的实现了函数的流程控制（可以去对比下Solidity中闪电贷的实现）。

## 引用安全

Move中还有一个引用安全保证了安全性。如果你习惯于c语言或者c++，你肯定会遇到这种所谓的dangling pinter的问题，也就是野指针，野指针可能导致修改任意对象的问题。所以在Move中引入了所有权，来保证这一问题。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-27.jpg)

"引用安全"简单来说就是遵循了Rust的所谓的ownership的规则：

- 任何一个函数里面的本地变量，在任何一个时间有且只能有一个owner来拥有。
- 同时对于一个全局变量，在整个blockchain里面，对于任何一个类型有且只有一个owner可以来拥有它

这里的owner代表的是最多一个写者，或者没有写者但有多个读者的情况。

# Move的形式化验证系统

形式化验证是一种使用严格的数学方法来描述行为和推理计算机系统的正确性的技术。现在已经在操作系统、编译器等对正确性要求高的领域有一定应用。这里存在一个很明显的问题就是为什么在有了类型系统之后，为什么我们还需要 Move Prover 这样一个形式化验证系统？

简单的说是因为：类型系统只能保证一个比较简单的逻辑，它并不能满足于我们表述真实世界中对应的更复杂的逻辑需求或者更复杂的意图。比如，类型系统不能保证某一种结构体中的变量不会到达一个非预期的状态，或者函数前后的关系的约束，以及 global state的状态，这些目前都是没有办法全部被类型系统所表示的。

所以需要通过一种更高层次的方式来表达，而这个更高层次的方式就是所谓的形式化验证。Move中提供了一种更高层次的语言也就是 [MSL](https://github.com/move-language/move/blob/main/language/move-prover/doc/user/spec-lang.md) (Move Specification Language) 语言来约束，规范Move代码，同时提供了一些工具来帮组我们来进行形式化验证，其中包含：

- 结构体不变量 Struct invariant
- 测试规范 Unit specification
  - 前置条件 Pre-condition
  - 异常条件Abort condition
  - 后置条件 Post-conditions

- 状态规范 State-machine specification
  - Global invariants (single-state invariant)
  - Global update invariants (two-state invariant)

## 结构体不变量 Struct Invariant

> Struct invariants allows you to specify complicated relations among the fields of a struct type which have to hold at runtime.

结构体不变量指的是允许你在运行时对结构体中不同的field之间复杂的关系作出规定，这是一个类型系统所达不到的。

一个最简单的例子，比如你根据一些事实或者商业逻辑，需要定义一个包含非0的64位的整数的结构体，最有效的方式是如果一个语言系统里已经存在一个非0的64位的整数类型，那么直接使用即可。但是这是目前在绝大多数语言里是没有办法做到，因为没有一个类型叫非0的64位整数。你能得到的最近的一个东西就是一个64位整数。所以你可以创造一个结构体，结构体的名字叫非零64位整数，然后里面有一个field，它叫结构体的值，然后它是64位整数。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-34.jpg)

但是这里存在一个问题，也就是整个所有的这些类型这个系统并不能阻止你写一个0进去，如果不小心写出来一个上图的函数 `fun create_zero()`它允许把一个0放到了结构体里面，这对运行时和编译器来说是完全没有问题的，但是这显然不是我们想要的。

所以这个时候就需要形式化验证的工具来给你提供能力。你就可以写一个形式化验证的一个规范，规定说在非零的结构体里面，它叫`value`的 `field`一定是要大于0。

然后在你有了这个规范之后，上文的函数 `fun create_zero()`，那么prover是会在这里报错。

而上图的第二个函数`fun create_x_checked()` 是允许的，因为这个函数是符合你所定义的规则，它一定不会把一个≤0的值放到这个结构体当中

所以当我们生成这样一个叫非0的64位整数的这样一个结构体的时候，其实就可以保证结构体的值一定不是0。这就是一个典型的 Move prove可以应用的一个例子。这一套结构体加结构体的规范，它其实起到的是一个“增强类型”的作用，它可以保证说结构体里面域之间的关系是符合某种关系。

刚刚是一个非常简单的例子，其实 Move Spec 可以支持写很复杂的关系，

## 函数规范 Function Specification

> For people not familiar with formal verification, function specification can be loosely considered as exhaustive unit testing.

Move Prover可以对一个函数做一个规范，其中就包括函数应该表现成什么样子，他在什么样情况下会异常报错，什么样的情况下会结束执行，而在结束执行的情况下，他又应该保持一个什么样的状态？这个就是函数规范所要达到的一个目的。

**可以把函数规范想象成一个人在写单元测试，但这个人比任何人写的测试都要好。**

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-37.jpg)

比如上图的代码，如果想知道这个代码对还是不对？那么你就可以开始对他做单元测试。

大家可能都听过一个单元测试的笑话，说QA工程师走进一家酒吧，要了一杯啤酒；要了-1杯啤酒，要了9999999杯啤酒，要了一杯洗脚水.... 然后这其实就是单元测试做的一个事情。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-39.jpg)

如果想去测试这个就取钱的函数，你会怎么做？你去走到一个ATM机前，然后你开始取钱，取10块钱，取1000块钱，取9999999999999999999999999099块钱，甚至从一个不应该取钱的一个地址，比如说零这个地址去取，取0块钱等，就是有目的的去测试一些边界值的可能性。

然后你去试图穷尽你写的代码里面所有的可能的路径，然后你发现穷尽所有路径都没问题，这个是单元测试想要做的。你可以想象成对于函数的规范形式化验证工具 Move prove它也是在做这样一件事情。虽然它并不是实际上去把所有的可能情况都遍历一下。

### 异常退出 Abort Conditions

Move prover 给你提供了允许你去更专业去表达这个函数应该怎么样做事务处理的机会，就是通过所谓的函数的规范你可以写一个取钱函数的规范。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-40.jpg)

上图是一个取钱的功能函数，取钱可能出现的错误存在两个地方：

- 首先如果地址就没有根本没有开户，那么会报错。这个就是第9行`abort if`的意思。
- 如果取的钱大于余额，那么同样会报错。也就是第10行的`abort if`的意思。

这里（从第8到11行）提供了一个取钱的专业的规范，它会报错并且它只会在这两个地方报错。不仅是它会报错，**同时是这两个报错是涵盖了所有可能的报错**，这个就是函数的规范。我们可以对上图的代码做一个简单的测试，跑一下形式化验证的工具，然后我们发现这个规范是对的，确实这个函数会在这两个情况下报错。

如果假设说我足够粗心，或者我们认为不存在开户的情况下程序不会报错，我们把第9行注释掉（相当于我们的规范不完整），在这个情况下，形式化验证的工具其实会报错的（如下图），它会提醒你有一个异常发生了，但是我们并没有在规范里面去提到它。

![img](https://pic1.zhimg.com/80/v2-a1caf50d0c9d65be880b7995aa28adc4_1440w.webp)

这个是不是这两个报错的条件，是不是真的是你期望看到的这种条件，我们都可以利用形式化验证工具来帮我们去理解这个程序的代码有什么问题。我们在实际的开发中经常会遇到说没想到它会报错，但是它实际上报错了的情况，这种时候这个 Prover 工具其实是非常有效的。

### 后置条件 Post-conditions

Prover除了关注它在什么情况下会出错，它还能够约束这个函数执行完要达到什么样的状态，这是比报错情况更重要的一个事情。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-44.jpg)

对于取钱这个函数的话，你得到的状态应该是当你的取钱函数结束了，你的钱肯定会减少，同时你减少的钱一定是等于你要取出来的钱。比如说你说要取100万，它一定是只减少100万，它不能减少200万，他也不能减少50万，他只能减少100万，这个东西就叫 后置条件post condition。

### 前置条件 Pre-conditions

除了可以写什么情况下会异常、没有异常的话状态会发生什么变化，我们还可以规范在什么情况下，我们可以真正的进入这个函数。比如要求“无论什么时候调用这个函数，必须要求这个人已经开过户了”。在调用内部取钱这个函数的时候，一定要求有一个账户存在，你就可以把这个东西列为所谓的前置条件，你在调用这个函数的时候前置条件一定要存在，有了这个前置条件，就可以简化你需要列出的异常退出条件。

这种比较类似于测试中的setup

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-46.jpg)

所以简单来说就是你有前置条件，你有异常条件和你有后置条件，你这个东西就可以完整的规范说你这个函数应该做什么，以及不能做什么，这就是函数的规范的内容。

### 全局不变量 Global Invariants

全局不变量类似于结构体不变量，只不过它能够限制所有拥有这个类型资源的状态。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-49.jpg)

如果把全局的区块链看作一个状态机的话，Global的资源状态就是区块链当前的状态。全局不变量提供了一个工具去描述什么状态是对的，什么状态是不对的这样一个功能，同时它还给你描述了在哪些情况下你可以做这个状态的转换。

具体的例子：你想写一个银行账户系统，每个账户里面存了每个人多少钱，你其实都不用去定义任何函数，只要定义了账户系统，就可以开始去定义一些全局的东西。比如说你想定义：

- 账户系统要保证任何现有的账户不能少于100块钱，

- 任何提取不能单次提取不能超过10%的现有的钱


那么形式化的规范可以按照下图来写，当你开始实现你的功能代码的时候，那么Prover可以根据规范来检查你的错误。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-50.jpg)

有了上图的规范代码后，你根本就不去关心它函数到底怎么实现，然后就可以去真正去写金融的实现，比如说你有一个取钱的函数，如下图的实现。

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-51.jpg)

当写完上述代码，**我们写完的这两个函数自动的就会被这两个全局的规范所限制**。刚刚那个代码去 Prover 验证的时候，我们会发现验证不过，因为写的函数里面并没有去支持我们想要的全局规范。

所以说全局规范简单点说，它是一个比函数的本身的规范更高层次的一个规范，它可以允许你去直接描述区块链的状态机，而不去关心具体实现。如果一个区块链的构架师不关心每个函数怎么执行，但是关心整个全局状态，他要整个区块链系统要保持一个什么样状态，就是这种全局的不变性或者不变量，那么全局规范可以满足这种要求。这个是关于整个全局变量的介绍。其实 Move Prover里还有很多其他高级的功能，如果大家感兴趣的话，可以持续关注 Move Prover的进展。

# 总结

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/MengXu-Move%20Prover-07.jpg)

以上就是Move为什么是一门安全的智能合约语言的介绍，简单的说就是通过类型系统和形式化验证系统来保证的。

类型系统通过类型安全，资源安全和Rust中类似的引用安全来保证。

而Move Prover也就是形式化验证系统，则通过各种约束条件，相比于类型系统可以提供的更多的保证。其中具有对类型的一种增强，也有限制函数前后关系的约束，此外通过全局不变量这个规范，可以这个全局的状态都是符合要求的。

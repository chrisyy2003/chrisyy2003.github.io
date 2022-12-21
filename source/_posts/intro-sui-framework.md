---
layout: post
title: Sui Framework分析
date: 2022-12-13 15:58:21
categories:
    - blockchain
---



https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/sources

# sui介绍

头等舱研报： [315-SUI.pdf](https://file.chrisyy.top/315-SUI.pdf) 

## 架构

![image-20221214141259428](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221214141259428.png)

每个 Objects 在 Sui 执行环境中都有一个唯一的 ID，并有指向所有者地址的内部指针。通过使用这些概念，很容易通过检查交易是否使用相同的 Objects 来识别关联。

通过将声明关联关系的工作转移给开发者，使执行引擎的实施变得更容易，这意味着理论上它可以有更好的性能和可扩展性。然而，这是以不太理想的开发者体验为代价的。

## Sui Move

![image-20221214141845093](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221214141845093.png)

# Framework分析

## 代币

1.   [sui](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/sui.move)

sui平台代币，通过coin模块铸造

```
public(friend) fun new(ctx: &mut TxContext): Supply<SUI> {
        let (treasury, metadata) = coin::create_currency(
            SUI {}, 
            9,
            b"SUI",
            b"Sui",
            // TODO: add appropriate description and logo url
            b"",
            option::none(),
            ctx
        );
        transfer::freeze_object(metadata);
        coin::treasury_into_supply(treasury)
    }
```

2.  [balance](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/balance.move)

balance可用于任何一般存储金额，余额的地方。例如可以用于 `Coin` ，可用于实现带有 `Supply` 和 `Balance` 的自定义硬币。

balance只用来存储金额，所以没有key的能力

```
struct Balance<phantom T> has store {
        value: u64
}
```

常见有如下的操作函数：

```
public fun zero<T>(): Balance<T> 
public fun join<T>(self: &mut Balance<T>, balance: Balance<T>): u64
public fun split<T>(self: &mut Balance<T>, value: u64): Balance<T>

public fun increase_supply<T>(self: &mut Supply<T>, value: u64): Balance<T>
public fun decrease_supply<T>(self: &mut Supply<T>, balance: Balance<T>): u64

```

3.   coin

`Coin` 类型是平台范围内的同质化代币， `Coin` 可以理解为安全包装过的`Balance`类型，它的结构如下：

```
struct Coin<phantom T> has key, store {
        id: UID,
        balance: Balance<T>
 }

```

常见有如下的操作函数：

基础代币功能

```

public entry fun join<T>(self: &mut Coin<T>, c: Coin<T>)
public fun split<T>(self: &mut Coin<T>, split_amount: u64, ctx: &mut TxContext): Coin<T>
public fun divide_into_n<T>(self: &mut Coin<T>, n: u64, ctx: &mut TxContext)
```

代币管理

```
public fun create_currency<T: drop>(
        witness: T,
        decimals: u8,
        symbol: vector<u8>,
        name: vector<u8>,
        description: vector<u8>,
        icon_url: Option<Url>,
        ctx: &mut TxContext
    ): (TreasuryCap<T>, CoinMetadata<T>)
    
    
public fun mint<T>(cap: &mut TreasuryCap<T>, value: u64, ctx: &mut TxContext): Coin<T>
public fun mint_balance<T>(cap: &mut TreasuryCap<T>, value: u64): Balance<T>
public fun burn<T>(cap: &mut TreasuryCap<T>, c: Coin<T>): u64
public entry fun mint_and_transfer<T>(
        c: &mut TreasuryCap<T>, amount: u64, recipient: address, ctx: &mut TxContext
    )
```

Balance <-> Coin 的访问器和类型转换方法

```
public fun from_balance<T>(balance: Balance<T>, ctx: &mut TxContext): Coin<T>
public fun into_balance<T>(coin: Coin<T>): Balance<T>
public fun take<T>(balance: &mut Balance<T>, value: u64, ctx: &mut TxContext): Coin<T>
```

**这里说说我对Balance和Coin区别的理解：**

Balance属于一个相对宽泛的记账概念，Balance表示的是一种记录数量的资产，而Coin是市场上流动的资产形式，Balance可以存在多种类型，例如货币、支票、保险、负债，Coin只是其中的的货币的表现形式，Coin在逻辑上对应的是可分割的代币，它需要一个记账的载体Balance。 所以Balance的用途并不局限于对于Coin的记账，实际上任何需要记账的地方都可以用到Balance。 

Balance保证了最基础的记账逻辑，例如不能无限增发，复制等等，并且Balance没有KEY的能力，所以也不能直接存放在区块链中。

4.   [pay](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/pay.move)

该模块为钱包和 sui::coin 管理提供方便的功能。



## NFT

https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/erc721_metadata.move

https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/devnet_nft.move



## 数据结构

1.  bag

bag是一个异构的map集合。该集合类似于 `sui::table` 它的键和值不存储在 `Bag` 值中，而是使用 Sui 的存储 对象系统。 `Bag` 结构仅充当**对象系统的句柄以检索**那些键和值。 

请注意，这意味着具有完全相同的键值映射的 `Bag` 值将不会被 在运行时使用 `==` 相等。

例如：

```
 let bag1 = bag::new();
 let bag2 = bag::new();
 bag::add(&mut bag1, 0, false);
 bag::add(&mut bag1, 1, true);
 bag::add(&mut bag2, 0, false);
 bag::add(&mut bag2, 1, true);
 // bag1 does not equal bag2, despite having the same entries
 assert!(&bag1 != &bag2, 0);
```

在它的核心，`sui::bag` 是 `UID` 的包装器，允许访问 `sui::dynamic_field` 同时防止意外搁浅字段值。 `UID` 可以是 删除，即使它有关联的动态字段，但另一方面，包必须是 空的被销毁。

**table和bag的区别 table的key和value类型初始化的时候就已经确定了，table只能存储同类型的key和value,。**

bag初始化的时候未限制具体类型，bag能存储不同类型的key和value。

dynamic_field和dynamic_object_field的区别 类似的table和object_table，bag和object_bag的区别，本质上也是dynamic_field和dynamic_object_field的区别

-   key的处理: 为了防止和dynamic_field key值冲突, dynamic_object_field对key类型进行了Wrapper。
-   加入存储的是一个object: 用dynamic_field存储，通过dynamic_field key值可以直接取到这个object的值，此时用object id在链上进行查询是查不到的 用dynamic_object_field存储, 通过dynamic_object_field key值取到的是这个object id, 需要用object id在链上再次查询才能获取到这个object的值

2.   [table](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/table.move)

3.   [priority_queue](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/priority_queue.move)

使用大根堆实现的优先级队列。



## 工具模块

1.	[transfer](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/transfer.move)

```

public fun transfer<T: key>(obj: T, recipient: address)
native fun transfer_internal<T: key>(obj: T, recipient: address);

public native fun freeze_object<T: key>(obj: T);

public native fun share_object<T: key>(obj: T);

```

2.   [tx_context](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/tx_context.move)

tx结构

```
struct TxContext has drop {
        /// The address of the user that signed the current transaction
        sender: address,
        /// Hash of the current transaction
        tx_hash: vector<u8>,
        /// The current epoch number.
        epoch: u64,
        /// Counter recording the number of fresh id's created while executing
        /// this transaction. Always 0 at the start of a transaction
        ids_created: u64
    }
```

常见有如下的操作函数：

```
public fun sender(self: &TxContext): address

public(friend) fun new_object(ctx: &mut TxContext): address {
        let ids_created = ctx.ids_created;
        let id = derive_id(*&ctx.tx_hash, ids_created);
        ctx.ids_created = ids_created + 1;
        id
    }
```

3.   [test_scenario](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/test_scenario.move)

测试框架

```
public fun begin(sender: address): Scenario

/// 将场景推进到新交易，其中 `sender` 是交易发送方 
/// 所有转移的对象将被移动到账户或全局的库存中 
/// 存货。换句话说，为了访问具有各种“取”之一的对象 
/// 下面的函数，例如`take_from_address_by_id`，交易必须先通过 
/// `next_tx`。 
/// 返回上一次交易的结果 
/// 如果共享或不可变对象被删除、传输或包装，将中止。 
/// 如果无法生成 TransactionEffects 将中止
public fun next_tx(scenario: &mut Scenario, sender: address): TransactionEffects

public fun end(scenario: Scenario): TransactionEffects

public fun ctx(scenario: &mut Scenario): &mut TxContext

public fun new_object(scenario: &mut Scenario): UID

public fun created(effects: &TransactionEffects): vector<ID>

```

资源操作

```move
public fun take_from_address<T: key>(scenario: &Scenario, account: address): T
public fun return_to_address<T: key>(account: address, t: T)

public fun take_shared<T: key>(scenario: &Scenario): T
public fun return_shared<T: key>(t: T)
```

## 密码学

1.  [bulletproof](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/crypto/bulletproofs.move)

```
public fun verify_full_range_proof(proof: &vector<u8>, commitment: &RistrettoPoint, bit_length: u64): bool
```

2.   [groth16](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/crypto/groth16.move)

```
// groth16 验证算法，参数
public fun verify_groth16_proof(prepared_verifying_key: &PreparedVerifyingKey, public_proof_inputs: &PublicProofInputs, proof_points: &ProofPoints): bool

// 获取verify key
public fun pvk_from_bytes(vk_gamma_abc_g1_bytes: vector<u8>, alpha_g1_beta_g2_bytes: vector<u8>, gamma_g2_neg_pc_bytes: vector<u8>, delta_g2_neg_pc_bytes: vector<u8>): PreparedVerifyingKey

```

# 案例

## hello_move

为了获得一个`Counter`计数器对象并且，使得`Counter`计数器对象中的值增加，我们需要调用合约中的函数。

在函数`getCounter`中，存在`public`，`entry`修饰符，这保证了我们拥有调用权限，并且可以通过命令行调用。`transfer`函数的参数为对象接收者地址，在代码中，我们通过`tx_context::sender(ctx)`来获取发送者地址，`ctx`是当前交易的上下文，包含此交易的相关信息。

所以我们首先需要调用`getCounter`函数，在命令行输入如下命令。

```
sui client call \
    --function getCounter \
    --module counter \
    --package 0x31f33e53a2c7a2620fc1bbf8140ffc7bde3984fa \
    --gas-budget 1000
```

这是一个相当复杂的命令，所以让我们一一解释它的所有参数：

- `--function`：要调用的函数的名称
- `--module`：包含函数的模块的名称
- `--package`：包含函数的模块所在的包对象的 `ID`。
- `--gas-budge`：是一个十进制数，表示我们交易的gas上限，以避免 gas pay 中所有 gas 的意外耗尽）

可以发现交易结果中返回了一个新创建的对象`ID`，很明显这就是我们获得的`Counter`对象

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221128180338202.png)

同时在浏览器上可以直接通过对象`ID`看到`counter`的`value`字段的具体值

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221128180507340.png)

最后我们试图调用`incr`来使得`value`的值+1

```
sui client call \
    --function incr \
    --module counter \
    --package 0x31f33e53a2c7a2620fc1bbf8140ffc7bde3984fa \
    --args 0x846e1db8383dd68373cd83c6ce5242951d7beb77 \
    --gas-budget 1000
```

其中`--args`用来传递我们的参数，参数格式参考 [Sui-JSON](https://docs.sui.io/build/sui-json)值的函数参数列表。

再次通过浏览器可以发现version（可以理解为修改的次数）被+1，同时`value`的字段值也成功+1

![image-20221128182935767](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221128182935767.png)

## test_example

## token



# 资料

sui官方课程：https://github.com/sui-foundation/sui-move-intro-course

sui测试：https://docs.sui.io/build/move/build-test

案例：https://examples.sui.io/

设计模式：https://move-patterns.chrisyy.top/


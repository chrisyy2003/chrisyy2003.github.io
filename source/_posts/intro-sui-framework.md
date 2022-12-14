---
layout: post
title: Sui Framework分析
date: 2022-12-13 15:58:21
tags:
---



https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/sources

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

Balance和Coin的理解：

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

table和bag的区别 table的key和value类型初始化的时候就已经确定了，table只能存储同类型的key和value,。

bag初始化的时候未限制具体类型，bag能存储不同类型的key和value。

dynamic_field和dynamic_object_field的区别 类似的table和object_table，bag和object_bag的区别，本质上也是dynamic_field和dynamic_object_field的区别

-   key的处理: 为了防止和dynamic_field key值冲突, dynamic_object_field对key类型进行了Wrapper。
-   加入存储的是一个object: 用dynamic_field存储，通过dynamic_field key值可以直接取到这个object的值，此时用object id在链上进行查询是查不到的 用dynamic_object_field存储, 通过dynamic_object_field key值取到的是这个object id, 需要用object id在链上再次查询才能获取到这个object的值

2.   [table](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/table.move)



## 工具模块

1.	[transfer](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/transfer.move)



1.	[tx_context](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/tx_context.move)

2.	[test_scenario](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/test_scenario.move)

测试框架

4.	.   





bsc：



1.  [priority_queue](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/priority_queue.move)

使用大根堆实现的优先级队列。







# 资料

sui官方课程：https://github.com/sui-foundation/sui-move-intro-course

sui测试：https://docs.sui.io/build/move/build-test

案例：https://examples.sui.io/

设计模式：https://move-patterns.chrisyy.top/


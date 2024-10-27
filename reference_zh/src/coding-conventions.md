# 编码规范

以下建议基于 2024 年的 Move。

## 添加章节标题

在代码注释中使用标题为您的 Move 代码文件创建章节。使用 `===` 在标题的两侧来构建您的标题。

```move
module conventions::comments {
    // === Imports ===

    // === Errors ===

    // === Constants ===

    // === Structs ===

    // === Method Aliases ===

    // === Public-Mutative Functions ===

    // === Public-View Functions ===

    // === Admin Functions ===

    // === Public-Package Functions ===

    // === Private Functions ===

    // === Test Functions ===
}
```

## CRUD 函数命名

以下是可用的 CRUD 函数：

- `add`: 添加一个值。
- `new`: 创建一个对象。
- `drop`: 删除一个结构体。
- `empty`: 创建一个结构体。
- `remove`: 移除一个值。
- `exists_`: 检查一个键是否存在。
- `contains`: 检查集合中是否包含一个值。
- `destroy_empty`: 销毁具有 **drop** 能力的对象或数据结构。
- `to_object_name`: 将对象 X 转换为对象 Y。
- `from_object_name`: 将对象 Y 转换为对象 X。
- `property_name`: 返回一个不可变引用或副本。
- `property_name_mut`: 返回一个可变引用。

## Potato 结构体

不要在结构体名称中使用 'potato'。缺乏能力定义它为一个 potato 模式。

```move
module conventions::request {
    // ✅ 正确
    struct Request {}

    // ❌ 错误
    struct RequestPotato {}
}
```

## 读取函数

在命名函数时要注意点语法。避免在函数名称中使用对象名称。

```move
module conventions::profile {

    struct Profile {
        age: u64
    }

    // ✅ 正确
    public fun age(self: &Profile): u64 {
        self.age
    }

    // ❌ 错误
    public fun profile_age(self: &Profile): u64 {
        self.age
    }
}

module conventions::defi {

    use conventions::profile::{Self, Profile};

    public fun get_tokens(profile: &Profile) {

     // ✅ 正确
     let name = profile.age();

     // ❌ 错误
     let name2 = profile.profile_age();
    }
}
```

## 空函数

将创建数据结构的函数命名为 `empty`。

```move
module conventions::collection {

    struct Collection has copy, drop, store {
        bits: vector<u8>
    }

    public fun empty(): Collection {
        Collection {
            bits: vector[]
        }
    }
}
```

## new 函数

将创建对象的函数命名为 `new`。

```move
module conventions::object {

    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;

    struct Object has key, store {
        id: UID
    }

    public fun new(ctx:&mut TxContext): Object {
        Object {
            id: object::new(ctx)
        }
    }
}
```

## 共享对象

共享对象的库模块应提供两个函数：一个用于创建对象，另一个用于共享对象。这样调用者可以访问其 UID 并在共享之前运行自定义功能。

```move
module conventions::profile {

    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    use sui::transfer::share_object;

    struct Profile has key {
        id: UID
    }

    public fun new(ctx:&mut TxContext): Profile {
        Profile {
            id: object::new(ctx)
        }
    }

    public fun share(profile: Profile) {
        share_object(profile);
    }
}
```

## 引用函数

将返回引用的函数命名为 `<PROPERTY-NAME>_mut` 或 `<PROPERTY-NAME>`，用实际的属性名称替换 `<PROPERTY-NAME>`。

```move
module conventions::profile {

    use std::string::String;

    use sui::object::UID;

    struct Profile has key {
        id: UID,
        name: String,
        age: u8
    }

    // profile.name()
    public fun name(self: &Profile): &String {
        &self.name
    }

    // profile.age_mut()
    public fun age_mut(self: &mut Profile): &mut u8 {
        &mut self.age
    }
}
```

## 关注点分离

围绕一个对象或数据结构设计您的模块。变体结构应有自己的模块以避免复杂性和错误。

```move
module conventions::wallet {

    use sui::object::UID;

    struct Wallet has key, store {
        id: UID,
        amount: u64
    }
}

module conventions::claw_back_wallet {

    use sui::object::UID;

    struct Wallet has key {
        id: UID,
        amount: u64
    }
}
```

## 错误

使用 PascalCase 为错误命名，以 E 开头并具有描述性。

```move
module conventions::errors {
    // ✅ 正确
    const ENameHasMaxLengthOf64Chars: u64 = 0;

    // ❌ 错误
    const INVALID_NAME: u64 = 0;
}
```

## 结构体属性注释

描述您的结构体的属性。

```move
module conventions::profile {

    use std::string::String;

    use sui::object::UID;

    struct Profile has key, store {
        id: UID,
        /// 用户的年龄
        age: u8,
        /// 用户的名字
        name: String
    }
}
```

## 销毁函数

提供删除对象的函数。使用 `destroy_empty` 函数销毁空对象。对于具有可删除类型的对象，使用 `drop` 函数。

```move
module conventions::wallet {

    use sui::object::{Self, UID};
    use sui::balance::{Self, Balance};
    use sui::sui::SUI;

    struct Wallet<Value> has key, store {
        id: UID,
        value: Value
    }

    // Value 具有 drop
    public fun drop<Value: drop>(self: Wallet<Value>) {
        let Wallet { id, value: _ } = self;
        object::delete(id);
    }

    // Value 不具有 drop
    // 如果 `wallet.value` 不为空则抛出。
    public fun destroy_empty(self: Wallet<Balance<SUI>>) {
        let Wallet { id, value } = self;
        object::delete(id);
        balance::destroy_zero(value);
    }
}
```

## 纯函数

保持您的函数纯净以维护可组合性。在核心函数中不要使用 `transfer::transfer` 或 `transfer::public_transfer`。

```move
module conventions::amm {

    use sui::transfer;
    use sui::coin::Coin;
    use sui::object::UID;
    use sui::tx_context::{Self, TxContext};

    struct Pool has key {
        id: UID
    }

    // ✅ 正确
    // 即使它们的值为零，也要返回多余的硬币。
    public fun add_liquidity<CoinX, CoinY, LpCoin>(pool: &mut Pool, coin_x: Coin<CoinX>, coin_y: Coin<CoinY>): (Coin<LpCoin>, Coin<CoinX>, Coin<CoinY>) {
        // 实现省略。
        abort(0)
    }

    // ✅ 正确
    public fun add_liquidity_and_transfer<CoinX, CoinY, LpCoin>(pool: &mut Pool, coin_x: Coin<CoinX>, coin_y: Coin<CoinY>, recipient: address) {
        let (lp_coin, coin_x, coin_y) = add_liquidity<CoinX, CoinY, LpCoin>(pool, coin_x, coin_y);
        transfer::public_transfer(lp_coin, recipient);
        transfer::public_transfer(coin_x, recipient);
        transfer::public_transfer(coin_y, recipient);
    }

    // ❌ 错误
    public fun impure_add_liquidity<CoinX, CoinY, LpCoin>(pool: &mut Pool, coin_x: Coin<CoinX>, coin_y: Coin<CoinY>, ctx: &mut TxContext): Coin<LpCoin> {
        let (lp_coin, coin_x, coin_y) = add_liquidity<CoinX, CoinY, LpCoin>(pool, coin_x, coin_y);
        transfer::public_transfer(coin_x, tx_context::sender(ctx));
        transfer::public_transfer(coin_y, tx_context::sender(ctx));

        lp_coin
    }
}
```

## 硬币参数

通过值传递 `Coin` 对象，并直接传递正确的金额，因为这对前端的交易可读性更好。

```move
module conventions::amm {

    use sui::coin::Coin;
    use sui::object::UID;

    struct Pool has key {
        id: UID
    }

    // ✅ 正确
    public fun swap<CoinX, CoinY>(coin_in: Coin<CoinX>): Coin<CoinY> {
        // 实现省略。
        abort(0)
    }

    // ❌ 错误
    public fun exchange<CoinX, CoinY>(coin_in: &mut Coin<CoinX>, value: u64): Coin<CoinY> {
        // 实现省略。
        abort(0)
    }
}
```

## 访问控制

为了保持可组合性，使用能力而不是地址进行访问控制。

```move
module conventions::access_control {

    use sui::sui::SUI;
    use sui::object::UID;
    use sui::balance::Balance;
    use sui::coin::{Self, Coin};
    use sui::table::{Self, Table};
    use sui::tx_context::{Self, TxContext};

    struct Account has key, store {
        id: UID,
        balance: u64
    }

    struct State has key {
        id: UID,
        accounts: Table<address, u64>,
        balance: Balance<SUI>
    }

    // ✅ 正确
    // 通过此函数，另一个协议可以代表用户持有 `Account`。
    public fun withdraw(state: &mut State, account: &mut Account, ctx: &mut TxContext): Coin<SUI> {
        let authorized_balance = account.balance;

        account.balance = 0;

        coin::take(&mut state.balance, authorized_balance, ctx)
    }

    // ❌ 错误
    // 这不太可组合。
    public fun wrong_withdraw(state: &mut State, ctx: &mut TxContext): Coin<SUI> {
        let sender = tx_context::sender(ctx);

        let authorized_balance = table::borrow_mut(&mut state.accounts, sender);
        let value = *authorized_balance;
        *authorized_balance = 0;
        coin::take(&mut state.balance, value, ctx)
    }
}
```

## 拥有对象与共享对象中的数据存储

如果您的 dApp 数据具有一对一关系，最好使用拥有的对象。

```move
module conventions::vesting_wallet {

    use sui::sui::SUI;
    use sui::coin::Coin;
    use sui::object::UID;
    use sui::table::Table;
    use sui::balance::Balance;
    use sui::tx_context::TxContext;

    struct OwnedWallet has key {
        id: UID,
        balance: Balance<SUI>
    }

    struct SharedWallet has key {
        id: UID,
        balance: Balance<SUI>,
        accounts: Table<address, u64>
    }

    /*
    * 一个归属钱包在一段时间内释放一定数量的硬币。
    * 如果整个余额属于一个用户并且钱包没有其他功能，最好将其存储在一个拥有的对象中。
    */
    public fun new(deposit: Coin<SUI>, ctx: &mut TxContext): OwnedWallet {
        // 实现省略。
        abort(0)
    }

    /*
    * 如果您希望为归属钱包添加额外功能，最好共享该对象。
    * 例如，如果您希望钱包的发行者能够在未来取消合同。
    */
    public fun new_shared(deposit: Coin<SUI>, ctx: &mut TxContext) {
        // 实现省略。
        // 它共享 `SharedWallet`。
        abort(0)
    }
}
```

## 管理能力

在管理权限控制的函数中，第一个参数应为能力。这有助于自动补全用户类型。

```move
module conventions::social_network {

    use std::string::String;

    use sui::object::UID;

    struct Account has key {
        id: UID,
        name: String
    }

    struct Admin has key {
        id: UID,
    }

    // ✅ 正确
    // cap.update(&mut account, b"jose");
    public fun update(_: &Admin, account: &mut Account, new_name: String) {
        // 实现省略。
        abort(0)
    }

    // ❌ 错误
    // account.update(&cap, b"jose");
    public fun set(account: &mut Account, _: &Admin, new_name: String) {
        // 实现省略。
        abort(0)
    }
}
```


请查看 [Sui 的 Move 编码规范](https://docs.sui.io/concepts/sui-move-concepts/conventions)

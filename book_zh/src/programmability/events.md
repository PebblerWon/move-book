# 事件

事件是一种通知链下监听器关于链上事件的方式。它们用于发布关于交易的额外信息，这些信息不会存储在链上，因此无法在链上访问。事件由[Sui Framework](./sui-framework.md)中的`sui::event`模块发布。

> 任何具有 [copy](./../move-basics/copy-ability.md) 和 [drop](./../move-basics/drop-ability.md) 能力的自定义类型都可以作为事件发布。Sui Verifier 要求类型是模块的内部类型。

```move
// 文件：sui-framework/sources/event.move
module sui::event {
    /// 发布自定义的Move事件，将数据发送到链下。
    ///
    /// 用于创建自定义索引并以最适合特定应用程序的方式跟踪链上活动。
    ///
    /// 类型 `T` 是索引事件的主要方式，可以包含幻象参数，例如 `emit(MyEvent<phantom T>)`。
    public native fun emit<T: copy + drop>(event: T);
}
```

## 发布事件

使用`sui::event`模块中的`emit`函数发布事件。该函数接受一个参数 —— 要发布的事件。事件数据以值传递。

```move
{{#include ../../../packages/samples/sources/programmability/events.move:emit}}
```

Sui Verifier 要求传递给`emit`函数的类型是 _模块的内部类型_。因此，发布自其他模块的类型将导致编译错误。尽管原始类型符合 _copy_ 和 _drop_ 要求，但它们不允许作为事件发布。

## 事件结构

事件是交易结果的一部分，存储在 _交易结果_ 中。因此，它们本质上具有`sender`字段，即发送交易的地址。因此，添加“sender”字段到事件中是不必要的。同样，事件元数据包含时间戳，但需要注意的是时间戳相对于节点，可能会因节点不同而有所变化。
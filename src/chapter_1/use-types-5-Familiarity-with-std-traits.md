# 第五条：熟悉标准特质


Rust 在其类型系统中通过一系列有着细行为粒度的标准特质(standard traits)来表达其关键行为，这些特质描述了这些行为（参见[方法 2]）

从C++转来的程序员会对许多标准特质感到面熟，这些特质对应于诸如拷贝-构造函数、析构函数、相等性和赋值运算符等概念。

和在 C++ 中一样，为自己的类型实现许多标准特质通常是个明智的选择；如果某个操作需要你的类型具有这些特质但你却没有实现之，Rust 编译器会给出有用的错误消息提示你。

虽然实现这么多的特质可能看起来有些吓人，但是大多数常见的特质都可以通过使用派生宏来自动应用到用户定义的类型上。这使得类型定义的形式会如下：

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)] enum MyBooleanOption { 
	Off, 
	On, 
}
```

这种细致入微的行为规范一开始可能会有点唬人，但熟悉了这些常见的标准特质,我们就能立即理解类型定义的可用行为，这相当重要。

这一节里所涵盖的每个标准特质可以被粗略总结为：

- `Clone`：具备 `Clone` 的类型在被要求时可以复制自身。
- `Copy`：编译器对该项的内存表示进行逐位复制，就可以得到一个全新的合法项。
- `Default`：可以使用默认值创建一个合适的该类型的新实例。
- `PartialEq`：若一个类型具备该特质，则各项之间存在[部分等价关系](https://en.wikipedia.org/wiki/Partial_equivalence_relation) - 任何两个项都可以明确地比较，但不能保证自反性 `x == x` 一定成立。
- `Eq`：该类型的项之间存在等价关系：任何两个项都可以明确定义地比较，且始终满足自反性 `x == x`. 
- `PartialOrd`：该类型的**某些**项可以进行比较和排序。
- `Ord`：该类型的**所有**项都可以进行比较和排序。
- `Hash`：该类型的项在被请求时可以生成其内容的稳定的哈希值。
- `Debug`：该类型的项可以显示给程序员。
- `Display`：该类型的项可以显示给用户。

 除了 `Display`（在这里列出了 `Display` 是因为它与 `Debug`在功能上有重合）,这些特质都可以通过`derive` 派生宏来为用户自定类型派生。不过，在某些情况下，需要手动实现或者选择不实现这些特质会更好。

Rust 还允许用户为定义的类型重载各种内置一元和二元运算符，方法是实现 std::ops 模块中的各种对应特质。这些特质不能派生，并且通常只适用于表示“代数”对象的类型。

其他（不可 由`derive` 派生）的标准特质在其他建议中已经涵盖了，因此在这里不再讨论。这些特质包括：

- `Fn`、`FnOnce` 和 `FnMut`：实现此特质的项表示可以调用的闭包。请参见[方法 2]。
- `Error`：实现此特质的项可以给用户或程序员显示错误信息，并且可能包含嵌套的子错误信息。请参见[方法 4]。
- `Drop`：实现此特质的项在销毁时执行处理，这是对RAII极为重要。请参见[方法 11]
- `From` 和 `TryFrom`：实现此特质的项可以自动从某些其他类型的项创建出来，但 `TryFrom` 意味着这个创建过程可能会失败。请参见[方法 6]
- `Deref` 和 `DerefMut`：实现此特质的项是类似指针的对象，可以对其进行解引用以访问其内部。请参见 [方法 9]
- `Iterator` 及其相关特质：实现此特质的项表示可以进行迭代的。请参见[方法 10].
- `Send`：实现此特质的项可以在多个线程之间安全传输。请参见[方法 17]。
- `Sync`：实现此特质的项可以被多个线程安全地引用。请参见[方法 17]。

## `Clone`

`Clone` 特质表示可以通过调用 `clone()` 方法创建项的新副本。这大致相当于 C++ 的拷贝构造函数，但更加明确：编译器永远不会在自己静默调用此方法（请继续阅读下一节了解详情）。
`Clone` 可以派生；宏实现按顺序克隆复合类型的每个成员，这也大致相当于 C++ 中的默认拷贝构造函数。这使得特质是“选择加入”的（通过添加 `#[derive(Clone)]`来引入），与 C++ 中的“选择拒绝加入”的行为相反（`MyType(const MyType&) = delete;`）。

这是一个非常常见和有用的操作，所以我们需要仔细研究不应该或不能实现 `Clone` 的情况，或者默认派生实现不合适的情况。

- 如果项具有对某些资源的唯一访问（例如 RAII 类型，[方法 11]）或者有其他原因限制复制（例如，如果项持有加密密钥材料），则不应该实现 Clone。
- 如果你的类型的某些部分无法被递归地复制，例如：
    - 是可变引用（&mut T）的字段，因为借用检查器（第15项）只允许一次可变引用。
    - 属于前面这一类的标准库类型，例如 `MutexGuard`（具有唯一访问权限）或 `Mutex`（为了线程安全限制复制）。
- 如果有关于项的任何内容都不能被（递归地）字段复制捕获，或者有额外的标注与项的生命周期相关联，那么应该手动实现 `Clone`。例如，一个在运行时跟踪现有项数量以进行度量的类型；手动 `Clone` 实现可以确保计数器保持准确。
---
title : '4.2 Ownership'
date : 2024-12-24T14:57:01+08:00
draft : false
tags:
 - Rust
 - ownership
 - move
---

如果你阅读过很多C或C++的代码，你可能会见到这样的注释：一个类的实例“拥有”它所指向的其他对象。这通常意味着拥有对象有权决定何时释放被拥有的对象。当拥有者被销毁时，它会连带销毁它所拥有的对象。

我们用一个C++代码做个演示：

```c++
std::string s = "frayed knot";
```

字符串`s`在内存中可以表示为：

![alt](/rust-note/ownership-and-moves/pics/string_mem_0401.png)
> C++ std::string value on the stack, pointing to its heap-allocated buffer

其中，`std::string`对象总是确定地占用3个字长（3 words long），包含以下部分：

1. A pointer to a heap-allocated buffer（指向堆内存的指针）
2. The buffer's overall capacity（堆内存分配的总容量） —— 为了容纳超过这一容量的更大的字符串信息，内存会进行重新分配，但目前这个容量是指在内存扩大重新分配之前，它可以容纳的最大字符串容量
3. 当前字符串的长度

在`std::string`类中，以上这些属性是私有的，对于字符串的使用者来说是不可访问的。

一个`std::string`拥有它的内存buffer：当程序销毁字符串时，字符串的析构函数会被执行并释放它所拥有的内存buffer。曾经，有些C++类库中，多个`std::string`对象会共享同一个内存buffer，通过一个引用计数来决定buffer应该什么时候被释放。更新版本的C++标准杜绝了这种方式，几乎所有现代的C++类库都是使用上图中表示的方式了。

在这些情况下，通常的理解是，尽管其它代码创建指向被拥有内存的临时指针是可以的，但确保在拥有者决定销毁被拥有对象之前这些指针已不复存在，是代码的责任。你可以创建一个指向存在于`std::string`缓冲区中的字符的指针，不过当该字符串被销毁时，你的指针就会变为无效，并且你有责任确保不再使用它。拥有者决定被拥有者对象的生命周期，而且他所有人都必须尊重它的决定。

我们使用了`std::string`作为一个例子来演示C++中的ownership看起来是什么样子的：这只是标准库通常遵循的一种约定，尽管该语言鼓励你遵循类似的做法，但如何设计你自己的类型最终还是由你自己决定。

然而，在Rust中，ownership的概念被内置在语言本身，并通过编译时检查来强制执行。每一个值都有唯一的所有者，由该所有者决定其生命周期。当所有者被释放时（在Rust术语中称为被丢弃*dropped*），被其拥有的值也会被丢弃。这些规则旨在让你能够通过简单地查看代码就能轻松确定任何给定值的生命周期，赋予你像系统编程语言应提供的那样对值生命周期的掌控能力。

变量拥有其对应的值。当控制流离开声明该变量的代码块时，这个变量就会被丢弃，那么与之对应的变量值也会被随之丢弃。

```rust
fn print_padovan() {
    let mut padovan = vec![1, 1, 1];  // allocated here
    for i in 3..10 {
        let next = padovan[i - 3] + padovan[i - 2];
        padovan.push(next);
    }
    println!("P(1..10) = {:?}", padovan);
}                                     // dropped here
```

变量`padovan`的类型是`Vec<i32>`，它是一个32-bit整型的向量集合。最终，`padovan`在内存中看起来是这样的：

![alt](/rust-note/ownership-and-moves/pics/rust_vec_mem_0402.png)

这与我们在上面看到C++中的`std::string`非常类似，唯一区别就是buffer中的元素不再是字符，而是32-bit的整数了。需要注意的是，用于存储`padovan`的`pointer`、`capacity`、`length`的这些存储单元，是直接存放在`print_padovan`函数的栈帧中的，只有向量的缓冲区是分配在堆内存里的。

就像之前的字符串变量`s`一样，vector也是持有它的元素的buffer的所有者。当变量`padovan`在方法最后离开方法的定义范围时，程序会丢弃vector，同时，vector所拥有的buffer也会被释放。

Rust的`Box`类型服务于ownership的另外一个示例。`Box<T>`是一个指向存储在堆内存的T类型数据的指针。当调用`Box::new(v)`时会在堆内存分配空间，将`v`移动到它里面，之后返回一个`Box`指针指向堆内存空间。因为`Box`是它所指向空间的所有者，当`Box`被丢弃时，它所指向的内存空间也会被释放。

例如，你可以通过这种方式在堆内存为元组（Tuple）分配一片空间：

```rust
{
    let point = Box::new((0.625, 0.5));  // point allocated here
    let label = format!("{:?}", point);  // label allocated here
    assert_eq!(label, "(0.625, 0.5)");
}                                        // both dropped here
```

当程序调用`Box::new`时，它会在堆内存上为持有两个f64数据的元组分配内存空间，然后将参数`(0.625, 0.5)`移动到这片区域，最后返回一个指向该区域的指针。当上述代码执行到`assert_eq!`时，内存表现形式如下：

![alt](/rust-note/ownership-and-moves/pics/rust_tuple_mem_0403.png)

变量`point`和`label`直接被栈帧持有，它们各自指向一片内存区域。当它们被丢弃时，被分配的空间会随之释放。

变量拥有它的值，对应的，structs拥有它们的字段，tuple、array、vector则拥有它们的元素：

```rust
struct Person { name: String, brith: i32 }

let mut composers = Vec::new();
composers.push(Person { name: "Palestrina".to_string(),
                        birth: 1525 });
composers.push(Person { name: "Dowland".to_string(),
                        birth: 1563 });
composers.push(Person { name: "Lully".to_string(),
                        birth: 1632 });

for composer in &composers {
    println!("{}, born {}", composer.name, composer.birth);
}
```

`composers`是一个`Vec<Person>`，一个结构体的向量，每一个都持有一个字符串和一个数字。`composers`在内存中的最终形态如下：

![alt](/rust-note/ownership-and-moves/pics/structs_vector_mem_0404.png)

这里存在许多ownership的关系，但每一种关系都相当简单明了：`composers`拥有一个vector，vector则拥有它自己的`Person`结构体，每个结构体又拥有它们自己的成员字段，而字符串字段则拥有其文本内容。当控制流离开`composers`的作用域时，程序会丢弃它的值，并连带处理与之相关的整个资源安排情况。如果在这种场景中涉及其他类型的集合 —— 比如哈希表（HashMap）或者二叉搜索树集合（BTreeSet），情况也会是一样的。

现在，让我们重新考虑一下目前为止引入ownership关系产生的影响。任何值都只有唯一一个所有者，这在决定是否要丢弃它们时非常简单。但是一个值也可能拥有很多其它值：例如，`composers`向量拥有它所有的元素。同时这些元素也分别拥有它们自己的值：每个`composers`中的元素都拥有一个string，这些string又拥有自己的文本。

这些ownership关系被组织成了一个树型结构：你的所有者就是你的父节点，而你所拥有的数据就是你的子节点。整棵树的根节点就是一个变量，当这个变量离开作用域时，整棵树都会被丢弃。可以说，在Rust中，所有的值都是树的成员，它们都来源于某一个变量。

Rust程序通常不会显示地丢弃一个值，并没有在C或C++中的`free`和`delete`的操作。丢弃一个值的方式就是通过将它从ownership树结构中移除它：通过离开变量定义的作用域，或是从一个向量中删除元素，或者诸如此类的操作。Rust能够保证该值以及它所拥有的一切都能被妥善地释放。

从某种意义上来讲，Rust语言不如其它编程语言强大：其它所有实用的编程语言都允许你按照自己认为合适的任何方式构建对象之间相互指向的任意图形结构。但恰恰是因为Rust没有那么强大，所以该语言能够对程序执行的分析才能更强大。Rust之所以能够提供安全保障，正是因为它在代码中可能遇到的各种关系更易于处理。这是我们之前提到的Rust的“大胆赌注”的一部分：Rust声称，在实际应用中，人们在解决问题的方式上通常有足够多的灵活性，以确保至少有一个非常好的解决方案能够落在该语言所施加的限制范围之内。

话虽如此，就我们目前所解释的ownership概念而言，它仍然太过刻板，实用性欠佳。Rust通过几种方式对这一简单概念进行了扩展：

- 你可以将值从一个所有者转移到另一个所有者那里。这使你能够构建、重新排列以及拆解（ownership）树结构。
- 像整数、浮点数和字符这类非常简单的类型不受ownership规则的约束。它们被称作“可复制（Copy）”类型。
- 标准库提供了引用计数指针类型`Rc`（单线程引用计数指针）和`Arc`（多线程引用计数指针），在某些限制条件下，它们允许值拥有多个所有者。
- 你可以“借用一个引用”指向某个值；引用是一种非拥有型指针，有着有限的生命周期。

上述的每一种策略都为所有权模型增添了灵活性，同时依然坚守着Rust语言所承诺的特性。
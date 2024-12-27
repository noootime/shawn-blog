---
title : '4.3 Moves'
date : 2024-12-25T14:57:01+08:00
draft : false
tags:
 - Rust
 - ownership
 - move
---

### Move

在Rust中，对于大多数类型，变量赋值、方法传参、方法返回值等操作不会拷贝值，而是“移动（`move`）”它。源操作会将该值的所有权让渡给目标操作，然后自身状态变为未初始化状态；此时目标操作便掌控了该值的生命周期。Rust程序就是通过一次处理一个值、一次进行一次移动这样的方式来构建和拆解复杂结构的。

你可能会感到惊讶，Rust竟然会改变这些基础操作的含义；毕竟赋值操作在编程发展到如今这个阶段，理应是已经确定好了的。然而，如果你仔细观察不同语言在处理赋值操作时所做的选择，就会发现实际上不同语言流派之间存在着显著的差异。这种对比也使得Rust所做选择的含义及产生的后果更易于理解。

先研究一段Python代码：

```python
s = ['udon', 'ramen', 'soba']
t = s
u = s
```

在Python中，每个对象都携带一个引用计数（reference count），用来跟踪记录值被指针引用的次数。因此，当变量`s`被赋值后，程序的状态看起来是这样的：

<!-- ![alt](/rust-note/ownership-and-moves/pics/python_list_mem_0405.png) -->
![alt](../pics/python_list_mem_0405.png)

因为只有`s`指向这个数组，所以这个数组的引用计数为1；又因为只有数组指向了这些字符串，因此每一个字符串的引用计数也分别是1。

当程序中剩余的两个赋值语句执行完成后，会发生什么呢？Python通过让目标指向与源相同的对象，并增加该对象的引用计数这种方式来实现赋值操作：

<!-- ![alt](/rust-note/ownership-and-moves/pics/python_list_mem2_0406.png) -->
![alt](../pics/python_list_mem2_0406.png)

Python将指针从`s`复制到了`t`和`u`中，并将列表的引用计数更新为3。Python中的赋值操作成本很低，但由于它为对象创建了一个新的引用，所以我们必须维护引用计数，以便知晓何时可以释放该值。

现在我们在C++中观察一下类似的情况：

```c++
using namespace std;
vector<string> s = { "udon", "ramen", "soba" };
vector<string> t = s;
vector<string> u = s;
```

`s`的最初内存分布情况如下图所示：

<!-- ![alt](/rust-note/ownership-and-moves/pics/cpp_vector_mem_0407.png) -->
![alt](../pics/cpp_vector_mem_0407.png)

当赋值语句执行完成，会发生什么呢？在C++中，对于`std::vector`的赋值操作会产生一个复制；`std::string`也有相似行为。因此，在完成赋值语句后，在内存中会分配3个向量以及9个字符串：

<!-- ![alt](/rust-note/ownership-and-moves/pics/cpp_vector_mem2_0408.png) -->
![alt](../pics/cpp_vector_mem2_0408.png)

根据所涉及的值的情况，在C++中进行赋值操作可能会消耗无限制的内存量和处理器时间。不过，其优势在于，程序很容易确定何时释放所有这些内存：当变量离开其作用域时，在此处分配的所有内存都会被自动情况。

从某种意义上来说，C++和Python做出了截然相反的权衡：Python使赋值操作的成本很低，但代价是需要进行引用计数（通常情况下还需要垃圾回收）。C++则让所有内存的所有权清晰明了，代价是赋值操作会对对象进行深拷贝。C++程序员往往不太热衷于这种选择，因为深拷贝的开销可能很大，而且通常还有更多实用的替代方案。

类似的代码，让我们观察一下在Rust中，会是怎么样的：

```rust
let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];
let t = s;
let u = s;
```

与C和C++一样，Rust会将诸如"udon"这样的普通字符串字面量存放在只读内存中，所以，为了能更清晰地与C++及Python的示例进行对比，我们在这里调用`to_string`函数来获取在堆上分配的`String`类型的值。当完成`s`的初始化后，Rust的内存分布情况看起来和C++差不多：

<!-- ![alt](/rust-note/ownership-and-moves/pics/rust_vector_mem_0409.png) -->
![alt](../pics/rust_vector_mem_0409.png)

但要记得，在Rust中，对大多数类型进行赋值操作时，会将值从源处移动到目标处，使得源处变为未初始化状态。那么，在`t`初始化后，会变成这样子：

<!-- ![alt](/rust-note/ownership-and-moves/pics/rust_vector_mem2_0410.png) -->
![alt](../pics/rust_vector_mem2_0410.png)

初始化语句`let t = s;`将向量的三个头字段从`s`移动到了`t`处；现在`t`拥有了这个向量。向量的元素仍然留在原来的位置，字符串也没有发生任何变化。每个值仍然只有一个所有者，尽管所有者已经易主了。这里不需要调整引用计数。而且编译器现在认为`s`处于未初始化状态。

所以最后一个初始化语句`let u = s;`，会将一个未初始化的`s`赋给`u`。Rust谨慎地禁止使用未初始化的值，所以编译器会拒绝这段代码，并给出如下错误提示：

```
error: use of moved value: `s`
  |
7 |     let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];
  |         - move occurs because `s` has type `Vec<String>`,
  |           which does not implement the `Copy` trait
8 |     let t = s;
  |             - value moved here
9 |     let u = s;
  |             ^ value used here after move
```

思考一下Rust在这里使用移动操作所产生的后果。和Python一样，赋值操作的成本很低：程序只是简单地将向量的三个字长的头信息从一处移动到另一处。但又和C++类似，所有权始终清晰明了：程序不需要引用计数或者垃圾回收机制来确定何时释放向量元素以及字符串内容。

你需要付出的代价是，当你想要进行复制时，必须显示地提出复制请求。如果你希望达到和C++程序一样的状态，即每个变量都持有该结构的一个独立副本，那你就必须调用向量的`clone`方法，该方法会对向量及其元素进行深拷贝操作：

```rust
let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];
let t = s.clone();
let u = s.clone();
```

你甚至可以使用Rust的引用计数类型指针（**Rc & Arc**）来重现Python的行为，这会在后面进行讨论。

### More Operations That Move

到目前为止的示例中，我们展示了初始化操作，也就是在`let`语句中，当变量进入其作用域时为它们提供相应的值。但是当你在移动一个值到一个已经初始化过的变量时，会有一些细微的差异，这种情况下，Rust会丢弃变量原有的值，例如：

```rust
let mut s = "Govinda".to_string();
s = "Siddhartha".to_string();  // value "Govinda" dropped here
```

在这段代码中，当程序为`s`赋值"Siddhartha"时，之前的值"Govinda"会被丢弃掉。但是继续考虑下面的代码：

```rust
let mut s = "Govinda".to_string();
let t = s;
s = "Siddhartha".to_string();  // nothing is dropped here
```

这时，`t`会取代`s`获取字符串"Govinda"的所有权，而`s`会成为未初始化状态。因此，在这个场景中，没有任何值被丢弃。

除了初始化操作和赋值操作所演示的简单示例外，Rust关于`move`语义几乎可以用在任何对于值的使用场景上。传递值到一个方法中，会将值的所有权移动到方法参数上；从方法中返回一个值，会将值的所有权移动给调用方。构建一个元组会将值的所有权移动到元组中，等等。

在此，让我们重新回顾一下之前的一个示例：

```rust
struct Person { name: String, birth: i32 }

let mut composers = Vec::new();
composers.push(Person { name: "Palestrina".to_string(),
                        birth: 1525 });
```

这段代码展示了在初始化和赋值操作之外，还会出现移动操作的若干情形：

**在方法中返回一个值**

> 调用`Vec::new()`会构造一个新的向量实例并返回，而不是返回一个向量的指针，就是向量自身：它的所有权从`Vec::new`移动到`composers`变量。类似的，调用`to_string`也会返回一个新的`String`实例。

**构造方法**

> `Person`结构体的`name`字段通过`to_string`方法的返回值进行初始化。这个结构体持有该字符串的所有权。

**方法传参**

> 整个`Person`结构（而非它的指针）被传递给了向量的`push`方法，该方法会将其移动到这个结构的末尾。向量获得了`Person`结构的所有权，因此也就间接地成为了`name`（字符串类型，属于`Person`结构的成员）的所有者。

像这样到处移动值听起来可能效率不高，但有两点需要牢记于心。首先，移动操作始终针对值本身进行的，而非它们所拥有的堆存储，值本身仅仅是三个字长的头信息；那些可能很大的元素数组以及文本缓冲区仍留在堆中的原有位置上。其次，Rust编译器的代码生成机制很擅长“看穿”所有这些移动操作；在实际应用中，机器代码通常会直接将值存储到它所属的位置上。

### Moves and Control Flow

前面的示例都有着非常简单的控制流；那么移动操作与更复杂的代码是如何相互作用的呢？一般原则是，如果一个变量有可能已经被移走了其值，并且自那之后有没有明确被赋予新的值，那么它就被视作未初始化状态。例如，如果一个变量在对`if`表达式的条件进行求值后仍然有值，那么我们就可以在`if`表达式的两个分支中使用它：

```rust
let x = vec![10, 20, 30];
if c {
    f(x);  // ... ok to move from x here
} else {
    g(x);  // ... and ok to also move from x here
}
h(x);  // bad: x is uninitialized here if either path uses it
```

基于相似的原因，在循环中移动变量是不能的：

```rust
let x = vec![10, 20, 30];
while (f) {
    g(x);  // bad: x would be moved in first iteration, uninitialized in second
}
```

除非我们在下一次遍历时赋予它一个新的值：

```rust
let mut x = vec![10, 20, 30];
while (f) {
    g(x);            // move from x
    x = h();         // give x a fresh value
}
e(x);
```

### Moves and Indexed Content

我们说过，移动操作会导致移动源变为未初始化状态，移动目标则会持有值的所有权。但并不是所有的值所有者都已经做好了成为未初始化状态的准备，考虑下面的代码：

```rust
// Build a vector of the strings "101", "102", ... "105"
let mut v = Vec::new();
for i in 101 .. 106 {
    v.push(i.to_string());
}

// Pull out random elements from the vector.
let third = v[2];  // error: Cannot move out of index of Vec
let fifth = v[4];  // here too
```

要实现这一点，Rust需要以某种方式记住向量的第三个和第五个元素已经变为未初始化状态，并跟踪这些信息，直至该向量被销毁。在最普遍的情况下，向量需要随身携带额外的信息，用以表明哪些元素是有效的，哪些已经变成未初始化状态了。但对于一门系统编程语言来说，这显然是不正确的行为；一个向量就应该仅仅只是一个向量而已。而对于上述代码，Rust在编译时会报这样的错误信息：

```
error: cannot move out of index of `Vec<String>`
   |
14 |     let third = v[2];
   |                 ^^^^
   |                 |
   |                 move occurs because value has type `String`,
   |                 which does not implement the `Copy` trait
   |                 help: consider borrowing here: `&v[2]`
```

对于移动`fifth`元素，Rust也是一样的错误提示。在提示信息中，Rust建议你使用引用，以防你想要访问元素但又不想移动它。这种方式基本可以应对大部分场景。但是如果你确实想要从向量中移出一个元素，那该怎么做呢？你需要找到一种能以符合该类型限制的方式来实现此操作的方法。

```rust
// Build a vector of the strings "101", "102", ... "105"
let mut v = Vec::new();
for i in 101 .. 106 {
    v.push(i.to_string());
}

// 1. Pop a value off the end of the vector:
let fifth = v.pop().expect("vector empty!");
assert_eq!(fifth, "105");

// 2. Move a value out of a given index in the vector, and move the last element into its spot:
let second = v.swap_remove(1);
assert_eq!(second, "102");

// 3. Swap in another value for the one we're taking out:
let third = std::mem::replace(&mut v[2], "substitute".to_string());
assert_eq!(third, "103");

// Let's see what's left of our vector.
assert_eq!(v, vec!["101", "104", "substitute"]);
```

上面代码中的每个方法都会将一个元素从向量中移出，不过是以一种能让向量处于填满状态（即便向量可能变小了）的方式来操作的。

Rust中类似`Vec`的集合类型，大多提供了一些方法用于在循环中消费它们的元素：

```rust
let v = vec!["liberté".to_string(), 
             "égalité".to_string(),
             "fraternité".to_string()];

for mut s in v {
    s.push('!');
    println!("{}", s);
}
```

当我们将向量直接传递给循环时，比如`for ... in v`这种形式，这会将向量从`v`中移出，使`v`变为未初始化状态。`for`循环的内部机制会获取该向量的所有权，并将其拆解为各个元素。在每一次迭代时，循环会将另一个元素移动到变量`s`中。由于现在`s`拥有了这个字符串，我们就能在循环体中对其进行修改，然后再将它打印出来。而且由于代码已经无法再看到这个向量本身了，所以在循环过程中，就不会出现有人观察到它处于某种部分被清空状态的情况了。

如果你确实发现自己需要从一个编译器无法追踪的所有者那里移出一个值，那你可以考虑将所有者的类型更改为某种能够动态追踪其是否有值的类型。把我们之前的一个示例做一些调整：

```rust
struct Person { name: Option<String>, birth: i32 }

let mut composers = Vec::new();
composers.push(Person { name: Some("Palestrina".to_string()),
                        birth: 1525 });
```

你不可以这么做：

```rust
let first_name = composers[0].name;
```

这会引发前面展示过的那种“cannot move out of index”的错误，不过，由于你已经将`name`字段的类型从`String`改为`Option<String>`，这意味着`None`是该字段可以持有的合法值，所以这样做是可行的：

```rust
let first_name = std::mem::replace(&mut composers[0].name, None);
assert_eq!(first_name, Some("Palestrina".to_string()));
assert_eq!(composers[0].name, None);
```

调用`replace`会将`composers[0].name`移出，在原处留下`None`，然后将原始值的所有权传递调用者。实际上，以这种方式使用`Option`类型是很常见的，以至于该类型专门为此提供了一个`take`方法。你可以将前面的操作写得更清晰易读，如下所示：

```rust
let first_name = composers[0].name.take();
```

调用`take`的效果与前面的`replace`一样。

### Copy Types: The Exception to Moves

到目前为止，我们展示的关于“移动”的示例包含向量、字符串以及其它可能会占用大量内存且复制成本较高的类型。移动操作能让这类类型的所有权清晰明了，并且使赋值操作成本低廉。但对于像整数或字符这样更简单的类型来说，这种谨慎地处理方式其实并非必要。

接下来我们比较一下分配`String`类型和`i32`类型在内存表现形式上有什么区别：

```rust
let string1 = "somnambulance".to_string();
let string2 = string1;

let num1: i32 = 36;
let num2 = num1;
```

代码执行后，内存表现如下：

<!-- ![alt](/rust-note/ownership-and-moves/pics/str_vs_i32_mem_0411.png) -->
![alt](../pics/str_vs_i32_mem_0411.png)

和前面的向量情况一样，赋值操作会将`string1`移动到`string2`，这样我们就不会出现两个`string`变量都负责释放同一个缓冲区的情况。然而，`num1`和`num2`的情况则有所不同。一个`i32`类型的值仅仅是内存中的一组比特位模式；它不拥有任何堆资源，并且除了它自身包含的字节之外，实际上并不依赖其它任何东西。当我们把它的比特位移动到`num2`时，就已经制作出了一个`num1`完全独立的副本。

移动一个值会使移动操作的源变为未初始化状态。不过，虽然将`string1`视作无值状态有着重要意义，但像这样对待`num1`却毫无意义；继续使用`num1`并不会造成什么危害。移动操作的优势在这里并不适用，而且还很不方便。

之前我们提到，大多数类型在赋值等操作时是被移动的；现在我们要说一下例外情况了，也就是那些被Rust指定为“可复制（Copy）”的类型。在给一个“可复制（Copy）”类型的变量赋值时，会赋值该值，而非移动它。进行赋值操作的源变量依然保持已初始化状态且可以使用，其值与赋值前相同。将“可复制（Copy）”类型的值传递给函数和构造函数时，行为也是类似的。

标准的`Copy`类型包含所有的机器整数类型、浮点型数值类型、`char`类型和`bool`类型，以及其它一些类型。有`Copy`类型组成的元组或固定大小的数组，其本身也是`Copy`类型。

只有那些进行简单的逐位复制就足够的类型才能是`Copy`类型。正如我们已经解释过的，`String`类型不是`Copy`类型，因为它拥有一个在堆上分配的缓冲区。出于类似的原因，`Box<T>`类型也不是`Copy`类型，它拥有其在堆上分配的被引用对象。`File`类型（代表操作系统中的文件句柄）也不是`Copy`类型，复制这样一个值意味着要向操作系统再请求一个文件句柄。同样的，`MutexGuard`类型（代表一个已锁定的互斥锁）也不是`Copy`类型：复制这个类型根本没有意义，因为在同一时间只有一个线程可以持有一个互斥锁。

凭经验来说，任何在值被销毁时需要执行特殊操作的类型都不能是`Copy`类型：比如`Vec`需要释放它的元素，`File`需要关闭它的文件句柄，`MutexGuard`需要解锁它的互斥锁，等等。对这类类型进行逐位复制会导致不清楚现在哪个值该对原始值的资源负责。

那么对于你自己定义的类型呢？默认情况下，`struct`和`enum`不是`Copy`类型：

```rust
struct Label { number: u32 }

fn print(l: Label) { println!("STAMP: {}", l.number); }

let l = Label { number: 3 };
print(l);
println!("My label number is: {}", l.number);
```

这段代码会编译失败：

```
error: borrow of moved value: `l`
   |
10 |     let l = Label { number: 3 };
   |         - move occurs because `l` has type `main::Label`,
   |           which does not implement the `Copy` trait
11 |     print(l);
   |           - value moved here
12 |     println!("My label number is: {}", l.number);
   |                                        ^^^^^^^^
   |                  value borrowed here after move
```

由于`Label`类型不是`Copy`类型，将它传递给`print`函数时，就会把该值的所有权移交给`print`函数，然后`print`函数在返回前会销毁这个值。但这很荒谬，`Label`只不过是一个自视甚高（有点特殊含义）的`u32`类型罢了。没有理由说把`l`传递给`print`函数就应该移动这个值。

不过，用户自定义类型默认是非`Copy`类型，这只是一种默认情况。如果你的结构体的所有字段本身都是`Copy`类型，那么你可以通过在结构体定义的上方添加`#[derive(Copy, Clone)]`属性，使该结构体类型也成为`Copy`类型：

```rust
#[derive(Copy, Clone)]
struct Label { number: u32 }
```

这样调整后，代码就可以通过编译正常运行了。但是，如果你的结构体并不是所有的成员属性都是`Copy`类型的，那么这种方式还是不行：

```rust
#[derive(Copy, Clone)]
struct StringLabel { name: String }
```

它会引起这样的报错：

```
error: the trait `Copy` may not be implemented for this type
  |
7 | #[derive(Copy, Clone)]
  |          ^^^^
8 | struct StringLabel { name: String }
  |                      ------------ this field does not implement `Copy`
```

为什么在符合条件的情况下，用户自定义类型不会自动成为`Copy`类型呢？一个类型是否为`Copy`类型，对代码能够如何使用它有着重大影响：`Copy`类型更加灵活，因为赋值及相关操作不会使原始值变成未初始化状态。但对于类型的实现者来说，情况恰恰相反：`Copy`类型在能够包含的类型方面有着很大限制，而非`Copy`类型则可以使用堆分配并拥有其它类型的资源。所以，将一个类型定义为`Copy`类型，对于实现者而言意味着一项重大的约束：如果日后有必要将其改为非`Copy`类型，那么使用该类型的大量代码很可能都需要进行调整。

虽然C++允许你重载赋值运算符，并定义专门的复制构造函数和移动构造函数，但Rust不允许这种定制化操作。在Rust中，每一次移动操作都是逐字节的浅复制，会时源对象变为未初始化状态。复制操作也是一样的情况，只不过源对象依然保持已初始化状态。这确实意味着C++类能够提供一些方便的接口，而Rust类型则无法提供，在C++中，那些看起来普通的代码能够隐式地调整引用计数、将成本较高的复制操作推迟到后续进行，或者使用其它复杂的实现技巧。

但这种灵活性对C++这门语言产生的影响是，使得像赋值、传递参数以及从函数返回值这类基本操作变得更不可预测了。例如，前面我们介绍过，在C++中把一个变量赋值给另外一个变量可能会需要任意数量的内存以及处理器时间。Rust的原则之一就是，各项成本应该对程序员来说是显而易见的。基本操作必须保持简单。那些可能会产生较高成本的操作应该是显示的，就像前面示例中对`clone`函数的调用那样，它会对向量及其包含的字符串进行深复制。

### Rc and Arc: Shared Ownership

尽管在典型的Rust代码中，大多数值都有唯一的所有者，但在某些情况下，很难为每个值都找到一个拥有你所需生命周期的单一所有者；你会希望这个值能一直存在，直到所有人都用完它为止。针对这些情况，Rust提供了引用计数指针类型`Rc`和`Arc`。正如你对Rust的预期那样，使用这些类型是完全安全的：你不会忘记去调整引用计数，不会创建出其它一些Rust无法察觉的指向被引用对象的指针，也不会遇到在C++中伴随引用计数指针类型出现的其它各类问题。

`Rc`和`Arc`这两种类型非常相似；它们之间唯一的区别在于，`Arc`可以直接在多个线程之间安全地共享 —— `Arc`这个名称是“原子引用计数（Atomic Reference Count）”的缩写 —— 而普通的`Rc`则使用速度更快的非线程安全代码来更新其引用计数。如果你不需要在线程之间共享指针，那就没有理由去承受使用`Arc`带来的性能损失，所以你应该使用`Rc`；Rust会防止你意外地将`Rc`类型的指针跨线程边界传递。除此以外，这两种类型是等效的，所以我们接下来只讨论`Rc`类型。

之前我们看过Python使用引用计数的方式来管理值的生命周期，在Rust中使用`Rc`可以达到类似的效果：

```rust
use std::rc::Rc;

// Rust can infer all these types; written out for clarity
let s: Rc<String> = Rc::new("shirataki".to_string());
let t: Rc<String> = s.clone();
let u: Rc<String> = s.clone();
```

对于任意类型`T`，一个`Rc<T>`值是指向堆上分配的`T`的指针，并且这个`T`上附加了一个引用计数。克隆一个`Rc<T>`值并不会复制`T`；相反，它只是简单地创建出另一个指向`T`的指针，并增加引用计数。上述代码的内存表示形式为：

<!-- ![alt](/rust-note/ownership-and-moves/pics/rc_mem_0412.png) -->
![alt](../pics/rc_mem_0412.png)

这三个`Rc<String>`指针都引用了同一片内存区域，这个内存保存着一个引用计数和`String`类型的成员。通常的所有权规则适用于这些`Rc`指针自身，并且当最后一个现存的`Rc`指针被销毁时，Rust也会销毁对应的`String`字符串。

你可以直接在`Rc<String>`上使用`String`类型的任何常用方法：

```rust
assert!(s.contains("shira"));
assert_eq!(t.find("taki"), Some(5));
println!("{} are quite chewy, almost bouncy, but lack flavor", u);
```

`Rc`指针所有的值是不可变的。假如你尝试在字符串的最后增加一些文本：

```rust
s.push_str(" noodles");
```

Rust会拒绝：

```
error: cannot borrow data in an `Rc` as mutable
   |
13 |     s.push_str(" noodles");
   |     ^ cannot borrow as mutable
   |
```

Rust的内存和线程安全保障依赖于确保任何值都不会同时处于被共享且可被修改的状态。Rust假定`Rc`指针所指向的对象通常可能会被共享，所以它（所指向的对象）绝不能是可变的。在将来学习“引用”相关只是时，可以发现这个限制的重要性。

使用引用计数来管理内存的一个众所周知的问题是，如果存在两个引用计数类型的值互相指向对方，那么每个值都会使对方的引用计数保持在大于零的状态，这样一来，这些值就永远不会被释放了。

<!-- ![alt](/rust-note/ownership-and-moves/pics/reference_count_loop_0413.png) -->
![alt](../pics/reference_count_loop_0413.png)

在Rust中，确实有可能以这种方式造成值无法被释放（内存泄露），但这种情况很罕见。如果不通过某种方式让一个旧值指向一个新值，是无法创建出循环引用的。显然，这需要旧值是可变的。由于`Rc`指针所指向的对象是不可变的，通常情况下就不可能创建出循环引用。不过，Rust确实提供了一些方法来为原本不可变的值创建可变部分，这被称为内部可变性（**interior mutability**），将来会专门讲解。如果你将这些计数与`Rc`指针结合起来使用，就有可能创建出循环引用并导致内存泄露。

有时候你可以使用弱指针（`std::rc::Weak`）来替代部分链接，以避免创建`Rc`指针的循环引用情况。具体请自行查看标准库文档。

移动操作和引用计数指针是两种放松所有权树严格限制的方式。在下一章中，我们会讨论第三种方式：借用对值的引用。一旦你对所有权和借用这两方面都能熟练掌握了，那你就已经翻过了Rust学习曲线中最陡峭的部分，并且也准备好去利用Rust独特的优势了。
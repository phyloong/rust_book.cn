# 使用 Packages、Crates、Modules 管理项目

[TOC]

在写大型程序时，如何组织代码变得更加重要。通过组织相关联的功能，根据不同特性划分代码，可以清晰地找到实现某一特性的代码位置，并更改其工作方式。

到现在为止，我们写的程序都被放在一个文件的一个 module 中。随着项目的发展，你应该通过划分多个 module，乃至多个文件的方式组织代码。一个 package 可以包含多个二进制 crate 和一个可选的库 crate。随着 package 的发展，你可以提取各部分到独立的 crate 中，从而成为外部依赖项。本章节覆盖了所有这些技术。对于由一组相互关联的 package 组成的非常大的项目，Cargo 提供了工作区（workspace），我们将在第 14 章的“Cargo 工作区”一节中介绍。

我们同样讨论封装的实现细节，这可以让你在一个更高的层面上复用代码：一旦你实现了一个操作，其他代码可以通过公共接口调用你的代码，而不必知道具体如何实现的。这种写代码的方式定义了哪部分是公开的从而可以被其他代码使用，哪部分是私有的实现细节从而保留了更改的权力。这是限制你必须记住的细节数量的方式

一个相关的概念是作用域：代码写在其中的嵌套上下文，有一系列定义为作用域中的名字。当读、写、编译代码时，程序员和编译器需要知道一个特定位置的一个特定名字是引用了变量、函数、结构体、枚举、module、常量，还是其他项，以及该项的含义。你可以创建作用域，改变作用域内外的名字。你不能在同一个作用域中有两个名字相同的项，可以使用工具来解决命名冲突。

Rust 有许多特性，允许你管理代码的组织结构，包：你的程序中哪些细节是暴露出来的，哪些细节是私有的，以及每个作用域中有哪些名字。这些特性又是统称为模块系统，包括：

- Packages：Cargo 的特性，允许你构建、测试、分享 crate；
- Crates：可以生成一个库或可执行程序的模块树；
- Modules 和 use：让你可以控制组织结构、作用域和 path 的私有性；
- Paths：一种命名一个项的方式，比如结构体、函数、module。

在本章中，我们将覆盖所有这些特性，讨论他们如何交互、如何使用他们管理作用域。在最后，你应该对模块系统有深入的理解，并可以像专家一样使用作用域来工作。

## Packages 和 Crates

我们首先来了解模块系统的第一个部分：packages 和 crates。

crate 是 Rust 编译器一次所能顾及到的最小数量的代码。即使你运行 rustc 而不是 cargo，同时传递一个源码文件，编译器也会认为这个文件是一个 crate。crate 可以包含 module，同时 module 可以定义在与该 crate 一同编译的其他文件中，就像我们将在接下来的章节中看到的一样。

crate 有两种形式：二进制 crate 或者库 crate。二进制 crate 是你可以编译为可执行文件的程序，比如命令行程序或者服务器。每个二进制 crate 必须有一个命名为 main 的函数，它定义了可执行程序运行时会发生什么。到现在为止我们创建的所有 crate 都是二进制 crate。

库 crate 没有 main 函数，他们也不编译为可执行程序。取而代之的是，他们定义了旨在多个项目中共享的功能。例如，我们在第二章中用过的 rand crate 提供了生成随机数的功能。绝大多数时候 Rustacean 谈及 crate 时，均意味着库 crate，他们用 crate 与一般程序中库的概念进行互换。

crate root 是 Rust 编译器开始的源文件，它构成了你的 crate 中的根 module（我们会在“定义 Module 来控制作用域和私有性”一节中深入解释 module）。

package 是一个或多个提供一系列功能的 crate 的集合。一个 package 包含了一个 `Cargo.toml` 文件，该文件描述了如何编译这些 crate。Cargo 就是一个包含了用于构建代码的命令行工具的二进制 crate 的 package。Cargo package 同样包含了二进制 crate 以来的库 crate。其他项目可以依赖 Cargo 的库 crate 以使用和 Cargo 命令行工具一样的逻辑代码。

一个 package 可以包含任意多个二进制 crate，但是只能有一个库 crate。一个 package 必须包含至少一个 crate，无论它是二进制 crate 还是库 crate。

让我们看一下当我们创建一个 package 时发生了什么。首先，我们键入命令 `cargo new`：

```shell
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

当我们运行 `cargo new` 之后，使用 `ls` 看一下 Cargo 创建了什么。在项目文件夹中，有一个 `Cargo.toml` 文件，给了我们一个 package。还有一个包含了 `main.rs` 文件的 `src` 文件夹。在你的编辑器中打开 `Cargo.toml` 文件，注意到并没有提及 `src/main.rs`。Cargo 遵循这样的约定：`src/main.rs` 是与 package 相同名字的二进制 crate 的 crate root。同样地，如果一个 package 文件夹中包含 `src/lib.rs`，Cargo 知道这个 package 包含了一个与 package 同名的库 crate，`src/lib.rs` 就是它的 crate root。Cargo 传递 crate root 文件给 rustc 从而编译这个库或者二进制。

这里，我们有一个只包含了 `src/main.rs` 的 package，这意味着该 package 只包含有一个名为 `my-project` 的二进制 crate。如果一个 package 同时包含 `src/main.rs` 和 `src/lib.rs`，那它有两个 crate：一个二进制 crate 和一个库 crate，都与这个 package 同名。一个 package 可以通过在 `src/bin` 文件夹中放多个文件的方式包含多个二进制 crate：每个文件都是一个独立的二进制 crate。

## 定义 Module 来控制作用域和私有性

在本节中，我们将谈论 module 和模块系统其他部分：允许你命名各个项的 path；将某个 path 带入作用域中的 use 关键字；以及是的各个项公开的 pub 关键字。我们还会讨论 as 关键字，外部 package 和 glob 操作符。

首先，我们从一个规则列表开始，在你将来组织代码的时候能够轻松参考。然后我们会解释每个规则的细节。

### Module 小抄

这里我们提供一个快速参考，关于 module、path、use 关键字、pub 关键字在编译器中的工作方式，以及大多数的开发者如何组织他们的代码。我们将在本章中逐一介绍这些规则的例子，这也是一个提示 module 工作方式以用于参考的好地方。

- 从 crate root 开始：当编译一个 crate 时，编译器首先在 crate root 文件（通常来说，库 crate 是 `src/lib.rs`，二进制 crate 是 `src/main.rs`）中查找代码去编译。
- 声明 module：在 crate root 文件中，你可以声明一个新的 module，也就是说，你可以通过 `mod graden;` 来声明一个“garden”module。编译器会在以下这些地方查找这个 module 的代码：
  - 内联，在取代 `mod graden` 后的分号的花括号中。
  - 在 `src/garden.rs` 文件中。
  - 在 `src/garden/mod.rs` 文件中。
- 声明子 module：在除 crate root 以外的任意文件中，你可以声明子 module。例如：你可以在 `src/garden.rs` 中声明 `mod vegetables;`。编译器将在父 module 的文件夹中的以下这些地方查找子 module 的代码：
  - 内联，直接跟在 `mod vegetables` 后取代掉分号的的花括号内。
  - 在 `src/garden/vegetables.rs` 文件中。
  - 在 `src/garden/vegetables/mod.rs` 文件中。
- module 中代码的 path：一旦一个 module 是你 crate 中的一部分，你可以在同一个 crate 中的其他任意地方引用该 module 中的代码，只要私有性规则允许，使用指向代码的 path。例如，在 garden vegetables module 中的一个 Asparagus 类型可以通过 `crate::garden::vegetables::Asparagus` 找到。
- 私有和公开：一个 module 中的代码，对于它的父 module 来说默认是私有的。在声明该 module 时使用 `pub mod` 而不是 `mod` 可以使得它公开。要让一个公开 module 中的项也变得公开，可以在它们的声明前使用 `pub`。
- `use` 关键字：在一个作用域中，`use` 关键字创建了项的快捷方式从而减少长 path 的重复。在可以引用到 `crate::garden::vegetables::Asparagus` 的任意作用域中，可以使用 `use crate::garden::vegetables::Asparagus;` 创建一个快捷方式，从而只需要写 `Asparagus` 就可以在作用域中使用这个类型。

在此我们创建一个叫做 backyard 的二进制 crate 来阐明这些规则。该 crate 的文件夹，同样叫 backyard，包含了这些文件和文件夹：

```
backyard
├── Cargo.lock
├── Cargo.toml
└── src
    ├── garden
    │   └── vegetables.rs
    ├── garden.rs
    └── main.rs

```

在这里 crate root 文件为 `src/main.rs`，它包含：

```rust
// Filename: src/main.rs
use crate::garden::vegetables::Asparagus;

pub mod garden;

fn main() {
    let plant = Asparagus {};
    println!("I'm growing {:?}!", plant);
}
```

`pub mod garden;` 行告诉编译器要包含在  `src/garden.rs` 中的代码，其为：

```rust
// Filename: src/garden.rs
pub mod vegetables;
```

这里，`pub mod vegetables;` 意味着 `src/garden/vegetables.rs` 中的代码也要被包含。代码为：

```rust
// Filename: src/garden/vegetables.rs
#[derive(Debug)]
pub struct Asparagus {}
```

现在让我们深入这些规则的细节，并用行动来进行说明。

### 在 module 中组织相关代码

module 让我们在一个 crate 中组织代码以提高可读性，更易复用。module 还允许我们控制各项的私有性，因为 module 中的代码默认是私有的。私有项是内部的实现细节，不可以在外部使用。我们可以选择让 module 和它们其中的项公开，暴露出来以允许外部代码使用和依赖他们。

作为一个例子，我们来写一个库 crate，提供一个餐厅相关的功能。我们将定义函数签名但对他们的函数体留空，以聚焦代码的组织而不是具体实现。

在餐厅行业中，一个餐厅有被称作前厅（front of house）和后场（back of house）的两部分。前厅中主要是顾客，包含了安置顾客、服务员接受订单和付款，以及调酒师制作饮料的地方。后场是大厨和厨师在厨房工作，洗碗机清理，和经理做行政工作的地方。

用这种方式组织我们的 crate，我们可以组织函数到嵌套 module 中。运行 `cargo new restaurant --lib` 来创建一个新的叫做 `restaurant` 的库；然后在 `src/lib.rs` 中键入 Listing 7-1 中的代码来定义一些 module和函数签名。这是前厅部份：

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

▲ Listing 7-1：`front_of_house` module 包含其他 module，这些 module 随后包含了函数。

我们用 `mod` 关键字，后面跟着这个 module 的名字（这里是 `front_of_house`）定义了一个 module。这个 module 的主体在后面的花括号中。在 module 中，我们可以放其他的 module，这里是 module `hosting` 和 `serving`。module 中还能有其他项的定义，比如 struct，enum，常量，trait，以及如同在 Listing 7-1 在的函数。

通过使用 module，我们可以组织相关的定义到一起，并命名他们相关的原因。使用这些代码的程序员可以基于组导航到这些代码，而不是不需要阅读所有的定义，这可以使得查找与此相关的代码更为简单。向这些代码中添加新功能的程序员能知道将代码放在哪里能够继续保持程序的组织性。

先前我们提到 `src/main.rs` 和 `src/lib.rs` 被叫做 crate root。它们如此命名的原因是这两个文件中的内容组织了一个叫做 `crate` 的 module 作为这个 crate 的 module 结构体的根，被称为模块树。

Listing 7-2 展示了 Listing 7-1 中的代码结构的模块树。

```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

▲ Listing 7-2：Listing 7-1 中的代码的模块树。

这棵树展示了 module 中的一部分如何潜逃到另一个中。例如，`hosting` 嵌套在 `front_of_house` 中。这棵树同样展示了一些 module 是兄弟模块，这意味着他们定义在同一个 module 中，`hosting` 和 `serving` 是定义在 `front_of_house` 中的兄弟模块。如果 module A 包含在 module B 中，我们说 module A 是 module B 的孩子，module B 是 module A 的双亲。注意：整个模块树以一个叫做 `crate` 的隐式 module 为根。

模块树可能让你想起来了你电脑上的文件系统的文件夹树，这是一个非常恰当的比较！就像文件系统中的文件夹，你可以使用 module 来组织你的代码。就像文件夹中的文件一样，我们需要一种找到我们的 module 的方法。

## 在模块树中使用 path 来引用项

为了展示 Rust 在模块树中查找项的位置，我们使用 path，这与导航文件系统时使用路径的方式相同。想要调用一个函数，我们需要知道它的 path。

path 可以有以下两种形式：

- 绝对路径，从 crate root 开始的完整路径；对于来自外部 crate 的代码，绝对路径以该 crate 的名字开头，对于当前 crate 中的代码，以字面值 `crate` 开头。
- 相对路径，从当前 module 开始，使用 `self`，`super`，或者当前 module 的标识符。

绝对路径和相对路径后面都跟着一个或多个以双冒号（`::`）分割的标识符。

再回到 Listing 7-1，假设我们想要调用函数 `add_to_waitlist`。这就是在问：函数 `add_to_waitlist` 的 path 是什么？Listing 7-3 包含了 Listing 7-1 中的一些 module 和函数。

我们演示从一个定义在 crate root 中的新函数 `eat_at_restaurant` 来调用函数 `add_to_waitlist` 的两种方式。这些 path 都是正确的，但是有另一个问题会阻止这个例子正常编译。我们稍后会解释原因。

函数 `eat_at_restaurant` 是我们库 crate 的公开 API 的一部分，所以我们使用 `pub` 关键字来标记它。在 “使用 pub 关键字暴露 path”一节中，我们会深入 `pub` 的细节。

```rust
// Filename: src/lib.rs
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

▲ Listing 7-3：使用绝对路径和相对路径调用函数 `eat_at_restaurant`。

我们第一次在 `eat_at_restaurant` 中调用 `add_to_waitlist` 时，使用了绝对路径。函数 `add_to_waitlist` 与 `eat_at_restaurant` 定义在同一个 crate 中，这意味着我们可以使用 `crate` 关键字作为绝对路径的开头。接下来我们包含每一个连续的 module 直到我们访问到 `add_to_waitlist`。你可以想象一个拥有同样结构的文件系统：我们指定路径 `/front_of_house/hosting/add_to_waitlist` 以运行 `add_to_waitlist` 程序；使用名字 `crate` 以从 crate root 开始，就像在 shell 中使用 `/` 以从文件系统的根开始一样。

我们第一次在 `eat_at_restaurant` 中调用 `add_to_waitlist` 时，使用了相对路径。该 path 以 `front_of_house` 开始，它是定义在模块树中与 `eat_at_restaurant` 同一级的 module 的名字。这里，在文件系统中的等价操作是使用路径 `front_of_house/hosting/add_to_waitlist`。以一个 module 的名字作为开始的 path 是相对的。

选择使用相对或绝对路径是你需要根据你的项目来做的决定，且取决于你是否更有可能将项的定义代码与使用该项的代码分开移动，还是一起移动。例如，如果我们移动 `front_of_house` module 和 `eat_at_restaurant` 函数到一个叫做 `customer_experience` 的 module 中，我们需要更新 `add_to_waitlist` 的绝对路径，但是它的相对路径仍然是合法的。然而，如果我们单独移动函数 `eat_at_restaurant` 到名为 `dining` 的module，`add_to_waitlist` 的绝对路径仍然是一样的，但是相对路径就需要更新了。通常我们更倾向于指定绝对路径，因为我们更有可能想要独立地移动代码定义和项的调用。

让我们尝试编译 Listing 7-3，找出无法编译的原因。我们得到的错误在 Listing 7-4 中。

```shell
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0603]: module `hosting` is private
 --> src/lib.rs:9:28
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                            ^^^^^^^ private module
  |
note: the module `hosting` is defined here
 --> src/lib.rs:2:5
  |
2 |     mod hosting {
  |     ^^^^^^^^^^^

error[E0603]: module `hosting` is private
  --> src/lib.rs:12:21
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                     ^^^^^^^ private module
   |
note: the module `hosting` is defined here
  --> src/lib.rs:2:5
   |
2  |     mod hosting {
   |     ^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
error: could not compile `restaurant` due to 2 previous errors
```

▲ Listing 7-4：构建 Listing 7-3 中的代码时的编译错误。

报错信息提示 module `hosting` 是私有的。换句话说，我们有 module `hosting` 和函数 `add_to_waitlist` 的正确 path，但是 Rust 不让我们使用，因为没有私有部分的访问权限。在 Rust 中，所有的项（函数、方法、结构体、枚举、module、常量）默认情况下在父 module 是私有的。如果你想让一个项比如函数或者结构体变成私有的，可以将它放入 module 中。

父 module 中的项不能使用子 module 中的私有项，但是子 module 中的项可以使用他们祖先 module 中的私有项。这是因为子 module 包装并隐藏了它们的实现细节，但是子 module 可以看到它们所被定义的上下文。继续我们的比喻，将这些私有性规则想作餐厅的后台：里面发生了什么对于餐厅顾客来说是私有的，但是经理们可以在工作的餐厅中看到并做任何事情。

Rust 选择具有这种模块系统功能，以便默认隐藏内部实现细节。这样，你会知道能够修改内部代码中的哪部分而不会破坏外部代码。然而，Rust 同样给你暴露子 module 的代码的内部部分到外部祖先 module 的选项，只要使用 `pub` 关键字使得一个项变得公开。

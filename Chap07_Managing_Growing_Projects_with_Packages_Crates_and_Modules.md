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

### 使用 pub 关键字暴露 path

我们回到 Listing 7-4 中的 `hosting` module 是私有的的报错。我们想让父 module 中的 `eat_at_restaurant` 函数能够访问子 module 中的 `add_to_waitlist` 权限，所以我们使用 `pub` 关键字标记 `hosting` module，如 Listing 7-5 所示。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
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
▲ Listing 7-5：使用 `pub` 声明 `hosting` module 以在 `eat_at_restaurant` 中使用。

不幸的是，Listing 7-5 中的代码仍然有错误，如 Listing 7-6 所示。

```shell
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0603]: function `add_to_waitlist` is private
 --> src/lib.rs:9:37
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                                     ^^^^^^^^^^^^^^^ private function
  |
note: the function `add_to_waitlist` is defined here
 --> src/lib.rs:3:9
  |
3 |         fn add_to_waitlist() {}
  |         ^^^^^^^^^^^^^^^^^^^^

error[E0603]: function `add_to_waitlist` is private
  --> src/lib.rs:12:30
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                              ^^^^^^^^^^^^^^^ private function
   |
note: the function `add_to_waitlist` is defined here
  --> src/lib.rs:3:9
   |
3  |         fn add_to_waitlist() {}
   |         ^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
error: could not compile `restaurant` due to 2 previous errors
```

▲ Listing 7-6：构建 Listing 7-5 中的代码时的编译错误。

发生了什么？在 `mod hosting` 前加上 `pub` 关键字使得该 module 公开。随着这点改变，如果我们可以访问 `front_of_house`，我们就可以访问 `hosting`。但是 `hosting` 中的内容仍是私有的，使该 module 公开并不会使它其中的内容公开。module 上的 `pub` 关键字仅仅使得它祖先 module 中的代码可以引用它，而不能访问它其中的代码。因为 module 是容器，我们只能公开这个 module，我们需要更进一步，选择该 module 中的一个或多个项，把它们也公开。

Listing 7-6 中的错误显示函数 `add_to_waitlist` 是私有的。私有性规则适用于结构体、枚举、函数、方法，以及 module。

通过在函数 `add_to_waitlist` 的定义前添加 `pub` 关键字来公开它，如 Listing 7-7 所示。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```
▲ Listing 7-7：为 `mod hosting` 和 `fn add_to_waitlist` 添加 `pub` 关键字，这样我们可以从 `eat_at_restaurant` 中访问该函数。

现在代码可以编译了。让我们从私有性规则的视角来看一下为什么添加 `pub` 关键字就可以在 `add_to_waitlist` 中使用这些 path，让我们看看绝对路径和相对路径。

在绝对路径中，我们以 `crate` 开始，它是我们 crate 的模块树的根。`front_of_house` module 定义在该 crate root 中。当 `front_of_house` 非公开时，因为 `eat_at_restaurant` 函数与 `front_of_house` 定义在同一个 module 中（也就是说，`eat_at_restaurant` 和 `front_of_house` 是兄弟），我们可以在 `eat_at_restaurant` 中引用它。接下来 `hosting` module 标记为 `pub`。我们可以访问 `hosting` 的父 module，所以我们可以访问 `hosting`。最终，函数 `add_to_waitlist` 也被标记为 `pub`，同时我们可以访问它的父 module，所以这个函数可以正常工作。

在相对路径中，逻辑与绝对路径的相同，除了第一步：并非从 crate root 开始，该 path 从 `front_of_house` 开始。`front_of_house` module 与 `eat_at_restaurant` 定义在同一个 module 中，所以从 `eat_at_restaurant` 定义其中的 module 开始的相对路径可以正常工作。然后，因为 `hosting` 和 `add_to_waitlist` 均被标记为 `pub`，该 path 的剩余部分也可以正常工作，最终该函数调用是合法的。

如果你计划分享你的库 crate 以让其他项目能够使用你的代码，你的公开 API 就是你与你的 crate 的用户的约定，它决定了他们如何与你的代码交互。关于管理对公开 API 的更改，有许多需要考虑的事项，以使人们更容易依赖于你的 crate。这些考虑事项不在本书范围，如果你对这个主题感兴趣，可参阅 [Rust API 指南](https://rust-lang.github.io/api-guidelines/)。

> **同时有一个二进制和库的 package 的最佳实践**
>
> 我们提到一个 package 可同时包含一个 `src/main.rs` 的 crate root 和一个 `src/lib.rs` 的 crate root，且两个 crate 默认都有该 package 名。通常，这种同时包含一个库和二进制 crate 的 package 会在二进制 crate 中有刚好足够的代码来启动一个可执行程序，该程序调用库 crate 中的代码。这可以让其他项目从该 package 提供的大部分功能中受益，因为库 crate 中的代码可以被共享。
>
> 模块树应当被定义在 `src/lib.rs` 中。这样，任何公开的项都可以被二进制 crate 通过从 package 名字开始的 path 使用。二进制 crate 成了库 crate 的用户，就像是一个完全外部的 crate 使用该库  crate：只能使用公开 API。这帮助你去设计良好的 API；你不仅仅是作者，你还是客户。
>
> 在[第 12 章](https://doc.rust-lang.org/book/ch12-00-an-io-project.html)中，我们将会通过一个同时包含二进制 crate 和库 crate 的命令行程序演示这种组织实践

### 以 super 开始的相对路径

我们可以通过在 path 的开始处，用 super 构造一个以父 module 开始的相对路径，而不是以当前 module 或 crate root 开始。就像以 `..` 语法开始的文件系统路径。使用 `super` 允许我们引用一个我们已知在父 module 中的项，这使得重组模块树时，当一个 module 与它的父 module 紧密相关，但是父 module 某天可能被移动到模块树中的其他地方时，变得更为简单。

考虑 Listing 7-8 中的代码，该代码模拟了厨师修复错误订单并亲自将其提交给客户的情况。函数 `fix_incorrect_order` 定义在 `back_of_house` module 中，通过以 `super` 开始的 path 来调用定义在父 module 中的 `deliver_order` 函数：

```rust
// Filename: src/lib.rs
fn deliver_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::deliver_order();
    }

    fn cook_order() {}
}
```

▲ Listing 7-8：使用以 `super` 开始的相对路径调用函数。

函数 `fix_incorrect_order` 在 `back_of_house` module 中，所以我们可以使用 `super` 转到 `back_of_house` 的父 module 中，此时是 `crate`，它是根。从那里，我们查找 `deliver_order` 并且找到了。成功！我们认为，如果我们决定重新组织 crate 的模块树， `back_of_house` module 和 `deliver_order` 函数很可能与彼此保持相同的关系并一同移动。因此，我们使用 `super`，从而将来如果这些代码被移动到不同 module 中时，要更新代码的地方更少。

### 公开结构体和枚举

我们还能用 `pub` 将结构体和枚举指定为公开的，但是关于 `pub` 在结构体和枚举上的使用，这里有一些额外的细节。如果我们在结构体的定义前使用了 `pub`，则会将该结构体公开。我们可以视情况使每个字段公开或不公开。在 Listing 7-9 中，我们定义了一个公开的 `back_of_house::Breakfast` 结构体，它有一个公开的 `toast` 字段和一个私有的 `seasonal_fruit` 字段。这模拟了餐厅中顾客可以选择一餐中的面包类型，但是厨师根据当季和库存决定一餐中配哪种水果的情况。可用的水果变化很快，所以顾客不能选择水果，或者甚至看不到他们会得到哪种水果。

```rust
// Filename: src/lib.rs
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    // Order a breakfast in the summer with Rye toast
    let mut meal = back_of_house::Breakfast::summer("Rye");
    // Change our mind about what bread we'd like
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);

    // The next line won't compile if we uncomment it; we're not allowed
    // to see or modify the seasonal fruit that comes with the meal
    // meal.seasonal_fruit = String::from("blueberries");
}
```

▲ Listing 7-9：某些字段公开，某些字段私有的结构体。

因为结构体 `back_of_house::Breakfast` 中的 `toast` 字段是公开的，所以在 `eat_at_restaurant` 中我们可以使用点符号读写 `toast` 字段。注意我们不能在 `eat_at_restaurant` 中使用 `seasonal_fruit` 字段，因为 `seasonal_fruit` 是私有的。尝试解除修改 `seasonal_fruit` 字段值这一行的注释，看你会得到什么错误！

同样地，因为 `back_of_house::Breakfast` 有一个私有字段，该结构体需要提供一个公开的关联函数以构造一个 `Breakfast` 的实例（这里我们为其命名为 `summer`）。如果 `Breakfast` 没有这样的函数，我们将无法在 `eat_at_restaurant` 中创建 `Breakfast` 的实例，因为我们无法在 `eat_at_restaurant` 中设置私有字段 `seasonal_fruit` 的值。

与之相反，如果我们使枚举公开，它的所有变元都会随之公开。我们只需要 `enum` 关键字前的 `pub`，如 Listing 7-10 所示。

```rust
// Filename: src/lib.rs
mod back_of_house {
    pub enum Appetizer {
        Soup,
        Salad,
    }
}

pub fn eat_at_restaurant() {
    let order1 = back_of_house::Appetizer::Soup;
    let order2 = back_of_house::Appetizer::Salad;
}
```

▲ Listing 7-10：标记枚举为公开的会让它所有的边缘公开。

因为我们使枚举 `Appetizer` 公开了，我们就能在 `eat_at_restaurant` 中使用 `Soup` 和 `Salad` 变元。

除非枚举的边缘是公开的，否则它没什么用；在所有情况下都必须用 `pub` 标记枚举的所有变元是非常恼人的，所以枚举变元默认情况下是公开的。即使字段不公开，结构体通常也是有用的，所以结构体的字段遵守通常的规则：除非使用 `pub` 标记，所有东西默认情况下都是私有的。

还有一个涉及 `pub` 的情况我们没有覆盖到，那就是我们最后一个模块系统特性：`use` 关键字。我们将首先覆盖 `use` 本身，然后展示如何结合 `pub` 和 `use`。

## 使用 use 关键字将 Path 引入作用域中

必须写出 path 才能调用函数让人感觉麻烦且重复。在 Listing 7-7 中，无论我们是选择指向 `add_to_waitlist` 函数的绝对路径还是相对路径，每次调用 `add_to_waitlist` 时都必须指定 `front_of_house` 和 `hosting`。幸运的是，有办法来简化这个过程：我们可以用 `use` 关键字创建一个 path 的快捷方式，然后在作用域中的任何地方使用这个更短的名字。

在 Listing 7-11 中，我们将 `crate::front_of_house::hosting` module 引入 `eat_at_restaurant` 函数的作用域中，然后我们可以仅仅指定 `hosting::add_to_waitlist` 就能在 `eat_at_restaurant` 中调用 `add_to_waitlist` 函数。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

▲ Listing 7-11：用 `use` 将一个 module 引入到作用域中。

在一个作用域中添加 `use` 和一个 path 与在文件系统中创建一个符号链接很像。通过在 crate root 中增加 `crate::front_of_house::hosting`，`hosting` 就是作用域中的一个合法名字，就像 `hosting` 已经定义在 crate root 中一样。用 `use` 引入到作用域中的 path 也会检查私有性，就像其他 path 一样。

注意 `use` 仅仅是为 `use` 出现的特定作用域中创建了一个快捷方式。Listing 7-12 移动 `eat_at_restaurant` 函数到一个新的名为 `customer` 的子 module 中，然后它就与 `use` 语句在不同的作用域中了，所以该函数体将无法编译。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

mod customer {
    pub fn eat_at_restaurant() {
        hosting::add_to_waitlist();
    }
}
```

▲ Listing 7-12：一个 `use` 语句只适用于它所在的作用域。

编译错误显示这个快捷方式不适用于 `customer` module：

```shell
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0433]: failed to resolve: use of undeclared crate or module `hosting`
  --> src/lib.rs:11:9
   |
11 |         hosting::add_to_waitlist();
   |         ^^^^^^^ use of undeclared crate or module `hosting`

warning: unused import: `crate::front_of_house::hosting`
 --> src/lib.rs:7:5
  |
7 | use crate::front_of_house::hosting;
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

For more information about this error, try `rustc --explain E0433`.
warning: `restaurant` (lib) generated 1 warning
error: could not compile `restaurant` due to previous error; 1 warning emitted
```

注意还有一个警告说 `use` 没有在它的作用域中被使用到！为了解决这个问题，把 `use` 也移动到 `customer` module 中，或者在 `customer` module 中使用 `super::hosting` 来引用父 module 中的快捷方式。

### 创建惯用的 use Path

在 Listing 7-11 中，你可以有疑问，为什么我们在 `eat_at_restaurant` 中指定 `use crate::front_of_house::hosting`，然后调用 `hosting::add_to_waitlist`，而不是指定 `add_to_waitlist` 函数的全部 `use` path 来达到相同的结果，如 Listing 7-13 所示。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
}
```

▲ Listing 7-13：用 `use` 将 `add_to_waitlist` 函数引入到作用域中，这不常用。

虽然 Listing 7-11 和 Listing 7-13 可以完成同样的任务，但是 Listing 7-11 是使用 `use` 将函数引入作用域中的惯用方式。用 `use` 将函数的父 module 引入作用域中意味着当我们调用函数时必须指定父 module。这样能在最小化减少完整 path 地同时，更加清晰地知道该函数未定义在本地。Listing 7-13 中地代码不能清晰地说明 `add_to_waitlist` 定义在何处。

另一方面，当用 `use` 引入结构体、枚举或其他项时，惯用方式时指定完整 path。Listing 7-14 展示了这种惯用方式，将标准库中地 `HashMap` 结构体引入到一个二进制 crate 的作用域中。

```rust
// Filename: src/main.rs
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

▲ Listing 7-14：以惯用方式将 `HashMap` 引入到作用域中。

这个惯例背后没有什么强有力的理由：它只是已经出现的一个约定，并且人们已经习惯以这种方式阅读和书写 Rust 代码。

该惯例的例外场景是，如果我们使用 `use` 语句将两个有相同名字的项引入到作用域中，因为 Rust 不允许这样做。Listing 7-15 演示了如何将两个有相同名字，但不同父 module 的 `Result` 类型引入到作用域中并引用它们。

```rust
// Filename: src/lib.rs
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
    Ok(())
}

fn function2() -> io::Result<()> {
    // --snip--
    Ok(())
}
```

▲ Listing 7-15：引入两个有相同名字的类型到同一作用域中要求使用它们的父 module。

如你所见，使用父 module 来区分两个 `Result` 类型。如果相反，我们指定 `use std::fmt::Result` 和 `use std::io::Result`，我们就会在同一作用域中有两个 `Result` 类型，Rust 不知道当我们使用 `Result` 时到底指的是哪一个。

### 使用 as 关键字提供新的名字

对于使用 `use` 关键字在同一作用域中引入两个具有相同名字的类型的问题，优良一种解决方式：在 path 后面，我们可以指定 `as` 和一个该类型的新的本地名字，或称别名。Listing 7-16 展示了书写 Listing 7-15 中的代码的另一种方式：使用 `as` 重命名两个 `Result` 类型中的其中一个。

```rust
// Filename: src/lib.rs
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
    Ok(())
}

fn function2() -> IoResult<()> {
    // --snip--
    Ok(())
}
```

▲ Listing 7-16：当引入一个类型到作用域中时，使用 `as` 关键字对其重命名。

在第二个 `use` 语句中，我们为 `std::io::Result` 类型选择一个新的名字 `IoResult`，这样就不会与我们同样引入到作用域中的 `std::fmt` 中的 `Result` 冲突。Listing 7-15 和 Listing 7-16 都被认为是惯用的，所以任君选择。

### 用 pub use 重新导出名字

当我们用 `use` 关键字向作用域中引入一个新的名字时，该名字在新的作用域中是私有的。要使调用我们代码的代码可以引用这个名字，就像它定义在我们代码的作用域中一样，我们可以合并使用 `pub` 和 `use`。该技术被称作重新导出，因为我们引入一个项到作用域中，同时又使该项可以被其他人引入到他们的作用域中。

Listing 7-17 展示了 Listing 7-11 中的代码，但是将根 module 中的 `use` 改为了 `pub use`。

```rust
// Filename: src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

▲ Listing 7-17：使用 `pub use` 使得一个名字可以被任意代码从一个新的作用域中使用。

在此修改前，外部代码只能通过 path `restaurant::front_of_house::hosting::add_to_waitlist()` 调用 `add_to_waitlist` 函数。现在 `pub use` 重新从根 module 中导出了 `hosting` module，外部代码就能使用 path `restaurant::hosting::add_to_waitlist()` 了。

当代码的内部结构与调用代码的程序员对域的看法不同时，重新导出是有用的。例如，在这个餐厅比喻中，经营餐厅的人会想到前厅和后场。但是光顾餐厅的顾客可能不会考虑到餐厅的哪些部分。使用 `pub use`，我们能使用一种结构来书写代码，但暴露不通的结构。这么做让我们的库对于从事该库和调用该库的程序员组织性更好。我们将在第 14 章的“使用 `pub use` 导出方便且公开的 API”一节看一个 `pub use` 的例子以及它如何影响你 crate 的文档。

### 使用外部 Package

在第二章中，我们编写了一个猜数字游戏项目，其中使用了名为 `rand` 的外部 package 来得到随机数。为了在项目中使用 `rand`，我们在 `Cargo.toml` 中添加了这一行：

```toml
// Filename: Cargo.toml
rand = "0.8.5"
```
在 `Cargo.toml` 中添加 `rand` 作为依赖告诉 Cargo 从 [crates.io](https://crates.io/) 下载 `rand` package 和任意依赖，从而 `rand` 对于我们的项目是可用的。

然后，为了在我们的 package 的作用域中引入 `rand` 定义，我们添加了一个 `use` 行，其以该 crate 的名字，`rand` 开头，并且列出了我们想要引入到作用域中的项。回顾第二章中[“生成随机数”](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html#generating-a-random-number)一节，我们将 `Rng` trait 引入到作用域中，并调用了 `rand::thread_rng` 函数：

```rust
use std::io;
use rand::Rng;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {guess}");
}
```

Rust 社区的成员已经在 [crates.io](https://crates.io/) 上提供了许多 package，拉去其中的任意 package 到你的 package 中包含了这些相同的步骤：在你 package 的 `Cargo.toml` 文件中列出来它们，然后用 `use` 将他们 crate 中的项引入到作用域中。

注意标准库 `std` 对我们的 package 来说同样是外部 crate。因为 Rust 附带了标准库，我们不需要更改 `Cargo.toml` 来包含 `std`。但是我们需要用 `use` 将其中的项引入到我们 package 的作用域中。例如，使用 `HashMap` 时，我们要写这一行：

```rust
#![allow(unused)]
fn main() {
use std::collections::HashMap;
}
```

这是一个以 `std` 开头的绝对路径，它是标准库 crate 的名字。

### 使用嵌套 Path 清理大量的 use 列表

如果我们使用多个定义在同一个 crate 或 module 中的项，逐一在单独行中列举它们会占用我们文件的大量纵向空间。例如，Listing 2-4 中我们在猜数字游戏中的这两个 `use` 语句从 `std` 引入项到作用域中：

```rust
// --snip--
use std::cmp::Ordering;
use std::io;
// --snip--
```

取而代之，我们可以使用嵌套 path 在一行中将相同的项引入到作用域中。我们通过指定 path 的公共部分，后跟两个冒号，然后是花括号包围的 path 的不同部分的列表，如 Listing 7-18 所示。

```rust
// Filename: src/main.rs
// --snip--
use std::{cmp::Ordering, io};
// --snip--
```

▲ Listing 7-18：指定嵌套 path 以将有相同前缀的多个项引入到作用域中。

在大型程序中，使用嵌套 path 从同一 crate 或 module 中引入多个项对作用域中可以大量减少独立 `use` 语句的数量！

我们可以在 path 的任一层级上使用嵌套 path，这在合并两个共享一个子 path 的 `use` 语句时很有用。比如，Listing 7-19 展示了两个 `use` 语句，其一将 `std::io` 引入到作用域中，另一个则引入了 `std::io::Write`。

```rust
// Filename: src/lib.rs
use std::io;
use std::io::Write;
```

▲ Listing 7-19：两个 `use` 语句，其一是里一个的子 path。

两个 path 的公共部分是 `std::io`，那是第一个 path 的全部。为了合并两个 path 到一个 `use` 语句中，我们可以在嵌套 path 中使用 `self`，如 Listing 7-20 所示。

```rust
// Filename: src/lib.rs
use std::io::{self, Write};
```

▲ Listing 7-20：合并 Listing 7-19 中的 path 到一个 `use` 语句中。

这一行将 `std::io` 和 `std::io::Write` 引入到作用域中。

### 通配符运算符

如果我们想将定义在一个 path 中的所有公开项引入到作用域中，我们可以在指定 path 是后跟 `*` 通配符运算符：

```rust
#![allow(unused)]
fn main() {
use std::collections::*;
}
```

这个 `use` 语句将定义在 `std::collections` 中的所有公开项引入到当前作用域中。要谨慎使用通配符运算符！通配符使得更难说明当前作用域中有哪些名字以及你程序中使用的名字定义在哪里。

通配符运算符经常在测试时使用，将所有待测试项引入到 `test` module 中；我们将在第 11 章中[“如何写测试”](https://doc.rust-lang.org/book/ch11-01-writing-tests.html#how-to-write-tests)一节讨论。通配符运算符有时用作 prelude 模式的一部分：查看[标准库文档](https://doc.rust-lang.org/std/prelude/index.html#other-preludes)以了解该模式的更多信息。

## 划分 Module 到不同文件中

截止目前，本章中所有例子都在一个文件中定义多个 module。当 module 逐渐膨胀，你可能想要移动它们的定义到单独的文件中，从而代码更易导航到。

例如，我们以 Listing 7-17 中的代码开始，有多个餐厅 module。我们可以提取 module 到文件中，而不是所有的 module 都定义在 crate root 文件中。此情况下，crate root 文件是 `src/lib.rs`，但是该过程同样可用于 crate root 文件是 `src/main.rs` 的二进制 crate 中。

首先，我们提取 `front_of_house` module 到它自己文件中。移除 `front_of_house` module 的花括号中的代码，仅留下 `mod front_of_house;` 声明，这样 `src/lib.rs` 中仅包含如 Listing 7-21 所示的代码。注意，直到我们创建 Listing 7-22 中的 `src/front_of_house.rs` 文件前，代码无法编译。

```rust
// Filename: src/lib.rs
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

▲ Listing 7-21：声明 `front_of_house` module，它的主体将在 `src/front_of_house.rs` 中。

接下来，将在花括号中的代码放到名为 `src/front_of_house.rs` 的新文件中，如 Listing 7-22 所示。编译器知道要查看此文件，因为它访问到 crate root 中的名为 `front_of_house` 的 module 的声明。

```rust
// Filename: src/front_of_house.rs
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

▲ Listing 7-22：`src/front_of_house.rs` 中 `front_of_house` module 的定义。

注意，你只需要使用在你的模块树中用 `mod` 声明加载文件一次。只要编译器知道这个文件是项目的一部分（同时知道这些代码存在于模块树中的位置，因为你已经放了 `mod` 语句），你项目中的其他文件应该使用指向它生命位置的 path 引用该已加载文件中的代码，这部分在[“在模块树中使用 path 来引用项”](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html)一节有覆盖到。换句话说，`mod` 不是一个你在其他编程语言中可以见到的“include”操作。

接下来，我们提取 `hosting` 到它自己文件中。该过程有些许不同，因为 `hosting` 是 `front_of_house` 的子 module，而非根 module。我们将会把 `hosting` 的文件放到名为模块树中它祖先的名字的新文件夹下，在这里是 `src/front_of_house/`。

开始移动 `hosting`，我们修改 `src/front_of_house.rs` 为仅包含 `hosting` module 的声明：

```rust
// Filename: src/front_of_house.rs
pub mod hosting;
```

随之我们创建 `src/front_of_house` 文件夹，以及包含 `hosting` module 中的定义的 `hosting.rs` 文件：

```rust
// Filename: src/front_of_house/hosting.rs
pub fn add_to_waitlist() {}
```

如果我们将 `hosting.rs` 放到 `src` 文件夹中，编译器会期望 `hosting.rs` 中的代码存在于声明在 crate root 的 `hosting` module 中，而非声明为 `front_of_house` module 的子 module。这些要检查哪些文件的哪些模块的编译器规则意味着文件夹和文件更接近于模块树。

> **备用文件路径**
>
> 截止目前，我们覆盖到了大多数 Rust 用到的惯用文件路径，但是 Rust 也支持更老旧的文件路径样式。对于一个声明在 crate root 中的名为 `front_of_house` 的 module，编译器将在以下路径查找 module 的代码：
>
> - *src/front_of_house.rs*（我们已覆盖到）
> - *src/front_of_house/mod.rs*（老旧样式，仍受支持的路径）
>
> 对于 `front_of_house` 的名为 `hosting` 的子 module，编译器将在以下路径查找 module 的代码：
>
> - *src/front_of_house/hosting.rs*（我们已覆盖到）
> - *src/front_of_house/hosting/mod.rs*（老旧样式，仍受支持的路径）
>
> 如果你在同一 module 中同时使用两种样式，你将得到一个编译错误。对于同一项目不同 module 混合使用两种样式是被允许的，但是这可能是让导航到你的项目的人感到困惑。
>
> 这种使用名为 `mod.rs` 的文件的样式的主要弊端是你的项目可能包含许多名为 `mod.rs` 的文件，你同一时间在编辑器中打开它们时，这可能使人迷惑。

我们已将每个 module 的代码移动到独立的文件中，模块树保持一致。在 `eat_at_restaurant` 中的函数调用不需要任何修改仍可正常工作，即使这些定义存在不同的文件中。该技术让你在 module 的大小持续增长时可以移动 module 到新文件中。

注意 `src/lib.rs` 中的 `pub use crate::front_of_house::hosting` 语句也无需更改，也不会影响哪些文件编译为 crate 的一部分。`mod` 关键字声明 module，Rust 查看有相同名字的文件，以查找 该 module 中的代码。

## 总结

Rust 让你能够划分一个 package 到多个 crate 中，划分一个 crate 对多个 module 中，如此你可以从其他 module 中引用定义在另一 module 中的项。你能通过指定绝对路径和相对路径的方式做到。这些 path 可以被 `use` 语句引入到作用域中，如此你可以在该作用域中使用更短的 path 多次使用这些项。module 代码默认是私有的，但是你可以通过添加 `pub` 关键字使得定义公开。

在下一章，我们看一下标准库中的一泻剂和数据结构，你可以在你整洁有序的代码中使用它们。

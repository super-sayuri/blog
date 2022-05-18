---
title: "rust程序解析参数"
date: 2022-05-18T18:10:40+08:00
tags: ["rust","参数","clap"]
category: "技术"
---

我们想要rust编写好的程序读取想要的参数，例如这样：

```bash
./rust-bin -a a1 --name name
```

在运行的时候会自动读入 _a_ 和 _name_ 的数据。

可以使用[clap](https://crates.io/crates/clap)库来进行操作。

首先在Cargo.toml内引入依赖

```toml
[dependencies]
clap = { version = "3.1.18", features = ["derive"] }
```

之后可以使用derive或builder的方法来完成，这里简单介绍derive的方法，builder的方法参照[官方的例子](https://github.com/clap-rs/clap/blob/v3.1.18/examples/tutorial_builder/README.md)

要接受参数首先要编写一个参数的结构

```rust
use clap::Parser

#[derive(Parser)]
#[clap(XXX)]
struct Args {
}
```

这里`#[derive(Parser)]`是必须带的后面的`#[clap(XXX)]`里可以跟各种参数，例如：

* version: 自动添加 -V 和 --version，返回项目的版本号。
* author: 返回项目作者
* about: 自动添加 -H 和 --help，返回项目的帮助文档。

以上条目都可以设定自定义值。

之后对于结构里的每一个数据：

```rust
#[clap(PARAM_ARGS)]
arg1: SomeType,
```

其中，PARAM_ARGS里面比较常用的有：

* short 一键表示名字，默认为第一个字母，可自己指定
* long 长名，默认为变量的名字，同样可以自己指定
* skip 可忽略
* default_value = "SOME VALUE" 默认值

[更多参考](https://github.com/clap-rs/clap/blob/v3.1.18/examples/derive_ref/README.md)

### 例子：
我们这样定义一个结构：

```rust
use clap::Parser;

#[derive(Parser)]
#[clap(author, version, about)]
struct Args {
    #[clap(short,long)]
    config: String,
    #[clap(short,long,default_value="key.json")]
    keyfile: String,
}
```

随后调用：

```rust
fn main() {
    let args = Args::parse();
    println!("config: {}, keyfile: {}", args.config, args.keyfile);
}
```

编译好之后运行：

```bash
./rust-cmd -c CONFIG -k abc.txt
```

就可以看见输出了。

[参考](https://github.com/clap-rs/clap/blob/v3.1.18/examples/tutorial_derive/README.md)
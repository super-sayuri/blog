---
title: "如何在rust里读取配置文件"
date: 2022-05-18T21:36:00+08:00
tags: ["rust", "config"]
categories: ["rust"]
---
## 场景

如果想看代码直接拉到最后。

程序在不同环境运行很多变量都不同，所以需要配置文件来适配。

rust里从文件读取数据到结构很简单，但是之后呢？

所以这次我们先来考虑配置文件怎么设计

## 结构

首先我们应该有一个结构来装所有的配置：

```rust
// in conf.rs
pub struct MyConfig {
    pub debug: bool,
}

pub fn config() -> MyConfig {
    MyConfig {
        debug: true,
    }
}
```

之后调用：
```rust
// in main.rs
mod conf;

fn main() {
    print!("debugging: {}", conf::config().debug);
}
```

看上去是可以了，不过有一个问题：如我我们每次调用config都要新建一个，现在hard code还好，但是如果是从文件或者其他服务读取，消耗也太大了。

## 放进内存

所以我们的思路就是把config初始化好了之后存到内存里，随时随地拿出来使用

```rust
// in conf.rs
pub struct MyConfig {
    pub debug: bool,
}

static CONFIG: MyConfig = MyConfig {debug: true};

pub fn get_config() -> MyConfig {
    CONFIG
}
```
```rust
// in main.rs
mod conf;

fn main() {
    println!("debugging: {}", conf::get_config().debug);
}
```

先别运行，回忆一下rust关于所有权的知识，想想这段代码有没有什么问题？

有！在get_config()里最后会剥夺CONFIG的所有权，而CONFIG又无法自动copy,所以在下一次调用的时候就无法实现了。所以这里应该改一下：

```rust
// in conf.rs
// --snip--
pub fn get_config() -> &MyConfig {
    &CONFIG
}
```

改成借用应该没问题了吧？

当然不是。少了生命周期。所有的借用都要考虑生命周期的问题，否则编译器无法知道这个借用原来的值是不是被销毁了。

由于配置信息的特殊性，这里可以使用`'static`全局的生命周期，也就是数据会一直放在内存里直到程序结束。不过这里是特殊情况，其他情况还是要考虑生命周期到底是什么，不要一股脑全部都用`'static`。

```rust
// in conf.rs
// --snip--
pub fn get_config() -> &'static MyConfig {
    &CONFIG
}
```
## 从文件读取

完美，试着把文件读取也给加进去，不过现在有个小小的问题，CONFIG现在是写死代码初始化的，但是我们需要从文件初始化，也就是给一个参数来定CONFIG的值，该怎么做呢？

我们先试试这样：

```rust
pub fn init(debug: bool) -> Result<(), i32>{
    CONFIG.debug = debug;
    Ok(())
}
```

当然不可以，CONFIG是不可变的但是这个函数改变了CONFIG的值。

那么把CONFIG改成mut不就可以了吗？也不行。

static的变量可以被多个线程同时读取，因此如果改成mut，那么它的线程安全性是无法保障的，因此static mut的声明都会被rust定义为unsafe。我们也别太高估自己了，unsafe的东西不要乱用。

既然知道了原理，那么就简单了，我们加个锁就可以了。我们将CONFIG变成读写锁模式：

```rust
use std::sync::RwLock;

// --snip--

static CONFIG: RwLock<MyConfig> = RwLock::new(MyConfig {debug: true});

pub fn init(debug: bool) -> Result<(), i32>{
    let mut c = CONFIG.write().unwrap();
    *c = MyConfig{debug};
    Ok(())
}
```

又报错了，原因是对于static只能指定现成的结构或者const 函数，RwLock::New不是，所以说这里不行。

## lazy_static

这里介绍一个工具`lazy_static`,它可以帮助static对象初始化，而且是lazy loading模式的。[具体看这里](https://crates.io/crates/lazy_static)

这样，CONFIG就可以这样写：

```rust
lazy_static!{
    static ref CONFIG: RwLock<MyConfig> = 
        RwLock::new(MyConfig {debug: true});
}
```

`get_config()`函数我们也应当完善一下

```rust
pub fn get_config() -> MyConfig {
    let g = CONFIG.read().unwrap();
    *g
}
```

这里就没有返回一个借用了，而是直接返回CONFIG的copy,别忘了加上`#[derive(Clone,Copy)]`到
`MyConfig`的上面。

看看效果：

```rust
fn main() {
    println!("debugging: {}", conf::get_config().debug);
    another_use();
    let _ = conf::init(false);
    println!("debugging: {}", conf::get_config().debug);
    another_use();
}

fn another_use() {
    println!("debug on other place: {}", conf::get_config().debug);
}
```
output:
```
debugging: true
debug on other place: true
debugging: false
debug on other place: false
```
## 最终实现
完美，最后我们完善一下，从文件读取

```toml
# in cargo.toml
# -snip-
[dependencies]
lazy_static = "1.4.0"
serde = { version = "1.0.137", features = ["derive"] }
toml = "0.5.9"
```

```toml
# in config.toml
debug = false
info = "super sayuri"
```

```rust
// conf.rs
use std::{sync::RwLock, fs::File, error::Error, io::Read};
use serde::Deserialize;
use lazy_static::lazy_static;

#[derive(Deserialize, Clone)]
pub struct MyConfig {
    pub debug: bool,
    pub info: String, // 注意这里不可以使用&str, 借用滚粗struct！
}

lazy_static!{
    static ref CONFIG: RwLock<MyConfig> = 
        RwLock::new(MyConfig {debug: true, info:String::new()});
}

pub fn init(filename: &str) -> Result<(), Box<dyn Error>> {
    let mut file = File::open(filename)?;
    let mut conf_str = String::new();
    file.read_to_string(&mut conf_str)?;
    let mut c = CONFIG.write()?;
    *c = toml::from_str(&conf_str)?;
    Ok(())
}

pub fn get_config() -> MyConfig {
    let g = CONFIG.read().unwrap();
    g.clone()
}
```

```rust
// in main.rs
use std::error::Error;

mod conf;

fn main() -> Result<(), Box<dyn Error>>{
    println!("debugging: {}", conf::get_config().debug);
    another_use();
    let _ = conf::init("config.local.toml")?;
    println!("debugging: {}", conf::get_config().debug);
    another_use();
    Ok(())
}

fn another_use() {
    println!("info: {}", conf::get_config().info);
}
```

～fin~
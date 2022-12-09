---
title: Rust-NoteBook
date: 2022-11-15 20:24:45
tags: Rust
categories: rust
cover: /img/topimg/rust.png
---


## 泛型 & 声明周期
### `T: 'static`
* `T` 没有生命周期泛型参数
* T 类型对象所包含的所有的生命周期参数都是 `'static`

['static 含义参考](https://www.jianshu.com/p/500ed3635a41)
#### 示例:
```rust
fn my_func<F>(handle: F) where F: 'static {
    todo!()
}
```
> 即函数my_func拥有入参handle的所有权

| 类型变量 | T | &T                          |&mut T   |
|:-----|:------ |:----------------------------|:--------|
| 例子   | i32, &i32, &mut i32, &&i32, &mut &mut i32, ... | &i32, &&i32, &&mut i32, ... |&mut i32, &mut &mut i32, &mut &i32, ...|


### 结构体中的引用成员变量
```rust
struct A<'a> {
    name: &'a str
}
```
> 表示程序运行过程中`name`应用的声明周期比实例`A`长, 即 **name lifetime >= A lifetime**

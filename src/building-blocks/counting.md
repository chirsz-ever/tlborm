# 计数

## 反复替换

在宏中计数是一项让人吃惊的难搞的活儿。
最简单的方式是采用反复替换 (repetition with replacement) 。

```rust,editable
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! count_tts {
    ($($tts:tt)*) => {0usize $(+ replace_expr!($tts 1usize))*};
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

对于小数目来说，这方法不错，但当输入量到达<del> 500 </del>左右的标记时，
很可能让编译器崩溃。想想吧，输出的结果将类似：

```rust,ignore
0usize + 1usize + /* ~500 `+ 1usize`s */ + 1usize
```

编译器必须把这一大串解析成一棵 AST ，
那可会是一棵完美失衡的 500 多级深的二叉树。

译者注：500 这个数据过时了，例子见下面的 [递归](#递归) 第三个代码块。

## 递归

递归 (recursion) 是个老套路。

```rust,editable
macro_rules! count_tts {
    () => {0usize};
    ($_head:tt $($tail:tt)*) => {1usize + count_tts!($($tail)*)};
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

> 注意：对于 `rustc` 1.2 来说，很不幸，
编译器在处理大数量的类型未知的整型字面值时将会出现性能问题。
我们此处显式采用 `usize` 类型就是为了避免这种不幸。
>
> 如果这样做并不合适（比如说，当类型必须可替换时），
可通过 `as` 来减轻问题。（比如， `0 as $ty`、`1 as $ty` 等）。

这方法管用，但很快就会超出宏递归的次数限制（
[目前](https://doc.rust-lang.org/reference/attributes/limits.html#the-recursion_limit-attribute)
是 128 ）。

与重复替换不同的是，可通过增加匹配分支来增加可处理的输入面值。
以下为增加匹配分支的改进代码，如果把前三个分支注释掉，看看编译器会提示啥 :)

```rust,editable
macro_rules! count_tts {
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
     $_k:tt $_l:tt $_m:tt $_n:tt $_o:tt
     $_p:tt $_q:tt $_r:tt $_s:tt $_t:tt
     $($tail:tt)*)
        => {20usize + count_tts!($($tail)*)};
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
     $($tail:tt)*)
        => {10usize + count_tts!($($tail)*)};
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $($tail:tt)*)
        => {5usize + count_tts!($($tail)*)};
    ($_a:tt
     $($tail:tt)*)
        => {1usize + count_tts!($($tail)*)};
    () => {0usize};
}

fn main() {
    assert_eq!(700, count_tts!(
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
    ));
}
```
译者注：上面列出的这种匹配分支可以处理最多 \(20 \times 128 =  2560\) 个标记。
（如果不显式提高 128 的递归限制的话）。可以复制下面的例子运行看看，
里面包含递归和反复匹配两种方法。

```rust
macro_rules! count_tts {
  ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
   $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
   $_k:tt $_l:tt $_m:tt $_n:tt $_o:tt
   $_p:tt $_q:tt $_r:tt $_s:tt $_t:tt
   $($tail:tt)*)
    	=> {20usize + count_tts!($($tail)*)};
  ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
   $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
   $($tail:tt)*)
    	=> {10usize + count_tts!($($tail)*)};
  ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
   $($tail:tt)*)
    	=> {5usize + count_tts!($($tail)*)};
  ($_a:tt
   $($tail:tt)*)
    	=> {1usize + count_tts!($($tail)*)};
  () => {0usize};
}

// 可试试“反复替代”的方式计数
// --snippet--
# // macro_rules! replace_expr {
# //     ($_t:tt $sub:expr) => {
# //         $sub
# //     };
# // }
# //
# // macro_rules! count_tts {
# //     ($($tts:tt)*) => {0usize $(+ replace_expr!($tts 1usize))*};
# // }

fn main() {
    assert_eq!(2500,
               count_tts!(
                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
                   ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

                   // --snippet-- 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
#                    ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
# 
                   // 默认的递归限制让改进的递归代码也无法继续下去了
                   // 反复替换的代码还能够运行，但明显效率不会很高
                   // ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
                   // ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
               ));
}
```


## 切片长度

第三种方法，是帮助编译器构建一个深度较小的 AST ，以避免栈溢出。
可以通过构造数组，并调用其 `len` 方法来做到。(slice length)

```rust,editable
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! count_tts {
    ($($tts:tt)*) => {<[()]>::len(&[$(replace_expr!($tts ())),*])};
}

fn main() {
    assert_eq!(count_tts!(0 1 2), 3);

    const N: usize = count_tts!(0 1 2);
    let array = [0; N];
    println!("{:?}", array);
}
```

经过测试，这种方法可处理高达 10000 个标记数，可能还能多上不少。

（译者注：这个具体的数据可能也过时了，但这个方法的确是高效的。）

而且可以用于常量表达式，比如当作在 `const` 值或定长数组的长度值。
（原作时这个方法无法用于常量，现在无此限制）

所以基本上此方法是 **首选** 。

## 枚举计数

当你需要统计 **互不相同的标识符** 的数量时，
可以利用枚举体的
[numeric cast](https://doc.rust-lang.org/reference/items/enumerations.html#custom-discriminant-values-for-fieldless-enumerations) 
功能来达到统计成员（即标识符）个数。

```rust,editable
macro_rules! count_idents {
    ($($idents:ident),* $(,)*) => {
        {
            #[allow(dead_code, non_camel_case_types)]
            enum Idents { $($idents,)* __CountIdentsLast } 
            const COUNT: u32 = Idents::__CountIdentsLast as u32;
            COUNT
        }
    };
}
# 
# fn main() {
#     const COUNT: u32 = count_idents!(A, B, C);
#     assert_eq!(COUNT, 3);
# }
```

此方法有两大缺陷：
1. 它仅能被用于数有效的标识符（同时还不能是关键词），而且不允许那些标识符有重复
2. 不具备卫生性：如果你的末位标识符（在 `__CountIdentsLast` 位置上的标识符）的字面值也是输入之一，
那么宏调用就会失败，因为 `enum` 中包含重复变量。

## bit twiddling

另一个递归方法，但是使用了 位操作 (bit operations) [^YatoRust]：

```rust,editable
macro_rules! count_tts {
    () => { 0 };
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 };
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 };
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

这种方法非常聪明。
只要它是偶数个，就能有效地将其输入减半，
然后将计数器乘以 2（或者在这种情况下，向左移1位）。
因为由于前一次左移位，此时最低位必须为 0 ，重复直到我们达到基本规则 `() => 0` 。
如果输入是奇数个，则从第二个输入开始减半，最终将结果进行 或运算（这等效于加 1）。

这样做的好处是，生成计数器的 AST 表达式将以 `O(log(n))` 而不是 `O(n)` 复杂度增长。
请注意，这仍然可能达到递归限制。

让我们手动分析中间的过程：

```rust,ignore
count_tts!(0 0 0 0 0 0 0 0 0 0);
```

由于我们的标记树数量为偶数（10），因此该调用将与第三条规则匹配。
该匹配分支把奇数项的标记树命名给 `$a` ，偶数项的标记树命名成 `$b` ，
但是只会对奇数项 `$a` 展开，这意味着有效地抛弃所有偶数项，切断了一半的输入。
因此，调用现在变为：

```rust,ignore
count_tts!(0 0 0 0 0) << 1;
```

现在，该调用将匹配第二条规则，因为其输入的令牌树数量为奇数。
在这种情况下，第一个标记树将被丢弃以再次让输入变成偶数个，
然后可以在调用中再次进行减半步骤。
此时，我们可以将奇数时丢弃的一项计数为1，然后再乘以2，因为我们也减半了。

```rust,ignore
((count_tts!(0 0) << 1) | 1) << 1;
```
```rust,ignore
((count_tts!(0) << 1 << 1) | 1) << 1;
```
```rust,ignore
(((count_tts!() | 1) << 1 << 1) | 1) << 1;
```
```rust,ignore
((((0 << 1) | 1) << 1 << 1) | 1) << 1;
```

现在，要检查是否正确分析了扩展过程，
我们可以使用 [`debugging`](../macros/minutiae/debugging.html) 调试工具。
展开宏后，我们应该得到：

```rust,ignore
((((0 << 1) | 1) << 1 << 1) | 1) << 1;
```

没有任何差错，太棒了！


> 译者注：以下内容补充这部分提到的调试。
> 
> 注意，我这里使用的加、乘运算与上面提到的位运算是一样的。

```rust,editable
#![allow(unused)]
macro_rules! count_tts {
    () => { 0 };
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) *2) + 1 };
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) *2 };
}

fn main() {
    count_tts!(0 1 2 3 4 5 6 7 8 9 10);
}
```

调试方法（必须在 nightly 版本下）：
1. 使用编译命令 `cargo rustc -- -Z trace-macros`
得到：

```RUST,ignore
note: trace_macro
 --> src/main.rs:9:5
  |
9 |     count_tts!(0 1 2 3 4 5 6 7 8 9 10);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: expanding `count_tts! { 0 1 2 3 4 5 6 7 8 9 10 }`
  = note: to `(count_tts! (1 3 5 7 9) * 2) + 1`
  = note: expanding `count_tts! { 1 3 5 7 9 }`
  = note: to `(count_tts! (3 7) * 2) + 1`
  = note: expanding `count_tts! { 3 7 }`
  = note: to `count_tts! (3) * 2`
  = note: expanding `count_tts! { 3 }`
  = note: to `(count_tts! () * 2) + 1`
  = note: expanding `count_tts! {  }`
  = note: to `0`
```

2. 上面的形式太不简洁，所以使用封装好的工具：[cargo expand](https://github.com/dtolnay/cargo-expand)。
使用编译命令 `cargo expand` ，得到：

```RUST,ignore
#![feature(prelude_import)]
#![allow(unused)]
#[prelude_import]
use std::prelude::rust_2018::*;
#[macro_use]
extern crate std;
fn main() {
    (((((0 * 2) + 1) * 2 * 2) + 1) * 2) + 1;
}
```

[^YatoRust]:这种方法的归功于 Reddit 用户 
[`YatoRust`](https://www.reddit.com/r/rust/comments/d3yag8/the_little_book_of_rust_macros/) 。

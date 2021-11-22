# tuple structs

有时候我们并不需要一个为每个字段提供一个名字，我们需要为tuple声明一个类型。注意不同的tuple struct类型，即使拥有相同的声明，它们的实例也是不能相互转换的。

```rust
struct Point (i32, i32);

fn main() {
    let pt = Point(0, 0); //初始化
    println!("{}", pt.0); //类似于tuple通过.下标来获取元素
}
```

# unit-like struct

一个struct允许没有任何字段，这样的struct称为unit-like struct。

```rust
    struct AlwaysEqual;
    let subject = AlwaysEqual;
```

# method

我们可以为struct实现特有的函数，这类函数称为关联函数。关联函数的名称可以于struct的某个field相同。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 { //不可变self引用
        self.width * self.height
    }
    fn init(&mut self){  //可变self引用
        self.width = 0;
        self.height = 0;
    }
}

impl Rectangle { //对于同一个struct，可以有多个impl块
    fn give_back(self) -> Rectangle { //获得所有权
        self
    }
    fn square(size: u32) -> Rectangle { //关联函数也可以没有self参数
        Rectangle{
            width:size, 
            height:size
        }
    }
}

fn main() {
    let mut rect = Rectangle {
        width: 30,
        height: 50,
    };
    let area = rect.area();
    rect.init(); //这个调用和下一行的调用是等价的，rust会自动创建引用作为第一个参数传入
    (&mut rect).init();
    rect = rect.give_back();
    rect = Rectangle::square(32);
}
```

这里`&self`是`self: &Self`的缩写，其中`Self`是impl后面接的类型在这个impl块中的别名。

# enum

rust中我们enum类型更像是一种类型的分组，它内部可以包含多个具有别名的类型。

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

impl Message{
    fn distance(&self) -> i32{
        match self {
            Message::Move { x, y  } => x + y,
            default => 0
        }
    }
}

fn main() {
    let msg = Message::Move{x : 1, y : 1};
    println!("{}", msg.distance());
}
```

# match

match可以用来处理整数、枚举、字符串等。

```rust
let roll = 1;
let mut x = 0;

let res = match roll { //match可以带返回值
    0 => 0,
    1 => 1,
    other => -1, //用other匹配所有情况
}

match roll {
    0 => x += 1,
    1 => x -= 1,
    _ => () //_表示匹配任意值并丢弃，()表示什么都不做，{}也是相同作用
}
```

枚举类型也可以同样操作：

```rust
    let max = Some(1);
    match max {
        Some(x) => println!("max = {}", x),
        _ => ()
    }
```

很多时候我们仅处理一种枚举类型，但是这要求我们总是加入`_ => ()`行，这比较麻烦，还有一种`if let`语法。

```rust
    let max = Some(1);
    if let Some(x) = max {
        println!("max = {}", x);
    } else { //else块是可选的
    }
```
# 第 19 章：Web Assembly 与 Rust

在本章中，我们将探索 Rust 在 Web Assembly（WASM）领域的应用。通过学习这些概念，你将能够理解如何使用 Rust 开发高性能的 Web 应用，在浏览器中运行 Rust 代码，并与 JavaScript 无缝集成。本章内容是理解 Rust 跨平台开发能力的重要部分。

**学习目标：**

- 理解 Web Assembly 的工作原理与优势
- 掌握使用 Rust 构建 WASM 模块的方法
- 学习如何在浏览器中与 JavaScript 交互
- 掌握 WASM 性能优化策略
- 通过实例项目加深对 Rust-WASM 开发的理解

## 19.1 Web Assembly 简介

Web Assembly（简称 WASM）是一种为 Web 设计的二进制指令格式，提供了一种能在现代浏览器中运行的低级别、高性能的编译目标。它不是用来直接编写的，而是作为其他语言（如 Rust、C++、AssemblyScript 等）的编译目标。

### WASM 的设计目标

WASM 的主要设计目标包括：

1. **高性能**：接近原生代码的执行速度
2. **安全性**：在沙箱环境中运行，内存隔离
3. **紧凑二进制格式**：减少下载大小
4. **硬件无关**：运行在各种设备和体系结构上
5. **开放标准**：由 W3C 管理的 Web 标准

### WASM 与 JavaScript 对比

WASM 不是为了替代 JavaScript，而是作为其补充，特别适用于计算密集型任务：

| 特性       | JavaScript              | Web Assembly                      |
| ---------- | ----------------------- | --------------------------------- |
| 性能       | 解释执行（JIT）         | 预编译，接近原生性能              |
| 起步开销   | 轻量级                  | 相对更重（需加载）                |
| 内存安全   | 垃圾回收                | 手动内存管理（Rust 提供安全保证） |
| 适用场景   | DOM 操作、通用 Web 开发 | 计算密集、性能关键场景            |
| 开发友好性 | 高（动态类型）          | 取决于源语言（Rust 相对更严格）   |

### WASM 和 Rust 的完美结合

Rust 成为 WASM 开发的首选语言之一，原因包括：

1. **零开销抽象**：高级语言特性不会带来运行时开销
2. **无 GC**：没有垃圾回收器，避免性能波动
3. **小巧的运行时**：编译后的二进制文件小巧高效
4. **内存安全**：所有权系统在编译时防止内存错误
5. **优秀的工具支持**：官方提供`wasm-pack`等工具简化开发

```rust
// Rust代码
fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2)
    }
}

// 编译为WASM后，可以在浏览器中高效运行此函数
```

### WASM 生态系统

Web Assembly 生态系统正在快速发展：

- **浏览器支持**：所有主流浏览器（Chrome、Firefox、Safari、Edge）均支持
- **服务器端**：通过 WASI（Web Assembly 系统接口）在服务器端运行
- **嵌入式应用**：在各种宿主环境中作为嵌入式运行时

## 19.2 构建与部署 Rust WASM 模块

要开始 Rust 的 WASM 开发，需要设置适当的工具链和项目结构。本节将详细介绍如何创建、构建和部署 Rust WASM 模块。

### 开发环境准备

首先，确保你已安装 Rust 工具链和 WebAssembly 相关工具：

```bash
# 安装Rust（如果尚未安装）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 安装wasm32编译目标
rustup target add wasm32-unknown-unknown

# 安装wasm-pack工具
cargo install wasm-pack

# 安装cargo-generate（可选，用于从模板创建项目）
cargo install cargo-generate
```

### 创建 WASM 项目

有两种主要方式创建 Rust WASM 项目：

#### 1. 使用 wasm-pack 模板

```bash
# 使用wasm-pack模板创建项目
cargo generate --git https://github.com/rustwasm/wasm-pack-template.git --name my-wasm-project
cd my-wasm-project
```

#### 2. 手动创建项目

```bash
# 创建新库
cargo new --lib my-wasm-project
cd my-wasm-project
```

然后编辑`Cargo.toml`添加必要的配置：

```toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <your.email@example.com>"]
description = "A Rust WASM example"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2.84"

[profile.release]
opt-level = "s"
lto = true
```

### 编写简单的 WASM 模块

下面是一个简单的 Rust WASM 模块示例（`src/lib.rs`）：

```rust
use wasm_bindgen::prelude::*;

// 导出函数到JavaScript
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// 导出结构体与方法
#[wasm_bindgen]
pub struct Point {
    x: f64,
    y: f64,
}

#[wasm_bindgen]
impl Point {
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    pub fn distance_from_origin(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }

    pub fn get_x(&self) -> f64 {
        self.x
    }

    pub fn set_x(&mut self, x: f64) {
        self.x = x;
    }
}

// 使用JavaScript函数
#[wasm_bindgen]
extern "C" {
    pub fn alert(s: &str);
}

// 调用JavaScript函数
#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```

### 构建 WASM 包

使用`wasm-pack`构建项目：

```bash
# 构建面向Web的包
wasm-pack build --target web

# 或构建面向npm的包
wasm-pack build --target bundler

# 或构建Node.js包
wasm-pack build --target nodejs
```

构建过程将：

1. 编译 Rust 代码为 WASM 二进制文件
2. 生成 JavaScript 包装代码
3. 创建`npm`包结构（位于`pkg/`目录）

### 在 Web 项目中使用 WASM 模块

#### 直接在 HTML 中使用（--target web）：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>WASM Demo</title>
  </head>
  <body>
    <script type="module">
      import init, { add, Point, greet } from "./pkg/my_wasm_project.js";

      async function run() {
        // 初始化WASM模块
        await init();

        // 使用导出的函数
        console.log("2 + 3 =", add(2, 3));

        // 使用导出的类
        const point = new Point(3.0, 4.0);
        console.log("Distance:", point.distance_from_origin());

        // 调用会触发JavaScript alert的函数
        greet("WebAssembly");
      }

      run();
    </script>
  </body>
</html>
```

#### 在 Webpack/Rollup 项目中使用（--target bundler）：

1. 将生成的包添加到项目：

```bash
npm install ../my-wasm-project/pkg
# 或者发布到npm后：npm install my-wasm-project
```

2. 在 JavaScript 中导入：

```javascript
import { add, Point, greet } from "my-wasm-project";

console.log("2 + 3 =", add(2, 3));
const point = new Point(3.0, 4.0);
console.log("Distance:", point.distance_from_origin());
greet("WebAssembly");
```

### 部署注意事项

部署 WASM 应用时需注意：

1. **MIME 类型**：服务器需要为`.wasm`文件提供正确的 MIME 类型：

   ```
   Content-Type: application/wasm
   ```

2. **跨域资源共享（CORS）**：如果从不同域加载 WASM 文件，需配置 CORS

3. **压缩**：WASM 文件应使用 gzip 或 brotli 压缩以减小传输大小

4. **流式实例化**：考虑使用`WebAssembly.instantiateStreaming()`加快加载

### 实践思考

> **思考问题：** 什么类型的 Web 应用最适合使用 Rust+WASM 实现？这种方法相比纯 JavaScript 实现有哪些优势和潜在缺点？

Rust+WASM 特别适合计算密集型 Web 应用，如图形/图像处理、游戏引擎、模拟和科学计算等。主要优势包括显著提升的性能和内存安全性，缺点包括增加的构建复杂性和潜在的交互开销。对于 DOM 操作密集型应用，纯 JavaScript 可能仍然是更简单有效的选择。

## 19.3 JavaScript 交互

在 Web Assembly 应用中，Rust 代码与 JavaScript 的交互是至关重要的一部分。本节将详细介绍如何在这两种语言之间传递数据、共享内存和处理复杂类型。

### wasm-bindgen 基础

`wasm-bindgen`是实现 Rust 和 JavaScript 交互的核心工具，它提供了以下功能：

1. 从 Rust 导出函数、结构体和方法到 JavaScript
2. 导入 JavaScript 函数和对象到 Rust
3. 处理复杂类型的转换（如字符串、数组等）
4. 生成 JavaScript 胶水代码

一个基本的例子：

```rust
use wasm_bindgen::prelude::*;

// 导出Rust函数到JavaScript
#[wasm_bindgen]
pub fn process_data(input: &str) -> String {
    format!("处理后的数据: {}", input.to_uppercase())
}

// 从JavaScript导入函数
#[wasm_bindgen]
extern "C" {
    fn console_log(s: &str);

    // 访问window.document
    type Document;
    #[wasm_bindgen(getter, static_method_of = Document)]
    fn body() -> HtmlElement;

    type HtmlElement;
    #[wasm_bindgen(method)]
    fn appendChild(this: &HtmlElement, node: &HtmlElement) -> HtmlElement;
}

// 调用JavaScript函数
#[wasm_bindgen]
pub fn log_message(msg: &str) {
    console_log(&format!("[Rust] {}", msg));
}
```

### 数据类型转换

Rust 和 JavaScript 使用不同的数据模型，`wasm-bindgen`处理它们之间的转换：

#### 基本类型

简单数据类型的转换是直接的：

| Rust 类型    | JavaScript 类型     |
| ------------ | ------------------- |
| i32, u32     | number (整数)       |
| f32, f64     | number (浮点)       |
| bool         | boolean             |
| &str, String | string              |
| Option<T>    | T 或 null/undefined |

使用示例：

```rust
#[wasm_bindgen]
pub fn calculate(x: f64, y: f64, operation: &str) -> Option<f64> {
    match operation {
        "add" => Some(x + y),
        "subtract" => Some(x - y),
        "multiply" => Some(x * y),
        "divide" => if y != 0.0 { Some(x / y) } else { None },
        _ => None
    }
}
```

在 JavaScript 中使用：

```javascript
import { calculate } from "my-wasm-module";

const result = calculate(5.2, 3.1, "add");
console.log(result); // 输出: 8.3

const invalidResult = calculate(10, 0, "divide");
console.log(invalidResult); // 输出: null
```

#### 复杂类型

对于复杂类型，有几种常见的传递方法：

##### 1. 序列化/反序列化（使用`serde`和`serde-wasm-bindgen`）

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
pub struct User {
    name: String,
    age: u32,
    email: String,
}

#[wasm_bindgen]
pub fn process_user(js_user: JsValue) -> Result<JsValue, JsValue> {
    // 从JavaScript值转换为Rust结构体
    let user: User = serde_wasm_bindgen::from_value(js_user)?;

    // 处理数据
    let processed_user = User {
        name: user.name.to_uppercase(),
        age: user.age,
        email: user.email,
    };

    // 转换回JavaScript值
    Ok(serde_wasm_bindgen::to_value(&processed_user)?)
}
```

在 JavaScript 中：

```javascript
import { process_user } from "my-wasm-module";

const user = {
  name: "张三",
  age: 28,
  email: "zhangsan@example.com",
};

const processedUser = process_user(user);
console.log(processedUser.name); // 输出: "张三"
```

##### 2. 直接暴露 Rust 结构体

```rust
#[wasm_bindgen]
pub struct Vector2D {
    x: f64,
    y: f64,
}

#[wasm_bindgen]
impl Vector2D {
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Vector2D {
        Vector2D { x, y }
    }

    pub fn magnitude(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }

    pub fn scale(&mut self, factor: f64) {
        self.x *= factor;
        self.y *= factor;
    }

    #[wasm_bindgen(getter)]
    pub fn x(&self) -> f64 {
        self.x
    }

    #[wasm_bindgen(setter)]
    pub fn set_x(&mut self, x: f64) {
        self.x = x;
    }
}
```

在 JavaScript 中：

```javascript
import { Vector2D } from "my-wasm-module";

const vector = new Vector2D(3, 4);
console.log(vector.magnitude()); // 输出: 5

vector.scale(2);
console.log(vector.x); // 输出: 6
```

### 内存管理与传递大量数据

对于大量数据（如图像处理、音频处理等），直接传递数据会导致性能问题。更高效的方法是共享内存：

#### 使用 ArrayBuffer 和 TypedArray

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Uint8Array, Float32Array};

#[wasm_bindgen]
pub fn process_image(data: &[u8], width: u32, height: u32) -> Uint8Array {
    // 创建输出缓冲区
    let output_len = (width * height * 4) as usize;  // RGBA数据
    let mut output_data = vec![0u8; output_len];

    // 处理图像数据
    for i in 0..output_len {
        output_data[i] = 255 - data[i];  // 简单的反转操作
    }

    // 创建JavaScript Uint8Array
    let result = Uint8Array::new_with_length(output_len as u32);
    result.copy_from(output_data.as_slice());

    result
}

#[wasm_bindgen]
pub fn process_audio(audio_buffer: &Float32Array) -> Float32Array {
    let length = audio_buffer.length() as usize;
    let mut result_buffer = vec![0.0f32; length];

    // 复制输入数据到Rust向量
    audio_buffer.copy_to(&mut result_buffer);

    // 处理音频数据（例如：应用增益）
    for i in 0..length {
        result_buffer[i] *= 1.5;  // 增加音量
    }

    // 创建输出缓冲区
    let result = Float32Array::new_with_length(length as u32);
    result.copy_from(result_buffer.as_slice());

    result
}
```

在 JavaScript 中：

```javascript
import { process_image, process_audio } from "my-wasm-module";

// 图像处理
const canvas = document.getElementById("myCanvas");
const ctx = canvas.getContext("2d");
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

const processed = process_image(
  new Uint8Array(imageData.data.buffer),
  canvas.width,
  canvas.height
);

const newImageData = new ImageData(
  new Uint8ClampedArray(processed.buffer),
  canvas.width,
  canvas.height
);
ctx.putImageData(newImageData, 0, 0);

// 音频处理
const audioContext = new AudioContext();
// ... 获取audioBuffer ...
const audioData = audioBuffer.getChannelData(0);
const processedAudio = process_audio(audioData);
// ... 使用处理后的音频数据 ...
```

#### 高级：直接内存访问

对于最高性能要求，可以使用直接内存访问：

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct ImageProcessor {
    width: u32,
    height: u32,
    buffer_ptr: *mut u8,
    buffer_len: usize,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> ImageProcessor {
        let buffer_len = (width * height * 4) as usize;
        let buffer = vec![0u8; buffer_len];
        let buffer_ptr = buffer.as_ptr() as *mut u8;

        // 使用Box防止缓冲区被释放
        Box::leak(buffer.into_boxed_slice());

        ImageProcessor {
            width,
            height,
            buffer_ptr,
            buffer_len,
        }
    }

    #[wasm_bindgen]
    pub fn buffer_ptr(&self) -> *const u8 {
        self.buffer_ptr
    }

    #[wasm_bindgen]
    pub fn buffer_len(&self) -> usize {
        self.buffer_len
    }

    #[wasm_bindgen]
    pub fn process(&mut self) {
        // 安全地访问缓冲区
        let buffer = unsafe {
            std::slice::from_raw_parts_mut(self.buffer_ptr, self.buffer_len)
        };

        // 对缓冲区执行操作
        for i in 0..self.buffer_len {
            buffer[i] = buffer[i].saturating_add(10);  // 增加亮度
        }
    }
}
```

在 JavaScript 中：

```javascript
import { ImageProcessor } from "my-wasm-module";

// 创建处理器
const processor = new ImageProcessor(640, 480);

// 获取指向WASM内存的指针
const ptr = processor.buffer_ptr();
const len = processor.buffer_len();

// 创建指向WASM内存的视图
const memory = new Uint8Array(wasmMemory.buffer, ptr, len);

// 可以直接从JavaScript写入数据
for (let i = 0; i < len; i++) {
  memory[i] = i % 255;
}

// 处理数据
processor.process();

// 数据已在WASM内存中被修改
// 现在可以直接使用memory视图读取结果，而不需要复制
```

### 错误处理

在跨语言边界处理错误需要特别注意：

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub enum ErrorKind {
    InvalidInput,
    OperationFailed,
    OutOfBounds,
}

// 返回Result<T, JsValue>
#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> Result<f64, JsValue> {
    if b == 0.0 {
        return Err(JsValue::from_str("除数不能为零"));
    }

    Ok(a / b)
}

// 使用自定义错误类型
#[wasm_bindgen]
pub fn parse_data(input: &str) -> Result<u32, ErrorKind> {
    let trimmed = input.trim();

    if trimmed.is_empty() {
        return Err(ErrorKind::InvalidInput);
    }

    match trimmed.parse::<u32>() {
        Ok(value) => Ok(value),
        Err(_) => Err(ErrorKind::OperationFailed),
    }
}
```

在 JavaScript 中处理错误：

```javascript
import { divide, parse_data, ErrorKind } from "my-wasm-module";

try {
  const result = divide(10, 0);
  console.log(result);
} catch (error) {
  console.error("错误:", error); // 输出: "错误: 除数不能为零"
}

try {
  const value = parse_data("abc");
} catch (error) {
  if (error === ErrorKind.InvalidInput) {
    console.error("输入无效");
  } else if (error === ErrorKind.OperationFailed) {
    console.error("操作失败");
  }
}
```

### 回调函数与异步操作

Rust 可以接受 JavaScript 回调函数，也可以返回 Promise：

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Function, Promise};
use wasm_bindgen_futures::{future_to_promise, JsFuture};

// 接受JavaScript回调函数
#[wasm_bindgen]
pub fn for_each_item(callback: &Function) {
    let data = vec![1, 2, 3, 4, 5];

    for item in data {
        let this = JsValue::NULL;
        let arg = JsValue::from(item);

        // 调用JavaScript回调
        match callback.call1(&this, &arg) {
            Ok(_) => {},
            Err(e) => console_error(&format!("回调失败: {:?}", e)),
        }
    }
}

// 返回Promise给JavaScript
#[wasm_bindgen]
pub fn process_async(value: i32) -> Promise {
    future_to_promise(async move {
        // 模拟异步操作
        let result = value * 2;

        // 模拟延迟
        wait_for_seconds(1).await?;

        Ok(JsValue::from(result))
    })
}

// 辅助函数：等待指定秒数
async fn wait_for_seconds(seconds: i32) -> Result<(), JsValue> {
    let promise = js_sys::Promise::new(&mut |resolve, _| {
        let window = web_sys::window().unwrap();
        window.set_timeout_with_callback_and_timeout_and_arguments_0(
            &resolve,
            seconds * 1000,
        ).unwrap();
    });

    JsFuture::from(promise).await?;
    Ok(())
}

// 导入console.error
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn error(s: &str);

    #[wasm_bindgen(js_namespace = console, js_name = error)]
    fn console_error(s: &str);
}
```

在 JavaScript 中使用：

```javascript
import { for_each_item, process_async } from "my-wasm-module";

// 使用回调
for_each_item((item) => {
  console.log(`回调收到: ${item}`);
});

// 使用Promise
process_async(21)
  .then((result) => console.log(`异步结果: ${result}`))
  .catch((err) => console.error(`异步错误: ${err}`));

// 使用async/await
async function run() {
  try {
    const result = await process_async(21);
    console.log(`异步结果: ${result}`);
  } catch (err) {
    console.error(`异步错误: ${err}`);
  }
}
run();
```

### 实践思考

> **思考问题：** 在设计 Rust 和 JavaScript 交互接口时，应考虑哪些因素来最小化性能开销？特别是对于需要频繁交换大量数据的应用，如何优化数据传输？

设计高效率 Rust-JavaScript 接口应考虑：1) 最小化跨边界调用次数；2) 减少数据序列化/反序列化；3) 对大型数据集使用共享内存而非复制；4) 批处理多个操作；5) 在可能的情况下将计算密集型处理完全移至 Rust 侧。对于频繁交换大数据的应用，最佳方案是使用`ArrayBuffer`和直接内存访问，避免不必要的数据复制，并考虑使用 WebWorkers 进行并行处理。

## 19.4 性能优化策略

Web Assembly 应用性能的优化涉及多个层面，从 Rust 代码的编写到 WASM 二进制文件的配置，再到浏览器中的运行，每个环节都能产生显著影响。本节将探讨 Rust WASM 应用的性能优化策略。

### 编译优化

首先，可以通过调整编译参数优化 WASM 输出：

#### Cargo.toml 配置

```toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2.84"

[profile.release]
# 优化等级 - 可以是's'(大小优化)或'3'(性能优化)
opt-level = 3
# 启用链接时优化
lto = true
# 减少代码段以缩小二进制大小
codegen-units = 1
# 启用一些不安全优化
panic = "abort"
# 启用更激进的LLVM精简优化
strip = true
```

#### wasm-opt 工具

使用 Binaryen 的`wasm-opt`进一步优化 WASM 二进制文件：

```bash
# 安装wasm-opt（通常作为wasm-pack的一部分）
npm install -g wasm-pack

# 手动优化
wasm-opt -O3 -o optimized.wasm input.wasm
```

### 内存优化

WASM 的内存管理对性能至关重要：

#### 预分配内存

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct DataProcessor {
    buffer: Vec<u8>,
}

#[wasm_bindgen]
impl DataProcessor {
    // 预分配适当大小的缓冲区，避免运行时重新分配
    #[wasm_bindgen(constructor)]
    pub fn new(capacity: usize) -> Self {
        let buffer = Vec::with_capacity(capacity);
        DataProcessor { buffer }
    }

    // 其他方法...
}
```

#### 避免不必要的内存复制

```rust
use wasm_bindgen::prelude::*;
use js_sys::Uint8Array;

#[wasm_bindgen]
pub fn process_in_place(data: &mut [u8]) {
    // 直接在输入缓冲区上操作，不创建新缓冲区
    for byte in data.iter_mut() {
        *byte = byte.saturating_add(10); // 示例操作
    }
}

#[wasm_bindgen]
pub fn process_efficiently(input: &Uint8Array) -> Uint8Array {
    let len = input.length() as usize;

    // 创建有足够容量的输出缓冲区
    let mut output_data = Vec::with_capacity(len);
    unsafe { output_data.set_len(len); }

    // 复制输入到我们的缓冲区
    input.copy_to(&mut output_data);

    // 处理数据
    for byte in output_data.iter_mut() {
        *byte = byte.saturating_add(10);
    }

    // 创建结果数组
    let result = Uint8Array::new_with_length(len as u32);
    result.copy_from(&output_data);

    result
}
```

### 算法优化

选择合适的算法和数据结构对 WASM 性能至关重要：

#### 利用 SIMD 指令

WebAssembly SIMD（单指令多数据）可以显著加速计算密集型操作：

```rust
#[cfg(target_feature = "simd128")]
use wasm_bindgen::prelude::*;
use core::arch::wasm32::*;

#[wasm_bindgen]
#[cfg(target_feature = "simd128")]
pub fn add_vectors_simd(a: &[f32], b: &[f32], result: &mut [f32]) {
    let len = a.len().min(b.len()).min(result.len());
    let chunks = len / 4;

    for i in 0..chunks {
        let index = i * 4;

        unsafe {
            // 加载4个f32值到SIMD寄存器
            let a_simd = v128_load(a[index..].as_ptr() as *const v128);
            let b_simd = v128_load(b[index..].as_ptr() as *const v128);

            // 执行SIMD加法
            let sum = f32x4_add(a_simd, b_simd);

            // 存储结果
            v128_store(result[index..].as_mut_ptr() as *mut v128, sum);
        }
    }

    // 处理剩余元素
    for i in (chunks * 4)..len {
        result[i] = a[i] + b[i];
    }
}
```

要启用 SIMD，需要在`Cargo.toml`中添加配置：

```toml
[package.metadata.wasm-pack.profile.release]
wasm-opt = ['-O3', '--enable-simd']
```

并在命令行使用特定的 target 特性编译：

```bash
RUSTFLAGS='-C target-feature=+simd128' wasm-pack build --target web
```

#### 使用更高效的数据结构

针对 WASM 环境选择适当的数据结构：

```rust
use wasm_bindgen::prelude::*;
use fnv::FnvHashMap; // 比标准HashMap更快的哈希映射实现

#[wasm_bindgen]
pub struct FastLookup {
    // 使用效率更高的哈希实现
    map: FnvHashMap<u32, f64>,
}

#[wasm_bindgen]
impl FastLookup {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        FastLookup {
            map: FnvHashMap::default(),
        }
    }

    pub fn insert(&mut self, key: u32, value: f64) {
        self.map.insert(key, value);
    }

    pub fn get(&self, key: u32) -> Option<f64> {
        self.map.get(&key).copied()
    }
}
```

### 多线程支持

WebAssembly 线程支持可以显著提高性能：

```rust
use wasm_bindgen::prelude::*;
use rayon::prelude::*;
use js_sys::Uint8Array;
use web_sys::console;

// 注意：需要启用特定的线程功能
#[wasm_bindgen]
pub fn parallel_process(data: &Uint8Array) -> Uint8Array {
    // 获取数据长度
    let len = data.length() as usize;
    let mut rust_data = vec![0; len];

    // 复制数据到Rust Vec
    for i in 0..len {
        rust_data[i] = data.get_index(i as u32);
    }

    // 并行处理数据（使用Rayon）
    rust_data.par_iter_mut().for_each(|byte| {
        // 执行一些计算密集型操作
        *byte = complex_calculation(*byte);
    });

    // 创建结果
    let result = Uint8Array::new_with_length(len as u32);
    for i in 0..len {
        result.set_index(i as u32, rust_data[i]);
    }

    result
}

fn complex_calculation(input: u8) -> u8 {
    // 模拟复杂计算
    let mut result = input;
    for _ in 0..100 {
        result = result.wrapping_mul(7).wrapping_add(3) % 255;
    }
    result
}
```

要启用多线程支持，需要添加特定依赖并使用特殊编译标志：

```toml
[dependencies]
wasm-bindgen = "0.2.84"
rayon = "1.5"
wasm-bindgen-rayon = "1.0"
```

### 减少 JavaScript/WASM 边界调用

频繁跨越 JavaScript 和 WASM 边界会显著影响性能：

```rust
use wasm_bindgen::prelude::*;

// 不推荐 - 多次跨边界调用
#[wasm_bindgen]
pub fn inefficient_sum(n: u32) -> u32 {
    let mut sum = 0;
    for i in 0..n {
        sum += add_one(i); // 每次循环都会跨越JS/WASM边界
    }
    sum
}

#[wasm_bindgen]
pub fn add_one(x: u32) -> u32 {
    x + 1
}

// 推荐 - 单次跨边界调用
#[wasm_bindgen]
pub fn efficient_sum(n: u32) -> u32 {
    let mut sum = 0;
    for i in 0..n {
        sum += i + 1; // 计算保持在WASM内
    }
    sum
}
```

### 懒加载与按需编译

对于大型 WASM 应用，考虑分割代码并实现懒加载：

```javascript
// JavaScript端实现
async function loadCalculationModule() {
  const { calculate_complex } = await import("./complex_calculations.js");
  // 现在可以使用calculate_complex函数
  return calculate_complex;
}

document.getElementById("btnStartCalc").addEventListener("click", async () => {
  const calculateFn = await loadCalculationModule();
  const result = calculateFn(getInputValues());
  displayResult(result);
});
```

这需要将 Rust 代码分割成多个模块，并单独编译为 WASM：

```bash
# 为不同功能编译不同模块
wasm-pack build src/core --out-dir pkg/core --target web
wasm-pack build src/complex --out-dir pkg/complex --target web
wasm-pack build src/visualization --out-dir pkg/visualization --target web
```

### 性能测量与分析

优化不能盲目进行，需要仔细测量：

#### 在 Rust 中添加性能测量

```rust
use wasm_bindgen::prelude::*;
use web_sys::console;

#[wasm_bindgen]
pub fn benchmark_function() -> f64 {
    // 获取开始时间
    let start = js_sys::Date::now();

    // 执行要测量的操作
    perform_heavy_computation();

    // 计算耗时
    let end = js_sys::Date::now();
    let elapsed = end - start;

    console::log_1(&format!("Computation took: {}ms", elapsed).into());

    elapsed
}

fn perform_heavy_computation() {
    // 实际执行的计算...
    let mut sum = 0.0;
    for i in 0..1_000_000 {
        sum += (i as f64).sqrt();
    }
}
```

#### 使用 Chrome 性能工具

1. 打开 Chrome 开发者工具
2. 切换到"Performance"面板
3. 点击"Record"开始记录
4. 执行要分析的操作
5. 点击"Stop"停止记录
6. 分析性能数据，特别关注 WASM 相关的执行时间

### 实践思考

> **思考问题：** 在性能和文件大小之间通常需要权衡。对于一个 Web 应用，什么情况下应该优先考虑 WASM 模块的性能，什么情况下应该优先考虑文件大小？如何决定正确的权衡点？

性能优先场景包括：计算密集型应用（图像/视频处理、3D 渲染、科学计算）、需要流畅用户体验的游戏、实时数据分析，以及运行在性能较好设备上的应用。文件大小优先场景包括：移动网络应用、首屏加载时间关键的应用、目标用户网络条件较差的应用、仅偶尔使用 WASM 功能的应用。理想的权衡点取决于用户指标分析（加载时间 vs 使用时性能），可以通过性能预算确定。在某些情况下，采用渐进式加载策略可以兼顾两者：先加载小型核心 WASM 模块，在后台加载更多功能。

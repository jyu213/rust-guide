# 第 18 章：嵌入式系统与 Rust

在本章中，我们将探索 Rust 语言在嵌入式系统开发中的应用。通过学习这些概念，你将能够理解如何将 Rust 的安全性和性能优势应用于资源受限的嵌入式环境，掌握裸机编程基础，以及如何与硬件直接交互。本章内容是理解 Rust 在系统级编程中特有优势的重要部分。

**学习目标：**

- 理解 Rust 在嵌入式系统中的优势与应用场景
- 掌握嵌入式 Rust 开发环境的配置方法
- 学习如何使用 Rust 进行裸机编程
- 了解 Rust 中断处理与硬件抽象层的实现方式
- 通过实例项目巩固嵌入式 Rust 开发技能

## 18.1 裸机编程基础

嵌入式系统通常指没有完整操作系统的计算设备，或运行资源受限操作系统的计算设备。在这类系统中，程序直接与硬件交互，称为"裸机编程"（Bare-metal programming）。Rust 凭借其零开销抽象、内存安全保证和强大的类型系统，成为嵌入式开发的理想选择。

### 嵌入式 Rust 概述

嵌入式 Rust 开发与标准 Rust 编程有几个关键区别：

1. **没有标准库依赖**：大多数嵌入式系统无法支持完整的 Rust 标准库
2. **直接硬件访问**：需要直接读写内存映射寄存器
3. **资源约束**：严格的内存和处理能力限制
4. **实时要求**：通常需要满足实时性能要求

为支持这些需求，Rust 提供了特殊的编译目标和功能：

```rust
#![no_std] // 不使用标准库
#![no_main] // 不使用标准main入口

use core::panic::PanicInfo;

// 当程序panic时被调用
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {} // 无限循环，简单的错误处理方式
}

// 应用程序入口点
#[no_mangle] // 防止函数名被修改
pub extern "C" fn _start() -> ! {
    // 初始化硬件，执行主要任务
    loop {
        // 主循环
    }
}
```

这个简单的示例展示了嵌入式 Rust 程序的基本结构：使用`#![no_std]`标记避免标准库依赖，使用自定义入口点，并提供 panic 处理程序。

### 内存模型与裸机编程

嵌入式系统中的内存模型与标准计算机不同。通常包括：

- **Flash 存储器**：用于存储程序代码
- **RAM**：用于存储运行时数据
- **特殊功能寄存器**：控制硬件外设

在 Rust 裸机编程中，我们使用链接脚本（linker script）定义内存布局：

```
MEMORY
{
  /* 示例内存布局 */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS
{
  .text :
  {
    *(.text) /* 代码段 */
  } > FLASH

  .data :
  {
    *(.data) /* 初始化数据段 */
  } > RAM AT > FLASH

  .bss :
  {
    *(.bss) /* 未初始化数据段 */
  } > RAM
}
```

### 直接硬件访问

在裸机编程中，通过读写内存映射寄存器直接与硬件交互。Rust 使用`volatile`内存访问和指针操作实现这一点：

```rust
// 定义寄存器地址
const GPIO_PORT_A: *mut u32 = 0x40020000 as *mut u32;
const GPIO_MODE_REG: *mut u32 = (GPIO_PORT_A as usize + 0x00) as *mut u32;
const GPIO_OUTPUT_REG: *mut u32 = (GPIO_PORT_A as usize + 0x14) as *mut u32;

// 安全地操作硬件寄存器
unsafe fn set_pin_as_output(pin: u8) {
    let mode = core::ptr::read_volatile(GPIO_MODE_REG);
    let new_mode = mode | (1 << (pin * 2));
    core::ptr::write_volatile(GPIO_MODE_REG, new_mode);
}

unsafe fn set_pin_high(pin: u8) {
    core::ptr::write_volatile(GPIO_OUTPUT_REG, 1 << pin);
}
```

这种直接访问非常强大，但也很危险，因此通常封装在安全的抽象中。Rust 的`unsafe`代码块明确标识潜在危险操作，同时允许构建安全抽象。

### 嵌入式特有的编译优化

嵌入式开发需要特殊的编译优化：

1. **代码大小优化**：使用`opt-level = "z"`配置

```toml
# Cargo.toml
[profile.release]
opt-level = "z"   # 优化代码大小
lto = true        # 链接时优化
codegen-units = 1 # 进一步减小代码大小
panic = "abort"   # 减小恐慌处理代码大小
```

2. **定制目标配置**（target specification）：

```json
{
  "llvm-target": "thumbv7m-none-eabi",
  "data-layout": "e-m:e-p:32:32-i64:64-v128:64:128-a:0:32-n32-S64",
  "arch": "arm",
  "target-endian": "little",
  "target-pointer-width": "32",
  "target-c-int-width": "32",
  "os": "none",
  "target-env": "",
  "target-vendor": "",
  "features": "+thumb2",
  "linker-flavor": "gcc",
  "linker": "arm-none-eabi-gcc",
  "panic-strategy": "abort",
  "executables": true,
  "relocation-model": "static",
  "disable-redzone": true,
  "emit-debug-gdb-scripts": true
}
```

### 实践思考

> **思考问题：** 相比 C/C++，Rust 在嵌入式开发中的安全机制如何帮助避免常见硬件交互问题？

Rust 的所有权系统和类型检查可以在编译时捕获许多潜在的硬件交互错误，如竞争条件、未初始化寄存器访问和不正确的位操作，这些通常在 C/C++中需要运行时才能发现。

## 18.2 嵌入式开发环境配置

要开始嵌入式 Rust 开发，需要配置专门的工具链和开发环境。本节将详细介绍嵌入式 Rust 开发环境的配置过程，以及常用开发板的设置方法。

### 工具链安装

嵌入式 Rust 开发需要以下核心组件：

1. **Rust 工具链**：包括适用于嵌入式目标的特定编译器
2. **交叉编译工具**：用于为目标架构编译代码
3. **调试工具**：用于在目标硬件上调试程序
4. **烧录工具**：将编译好的程序写入硬件

以下是针对常见 ARM Cortex-M 微控制器的环境配置步骤：

```bash
# 安装基本Rust工具链
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 添加特定目标支持
# 对于Cortex-M0/M0+
rustup target add thumbv6m-none-eabi

# 对于Cortex-M3
rustup target add thumbv7m-none-eabi

# 对于Cortex-M4/M7（无FPU）
rustup target add thumbv7em-none-eabi

# 对于Cortex-M4/M7（有FPU）
rustup target add thumbv7em-none-eabihf

# 安装embedded-hal支持库
cargo install cargo-embed cargo-flash cargo-binutils

# 安装调试工具
cargo install probe-run
```

### 项目配置

嵌入式 Rust 项目需要特殊的配置。以下是一个典型的项目结构：

```
embedded-project/
├── Cargo.toml              # 项目配置
├── memory.x                # 内存布局脚本
├── .cargo/
│   └── config.toml         # Cargo配置
└── src/
    ├── main.rs             # 主程序
    └── lib.rs              # 库代码
```

`Cargo.toml`文件包含项目依赖和配置：

```toml
[package]
name = "embedded-project"
version = "0.1.0"
edition = "2021"
authors = ["你的名字 <your.email@example.com>"]

# 不使用标准库
[dependencies]
cortex-m = "0.7.7"            # Cortex-M CPU支持
cortex-m-rt = "0.7.3"         # Cortex-M启动和异常处理
panic-halt = "0.2.0"          # panic处理
embedded-hal = "0.2.7"        # 硬件抽象层

# 特定开发板支持（以STM32F4为例）
[dependencies.stm32f4xx-hal]
version = "0.15.0"
features = ["stm32f411"]      # 指定芯片型号

[profile.release]
opt-level = "z"               # 优化尺寸
lto = true                    # 链接时优化
codegen-units = 1             # 减小代码大小
debug = true                  # 保留调试信息
```

`.cargo/config.toml`文件配置构建目标和工具：

```toml
[target.thumbv7em-none-eabihf]
# 可以添加custom-runner以使用probe-run
runner = "probe-run --chip STM32F411CEUx"

# 所有编译使用rustflags
rustflags = [
  "-C", "link-arg=-Tlink.x",
]

[build]
# 设置默认目标
target = "thumbv7em-none-eabihf"
```

### 硬件抽象层（HAL）

嵌入式 Rust 使用硬件抽象层将底层硬件细节与应用代码分离。`embedded-hal`定义了标准接口，各种具体实现（如`stm32f4xx-hal`）提供特定芯片的实现。

```rust
use cortex_m::asm;
use cortex_m_rt::entry;
use stm32f4xx_hal::{
    pac,                  // 外设访问控制
    prelude::*,           // 常用特性和扩展
    gpio::{Output, PushPull},
    delay::Delay,
};
use panic_halt as _;     // panic处理程序

#[entry]
fn main() -> ! {
    // 获取设备特定外设
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // 创建延迟函数
    let mut delay = Delay::new(cp.SYST, clocks);

    // 配置GPIO
    let gpioc = dp.GPIOC.split();
    let mut led = gpioc.pc13.into_push_pull_output();

    loop {
        // 闪烁LED
        led.set_high().unwrap();
        delay.delay_ms(500u32);
        led.set_low().unwrap();
        delay.delay_ms(500u32);
    }
}
```

### 调试与烧录

嵌入式 Rust 支持多种调试和烧录方法：

1. **probe-run**：直接从命令行运行和调试程序
2. **OpenOCD**：开源调试器支持
3. **J-Link**：专业调试器支持
4. **ST-Link**：STM32 系列的官方调试器

使用`probe-run`进行开发的示例：

```bash
# 运行程序（编译并烧录到目标设备）
cargo run --release

# 使用特定的probe配置
probe-run --chip STM32F411CEUx target/thumbv7em-none-eabihf/release/my-program
```

使用 GDB 进行调试：

```bash
# 在一个终端启动OpenOCD
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

# 在另一个终端启动GDB
arm-none-eabi-gdb target/thumbv7em-none-eabihf/debug/my-program
```

### 实践思考

> **思考问题：** 为什么在嵌入式开发中选择特定的目标架构（如 thumbv7em-none-eabihf）如此重要？不同架构之间有什么差异？

针对特定架构进行优化可以最大限度地利用硬件功能，如 FPU（浮点单元）。例如，thumbv7em-none-eabihf 与 thumbv7em-none-eabi 的主要区别在于前者启用了硬件浮点支持，这对浮点密集型应用性能有显著影响。选择错误的架构可能导致性能问题，甚至无法访问某些硬件特性。

## 18.3 外设操作与硬件抽象层

嵌入式系统的核心功能是与外设交互，如 GPIO、定时器、ADC（模数转换器）和通信接口（I2C、SPI、UART）等。Rust 通过硬件抽象层（HAL）为这些外设提供了安全、抽象的接口，同时保持高性能。

### 外设访问模型

Rust 嵌入式开发中使用两种主要抽象级别操作外设：

1. **PAC (Peripheral Access Crate)**：低级别直接访问寄存器
2. **HAL (Hardware Abstraction Layer)**：高级别接口，提供类型安全的 API

#### PAC 层访问

PAC 提供最底层的寄存器访问，通常由 svd2rust 工具从芯片 SVD 文件自动生成：

```rust
// 使用PAC直接操作寄存器
use stm32f4xx_pac as pac;

fn configure_gpio_with_pac() {
    let peripherals = pac::Peripherals::take().unwrap();

    // 启用GPIOA时钟
    peripherals.RCC.ahb1enr.modify(|_, w| w.gpioaen().set_bit());

    // 配置PA5为输出模式
    peripherals.GPIOA.moder.modify(|_, w| w.moder5().output());

    // 设置PA5为高电平
    peripherals.GPIOA.bsrr.write(|w| w.bs5().set_bit());
}
```

虽然功能强大，但 PAC 级别的代码容易出错，需要仔细阅读芯片数据手册。

#### HAL 层访问

HAL 提供更高级别的抽象，利用 Rust 类型系统确保安全性：

```rust
use stm32f4xx_hal::{pac, prelude::*, gpio::{Output, PushPull, gpioa::PA5}};

fn configure_gpio_with_hal() {
    let dp = pac::Peripherals::take().unwrap();

    // 获取GPIO外设
    let gpioa = dp.GPIOA.split();

    // 配置PA5为输出模式
    let mut led: PA5<Output<PushPull>> = gpioa.pa5.into_push_pull_output();

    // 设置为高电平
    led.set_high().unwrap();

    // 设置为低电平
    led.set_low().unwrap();
}
```

HAL 提供类型安全的 API，编译器可以捕获许多常见错误，例如在配置为输入时尝试设置输出。

### 常见外设操作

#### GPIO 操作

GPIO (通用输入/输出) 是最基本的外设，用于数字信号的输入和输出：

```rust
use stm32f4xx_hal::{pac, prelude::*};

fn gpio_examples() {
    let dp = pac::Peripherals::take().unwrap();
    let gpioa = dp.GPIOA.split();

    // 配置输出引脚
    let mut output_pin = gpioa.pa5.into_push_pull_output();

    // 配置输入引脚（带上拉电阻）
    let input_pin = gpioa.pa0.into_pull_up_input();

    // 读取输入
    if input_pin.is_high().unwrap() {
        // 按钮释放状态
        output_pin.set_high().unwrap();
    } else {
        // 按钮按下状态
        output_pin.set_low().unwrap();
    }

    // 输出切换
    output_pin.toggle().unwrap();
}
```

#### 定时器操作

定时器用于精确计时和生成 PWM 信号：

```rust
use stm32f4xx_hal::{pac, prelude::*, timer::{Timer, Event}};

fn timer_example() {
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // 创建一个1Hz的定时器（1秒触发一次）
    let mut timer = Timer::tim2(dp.TIM2, 1.Hz(), clocks);

    // 启动定时器
    timer.listen(Event::TimeOut);

    // 等待定时器事件
    loop {
        if timer.wait().is_ok() {
            // 定时器触发
            break;
        }
    }
}
```

#### UART 通信

UART (通用异步收发器) 是一种常见的串行通信协议：

```rust
use stm32f4xx_hal::{pac, prelude::*, serial::{config::Config, Serial}};
use core::fmt::Write;

fn uart_example() {
    let dp = pac::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // 获取GPIO
    let gpioa = dp.GPIOA.split();

    // 配置UART引脚
    let tx_pin = gpioa.pa9.into_alternate::<7>();
    let rx_pin = gpioa.pa10.into_alternate::<7>();

    // 配置USART1 (115200 波特率)
    let serial_config = Config::default().baudrate(115200.bps());
    let mut serial = Serial::usart1(
        dp.USART1,
        (tx_pin, rx_pin),
        serial_config,
        clocks,
    ).unwrap();

    // 发送数据
    writeln!(serial, "Hello from Rust!").unwrap();

    // 接收数据
    if let Ok(received) = nb::block!(serial.read()) {
        // 处理接收到的字节
    }
}
```

#### I2C 通信

I2C 是一种常用于连接低速外设的总线协议：

```rust
use stm32f4xx_hal::{pac, prelude::*, i2c::{I2c, Mode}};

fn i2c_example() {
    let dp = pac::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // 获取GPIO
    let gpiob = dp.GPIOB.split();

    // 配置I2C引脚
    let scl = gpiob.pb8.into_alternate_open_drain::<4>();
    let sda = gpiob.pb9.into_alternate_open_drain::<4>();

    // 配置I2C1 (100kHz)
    let mut i2c = I2c::i2c1(
        dp.I2C1,
        (scl, sda),
        Mode::Standard {
            frequency: 100.kHz(),
        },
        clocks,
    );

    // 设备地址
    const DEVICE_ADDR: u8 = 0x68;

    // 写入数据
    let data_to_write = [0x00, 0x42];
    i2c.write(DEVICE_ADDR, &data_to_write).unwrap();

    // 读取数据
    let mut buffer = [0u8; 2];
    i2c.read(DEVICE_ADDR, &mut buffer).unwrap();
}
```

### 硬件抽象层设计模式

`embedded-hal`是一个核心 trait 集合，定义了嵌入式设备常见功能的标准接口：

```rust
// embedded-hal中的基本特性
pub trait InputPin {
    type Error;

    fn is_high(&self) -> Result<bool, Self::Error>;
    fn is_low(&self) -> Result<bool, Self::Error>;
}

pub trait OutputPin {
    type Error;

    fn set_high(&mut self) -> Result<(), Self::Error>;
    fn set_low(&mut self) -> Result<(), Self::Error>;
}

pub trait ToggleableOutputPin {
    type Error;

    fn toggle(&mut self) -> Result<(), Self::Error>;
}
```

通过这些统一接口，可以编写与具体硬件无关的驱动程序，实现跨平台兼容性：

```rust
// 设备驱动示例 - LED闪烁器
use embedded_hal::digital::v2::{OutputPin, ToggleableOutputPin};
use cortex_m::delay::Delay;

struct Blinker<LED> {
    led: LED,
    delay_ms: u32,
}

impl<LED, E> Blinker<LED>
where
    LED: OutputPin<Error = E> + ToggleableOutputPin<Error = E>
{
    pub fn new(led: LED, delay_ms: u32) -> Self {
        Blinker { led, delay_ms }
    }

    pub fn blink(&mut self, delay: &mut Delay) -> Result<(), E> {
        self.led.toggle()?;
        delay.delay_ms(self.delay_ms);
        Ok(())
    }
}
```

这种设计使驱动程序可以在任何实现了`OutputPin`和`ToggleableOutputPin`特性的硬件上工作，无需修改。

### 实践思考

> **思考问题：** 使用 HAL 抽象与直接使用 PAC 访问寄存器相比，在代码安全性、可维护性和性能方面有什么权衡？

HAL 抽象提供更高的安全性和可维护性，通过类型系统防止错误配置，并使代码更易于理解。理论上可能有轻微性能开销，但由于 Rust 的零成本抽象，编译后的代码通常与直接寄存器访问一样高效。选择取决于项目需求 - 通用性和安全性（HAL）vs. 极致控制和特定优化（PAC）。

## 18.4 中断处理与实时特性

嵌入式系统通常需要快速响应外部事件，这就需要高效的中断处理机制。同时，许多嵌入式应用需要满足实时性要求，确保任务在特定时间内完成。Rust 提供了安全的中断处理和实时编程支持。

### 中断基础

中断是处理器响应外部事件或定时器触发的机制，使处理器暂停正常执行流程并执行中断处理程序：

1. **硬件中断**：由外部事件（如按钮按下、传感器信号）触发
2. **软件中断**：由软件指令触发
3. **异常**：由处理器内部条件（如内存访问错误）触发

Rust 提供了类型安全的中断处理框架，确保中断处理安全可靠。

### 配置中断处理程序

在 ARM Cortex-M 架构上，可以使用`cortex-m-rt`提供的中断属性宏定义中断处理程序：

```rust
use cortex_m::asm;
use cortex_m_rt::entry;
use stm32f4xx_hal::{pac, prelude::*};
use stm32f4xx_pac::interrupt;

static mut LED: Option<stm32f4xx_hal::gpio::Pin<Output<PushPull>, PC13>> = None;

#[entry]
fn main() -> ! {
    // 获取设备外设
    let dp = pac::Peripherals::take().unwrap();

    // 配置GPIO
    let gpioc = dp.GPIOC.split();

    // 配置LED引脚
    unsafe {
        LED = Some(gpioc.pc13.into_push_pull_output());
    }

    // 配置EXTI中断线
    let syscfg = dp.SYSCFG.constrain();

    // 配置PA0为外部中断源（通常连接按钮）
    let gpioa = dp.GPIOA.split();
    let button = gpioa.pa0.into_pull_up_input();

    // 连接PA0到EXTI0中断线
    syscfg.exti.link_pin(&button);

    // 配置中断触发条件（下降沿）
    syscfg.exti.configure_gpio(
        &button,
        stm32f4xx_hal::syscfg::Edge::FALLING
    );

    // 启用EXTI0中断
    unsafe {
        cortex_m::peripheral::NVIC::unmask(pac::Interrupt::EXTI0);
    }

    loop {
        // 主循环
        asm::wfi(); // 等待中断
    }
}

#[interrupt]
fn EXTI0() {
    // 清除中断标志
    // 注意：具体清除方式取决于芯片家族
    let exti = unsafe { &(*pac::EXTI::ptr()) };
    exti.pr.write(|w| w.pr0().set_bit());

    // 处理中断，例如切换LED状态
    unsafe {
        if let Some(led) = &mut LED {
            led.toggle().unwrap();
        }
    }
}
```

### 中断安全与共享变量

处理中断时，Rust 的所有权系统帮助避免常见问题，如数据竞争。但由于中断可能随时发生，需要特殊机制共享数据：

1. **静态可变**：使用`static mut`并在访问时使用`unsafe`
2. **临界区**：使用`cortex_m::interrupt::free`创建禁用中断的临界区
3. **并发原语**：使用无锁数据结构如`core::sync::atomic`或`heapless::spsc`队列

以下示例展示了使用临界区安全共享数据：

```rust
use cortex_m::interrupt::{free, Mutex};
use core::cell::RefCell;

// 全局共享变量
static G_COUNT: Mutex<RefCell<u32>> = Mutex::new(RefCell::new(0));

fn increment_count() {
    free(|cs| {
        // 在临界区内安全访问共享数据
        let mut count = G_COUNT.borrow(cs).borrow_mut();
        *count += 1;
    });
}

#[interrupt]
fn TIMER0() {
    increment_count();
}
```

### 中断优先级

嵌入式系统通常允许设置中断优先级，确保关键中断优先处理：

```rust
use cortex_m::peripheral::NVIC;

fn configure_interrupts() {
    unsafe {
        // 配置中断优先级 (0-255，0为最高优先级)
        NVIC::unmask(pac::Interrupt::EXTI0);
        cortex_m::peripheral::NVIC::set_priority(
            pac::Interrupt::EXTI0,
            128 // 中等优先级
        );

        // 配置另一个更高优先级中断
        NVIC::unmask(pac::Interrupt::TIM2);
        cortex_m::peripheral::NVIC::set_priority(
            pac::Interrupt::TIM2,
            64 // 更高优先级
        );
    }
}
```

### 实时系统特性

实时系统需要在确定的时间内响应事件。Rust 支持构建实时系统，但需要注意几个关键方面：

#### 可预测的内存管理

Rust 的"零成本抽象"和无运行时垃圾回收使其非常适合实时系统。但需注意：

1. **避免动态内存分配**：使用静态分配或预分配策略
2. **限制栈使用**：避免大型局部变量和深递归

```rust
// 使用静态缓冲区代替动态分配
use heapless::Vec;

fn process_data() {
    // 使用栈上预分配的固定大小向量
    let mut buffer: Vec<u8, 64> = Vec::new();

    // 填充数据（安全限制在容量内）
    for i in 0..60 {
        buffer.push(i).unwrap();
    }

    // 处理数据...
}
```

#### 定时精度

实时系统需要精确计时，Rust 支持各种定时方式：

```rust
use stm32f4xx_hal::{pac, prelude::*, timer::{Timer, Event}};

fn precise_timing() {
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // 高精度定时器
    let mut timer = Timer::tim2(dp.TIM2, 10.kHz(), clocks);

    // 获取SysTick定时器
    let mut systick = cp.SYST;
    systick.set_clock_source(cortex_m::peripheral::syst::SystClkSource::Core);
    systick.set_reload(SystemFrequency::get() / 1000); // 1ms定时
    systick.enable_counter();

    // 等待精确时间
    timer.start(1000.millis());
    nb::block!(timer.wait()).unwrap();
}
```

#### 任务调度

实时系统通常使用任务调度器管理多个任务。RTIC (Real-Time Interrupt-driven Concurrency) 是 Rust 中流行的实时框架：

```rust
#[rtic::app(device = stm32f4xx_hal::pac)]
mod app {
    use stm32f4xx_hal::{pac, prelude::*};

    #[shared]
    struct Shared {
        // 共享资源
        count: u32,
    }

    #[local]
    struct Local {
        // 任务本地资源
        led: stm32f4xx_hal::gpio::Pin<Output<PushPull>, PC13>,
    }

    #[init]
    fn init(ctx: init::Context) -> (Shared, Local, init::Monotonics) {
        let dp = ctx.device;

        // 配置外设
        let gpioc = dp.GPIOC.split();
        let led = gpioc.pc13.into_push_pull_output();

        // 初始化系统

        // 计划定期任务
        blink::spawn_after(1.secs()).unwrap();

        (
            Shared { count: 0 },
            Local { led },
            init::Monotonics()
        )
    }

    // 定期执行的任务
    #[task(local = [led], shared = [count])]
    fn blink(ctx: blink::Context) {
        // 访问本地资源
        let led = ctx.local.led;
        led.toggle().unwrap();

        // 安全访问共享资源
        ctx.shared.count.lock(|count| {
            *count += 1;
        });

        // 重新调度任务
        blink::spawn_after(1.secs()).unwrap();
    }

    // 由外部中断触发的任务
    #[task(binds = EXTI0, shared = [count])]
    fn button_press(ctx: button_press::Context) {
        // 处理按钮中断

        // 访问共享资源
        ctx.shared.count.lock(|count| {
            *count = 0; // 重置计数器
        });
    }
}
```

RTIC 提供的关键功能包括：

- 基于优先级的抢占式调度
- 静态验证的资源管理（避免数据竞争）
- 支持定时器和外部事件驱动的任务
- 显式的资源访问控制，避免死锁

### 实时性能分析

评估实时系统需要测量关键指标：

- **响应时间**：从事件发生到响应开始的时间
- **执行时间**：完成任务所需的时间
- **抖动**：任务执行时间的变异性

可以使用片上调试功能或 GPIO 引脚测量这些指标：

```rust
fn measure_execution_time() {
    let dp = pac::Peripherals::take().unwrap();
    let gpioa = dp.GPIOA.split();

    // 使用GPIO引脚进行性能分析
    let mut timing_pin = gpioa.pa8.into_push_pull_output();

    // 开始测量
    timing_pin.set_high().unwrap();

    // 执行要测量的操作
    perform_critical_task();

    // 结束测量
    timing_pin.set_low().unwrap();

    // 可以使用逻辑分析仪或示波器测量引脚高电平持续时间
}
```

### 实践思考

> **思考问题：** 在设计嵌入式实时系统时，Rust 的所有权系统如何帮助避免传统 C/C++中常见的中断相关问题？

Rust 的所有权系统通过编译时检查防止多个中断处理程序同时访问可变数据，强制使用安全机制（如`Mutex<T>`和临界区）进行访问。这消除了传统 C/C++中常见的数据竞争和时序相关 bug，提高了系统可靠性。此外，Rust 的类型系统能确保中断资源的正确初始化和使用，减少了运行时错误的可能性。

## 18.5 实例：简易嵌入式项目

在本节中，我们将通过一个完整的微控制器项目，将前面学习的概念付诸实践。我们将构建一个基于 STM32F4 微控制器的环境监测器，它能测量温度和湿度，将数据显示在 OLED 显示屏上，并通过按钮交互改变显示模式。

### 项目概述

项目功能包括：

1. 读取 DHT11/DHT22 温湿度传感器数据
2. 在 SSD1306 OLED 显示屏上显示温度和湿度信息
3. 通过按钮切换显示模式（温度/湿度/两者）
4. LED 指示系统状态

### 硬件准备

项目需要以下硬件：

- STM32F4 开发板（如 STM32F411 "Black Pill"）
- DHT11 或 DHT22 温湿度传感器
- SSD1306 OLED 显示屏（I2C 接口）
- 按钮
- LED
- 面包板和连接线

### 项目结构

首先，创建一个新的 Rust 项目：

```bash
cargo new --bin temp_humidity_monitor
cd temp_humidity_monitor
```

配置`Cargo.toml`文件：

```toml
[package]
name = "temp_humidity_monitor"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <your.email@example.com>"]

[dependencies]
cortex-m = "0.7.7"
cortex-m-rt = "0.7.3"
panic-halt = "0.2.0"
embedded-hal = "0.2.7"
nb = "1.0.0"
heapless = "0.7.16"

# 显示屏驱动
ssd1306 = "0.7.1"
embedded-graphics = "0.7.1"

# STM32F4支持
[dependencies.stm32f4xx-hal]
version = "0.15.0"
features = ["stm32f411", "rt"]

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
```

创建`.cargo/config.toml`配置文件：

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-run --chip STM32F411CEUx"
rustflags = [
  "-C", "link-arg=-Tlink.x",
]

[build]
target = "thumbv7em-none-eabihf"
```

### 传感器驱动实现

在`src/dht.rs`中实现 DHT11/DHT22 传感器驱动：

```rust
//! DHT11/DHT22 温湿度传感器驱动
use embedded_hal::digital::v2::{InputPin, OutputPin};
use cortex_m::asm::delay;

pub struct Reading {
    pub temperature: f32,
    pub humidity: f32,
}

pub struct Dht<PIN> {
    pin: PIN,
    delay_us: fn(u32),
    is_dht22: bool,
}

impl<PIN, E> Dht<PIN>
where
    PIN: OutputPin<Error = E> + InputPin<Error = E>,
{
    pub fn new(pin: PIN, delay_us: fn(u32), is_dht22: bool) -> Self {
        Dht { pin, delay_us, is_dht22 }
    }

    pub fn read(&mut self) -> Result<Reading, &'static str> {
        // 发送开始信号
        self.pin.set_low().map_err(|_| "Error setting pin low")?;
        (self.delay_us)(20_000);  // 等待20ms
        self.pin.set_high().map_err(|_| "Error setting pin high")?;
        (self.delay_us)(40);      // 等待40us

        // 等待传感器响应
        self.pin.set_low().map_err(|_| "Error setting pin as input")?;

        // 读取40位数据
        let mut data = [0u8; 5];
        // ... 省略详细的位读取逻辑 ...

        // 检查校验和
        if data[4] != ((data[0] + data[1] + data[2] + data[3]) & 0xFF) {
            return Err("Checksum error");
        }

        // 解析数据
        let humidity = if self.is_dht22 {
            ((data[0] as u16) << 8 | data[1] as u16) as f32 / 10.0
        } else {
            data[0] as f32
        };

        let temperature = if self.is_dht22 {
            let temp_raw = ((data[2] as u16) << 8 | data[3] as u16) as i16;
            temp_raw as f32 / 10.0
        } else {
            data[2] as f32
        };

        Ok(Reading {
            temperature,
            humidity,
        })
    }
}
```

### 主程序实现

在`src/main.rs`中实现主程序：

```rust
#![no_std]
#![no_main]

use cortex_m::asm;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    pac,
    prelude::*,
    i2c::I2c,
    gpio::{Edge, Input, PullUp},
};
use heapless::String;
use heapless::consts::U64;
use embedded_graphics::{
    mono_font::{ascii::FONT_6X10, MonoTextStyleBuilder},
    pixelcolor::BinaryColor,
    prelude::*,
    text::{Baseline, Text},
};
use ssd1306::{prelude::*, I2CDisplayInterface, Ssd1306};

mod dht;
use dht::Dht;

// 显示模式
enum DisplayMode {
    Temperature,
    Humidity,
    Both,
}

#[entry]
fn main() -> ! {
    // 获取外设
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    // 配置时钟
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.sysclk(84.MHz()).freeze();

    // 配置延迟函数
    let mut delay = cp.SYST.delay(&clocks);

    // 配置GPIO
    let gpioa = dp.GPIOA.split();
    let gpiob = dp.GPIOB.split();

    // DHT传感器引脚
    let mut dht_pin = gpioa.pa1.into_open_drain_output();
    // 创建微秒延迟函数
    let delay_us = |us| {
        // 简化版，实际应调整为准确的微秒延迟
        for _ in 0..us * 8 {
            asm::nop();
        }
    };
    let mut dht = Dht::new(dht_pin, delay_us, true); // true表示DHT22

    // 配置I2C总线用于OLED
    let scl = gpiob.pb8.into_alternate_open_drain::<4>();
    let sda = gpiob.pb9.into_alternate_open_drain::<4>();
    let i2c = I2c::i2c1(
        dp.I2C1,
        (scl, sda),
        400.kHz(),
        clocks,
    );

    // 配置OLED显示屏
    let interface = I2CDisplayInterface::new(i2c);
    let mut display = Ssd1306::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
        .into_buffered_graphics_mode();
    display.init().unwrap();
    display.clear();

    // 配置按钮
    let mut button = gpioa.pa0.into_pull_up_input();

    // 配置LED
    let mut led = gpioc.pc13.into_push_pull_output();

    // 初始化显示模式
    let mut mode = DisplayMode::Both;

    // 主循环
    loop {
        // 读取传感器数据
        led.set_high().unwrap(); // 指示开始读取
        match dht.read() {
            Ok(reading) => {
                led.set_low().unwrap(); // 指示读取成功

                // 清除显示
                display.clear();

                // 创建文本风格
                let text_style = MonoTextStyleBuilder::new()
                    .font(&FONT_6X10)
                    .text_color(BinaryColor::On)
                    .build();

                // 根据模式显示数据
                match mode {
                    DisplayMode::Temperature => {
                        let mut text: String<U64> = String::new();
                        write!(text, "Temp: {:.1}C", reading.temperature).unwrap();
                        Text::with_baseline(
                            &text,
                            Point::new(0, 16),
                            text_style,
                            Baseline::Top,
                        )
                        .draw(&mut display).unwrap();
                    },
                    DisplayMode::Humidity => {
                        let mut text: String<U64> = String::new();
                        write!(text, "Humidity: {:.1}%", reading.humidity).unwrap();
                        Text::with_baseline(
                            &text,
                            Point::new(0, 16),
                            text_style,
                            Baseline::Top,
                        )
                        .draw(&mut display).unwrap();
                    },
                    DisplayMode::Both => {
                        let mut temp_text: String<U64> = String::new();
                        let mut hum_text: String<U64> = String::new();
                        write!(temp_text, "Temp: {:.1}C", reading.temperature).unwrap();
                        write!(hum_text, "Humidity: {:.1}%", reading.humidity).unwrap();

                        Text::with_baseline(
                            &temp_text,
                            Point::new(0, 16),
                            text_style,
                            Baseline::Top,
                        )
                        .draw(&mut display).unwrap();

                        Text::with_baseline(
                            &hum_text,
                            Point::new(0, 32),
                            text_style,
                            Baseline::Top,
                        )
                        .draw(&mut display).unwrap();
                    }
                }

                // 更新显示
                display.flush().unwrap();
            },
            Err(_) => {
                // 读取失败，快速闪烁LED指示错误
                for _ in 0..5 {
                    led.set_high().unwrap();
                    delay.delay_ms(100_u32);
                    led.set_low().unwrap();
                    delay.delay_ms(100_u32);
                }
            }
        }

        // 检查按钮状态
        if button.is_low().unwrap() {
            // 按钮被按下，切换模式
            mode = match mode {
                DisplayMode::Temperature => DisplayMode::Humidity,
                DisplayMode::Humidity => DisplayMode::Both,
                DisplayMode::Both => DisplayMode::Temperature,
            };

            // 等待按钮释放
            while button.is_low().unwrap() {
                delay.delay_ms(10_u32);
            }

            // 防抖
            delay.delay_ms(50_u32);
        }

        // 等待下一个读取周期
        delay.delay_ms(2000_u32);
    }
}
```

### 烧录与运行

使用以下命令编译并烧录程序：

```bash
cargo run --release
```

### 项目扩展建议

1. **添加数据记录功能**：将数据保存到片上 Flash 或外部存储器
2. **添加蓝牙/WiFi 连接**：通过无线模块发送数据到智能手机或云服务
3. **增加节能模式**：实现低功耗睡眠模式延长电池寿命
4. **添加更多传感器**：如气压、光照或空气质量传感器
5. **实现校准功能**：允许用户校准传感器读数

### 实践思考

> **思考问题：** 在这个项目中，哪些设计决策体现了 Rust 在嵌入式系统中的优势？如何进一步提高系统的可靠性和实时性能？

这个项目利用了 Rust 的类型安全性和零成本抽象，特别是在传感器接口、I2C 通信和状态管理方面。进一步改进可以包括：使用 RTIC 框架实现基于优先级的任务调度、实现错误恢复机制、利用 const 泛型优化内存使用，以及添加单元测试验证关键功能。此外，通过静态分配替代动态内存分配可以提高实时性能和可预测性。

## 小结

本章介绍了 Rust 在嵌入式系统开发中的应用。通过学习这些内容，你现在应该能够：

- 理解裸机编程的基本概念和挑战
- 配置嵌入式 Rust 开发环境和工具链
- 使用 PAC 和 HAL 与嵌入式系统外设交互
- 编写安全的中断处理代码和实时系统
- 实现一个简单但功能完整的嵌入式项目

Rust 凭借其内存安全保证、零成本抽象和强大的类型系统，为嵌入式开发带来了前所未有的可靠性和生产力。虽然嵌入式 Rust 生态系统仍在发展中，但其已经成为开发可靠、高性能嵌入式应用的强大选择。

在下一章中，我们将探索 Rust 在 Web Assembly 领域的应用，这是另一个展示 Rust 跨平台能力的重要方向。

## 扩展阅读

想要深入了解本章内容，推荐以下资源：

1. [Embedded Rust Book](https://docs.rust-embedded.org/book/) - 官方嵌入式 Rust 文档
2. [rust-embedded/awesome-embedded-rust](https://github.com/rust-embedded/awesome-embedded-rust) - 嵌入式 Rust 资源集合
3. [RTIC 框架文档](https://rtic.rs) - 实时中断驱动并发框架
4. [probe-rs 项目](https://probe.rs/) - 现代嵌入式调试工具
5. [The Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) - 深入理解嵌入式 Rust 底层原理

# STM32F411RET6 CubeMX 引脚配置指南

> 论文：公路巡检智能机器人研究与设计  
> 主控：STM32F411RET6 (LQFP64)  
> 基于 CubeMX + HAL 库  
> 修订：HC-15B STA 引脚改接至 PC10（原论文 PB4 冲突，改留给后轮 IN2）

---

## 1. 时钟树 (Clock Configuration)

| 项 | 值 | 说明 |
|:---|:---|:---|
| HSE | 8MHz 晶振 | 主时钟源 |
| LSE | 32.768kHz 晶振 | RTC 时钟源 |
| PLL Source | HSE | — |
| SYSCLK | **84MHz** | 系统主频 |
| APB1 | 42MHz (Timer=84MHz) | 低速外设总线 |
| APB2 | 84MHz (Timer=84MHz) | 高速外设总线 |

---

## 2. RCC

| 配置项 | 值 |
|:---|---:|
| High Speed Clock (HSE) | Crystal/Ceramic Resonator |
| Low Speed Clock (LSE) | Crystal/Ceramic Resonator |

---

## 3. 各外设引脚配置

### 3.1 ADC1 — 四通道 DMA 扫描

**用途：** MQ-9 / MQ-135 / 火焰传感器 / 电池电压 模拟采样

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PA1 | ADC1_IN1 单端 | `ADC_MQ9` |
| PC0 | ADC1_IN10 单端 | `ADC_MQ135` |
| PC1 | ADC1_IN11 单端 | `ADC_FLAME` |
| PC2 | ADC1_IN12 单端 | `ADC_BAT` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| Scan Conversion Mode | **Enabled** |
| Continuous Conversion Mode | **Disabled** |
| Discontinuous Conversion Mode | **Disabled** |
| Number of Conversion | **4** |
| External Trigger Conversion Source | **Timer 2 Trigger Out event** |
| End of Conversion Selection | **EOC flag at the end of all conversions** |

**Rank 设置（4 个通道，各 12bit / 3 cycles）：**

| Rank | Channel | Sampling Time |
|:---:|:---|---:|
| 1 | IN1 (PA1) | 3 Cycles |
| 2 | IN10 (PC0) | 3 Cycles |
| 3 | IN11 (PC1) | 3 Cycles |
| 4 | IN12 (PC2) | 3 Cycles |

> **额定电压对应关系：**
> - MQ-9 (CO传感器): 0~5V → 2kΩ+3.3kΩ分压(0.622) → 0~3.3V
> - MQ-135 (NH₃/硫化物): 同上分压
> - 火焰传感器: 电压越低火焰越强
> - 电池分压: Vin × 3.3/(10+3.3) = Vin × 0.248，满电12.6V→3.12V

---

### 3.2 DMA2 — ADC 数据搬运

| 配置项 | 值 |
|:---|---:|
| DMA2 Stream0 | ADC1 |
| Mode | **Circular** |
| Direction | **Peripheral-to-Memory** |
| Data Width (Periph) | **Half Word (16bit)** |
| Data Width (Memory) | **Half Word (16bit)** |
| Increment Address (Memory) | **Enabled** |

---

### 3.3 TIM2 — ADC 触发源 (20Hz)

**用途：** 产生 20Hz 的 TRGO 事件触发 ADC 采样

| 参数 | 值 |
|:---|---:|
| Prescaler (PSC) | **41999** |
| Counter Mode | Up |
| Counter Period (ARR) | **99** |
| Auto-reload preload | **Enable** |
| Trigger Output (TRGO) | **Update Event** |

> 计算：`84MHz ÷ (41999+1) ÷ (99+1) = 20Hz`

---

### 3.4 TIM3 — 前轮 PWM (1kHz)

**用途：** 前轮电机 L298N 调速

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PA6 | TIM3_CH1 | `PWM_FRONT_ENA` |
| PA7 | TIM3_CH2 | `PWM_FRONT_ENB` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| Prescaler (PSC) | **0** |
| Counter Mode | Up |
| Counter Period (ARR) | **8399** |
| Auto-reload preload | **Enable** |
| CH1 Mode | PWM Mode 1 |
| CH1 Pulse | 0（代码动态调整） |
| CH1 Output Polarity | High |
| CH2 Mode | PWM Mode 1 |
| CH2 Pulse | 0（代码动态调整） |
| CH2 Output Polarity | High |

> 计算：`84MHz ÷ (0+1) ÷ (8399+1) = 1kHz`  
> 论文速度档位（CCR 值）：低速 2800 (33%), 中速 5000 (60%), 高速 7500 (89%)

---

### 3.5 TIM4 — 后轮 PWM (1kHz)

**用途：** 后轮电机 L298N 调速

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PB6 | TIM4_CH1 | `PWM_REAR_ENA` |
| PB7 | TIM4_CH2 | `PWM_REAR_ENB` |

**Parameter Settings:** 完全同 TIM3 (PSC=0, ARR=8399)

---

### 3.6 USART1 — HC-15B LoRa 通信

**用途：** 与基站 LoRa 通信（半双工透传）

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PA9 | USART1_TX | `LORA_TX` |
| PA10 | USART1_RX | `LORA_RX` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| Baud Rate | **9600** |
| Word Length | **8 bit (including Parity)** = 8 |
| Parity | **None** |
| Stop Bits | **1** |
| Mode | **Asynchronous** |
| NVIC → USART1 global interrupt | **Enabled** |

> **半双工处理：** HC-15B 收发共用，软件层控制收发切换。发送完后需等待模块空闲。

---

### 3.7 USART2 — ESP32-CAM 通信

**用途：** 控制 ESP32-CAM 拍照/休眠/唤醒

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PA2 | USART2_TX | `CAM_TX` |
| PA3 | USART2_RX | `CAM_RX` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| Baud Rate | **115200** |
| Word Length | **8 bit** |
| Parity | **None** |
| Stop Bits | **1** |
| Mode | **Asynchronous** |
| NVIC → USART2 global interrupt | **Enabled** |

---

### 3.8 USART6 — GP10-A GPS 接收

**用途：** 接收 GPS NMEA-0183 数据

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PC6 | USART6_TX | `GPS_TX` |
| PC7 | USART6_RX | `GPS_RX` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| Baud Rate | **9600** |
| Word Length | **8 bit** |
| Parity | **None** |
| Stop Bits | **1** |
| Mode | **Asynchronous** |
| NVIC → USART6 global interrupt | **✅ Enabled** |

---

### 3.9 I2C1 — AHT20 + BMP280 温湿度气压

**用途：** 读取环境温湿度和气压

| Pin | 功能 | 标签 |
|:---|:---|---:|
| PB8 | I2C1_SCL | `I2C_SCL` |
| PB9 | I2C1_SDA | `I2C_SDA` |

**Parameter Settings:**

| 参数 | 值 |
|:---|---:|
| I2C Speed Mode | **Standard Mode (100kHz)** |
| I2C peripheral | **Enabled** |

> AHT20 地址 0x38，BMP280 地址 0x77

---

### 3.10 RTC — 休眠唤醒

**用途：** 低功耗模式下定时唤醒

| 配置项 | 值 |
|:---|---:|
| Activate Clock Source | **LSE (32.768kHz)** |
| Activate Calendar | **Enabled** |
| Wake Up | **Internal Wake Up** |
| Wake Up Clock | **RTC_WAKEUPCLOCK_CK_SPRE_16BIT** ≈ 1Hz |
| NVIC → RTC_WKUP_IRQn | **✅ Enabled** |

> 默认唤醒周期 30 分钟由代码设定
> RTC 闹钟唤醒后，系统从 STOP 模式退出，执行快速初始化后恢复巡检

---

## 4. GPIO 引脚配置

### 4.1 报警输出

| Pin | 配置 | 方向 | 输出类型 | 上下拉 | 初始电平 | 标签 | 说明 |
|:---|:---|---:|:---:|:---:|:---:|:---:|---|
| PB0 | GPIO_Output | Output | Push-Pull | No pull | Low | `BUZZER` | 蜂鸣器，经 S8050 NPN 驱动，高电平导通 |
| PB1 | GPIO_Output | Output | Push-Pull | No pull | Low | `LED_RED` | 红色 LED，4.7kΩ 限流 |
| PC5 | GPIO_Output | Output | Push-Pull | No pull | Low | `LED_BLUE` | 蓝色 LED，4.7kΩ 限流 |

### 4.2 电机方向控制

| Pin | 配置 | 方向 | 输出类型 | 标签 |
|:---|:---|---:|:---:|:---:|
| PB12 | GPIO_Output | Output | Push-Pull | `FRONT_IN1` |
| PB13 | GPIO_Output | Output | Push-Pull | `FRONT_IN2` |
| PB14 | GPIO_Output | Output | Push-Pull | `FRONT_IN3` |
| PB15 | GPIO_Output | Output | Push-Pull | `FRONT_IN4` |
| PB3 | GPIO_Output | Output | Push-Pull | `REAR_IN1` |
| PB4 | GPIO_Output | Output | Push-Pull | `REAR_IN2` |
| PB5 | GPIO_Output | Output | Push-Pull | `REAR_IN3` |
| PD2 | GPIO_Output | Output | Push-Pull | `REAR_IN4` |

> 电机方向控制真值表：
> | IN1 | IN2 | 前轮左 | IN3 | IN4 | 前轮右 |
> |:---:|:---:|:---:|:---:|:---:|:---:|
> | 1 | 0 | 正转 | 1 | 0 | 正转 |
> | 0 | 1 | 反转 | 0 | 1 | 反转 |
> | 0 | 0 | 停止 | 0 | 0 | 停止 |
>
> 后轮（PB3/PB4/PB5/PD2）同理。

### 4.3 红外对接

**对接原理：** 小车前后共对称装配 **4 只红外接收管** + 基站 4 只红外发射管。
NPN 三极管放大电路：无光时 GPIO 为高电平，**检测到红外光时 GPIO 被拉低**。

| Pin | 配置 | 方向 | 上下拉 | 标签 | 安装位置 |
|:---|:---|---:|:---:|:---:|:---:|
| PC8 | GPIO_Output | Output | No pull | `IR_TX_CTRL` — 控制基站发射管通断（基站端） | 基站 |
| PA8 | GPIO_Input | — | **Pull-down** | `IR_RX1` — 1号接收管 | 车前部左侧 |
| PC9 | GPIO_Input | — | **Pull-down** | `IR_RX2` — 2号接收管 | 车前部右侧 |
| **PA4** ⚠️ | GPIO_Input | — | **Pull-down** | `IR_RX3` — 3号接收管 | 车后部左侧 |
| **PA5** ⚠️ | GPIO_Input | — | **Pull-down** | `IR_RX4` — 4号接收管 | 车后部右侧 |

> ⚠️ **PA4、PA5** 为新增引脚，用于接入第 3、4 路红外接收管（论文原设计 4 管）。
> 对接判定：**4 路中 ≥3 路检测到低电平** → 对接成功。

---

### 4.4 LoRa 控制引脚

| Pin | 配置 | 方向 | 输出/输入类型 | 上下拉 | 标签 |
|:---|:---|---:|:---:|:---:|:---:|
| PB2 | GPIO_Output | Output | Push-Pull | No pull | `LORA_KEY` — HC-15B KEY 脚，10kΩ 外部上拉到 3.3V，拉低进入 AT 模式 |
| PB10 | GPIO_Output | Output | Push-Pull | No pull | `LORA_AUX` — 唤醒脚，10kΩ 外部上拉到 3.3V |
| **PC10** ⚠️ | **GPIO_Input** | Input | — | **Pull-up** | `LORA_STA` — HC-15B 模块状态检测，高电平=空闲，低电平=忙碌 |

> ⚠️ **与论文差异：** 原论文 PB4 被同时分配给 LORA_STA 和后轮 IN2。此处改 PB4→后轮 IN2，LORA_STA→**PC10**。

### 4.5 GPS 控制

| Pin | 配置 | 方向 | 输出/输入类型 | 标签 |
|:---|:---|---:|:---:|:---:|
| PB2 | GPIO_Output | Output | Push-Pull | `GPS_WAKE` — GP10-A 唤醒/休眠控制，高电平=全工作，低电平=休眠 |
| PB10 | GPIO_Input | — | — | `GPS_1PPS` — 秒脉冲（可选） |

---

## 5. NVIC 中断优先级

| 中断 | 使能 | Preemption Priority | Sub Priority | 说明 |
|:---:|:---:|:---:|:---:|---|
| TIM2 global | ✅ | 1 | 0 | ADC 触发（20Hz 时基） |
| USART1 global | ✅ | 2 | 0 | LoRa 接收 |
| USART2 global | ✅ | 2 | 0 | ESP32-CAM 接收 |
| USART6 global | ✅ | 2 | 0 | GPS 数据接收 |
| RTC_WKUP | ✅ | 3 | 0 | 休眠唤醒 |
| DMA2 Stream0 | ✅ | 3 | 0 | ADC DMA 传输完成 |
| TIM3 global | ❌ | — | — | PWM 无需中断 |
| TIM4 global | ❌ | — | — | PWM 无需中断 |

---

## 6. 生成代码前检查清单

- [ ] 时钟树：HSE 8MHz + LSE 32.768kHz, SYSCLK=84MHz
- [ ] ADC1: 四通道 DMA + TIM2 TRGO 触发
- [ ] DMA2 Stream0: Circular, HalfWord
- [ ] TIM2: PSC=41999, ARR=99, TRGO=Update
- [ ] TIM3: PSC=0, ARR=8399, CH1/CH2=PWM1
- [ ] TIM4: PSC=0, ARR=8399, CH1/CH2=PWM1
- [ ] USART1: 9600-8-N-1, RX 中断
- [ ] USART2: 115200-8-N-1, RX 中断
- [ ] USART6: 9600-8-N-1, RX 中断
- [ ] I2C1: Standard 100kHz
- [ ] RTC: LSE, WakeUp IRQ
- [ ] GPIO: 见第 4 节全部配完
- [ ] 项目名称 → 任意，如 `patrol_car`
- [ ] Toolchain → **SW4STM32** 或 **Makefile**（配合 ARM GCC + VSCode EIDE）
- [ ] Generate Code

---

## 附录：引脚功能汇总表

| Pin | 功能 | 方向 | 标签 |
|:---:|:---:|:---:|:---:|
| PA1 | ADC1_IN1 | 输入 | `ADC_MQ9` |
| PA2 | USART2_TX | 输出 | `CAM_TX` |
| PA3 | USART2_RX | 输入 | `CAM_RX` |
| PA4 | GPIO_IN (下拉) | 输入 | `IR_RX3` |
| PA5 | GPIO_IN (下拉) | 输入 | `IR_RX4` |
| PA6 | TIM3_CH1 | 输出 | `PWM_FRONT_ENA` |
| PA7 | TIM3_CH2 | 输出 | `PWM_FRONT_ENB` |
| PA8 | GPIO_IN | 输入 | `IR_RX1` |
| PA9 | USART1_TX | 输出 | `LORA_TX` |
| PA10 | USART1_RX | 输入 | `LORA_RX` |
| PB0 | GPIO_OUT | 输出 | `BUZZER` |
| PB1 | GPIO_OUT | 输出 | `LED_RED` |
| PB2 | GPIO_OUT | 输出 | `LORA_KEY` |
| PB3 | GPIO_OUT | 输出 | `REAR_IN1` |
| PB4 | GPIO_OUT | 输出 | `REAR_IN2` |
| PB5 | GPIO_OUT | 输出 | `REAR_IN3` |
| PB6 | TIM4_CH1 | 输出 | `PWM_REAR_ENA` |
| PB7 | TIM4_CH2 | 输出 | `PWM_REAR_ENB` |
| PB8 | I2C1_SCL | 双向 | `I2C_SCL` |
| PB9 | I2C1_SDA | 双向 | `I2C_SDA` |
| PB10 | GPIO_IN | 输入 | `GPS_1PPS` |
| PB12 | GPIO_OUT | 输出 | `FRONT_IN1` |
| PB13 | GPIO_OUT | 输出 | `FRONT_IN2` |
| PB14 | GPIO_OUT | 输出 | `FRONT_IN3` |
| PB15 | GPIO_OUT | 输出 | `FRONT_IN4` |
| PC0 | ADC1_IN10 | 输入 | `ADC_MQ135` |
| PC1 | ADC1_IN11 | 输入 | `ADC_FLAME` |
| PC2 | ADC1_IN12 | 输入 | `ADC_BAT` |
| PC5 | GPIO_OUT | 输出 | `LED_BLUE` |
| PC6 | USART6_TX | 输出 | `GPS_TX` |
| PC7 | USART6_RX | 输入 | `GPS_RX` |
| PC8 | GPIO_OUT | 输出 | `IR_TX` |
| PC9 | GPIO_IN | 输入 | `IR_RX2` |
| **PC10** ⚠️ | GPIO_IN (上拉) | 输入 | `LORA_STA` |
| PD2 | GPIO_OUT | 输出 | `REAR_IN4` |

> ⚠️ **PC10** 为本文档修正引脚，避开原论文 PB4 冲突。

---

*文档版本: v1.0*  
*日期: 2026-06-01*

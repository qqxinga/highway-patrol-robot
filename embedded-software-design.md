# 公路巡检智能机器人 — 嵌入式软件设计文档

> 对应论文：公路巡检智能机器人研究与设计  
> 主控芯片：STM32F411RET6  
> 开发框架：STM32 HAL 库 (CubeMX 生成)  
> 文档版本：v1.0

---

## 1. 软件架构

### 1.1 前后台系统

| 层级 | 角色 | 机制 |
|:---:|---|---|
| **前台 (中断)** | 定时基准、数据采集、通信接收 | TIM2 20Hz 中断设 `g_tick_20hz` 标志 |
| **后台 (主循环)** | 状态机调度、数据处理、控制输出 | main loop 轮询标志，依次执行各模块 |

### 1.2 系统时基

| 时钟源 | 频率 | 用途 |
|:---:|:---:|---|
| TIM2 TRGO | **20 Hz** (PSC=41999, ARR=99) | ADC 触发、系统调度 tick |
| TIM3 / TIM4 | **1 kHz** (PSC=0, ARR=8399) | 前后轮 PWM 输出 |
| LSE + RTC | 32.768 kHz | 休眠唤醒 (30min) |

---

## 2. 六状态机

### 2.1 状态定义

```
                    ┌──────────┐
                    │  INIT    │ ◄── 上电/复位
                    └────┬─────┘
                         │ 初始化完成
                    ┌────▼─────┐
              ┌─────│  PATROL  │◄───────────────────────────┐
              │     └────┬─────┘                            │
              │          │ 传感器超标 / LoRa指令            │
              │     ┌────▼───────┐                          │
              │     │  EMERGENCY │─── 执行完毕 ──── 电量>50% ─┘
              │     └────┬───────┘                          │
              │          │ 拍照返航完成, 电量<50%            │
              │     ┌────▼───────┐                          │
              ├─────│  CHARGING  │─── 充满 ──────────────────┘
              │     └────┬───────┘
              │          │ 充满且无任务
              │     ┌────▼─────┐
              └─────│  SLEEP   │─── RTC 30min 唤醒 ──→ PATROL
                    └──────────┘

  ALERT 为 PATROL/EMERGENCY 内部子状态，不独立切换。
```

### 2.2 各状态说明

| 状态 | 入口 | 行为 | 出口条件 |
|:---:|---|---|---|
| **INIT** | 上电/复位 | GPIO/TIM/ADC/DMA/I2C/UART 初始化 → AHT20 校准 → GPS 等待定位/超时 → 切 PATROL | 初始完成 / 定位超时（30s） |
| **PATROL** | 默认工作态 | 20Hz 传感器采集 → 阈值判定 → 每 30s LoRa 上报 STATUS → 正常行驶 | 超标→EMERGENCY / 低电量→CHARGING / 无任务→SLEEP |
| **EMERGENCY** | 紧急响应 | 见第 2.3 节 | 执行完毕 |
| **ALERT** | 报警子状态 | 蜂鸣器+LED 闪 → 拍照存 SD → LoRa 发 ALARM 帧 → 持续 60s | 异常消除保持 30s→退出 |
| **CHARGING** | 充电 | 关闭电机 → 监测电压 → 充满判定 | 满电→PATROL 或 SLEEP |
| **SLEEP** | 休眠 | 关闭外设时钟 → STOP 模式 → RTC 30min → 唤醒 | RTC 中断 → 快速初始化 → PATROL |

### 2.3 EMERGENCY 子流程

| Phase | 动作 | 触发条件 | 详细 |
|:---:|---|---|---|
| **① 赶路** | PWM 90% 全速前进 | 进入 EMERGENCY | GPS 更新频率提至 5Hz，持续计算距目标距离 |
| **② 🚨 报警** | 蜂鸣器 + 红蓝 LED 闪烁 | 距目标 **≤ 50m** | 警示后方来车，持续到 Phase ④ 结束 |
| **③ 📸 到达** | 制动停车，连拍多张照片→SD 卡 | 距目标 **< 10m** | 发送 EMERGENCY_RESP 帧告知基站"已到达" |
| **④ 返航** | 返回基站 WiFi 覆盖 | 拍照完成 | 进入基站 WiFi 范围后上传照片到数据中心 |
| **⑤ 决断** | 判断电量 | 上传完成 | Vbat ≥ 11.0V → PATROL；Vbat < 11.0V → CHARGING |

### 2.4 EMERGENCY 触发条件（OR 关系）

| 触发源 | 判定条件 |
|:---|---:|
| **LoRa 命令** | 收到 `EMERGENCY_TASK (0x07)` 帧，含目标经纬度 |
| **火情自检** | 火焰传感器 ADC 连续 3 次采样 < 1.0V + GPS 有效定位 |
| **复合异常** | 任意 2 个传感器同时超标，持续 3 次采样 |

---

## 3. 传感器驱动

### 3.1 ADC 多通道 DMA 扫描

**硬件配置：**

| 通道 | Pin | 信号源 | 分压 | ADC 分辨率 |
|:---:|:---:|---|---:|---:|
| ADC1_IN1 | PA1 | MQ-9 (CO) | 2k+3.3k (0.622) | 12bit |
| ADC1_IN10 | PC0 | MQ-135 (NH₃/硫化物) | 2k+3.3k (∝0.622) | 12bit |
| ADC1_IN11 | PC1 | 火焰传感器 | 直连 | 12bit |
| ADC1_IN12 | PC2 | 电池电压 (Vbat×0.248) | 10k+3.3k (0.248) | 12bit |

**软件处理：**

- 触发：TIM2 TRGO → 20Hz
- 搬运：DMA2 Stream0 Circular → `adc_raw[4]` (Half Word)
- 滤波：滑动平均，窗口宽度 **8 次**
- 就绪标志：DMA 传输完成中断置位 `g_adc_ready`

**数值还原公式：**

```
V_sensor = adc_raw × (3.3 / 4096)   // ADC 引脚处的电压
V_mq9    = V_sensor / 0.622          // 还原分压前电压
V_bat    = V_sensor / 0.248          // 还原电池电压
```

### 3.2 AHT20 + BMP280 (I2C1)

**总线参数：** 100kHz Standard Mode

| 器件 | 地址 | 测量值 | 精度 |
|:---:|:---:|---|---|
| AHT20 | 0x38 | 温度 -40~+85°C / 湿度 0~100%RH | ±0.3°C / ±2%RH |
| BMP280 | 0x77 | 气压 300~1100hPa | ±1hPa |

**AHT20 初始化序列：**

```
上电等待 40ms → 发送 [0xBE, 0x08, 0x00] 校准
触发测量 [0xAC, 0x33, 0x00] → 等待 80ms → 读取 6 字节
```

**BMP280 流程：**

```
读取 12 字节校准系数 → 等待 9ms 转换 → 读取 0xF7~0xFC (6 字节原始值)
→ 代入补偿算法 → 温度/气压
```

### 3.3 GP10-A GPS (USART6)

**串口参数：** 9600-8-N-1, RX 中断

**数据流：** UART RX 中断 → 环形缓冲区 → '\n' 触发解析

**解析语句：**

| 语句 | 提取字段 |
|:---|---:|
| **$GPGGA** | UTC 时间、纬度 (ddmm.mmmm→dd.dddd)、经度、定位质量、卫星数、海拔 |
| **$GPRMC** | 对地速度、航向角 |

**异常检测：** 连续 3s 定位质量=0 → 置位 `gps_lost` 标志 → 触发 LoRa 报警

### 3.4 报警阈值

| 传感器 | 报警阈值 | 恢复条件 |
|:---|---:|---:|
| MQ-9 (CO) | ADC 还原 ≥ **2000 ppm** | < 阈值保持 30s |
| MQ-135 (NH₃/硫化物) | ADC 电压 ≥ **2.0 V** | < 阈值保持 30s |
| 火焰传感器 | ADC 电压 < **1.0 V** | ≥ 阈值保持 30s |
| 电池电压 | **< 10.5 V** | ≥ 10.5V |

---

## 4. 电机控制

### 4.1 PWM 参数

| 参数 | 值 |
|:---|---:|
| 频率 | **1 kHz** (ARR=8399) |
| 低速档 | CCR = **2800** (≈33%) |
| 中速档 | CCR = **5000** (≈60%) |
| 高速档 | CCR = **7500** (≈89%) |
| EMERGENCY 档 | CCR = **7500** (≈89%) |

### 4.2 电池电压前馈补偿

```
CCR_comp = (int)(CCR_base × (12.6f / Vbat))
```

**钳位逻辑：**
```c
if (CCR_comp > 7500) CCR_comp = 7500;  // 不超高速档
if (CCR_comp < 2800) CCR_comp = 2800;  // 不低于低速档
```

**当 Vbat < 10.5V：** 补偿 + 触发低电量报警

### 4.3 电机方向控制 (L298N)

| 前轮 IN1( PB12) | IN2(PB13) | 前轮左 | IN3(PB14) | IN4(PB15) | 前轮右 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | 0 | 正转 | 1 | 0 | 正转 |
| 0 | 1 | 反转 | 0 | 1 | 反转 |
| 0 | 0 | 停止 | 0 | 0 | 停止 |

> 后轮同理：REAR_IN1(PB3)、IN2(PB4)、IN3(PB5)、IN4(PD2)

---

## 5. LoRa 通信协议 (HC-15B)

### 5.1 串口参数

| 项 | 值 |
|:---|---:|
| 波特率 | **9600** |
| 数据位 | 8, None, 1 |
| 半双工 | TX/RX 共用，软件切换 |
| STA 检测 | PC10 (高电平=空闲，低电平=忙碌) |
| KEY | PB2（高电平=透传，拉低=AT 模式） |
| AUX | PB10（拉低唤醒模块） |

### 5.2 收发时序

```
TX 时隙 (500ms)          RX 时隙 (500ms)
┌────────────────────┐  ┌────────────────────┐
│ 检测 STA=空闲       │  │ 等待接收           │
│ 发送数据帧          │  │ CRC 校验            │
│ 等待 ACK (2~8s)    │  │ 回复 ACK            │
│ STA=忙碌? → 等待     │  │                     │
└────────────────────┘  └────────────────────┘
         ↑ 每 1s 轮换 ↑
```

### 5.3 帧结构

| 字段 | 帧头 | 帧类型 | 数据长度 | 数据域 | CRC16 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 字节数 | 2 | 1 | 1 | N (≤236) | 2 |
| 内容 | 0xAA 0x55 | TYPE | LEN | DATA | CRC16-IBM |

### 5.4 帧类型

| TYPE | 名称 | 方向 | 说明 | ACK 要求 |
|:---:|:---:|:---:|---|---:|
| 0x01 | STATUS | 车→基站 | 周期性传感器状态上报 (30s) | ❌ |
| 0x02 | ALARM | 车→基站 | 异常报警 (定位+类型+数值) | ✅ 必须 ACK |
| 0x03 | CTRL | 基站→车 | 启动/停止/拍照/休眠/唤醒 | ❌ |
| 0x04 | IMAGE_NOTIFY | 车→基站 | 通知基站有待传照片 | ❌ |
| 0x05 | CHARGE_STATUS | 车→基站 | 充电状态上报 | ❌ |
| 0x06 | ACK | 双向 | 确认应答 | — |
| 0x07 | EMERGENCY_TASK | 基站→车 | 紧急任务 (目标纬度4B+经度4B+优先级1B) | ✅ 必须 ACK |
| 0x08 | EMERGENCY_RESP | 车→基站 | 任务响应 (到达/拍照完成/失败) | ❌ |

### 5.5 重传机制 (指数退避)

```
ALARM / EMERGENCY_TASK 帧:
  第 1 次: 等待 2s → 未收到 ACK → 重传
  第 2 次: 等待 4s → 未收到 ACK → 重传
  第 3 次: 等待 8s → 未收到 ACK → 放弃，记录通信故障

STATUS / 其他帧: 不要求 ACK，不重传
```

### 5.6 CRC16 校验

- 算法：**CRC16-IBM** (多项式 x¹⁶+x¹⁵+x²+1, 初始值 0xFFFF)
- 实现：**256 字节查表法**（低 MCU 开销）
- 范围：从帧头 0xAA 到数据域结束
- 比对：接收方计算结果与帧末尾 2 字节 CRC 比较，不匹配则丢弃

---

## 6. ESP32-CAM 控制 (USART2)

**串口参数：** 115200-8-N-1

**指令协议：**

| 命令 | 方向 | 说明 |
|:---:|:---:|---|
| `SNAP` | STM32→CAM | 触发拍照，返回 JPEG 数据 |
| `SLEEP` | STM32→CAM | 进入深度睡眠 (~6mA) |
| `WAKE` | STM32→CAM | 唤醒模块 |

**使用场景：**
- ALERT 触发时 → 发 `SNAP` → 照片存 SD 卡
- EMERGENCY Phase ③ → 发 `SNAP` 连续多张
- 返航到基站 WiFi 范围 → 通过 WiFi 上传照片

---

## 7. 红外对接与充电控制

### 7.1 硬件连接

| 信号 | Pin | 说明 |
|:---:|:---:|---|
| IR_TX_CTRL | **PC8** | 基站红外发射管控制（小车端控制基站） |
| IR_RX1 | **PA8** | 车前部左侧接收管 |
| IR_RX2 | **PC9** | 车前部右侧接收管 |
| IR_RX3 | **PA4** | 车后部左侧接收管 |
| IR_RX4 | **PA5** | 车后部右侧接收管 |

### 7.2 对接判定

```c
// 红外接收管：检测到红外光时 GPIO 被拉低
uint8_t ir_count = 0;
if (!HAL_GPIO_ReadPin(IR_RX1_GPIO_Port, IR_RX1_Pin)) ir_count++;
if (!HAL_GPIO_ReadPin(IR_RX2_GPIO_Port, IR_RX2_Pin)) ir_count++;
if (!HAL_GPIO_ReadPin(IR_RX3_GPIO_Port, IR_RX3_Pin)) ir_count++;
if (!HAL_GPIO_ReadPin(IR_RX4_GPIO_Port, IR_RX4_Pin)) ir_count++;

if (ir_count >= 3) {
    // 对接成功 → 制动停车 → 切 CHARGING
}
```

### 7.3 充电流程

```
对接成功 → 关闭 L298N 使能 → LoRa 发送 CHARGE_STATUS(对接成功)
→ 基站开启无线充电 → 持续监测 Vbat
→ Vbat ≥ 12.6V → 充满 → 根据指令切 PATROL 或 SLEEP
```

---

## 8. 低功耗管理

### 8.1 SLEEP 状态行为

```
进入 SLEEP:
  1. 关闭 L298N 电机驱动使能
  2. 关闭 ESP32-CAM（已发 SLEEP 指令）
  3. 关闭 GPS（拉低 WAKE 引脚）
  4. 失能 USART1/USART2 外设时钟
  5. 设置 RTC 30min 唤醒
  6. 进入 STOP 模式

RTC 唤醒:
  1. 恢复外设时钟
  2. 快速初始化传感器
  3. 切 PATROL
```

### 8.2 唤醒周期

| 条件 | 休眠时长 |
|:---:|:---:|
| 默认（无异常、电量充足） | **30 min** 固定 |
| 电量过低 | 仍然 30min，但每次唤醒检测到低电量自动回充 |

---

## 9. 报警 (ALERT)

### 9.1 触发条件

进入 ALERT 的触发条件已在 EMERGENCY 部分定义（见 2.4）。EMERGENCY 在赶路阶段的 Phase ② 也会启用报警。

### 9.2 报警行为

| 动作 | 说明 |
|:---:|---|
| **蜂鸣器** | PB0 输出 1kHz 方波 |
| **LED 红蓝交替闪烁** | PB1(LED_RED) 和 PC5(LED_BLUE) 交替高电平，周期 500ms |
| **拍照** | 向 ESP32-CAM 发 `SNAP` 指令，照片存 SD 卡 |
| **LoRa 报警** | 发送 ALARM(0x02) 帧，含定位+异常类型+数值，高优先级 |
| **持续时长** | 60s，若异常消除后保持 30s 正常值 → 退出 ALERT |

---

## 10. 文件模块划分

```
Core/
├── Inc/
│   ├── main.h             全局宏定义、引脚映射、状态枚举
│   ├── sensor.h           ADC + I2C 传感器驱动接口
│   ├── gps.h              NMEA 解析接口
│   ├── motor.h            PWM + 电压补偿接口
│   ├── lora.h             LoRa 帧协议接口
│   ├── camera.h           ESP32-CAM 控制接口
│   ├── charger.h          红外对接 + 充电接口
│   ├── power.h            低功耗管理接口
│   └── alarm.h            报警控制接口
│
└── Src/
    ├── main.c             主循环 + 状态机调度
    ├── sensor.c           ADC DMA / I2C 传感器驱动实现
    ├── gps.c              NMEA 环形缓冲区 + 解析实现
    ├── motor.c            PWM + 电压补偿实现
    ├── lora.c             CRC16 / 帧封装/解析 / 收发控制
    ├── camera.c           UART 指令发送实现
    ├── charger.c          红外对接 + 充电逻辑
    ├── power.c            RTC 唤醒 / 时钟门控实现
    └── alarm.c            蜂鸣器 / LED / LoRa 报警实现
```

---

## 11. 全局 tick 与调度

```c
// main.c 中定义
volatile uint32_t g_tick_20hz = 0;   // TIM2 中断中 ++
volatile uint32_t g_tick_1khz = 0;   // TIM3/TIM4（可选）
volatile uint8_t  g_adc_ready = 0;    // DMA 完成中断置位
volatile uint8_t  g_lora_rx_flag = 0; // USART1 RX 完成置位
volatile uint8_t  g_gps_rx_flag = 0;  // USART6 RX 完成置位

// 软件定时器（在 main loop 20Hz 中计数）
static uint32_t s_lora_tx_counter = 0;  // 30s = 600 ticks

void main_loop_20hz(void) {
    if (g_adc_ready) {
        sensor_process_adc();     // 滑动平均 + 阈值判定
        g_adc_ready = 0;
    }

    // 30s LoRa STATUS 上报
    s_lora_tx_counter++;
    if (s_lora_tx_counter >= 600) {
        s_lora_tx_counter = 0;
        if (current_state == PATROL) {
            lora_send_status();
        }
    }

    // Lora 半双工 1s 切片
    lora_timeslice();

    // 状态机执行
    state_machine_run();
}
```

---

## 12. 状态机调度伪代码

```c
typedef enum {
    STATE_INIT,
    STATE_PATROL,
    STATE_EMERGENCY,
    STATE_CHARGING,
    STATE_SLEEP
} robot_state_t;

robot_state_t current_state = STATE_INIT;

void state_machine_run(void) {
    switch (current_state) {
        case STATE_INIT:
            // 执行一次初始化
            if (init_complete) {
                current_state = STATE_PATROL;
            }
            break;

        case STATE_PATROL:
            // 采集 → 判定阈值 → 上报
            // 检测异常 → ALERT
            // 检测低电量 → CHARGING
            // 检测空闲 → SLEEP
            break;

        case STATE_EMERGENCY:
            emergency_execute_phases();
            break;

        case STATE_CHARGING:
            charger_monitor();
            break;

        case STATE_SLEEP:
            power_enter_stop();
            // RTC 唤醒后恢复
            break;
    }
}
```

---

## 13. 关键全局宏

```c
// === 系统 ===
#define SYS_TICK_HZ             20
#define SYS_ADC_WINDOW          8           // 滑动平均窗口

// === 电机 ===
#define PWM_ARR                 8399
#define PWM_SPEED_LOW           2800        // ≈33%
#define PWM_SPEED_MED           5000        // ≈60%
#define PWM_SPEED_HIGH          7500        // ≈89%
#define PWM_SPEED_EMERGENCY     7500
#define PWM_CCR_MIN             2800
#define PWM_CCR_MAX             7500
#define BAT_REF_VOLTAGE         12.6f       // 满电参考值
#define BAT_LOW_THRESHOLD       10.5f       // 低电量阈值

// === 传感器阈值 ===
#define MQ9_ALARM_THRESHOLD     2000.0f     // ppm
#define MQ135_ALARM_THRESHOLD   2.0f        // V (分压后ADC电压)
#define FLAME_ALARM_THRESHOLD   1.0f        // V (ADC电压)
#define BAT_ALARM_THRESHOLD     10.5f       // V

// === LoRa ===
#define LORA_TX_INTERVAL_MS     30000       // STATUS 上报周期
#define LORA_RETRY_1_MS         2000
#define LORA_RETRY_2_MS         4000
#define LORA_RETRY_3_MS         8000
#define LORA_RETRY_MAX          3
#define LORA_ACK_TIMEOUT_MS     2000
#define LORA_STA_BUSY_TIMEOUT_MS 500

// === EMERGENCY ===
#define EMERGENCY_ALERT_DIST    50          // 米，开始报警
#define EMERGENCY_ARRIVE_DIST   10          // 米，到达停车拍照
#define EMERGENCY_SPEED_CCR     7500        // 全速

// === 充电 ===
#define CHARGING_FULL_VOLTAGE   12.6f
#define IR_DETECT_COUNT         3           // ≥3 路触发判定成功

// === 休眠 ===
#define SLEEP_WAKEUP_INTERVAL   1800        // RTC 1Hz × 1800 = 30min
#define GPS_LOST_TIMEOUT_MS     3000        // 3s 无定位
#define ALERT_DURATION_MS       60000       // 报警持续
#define ALERT_RECOVER_MS        30000       // 报警恢复保持

// === 报警类型码 ===
#define ALARM_TYPE_GAS          0x01
#define ALARM_TYPE_FIRE         0x02
#define ALARM_TYPE_LOW_BAT      0x03
#define ALARM_TYPE_GPS_LOST     0x04
```

---

---

## 14. LoRa 帧数据域详细排布

### 通用约定

- **字节序：** 大端 (Big Endian)
- **浮点数：** IEEE 754 32bit float，4 字节大端
- **GPS 经纬度：** 十进制度 (float)

### 14.1 STATUS (0x01) — 车→基站

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 2 | 电池电压 | uint16 | Vbat × 100，如 1260 = 12.60V |
| 2 | 2 | MQ-9 | uint16 | ADC 原始值 (0~4095) |
| 4 | 2 | MQ-135 | uint16 | ADC 原始值 (0~4095) |
| 6 | 1 | 火焰传感器 | uint8 | ADC 原始值高 8 位 (0~255) |
| 7 | 2 | 温度 | int16 | °C × 100，如 2560 = 25.60°C |
| 9 | 2 | 湿度 | uint16 | %RH × 100，如 6550 = 65.50% |
| 11 | 2 | 气压 | uint16 | hPa × 100，如 101325 = 1013.25hPa |
| 13 | 4 | 纬度 | float | 十进制度 |
| 17 | 4 | 经度 | float | 十进制度 |
| 21 | 1 | 系统状态 | uint8 | Bit0:GPS有效 / Bit1:充电中 / Bit2:低电量 / Bit3:LoRa链路正常 |
| **22** | — | **总计** | | |

### 14.2 ALARM (0x02) — 车→基站

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 1 | 报警类型码 | uint8 | 0x01=气体 / 0x02=火焰 / 0x03=低电量 / 0x04=GPS失锁 |
| 1 | 2 | 报警值 | uint16 | 触发报警时的传感器值 |
| 3 | 4 | 纬度 | float | 报警位置 |
| 7 | 4 | 经度 | float | 报警位置 |
| 11 | 1 | GPS 质量 | uint8 | 0=无效 / 1=单点 / 2=差分 |
| **12** | — | **总计** | | |

### 14.3 CTRL (0x03) — 基站→车

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 1 | 指令码 | uint8 | 0x01=启动 / 0x02=停止 / 0x03=拍照 / 0x04=休眠 / 0x05=唤醒 / 0x06=切充电 |
| **1** | — | **总计** | | |

### 14.4 IMAGE_NOTIFY (0x04) — 车→基站

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 4 | 图片数量 | uint32 | 待上传图片张数 |
| 4 | 4 | 图片总大小 | uint32 | 字节总和 |
| 8 | 1 | 最高紧急度 | uint8 | 0=普通 / 1=紧急 |
| **9** | — | **总计** | | |

### 14.5 CHARGE_STATUS (0x05) — 车→基站

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 1 | 充电状态码 | uint8 | 0x01=对接成功 / 0x02=充电中 / 0x03=充满 / 0x04=对接失败 |
| 1 | 2 | 当前电压 | uint16 | Vbat × 100 |
| **3** | — | **总计** | | |

### 14.6 ACK (0x06) — 双向

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 1 | 应答类型 | uint8 | 原帧的 TYPE 值 |
| 1 | 1 | 结果 | uint8 | 0x00=成功 / 0x01=失败 |
| 2 | 2 | 保留 | — | 预留 |
| **4** | — | **总计** | | |

### 14.7 EMERGENCY_TASK (0x07) — 基站→车

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 4 | 目标纬度 | float | 十进制度 |
| 4 | 4 | 目标经度 | float | 十进制度 |
| 8 | 1 | 优先级 | uint8 | 0=普通 / 1=紧急 / 2=最高 |
| **9** | — | **总计** | | |

### 14.8 EMERGENCY_RESP (0x08) — 车→基站

| 偏移 | 大小 | 字段 | 类型 | 说明 |
|:---:|:---:|---|---|---|
| 0 | 1 | 执行状态 | uint8 | 0x01=到达 / 0x02=拍照完成 / 0x03=失败:无法到达 / 0x04=失败:拍照失败 / 0x05=返航中 |
| 1 | 4 | 当前纬度 | float | 当前 GPS 位置 |
| 5 | 4 | 当前经度 | float | 当前 GPS 位置 |
| **9** | — | **总计** | | |

---

## 15. ESP32-S3 基站边缘网关软件

### 15.1 系统架构

```
┌────────────────────────────────────────────────────────────────┐
│                     ESP32-S3                                   │
│                                                                │
│  ┌──────────┐   UART1     ┌────────────┐  RingBuf ┌───┴──────┐│
│  │  HC-15B  │◄──────────► │  Task A    │─────────►│          ││
│  │  (LoRa)  │  9600bps    │ LoRa接收   │lora_rb[] │ Task B   ││
│  └──────────┘             └────────────┘          │USB CDC   ││
│                                                    │转发      ││
│  ┌──────────┐   WiFi     ┌────────────┐ JPEG data │JSON行封装││
│  │ESP32-CAM │◄──────────►│ TCP Server │──────────►│tud_cdc_  ││
│  │ (小车)   │   SoftAP   │ Port 8080  │buf_a/buf_b│write()   ││
│  └──────────┘            └────────────┘           └─────┬────┘│
│                                                          │ USB │
│                                                   ┌──────▼───┐│
│                                                   │数据中心/PC ││
│                                                   └──────────┘│
└────────────────────────────────────────────────────────────────┘
```

### 15.2 任务划分

| 任务 | 功能 | 通信方式 | 优先级 |
|:---:|---|---|:---:|
| **Task A** (LoRa 接收) | UART1 收帧 → CRC 校验 → 写入 `lora_rb` | UART1 @9600 | Medium |
| **Task B** (USB 转发) | 从 `lora_rb` 取帧 + 从图像缓冲取 JPEG → JSON 封装 → CDC 发送 | TinyUSB CDC | High |
| **TCP Server** (图像接收) | SoftAP → CAM 连接 → JPEG 数据写入双缓冲 | WiFi TCP 8080 | High |

### 15.3 图像双缓冲

| 参数 | 值 |
|:---:|:---:|
| 缓冲区大小 | **64 KB × 2** (buf_a, buf_b) |
| 切换条件 | 写满 64KB 或收到帧结束标记 |

### 15.4 USB CDC 输出格式

每行一条 JSON，以 `\n` 分隔。

**传感器/状态数据 (来自 LoRa)：**
```json
{"type":"lora","time":1717200000,"rssi":-87,"frame":[0xAA,0x55,0x01,0x16,...]}
```

**图像数据 (来自 WiFi)：**
```json
{"type":"image","time":1717200000,"seq":1,"total":3,"size":65536,"data":"<BASE64_CHUNK>"}
```

| 字段 | 说明 |
|:---|---:|
| `type` | 数据类型：`lora` / `image` / `system` |
| `time` | Unix 时间戳 (s) |
| `rssi` | LoRa 信号强度（仅 `lora` 类型） |
| `frame` | 原始 LoRa 帧 HEX 数组（仅 `lora` 类型） |
| `seq` | 图像分片序号（仅 `image` 类型） |
| `total` | 总片数（仅 `image` 类型） |
| `size` | 本片字节数（仅 `image` 类型） |
| `data` | BASE64 编码的图像数据（仅 `image` 类型） |

---

## 16. 车→基站 WiFi 图像传输

### 16.1 传输流程

```
[小车靠近基站 WiFi 覆盖范围 ~100m]

1. STM32 唤醒 ESP32-CAM（发 WAKE 指令）
2. STM32 发时间同步：SETTIME 2026 06 01 14 35 20\n
3. STM32 发 START_UPLOAD\n 指令
4. ESP32-CAM 扫描并连接 SSID: "PATROL_BASE" 的 SoftAP
5. CAM 建立 TCP 连接 → 基站 192.168.4.1:8080
6. CAM 读取 SD 卡中所有未上传的 IMG_*.jpg 文件
7. 逐一发送：
   a. 先发 4 字节文件名长度 (uint32 BE)
   b. 发文件名 (UTF-8 字符串, 无后缀 \0)
   c. 发 4 字节文件大小 (uint32 BE)
   d. 发 JPEG 裸数据
8. 每发完一张，等待基站 ACK (1 字节 0x06)
9. 收到 ACK → 删除 SD 卡上该文件 → 发下一张
10. 全部发完 → 发 UPLOAD_DONE\n 通知 STM32
11. STM32 发 SLEEP 让 CAM 休眠
```

### 16.2 照片命名规则

```
IMG_YYYYMMDD_HHMMSS_N.jpg
```

| 片段 | 含义 | 示例 |
|:---:|---|---|
| `YYYYMMDD` | 拍摄日期 | `20260601` |
| `HHMMSS` | 拍摄时间 (UTC+8) | `143520` |
| `N` | 连拍序号 | `1`、`2`、`3` |

### 16.3 CAM 指令协议 (UART2 @115200)

| 命令 | 方向 | 说明 |
|:---:|:---:|---|
| `SETTIME YYYY MM DD HH MM SS\n` | STM32→CAM | 时间同步 |
| `SNAP\n` | STM32→CAM | 拍照 → 存 SD 卡 (IMG_...) |
| `START_UPLOAD\n` | STM32→CAM | 连接基站 WiFi 并上传照片 |
| `SLEEP\n` | STM32→CAM | 深度睡眠 (~6mA) |
| `WAKE\n` | STM32→CAM | 唤醒 |
| `UPLOAD_DONE\n` | CAM→STM32 | 上传完成通知 |

---

*文档版本: v1.1*
*日期: 2026-06-01*

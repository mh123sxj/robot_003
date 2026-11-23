# 履带式底盘 + EEZYbotARM MK2 机械臂（校内机器人竞赛）

> STM32F103C8T6 主控的履带车 + 三轴机械臂 + 平行两指夹爪，用于校内机器人赛。
> 本仓库为主控固件、接线与装配文档。

---

## 一、项目简介

| 项 | 内容 |
|---|---|
| 载体 | 履带式底盘（260×158 铝板 + 2020 型材/角码，双侧履带 56 节） |
| 机械臂 | EEZYbotARM MK2，**取消底座旋转**，3 舵机（肩/肘/爪） |
| 夹爪 | 平行两指 + 齿轮齿条，开口 30–62 mm |
| 主控 | STM32F103C8T6（蓝丸），HAL + CMake + Ninja |
| 遥控 | 富斯 iA10B，iBUS 协议（USART3） |
| 驱动 | 双 BTS7960（每块 1 路），MG540 减速电机 ×2 + 霍尔编码器 |
| 供电 | 3S 锂电 → 保险 → 总开关 → 12V 端子 → UBEC 6V（舵机）/ MP1584 5V（逻辑） |

## 二、主要功能

| 功能 | 说明 |
|---|---|
| 遥控解析 | iBUS 10 通道，校验 + 丢包重同步 |
| 速度闭环 | 左右轮独立 PI + 前馈（KFF），编码器反馈 |
| 混控 | 油门/转向混控成左右轮目标 |
| 保护机制 | 失控保护（100 ms 停车）、低压保护（9.9 V）、堵转保护（单轮 5 次采样 500 ms）、IWDG 看门狗 |
| 舵机控制 | 肩/肘/爪 3 路 PWM，带缓动（slew） |
| 调试 | 非阻塞串口输出（USART3 + USART1 双路），200 ms 一行 |
| 故障处理 | HardFault/MemManage/BusFault/UsageFault 统一切电机 |

## 三、引脚分配

| 功能 | 引脚 |
|---|---|
| 左电机 PWM | PA0（RPWM）/ PA1（LPWM），EN = PB1 |
| 右电机 PWM | PA2 / PA3，EN = PB9 |
| 编码器 | 左 PA10 / PA11，右 PB4 / PB5（EXTI） |
| 舵机 | 肩 PA6、肘 PA7、夹爪 PB0 |
| iBUS 接收 | PB11（USART3 RX） |
| 调试串口 | PB10（USART3 TX）+ PA9（USART1 TX） |
| 电池分压 | PA5（ADC1） |
| 状态灯 | PC13（低有效） |

完整表见 `docs/引脚表.csv`。

## 四、目录结构

```
App/                 应用层（自己写的）
  app.c/h            主循环、任务调度
  app_config.h       全部参数（PWM、PID、保护阈值、通道映射）
  motor.c/h          电机驱动 + 编码器测速 + PI 闭环 + 堵转保护
  encoder.c/h        正交解码
  rc.c/h             iBUS 解析
  servo.c/h          舵机 PWM + 缓动
  gripper.c/h        夹爪开合
  mixer.c/h          混控
  battery.c/h        电压采样 + 低压保护
  debug.c/h          非阻塞串口打印
  bsp.c/h            板级初始化（时钟、GPIO、EXTI、ADC、IWDG）
  pins.h             引脚宏
Core/                CubeMX 生成的启动与中断（只改 USER CODE 区）
Drivers/             STM32F1 HAL + CMSIS
docs/                装配流程、接线方案、引脚表
CMakeLists.txt       构建脚本
robot_003.ioc        CubeMX 工程文件
```

## 五、编译与烧录

### 编译

需要：STM32CubeMX 6.15、CMake ≥3.22、Ninja、arm-none-eabi-gcc（本机用 STM32CubeCLT 1.22.0）。

```bash
cmake --preset Debug
cmake --build build/Debug -j
```

产物：`build/Debug/robot_003.elf` / `.hex`

> 若重新生成代码：用 CubeMX 打开 `robot_003.ioc` 生成，**只保留 USER CODE 区内容**。

### 烧录

- UART bootloader：BOOT0=1，USART1（PA9/PA10）接 CH340，用 STM32CubeProgrammer 或 FlyMcu 烧 `.hex`，烧完 BOOT0=0。
- 或 ST-Link / DAP-Link：SWD（PA13/PA14）直接烧 `.elf`。

### 上电自检

串口 115200 应看到启动横幅与周期行：

```
BOOT <date> <time> | CPR=4680 GEAR=90 KFFx10=26 VMAX=260 BRK=150
CH:<10 通道> | N:<左 右>| SPD:<左 右> | D:<左 右> | V:x.x| CNT:<左 右> | E:<错误数>
```

## 六、关键参数（`App/app_config.h`）

| 参数 | 值 | 说明 |
|---|---|---|
| PWM_ARR | 3599 | 电机 PWM 周期（20 kHz） |
| SERVO_ARR | 19999 | 舵机 50 Hz |
| SPEED_MAX | 260 | 最大占空比 |
| PID_KFF / KP / KI | 2.6 / 2.0 / 6.0 | 速度环 |
| RC_TIMEOUT_MS | 100 | 失控判定 |
| BATT_LOW_V | 9.9 | 低压停车 |
| STALL_CMD_MIN | 150 | 堵转检测最低指令 |
| ENC_CPR | 13×4×90 = 4680 | 编码器每输出轴一圈计数 |

标定方法：手转输出轴一圈读 `CNT:`，若为 1560 则把 GEAR_RATIO 由 90 改 30。

## 七、个人完成部分

> 按实际分工填写/调整；以下为本人在本项目中的工作范围。

1. **机械结构**
   - 车架铝件（D01–D06）图纸尺寸复核，比对孔位与实物，输出孔位编号图与装配图解；
   - 车架装配方案设计（无型材版 / 型材版两套）、角码与螺丝选型、钻孔纸样；
   - EEZYbotARM MK2 机械臂装配（001–013 件识别、肩/肘舵机装配、连杆机构）；
   - 平行两指夹爪的现件改造方案（不重新打印条件下的胶接 + 自攻 + 螺栓当轴）。
2. **电控与接线**
   - 整车电控布局（电池/双 BTS7960/主控/接收机/端子排分层布置）；
   - 电源链设计（3S → 保险 → 开关 → 12V 端子 → UBEC 6V / MP1584 5V）与分压采样；
   - 全部接线（电机、编码器、舵机、iBUS、调试串口），并绘制接线图 D1–D6。
3. **固件与调试**
   - 主控固件（本仓库 `App/`）：电机闭环、编码器解码、iBUS 解析、舵机控制、混控、保护机制、串口调试；
   - 参数整定（速度环 KFF/PI、死区、堵转阈值、低压阈值）；
   - 台架与落地测试：方向、堵转、失控、低压、直线 3 m 偏差、原地转 90°。
4. **文档**
   - 整车装配流程、机械臂装配流程、夹爪方案、飞线接线全流程、引脚表。

## 八、文档

| 文件 | 内容 |
|---|---|
| `docs/整车装配流程.pdf` | 车架 16 步装配（含孔位图） |
| `docs/机械臂装配流程.pdf` | 机械臂 12 步装配 |
| `docs/夹爪方案.pdf` | 夹爪免打印方案与应急路线 |
| `docs/接线全流程.pdf` | 飞线接线与上电顺序 |
| `docs/引脚表.csv` | 完整引脚分配 |

## 九、已知限制

- 编码器 CPR 需按实际减速比标定（默认按 90:1）。
- 速度环参数在空载台架整定，重载需重调。
- 夹爪为改造件（非原设计孔位），夹持力受 SG90 限制，重物需降速。
- 舵机断电不自锁，夹取时需保持通电。

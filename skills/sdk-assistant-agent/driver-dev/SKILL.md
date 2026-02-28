---
name: driver-dev
description: ARCS SDK lisa_device 驱动辅助统一入口。当用户涉及任何 lisa_xxx 驱动的使用、配置、示例查询，或创建/移植驱动时触发。路由到对应设备的子 skill。
---

# ARCS 驱动开发辅助

## 设备路由（必须先做）

识别用户描述中提到的设备，读取对应子 skill 并执行其流程：

| 用户提到的设备 / 关键词 | 读取并执行 |
|------------------------|-----------|
| gpio / 引脚 / 输入输出 / 中断引脚 | `lisa_gpio/SKILL.md` |
| uart / 串口 / 收发 / 波特率 | `lisa_uart/SKILL.md` |
| spi / spi master / spi slave | `lisa_spi/SKILL.md` |
| i2c / i2c master / 从机扫描 | `lisa_i2c/SKILL.md` |
| flash / nor flash / 读写擦除 | `lisa_flash/SKILL.md` |
| adc / 模数转换 / 采样 / 温度传感器 | `lisa_adc/SKILL.md` |
| pwm / 占空比 / 脉宽调制 | `lisa_pwm/SKILL.md` |
| hwtimer / 硬件定时器 / gpt / aon timer | `lisa_hwtimer/SKILL.md` |
| rtc / 实时时钟 / 闹钟 / 日历 | `lisa_rtc/SKILL.md` |
| wdt / watchdog / 看门狗 | `lisa_wdt/SKILL.md` |
| sdmmc / sd卡 / mmc / tf卡 | `lisa_sdmmc/SKILL.md` |
| i2s / 音频接口 / pcm 流 | `lisa_i2s/SKILL.md` |
| display / 显示屏 / lcd 刷屏 / lvgl 显示 | `lisa_display/SKILL.md` |
| touch / 触摸 / 触控 | `lisa_touch/SKILL.md` |
| audio / 录音 / 播放 / codec / aec | `lisa_audio/SKILL.md` |
| qspilcd / qspi lcd / 四线 lcd | `lisa_qspilcd/SKILL.md` |
| dvp / 摄像头接口 / camera 接口 | `lisa_dvp/SKILL.md` |
| camera / 摄像头 / sensor / 拍照 | `lisa_camera/SKILL.md` |
| rgb / led灯带 / ws2812 / sk6812 | `lisa_rgb/SKILL.md` |
| pinmux / 引脚复用 / iomux / 引脚功能 | `pinmux/SKILL.md` |

## 不明确时

若用户描述涉及多个设备，或设备不明确：

1. 列出可能的候选设备，询问用户确认
2. 确认后再读取对应 SKILL.md

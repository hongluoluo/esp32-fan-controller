# ESP32 风扇调压控制器固件（ESPHome）

基于 **ESP32-WROOM-32E** 的 12V 风扇无级调速控制器固件。用一路 18kHz PWM 驱动「双 MOS 调压模块」，把风扇转速做成 1%~100% 的无级滑块；同时把**星期定时**和**倒计时**做进设备本体，手机不在线、App 未运行也能按时执行。

配套安卓 App：[smartfan-android](https://github.com/hongluoluo/smartfan-android)

---

## 目录

- [它能做什么](#它能做什么)
- [硬件](#硬件)
- [接线](#接线)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [转速标定](#转速标定)
- [MQTT 接口（App 用）](#mqtt-接口app-用)
- [设备端定时规则](#设备端定时规则)
- [设备端倒计时](#设备端倒计时)
- [Home Assistant 集成](#home-assistant-集成)
- [常见问题与踩过的坑](#常见问题与踩过的坑)
- [目录结构](#目录结构)
- [许可证](#许可证)

---

## 它能做什么

| 能力 | 说明 |
|---|---|
| 无级调速 | 风扇实体 0~100% 滑块，内部映射成 PWM 占空比 40%~80%，1% 步进 |
| 真 0V 关断 | 0% 时 PWM 引脚恒低，MOS 完全关断，不存在"低速常转" |
| 星期定时 | 设备内 6 个规则槽，支持「每天 / 工作日 / 周末 / 任意星期组合」，动作为开 / 关 / 设为指定转速 |
| 倒计时 | 设备内倒计时，1~1440 分钟，到点开或关；手机断网、关机都不影响 |
| 掉电保存 | 6 条规则、倒计时动作与结束时刻写入 flash（NVS），重启不丢 |
| 双控制通道 | Home Assistant 原生 API（加密）+ MQTT（供自研 App 直连），互不干扰 |
| 状态快照 | retained JSON 快照，任何客户端一连上就能拿到完整真实状态 |

---

## 硬件

| 元件 | 规格 | 备注 |
|---|---|---|
| 主控 | ESP32-WROOM-32E 开发板（DevKitC 类） | 用 `esp32dev` 板型，Arduino 框架 |
| 调压模块 | **PWM 输入型双 MOS 调压模块** | 规格：PWM 频率 0~20kHz，信号电压 3.3~20V，输入 DC 5~36V |
| 风扇 | 12V 2 线无刷风扇 | **必须带载测试**，见「常见问题」 |
| 电源 | 12V 直流 | 输入电压决定最大输出；换电源要重新标定 |

> ⚠️ 不要用 ESP32 的 DAC（`esp32_dac`）方案：DAC 零码有 20~100mV 偏置，会让风扇低速常转；PWM 方案 0% 是真正的引脚恒低。

---

## 接线

```
                        ┌──────────────── 12V 电源 (+) ────────────────┐
                        │                                             │
            ┌───────────┴──────────┐                    ┌─────────────┴──────────┐
            │  双 MOS 调压模块      │                    │   12V 风扇              │
            │  DC+  ← 12V (+)      │                    │                        │
            │  DC-  ─ 12V (-)      │   OUT+ ────────────┤ +                      │
            │  PWM+ ← GPIO17 (3.3V)│   OUT- ────────────┤ -                      │
            │  GND  ─┬─ 12V (-)    │                    └────────────────────────┘
            └────────┼─────────────┘
                     │
            ┌────────┴─────────┐
            │  ESP32  GND      │      ESP32 由 USB 或 5V 供电
            │  GPIO17 ── PWM+  │
            └──────────────────┘
```

要点：

1. **必须共地**：ESP32 GND、模块 GND、12V 电源负极三者相连，否则 PWM 无参考电平。
2. 模块「信号电压 3.3~20V」意味着 ESP32 的 3.3V 逻辑可以直接驱动，**不需要电平转换或运放**。
3. PWM 走 GPIO17（LEDC 硬件通道，与 WiFi 无冲突）。
4. 12V 与 ESP32 供电建议分开（或用带隔离的降压），风扇启动瞬间电流冲击较大。

---

## 快速开始

```bash
# 1. 准备凭据
cp secrets.yaml.example secrets.yaml
#   编辑 secrets.yaml：WiFi、MQTT broker、API 密钥

# 2. 校验配置
esphome config psu-fan.yaml

# 3. 首次烧写（USB）
esphome run psu-fan.yaml --device COM3

# 4. 之后无线升级
esphome upload psu-fan.yaml --device <设备IP>
```

Home Assistant 里打开 ESPHome 集成 → 输入设备 IP 与 `fan_api_key` 即可获得「风扇」实体（开关 + 0~100% 滑块）。

> Windows 用户注意：如果你的 shell 是 Git Bash / MSYS，ESPHome 的 `idf_tools.py` 会因检测到 `MSYSTEM` 环境变量直接 abort。用 PowerShell 执行，或先 `Remove-Item Env:MSYSTEM`。

---

## 配置说明

`psu-fan.yaml` 主要段落：

| 段落 | 作用 |
|---|---|
| `esphome.on_boot` | 上电先把占空比拉到 0，避免启动瞬间风扇乱转 |
| `api.encryption` | HA 原生 API 加密密钥（`!secret fan_api_key`） |
| `mqtt` | broker 连接、`topic_prefix: fan32`、关闭 discovery（避免与 API 集成重复注册实体） |
| `time.sntp` | 授时，`Asia/Shanghai`；`on_time: seconds: 0` 每分钟检查规则槽 |
| `globals.cd_end` | 倒计时结束时间戳（epoch），`restore_value: true` 掉电保存 |
| `text.rule1..rule6` | 6 个定时规则槽，`restore_value: true` |
| `text.cmd` | 固定命令通道（载荷 `on` / `off` / `speed 60`） |
| `number.cd_minutes` | 倒计时分钟数 |
| `button.cd_start` / `cd_cancel` | 启动 / 取消倒计时 |
| `sensor.cd_remaining` | 剩余秒数（3 秒刷新） |
| `text_sensor.dev_time` | 设备当前时间（排障用，未授时显示 `--unsynced--`） |
| `interval: 20s` | 发布 retained 状态快照（含规则与倒计时） |
| `interval: 3s` | 倒计时到点判定 |
| `output.ledc` | GPIO17，18kHz，12 位分辨率（占空比步进 1/4096） |
| `output.template fan_map` | 风速 % → 占空比 % 的标定层 |
| `fan.speed` | 风扇实体，`speed_count: 100`，`RESTORE_DEFAULT_OFF` |

---

## 转速标定

`fan_map` 把「滑块百分比」映射为「PWM 占空比」，是唯一的标定点：

```cpp
float s = constrain(state, 0.0f, 1.0f);
float duty;
if (s <= 0.001f) {
  duty = 0.0f;                              // 0% = 关断，引脚恒低
} else {
  duty = 0.4000f + (s - 0.01f) * 0.40404f;  // 1%→40%，100%→80%
}
id(pwm_out).set_level(constrain(duty, 0.0f, 1.0f));
```

本机实测（12V 输入、带载、双 MOS 模块）：

| 占空比 | 实测输出电压 |
|---|---|
| 40% | 4.75V |
| 50% | 5.78V |
| 80% | 9.50V |

即 `Vout ≈ 占空比 × Vin`，故取「1% → 40%（4.75V，风扇可靠起转且风可感）」「100% → 80%（9.5V）」，中间线性插值。

**换电源 / 换模块 / 换风扇 / 改 PWM 频率，都必须重新标定**，详见 [docs/calibration.md](docs/calibration.md)。

---

## MQTT 接口（App 用）

主题前缀 `fan32`（`mqtt.topic_prefix`）。

### 1. 命令通道（推荐，固定主题）

| 主题 | 载荷 | 说明 |
|---|---|---|
| `fan32/text/cmd/command` | `on` | 开风扇（保持当前转速） |
| | `off` | 关风扇（PWM 归零） |
| | `speed 60` | 设为 60% 并自动开启（1~100） |

### 2. 状态快照（retained，20 秒兜底 + 状态变化立即发）

```
主题: fan32/state     retain: true
载荷: {"on":true,"speed":60,"rules":["08:00|*|OFF|60","0","0","0","0","0"],"cd_action":"OFF","cd_rem":0}
```

| 字段 | 含义 |
|---|---|
| `on` | 风扇开关 |
| `speed` | 当前风速百分比 |
| `rules` | 6 个规则槽的原始字符串，`"0"` 表示空槽 |
| `cd_action` | 倒计时到点动作 `ON` / `OFF` |
| `cd_rem` | 剩余秒数（0 = 无倒计时） |

> 因为设备状态消息本身**不是 retained**，这份快照是客户端「一连接就能拿到真实状态」的关键。

### 3. 定时规则槽

| 主题 | 方向 | 载荷 |
|---|---|---|
| `fan32/text/ruleN/command` | 写 | `HH:MM\|DAYS\|ACTION\|SPEED`，`0` 表示清空 |
| `fan32/text/ruleN/state` | 读 | 当前槽内容 |

### 4. 倒计时

| 主题 | 方向 | 载荷 |
|---|---|---|
| `fan32/text/cd_action/command` | 写 | `ON` / `OFF` |
| `fan32/number/cd_minutes/command` | 写 | `1`~`1440` |
| `fan32/button/cd_start/command` | 写 | `PRESS` |
| `fan32/button/cd_cancel/command` | 写 | `PRESS` |
| `fan32/sensor/cd_remaining/state` | 读 | 剩余秒（3 秒刷新） |
| `fan32/text_sensor/dev_time/state` | 读 | 设备时间，未授时为 `--unsynced--` |

### 5. 原生实体主题（ESPHome 自动生成，可选）

| 主题 | 说明 |
|---|---|
| `fan32/fan/______/command` | `ON` / `OFF`（实体名为中文"风扇"，净化后是 6 个下划线） |
| `fan32/fan/______/speed_level/command` | `1`~`100` |
| `fan32/fan/______/state`、`.../speed_level/state` | 状态回显 |
| `fan32/status` | `online` / `offline`（retained birth 消息） |

> 这些主题名依赖实体名净化规则，**不要写死在客户端里**；用 `cmd` + `state` 快照即可。详细协议见 [docs/mqtt-protocol.md](docs/mqtt-protocol.md)。

---

## 设备端定时规则

格式：

```
HH:MM|DAYS|ACTION|SPEED
```

| 字段 | 取值 | 说明 |
|---|---|---|
| `HH:MM` | `08:30` | 24 小时制，精确到分钟 |
| `DAYS` | `12345` / `*` / `0` | 数字位 1=周一 … 7=周日；`*` 每天；`0` 停用 |
| `ACTION` | `ON` / `OFF` / `SPEED` | 开 / 关 / 设为指定转速（并开启） |
| `SPEED` | `1`~`100` | 仅 `SPEED` 用，缺省 60 |

示例：

```
08:00|12345|SPEED|60      工作日 08:00 开 60%
23:30|*|OFF               每天 23:30 关闭
19:00|67|SPEED|30         周六周日 19:00 开 30%
0                         空槽
```

执行逻辑：`time.sntp` 的 `on_time: seconds: 0` 每分钟触发一次，遍历 6 个槽，匹配「小时+分钟+星期位」后调用风扇动作。设备停机、断网错过的时刻**不会补执行**（避免误触）。

---

## 设备端倒计时

- 设定 `cd_minutes`（分钟）与 `cd_action`（到点开还是关）
- 按 `cd_start` 后，设备把「结束时刻（epoch）」写入全局变量并落盘
- 每 3 秒比对一次当前时间，到点执行动作并清零
- 剩余时间通过 `cd_remaining` 与状态快照暴露；按 `cd_cancel` 可随时取消
- **依赖 NTP 授时**：未授时（`dev_time` 显示 `--unsynced--`）时按下开始会被拒绝，日志打印 `time not synced, cannot start`

---

## Home Assistant 集成

1. 设置 → 设备与服务 → 添加集成 → **ESPHome**
2. 填设备 IP（或等自动发现），粘贴 `fan_api_key`
3. 得到 `fan.电源风扇调压32_风扇` 实体：开关 + 0~100% 滑块
4. 若同时连了 MQTT，本配置已设 `discovery: false`，不会重复注册实体

---

## 常见问题与踩过的坑

**1. 空载测输出电压永远接近 12V**

PWM 斩波 + 输出端有电容、无放电通路时，万用表直流档读的是**峰值**。任何占空比 >0 都读到满压是正常现象，**必须带载测**。

**2. 2 线无刷风扇在 50% 斩波下不转，只有 100% 才转**

风扇驱动板输入储能不足（LDO 供电 / 无 bulk cap / 自举 / OCP），每个周期掉到 0V 时 UVLO 反复复位，软启动永远走不完。
解决：① 换兼容风扇（最有效）② 加 LC 滤波得平滑直流 ③ 换内部带电感的调压模块 ④ 4 线风扇直接驱动其 PWM 脚。
**并电容无效**——纯电容是峰值检波，任何占空比都会变全速。

**3. 定时/倒计时"静默失效"**

九成是**没授时**。先看 `fan32/text_sensor/dev_time/state`：
- `--unsynced--` → NTP 不通。**`223.5.5.5` 是阿里 DNS，不提供 NTP 服务**（本机实测超时）；用 `ntp.aliyun.com` / `ntp1.aliyun.com` / `time1.cloud.tencent.com`。
- 时间正确但时区差 8 小时 → `timezone` 写错。

**4. 改完配置风扇自己转起来**

`on_boot` 里已经把占空比拉 0；如果仍乱转，检查 `fan.restore_mode` 与 `on_turn_on` 的默认档位逻辑。

**5. Windows 下编译直接失败**

Git Bash / MSYS 会注入 `MSYSTEM`，导致 `idf_tools.py` abort。用 PowerShell，或 `Remove-Item Env:MSYSTEM`。

---

## 目录结构

```
esp32-fan-controller/
├── psu-fan.yaml           # 固件主配置（已脱敏，凭据走 secrets）
├── secrets.yaml.example   # 凭据模板
├── docs/
│   ├── calibration.md     # 转速标定流程与实测数据
│   └── mqtt-protocol.md   # MQTT 协议细节
├── .gitignore
└── README.md
```

编译耗时参考：首次（含工具链下载）550~670 秒；增量 70~80 秒；改配置触发组件重编 280~300 秒。

---

## 许可证

MIT，见 [LICENSE](LICENSE)。

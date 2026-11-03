# MQTT 协议细节

主题前缀：`fan32`（`mqtt.topic_prefix`，可自行修改，App 中需保持一致）。

## 命令（客户端 → 设备）

| 主题 | QoS | 载荷 | 效果 |
|---|---|---|---|
| `fan32/text/cmd/command` | 1（建议） | `on` | 开启，保持当前转速 |
| | | `off` | 关闭，PWM 引脚恒低 |
| | | `speed 60` | 设 60% 并自动开启 |

`cmd` 是 ESPHome `text` 实体（名为 ASCII `cmd`），固件在 `set_action` 里解析并执行，**主题名固定可预测**，不依赖实体名净化规则。

## 状态快照（设备 → 客户端，retained）

```
topic : fan32/state
retain: true
qos   : 0
```

载荷（单行 JSON）：

```json
{"on":true,"speed":60,"rules":["08:00|*|OFF|60","0","0","0","0","0"],"cd_action":"OFF","cd_rem":0}
```

发布时机：
1. `fan.on_state` —— 风扇开关状态变化立即发；
2. `interval: 20s` —— 每 20 秒兜底刷新一次（覆盖规则/倒计时变化，以及 broker 重启后补齐 retained）。

客户端**只需订阅这一个主题**即可获得全量状态；单独订阅实体主题并不可靠，因为 ESPHome 的实体状态消息不是 retained，冷启动时设备不会重发。

## 定时规则槽

| 主题 | 载荷 |
|---|---|
| `fan32/text/rule1/command` … `rule6/command` | `HH:MM\|DAYS\|ACTION\|SPEED` 或 `0`（清空） |
| `fan32/text/ruleN/state` | 当前内容（保留在 flash） |

字段语义见主 README「设备端定时规则」。

## 倒计时

| 主题 | 载荷 | 说明 |
|---|---|---|
| `fan32/text/cd_action/command` | `ON` / `OFF` | 到点执行的动作 |
| `fan32/number/cd_minutes/command` | `1`~`1440` | 时长 |
| `fan32/button/cd_start/command` | `PRESS` | 启动 |
| `fan32/button/cd_cancel/command` | `PRESS` | 取消 |
| `fan32/sensor/cd_remaining/state` | 秒 | 每 3 秒刷新 |

## 在线状态与时间

| 主题 | 载荷 |
|---|---|
| `fan32/status` | `online` / `offline`（retained LWT/birth） |
| `fan32/text_sensor/dev_time/state` | `2026-11-03 20:34:20` 或 `--unsynced--` |

`dev_time` 是排障关键：定时/倒计时不生效时先看它，`--unsynced--` 表示 NTP 未同步，此时任何时间相关功能都不会执行。

## 客户端实现注意事项

1. **订阅 QoS 1**，发送也用 QoS 1，避免局域网抖动丢命令。
2. **不要把实体主题写死**（中文实体名会被净化成下划线，如 `fan32/fan/______/command`）。
3. 收到 `fan32/state` 之前，UI 不要显示"确定状态"，应显示"同步中"；否则会出现「设备实际在转、App 显示已关闭」。
4. 解析时**严格按主题前缀分类**——曾出现 `fan32/number/cd_minutes/state`（值 `30.000000`）被兜底分支当成 `ON/OFF` 判断，导致开关间歇性闪烁。

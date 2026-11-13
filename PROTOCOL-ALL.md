# WeLoop / COROS 全机型 BLE 协议整理

> 逆向自官方 app（`com.yf.*`）反编译源码。本文覆盖**所有 14 个机型**，而不仅是 Hey 3S。
> 姊妹文档 `weloop-time-sync-master/PROTOCOL.md` 只描述 Hey 3S 的时间同步，本文是它的超集，并修正了其中关于字节序的错误结论。

---

## 目录

1. [三代协议总览与机型归属](#1-三代协议总览与机型归属)
2. [连接与识别流程（通用）](#2-连接与识别流程通用)
3. [V2 协议（新机型）](#3-v2-协议新机型)
4. [V1 协议（老机型）](#4-v1-协议老机型)
5. [V0 / DFU（固件升级）](#5-v0--dfu固件升级)
6. [功能码（Feature Code）](#6-功能码feature-code)
7. [时间同步速查表](#7-时间同步速查表)
8. [代码定位索引](#8-代码定位索引)
9. [尚未确认 / 需实测的点](#9-尚未确认--需实测的点)

---

## 1. 三代协议总览与机型归属

### 1.1 协议探测顺序

app 在 `onServicesDiscovered` 后**按 GATT 服务 UUID 依次试加载协议，与机型无关**。

`sources/com/yf/lib/bluetooth/b/e.java:33`

```java
private static final Class[] y = {
    com.yf.lib.bluetooth.b.c.j.class,   // V2
    com.yf.lib.bluetooth.b.b.i.class,   // V1
    com.yf.lib.bluetooth.b.a.a.class    // V0 = Nordic DFU
};
```

`e.java:496-521` 逐个 `newInstance()` 调 `a(deviceContext)`，第一个返回 `true` 的即被采用。

| 协议 | 判定条件 | 入口类 |
|------|----------|--------|
| **V2** | **同时**存在 命令通道 `…77656C6F6F70` **和** 数据通道 `…E50E24DCCA9E`（各含 0x002 写 / 0x003 通知） | `b/c/b.java:65-90` |
| **V1** | 只有 `…77656C6F6F70` 服务（含 0x002 写 / 0x003 通知） | `b/b/i.java:54-65` |
| **V0** | 存在 Nordic DFU `00001530-1212-efde-1523-785feabcd123` | `b/a/a.java:94-105` |

另外 V2 还有一个**可选**的 Notification 服务 `…77757C7F7F70`（只含 0x002 写），存在时通知推送走它（`b/c/b.java:80-86`）。

### 1.2 机型枚举

`sources/com/yf/lib/bluetooth/b/g.java:8-22`

| 枚举名 | model 字符串 | 广播名前缀 | index | 归属协议 |
|--------|--------------|------------|-------|----------|
| `MYLINK` | `MyLink+` | `MyLink+` | 0 | V2 |
| `BN01L02` | `BN01L02` | `BN01L02`, `Now` | 1 | **V1** |
| `BN01SH02` | `BN01SH02` | `BN01SH02`, `NowC` | 2 | **V1** |
| `WN02B01` | `WN02B01` | `WeLoop`, `Tommy`, `DfuTarg` | 3 | **V1** |
| `WN08B08` | `WN08B08` | `Tommy2`, `WeLoop B08` | 4 | **V1** |
| `NOW2` | `WeLoop Now2` | `WeLoop Now2` | 5 | **V1** |
| `WELOOPXH3` | `WELOOPXH3` | `WeLoop XH3` | 6 | V2 |
| `WELOOPHEY3S` | `WELOOPHEY3S` | `WeLoop Hey 3S` | 7 | V2 |
| `COROS_T1` | `COROS T1` | `COROS T1` | 8 | V2 |
| `COROS_PACE` | `COROS M1` | `COROS PACE`, `COROS M1` | 9 | V2 |
| `WELOOPNOW3` | `WeLoop Neo` | `WeLoop Neo`, `Neo`, `WeLoop Now 3`, `Now3` | 10 | V2 |
| `COROS_AERO` | `COROS OMNI` | `COROS OMNI`, `COROS OMNI BLE` | 11 | V2 |
| `X9K` | `X9K` | `Trumpchi X9K`, `X9K` | 12 | V2 |
| `COROS_LINX` | `COROS HELMET` | `COROS HELMET`, `COROS HELMET BLE` | 13 | V2 |

> **归属推断依据**：协议栈里显式按机型分支的代码只覆盖 BN01L02 / BN01SH02 / WN02B01 / WN08B08 / NOW2 这五个（`b/e.java:551-578`、`b/b/b.java:108-119`、`b/b/e.java:550,711-717`），它们正是 V1 的五个机型；其余 9 个机型在协议层没有任何硬编码分支，完全由设备上报决定。
> 广播名 `DfuTarg` 属于 WN02B01，即老 Tommy 进入 DFU 时走 V0。

### 1.3 各机型命令差异总览

| 机型 | 初始化 cmd 112 变体 | 闹钟 | 复位 | 勿扰 | OTA | 专属 |
|------|---------------------|------|------|------|-----|------|
| Tommy / Now / NowC / B08 / Now2 | **V1**，见第 4 节 | V1 cmd 158 | V1 cmd 137 | V1 cmd 173 | V1 cmd 150/151 | 固件版本阈值决定功能码 |
| XH3 | 通用 `m.java` 21B | 135 | 133 | 134 | 148+120/7 | **固件 "V 1.7" 时 OTA 后自动发 cmd 132 重启**（`b/c/e.java:190-193`） |
| Hey 3S | 通用 21B | 135 | 133 | 134 | 148+120/7 | 无 |
| MyLink+ | 通用 21B | 135 | 133 | 134 | 148+120/7 | 无 |
| Neo / Now 3 | 通用 21B，但解析后 `dVar.e(1)`（`b/c/b/m.java:296-302`） | 135 | 133 | 134 | 148+120/7 | 唯一在协议层被区分的 WeLoop 机型 |
| X9K | 通用 21B | 135 | 133 | 134 | 148+120/7 | 无 |
| **COROS PACE / M1 / T1** | **COROS 手表变体 `b/c/b/e.java` 37B 位打包，含心率区间，发前 sleep 500ms** | 135（`r.a(z)`，**含单次闹钟位**） | **133 + payload [0]** | **134 5B（含起止时段）** | **sendFirmwareCoros** + 多资源包，结束发 132 | `unBindM1` = 133+[1] |
| **COROS OMNI** | 通用 / COROS（协议层未区分） | 135 | 133 | 134 | 148+120 | 尾灯 cmd 159、被动尾灯事件 cmd 20、ANT cmd 161 |
| **COROS HELMET** | **头盔变体 `b/c/b/l.java` 仅 7B** | — | — | — | — | `helmetSpeaker` = cmd 160 |

---

## 2. 连接与识别流程（通用）

`com/yf/lib/bluetooth/b/e.java` + `b/f.java`

1. 连接后 `discoverServices()`（3 秒后触发，10 秒超时 `e.java:67`）
2. **读 DIS（Device Information Service `0000180A`）**，顺序见 `b/f.java`：
   - `00002A24` Model Number → `g.fromModel(str)`（`e.java:257-260`）
   - `00002A26` Firmware Revision → 软件版本串（形如 `"V 4.3 02"`）
   - `00002A25` Serial Number
   - `00002A27` Hardware Revision → **功能码字符串**（见第 6 节）
   - `00002A29` Manufacturer / `00002A28` Software Revision
3. 型号识别失败则退化为广播名匹配：`g.fromDeviceName(name)`，`g.java:45-49` 会**截掉最后一个空格之后的内容**，所以 `"COROS PACE 1234"` → `"COROS PACE"`
4. 读标准电量服务 `0000180F` / `00002A19` 并开启通知（`b/b/d/x.java:76-78`）
5. 加载协议（1.1 节）

> **注意**：功能码完全依赖 DIS `2A27` 的字符串；读不到就清空功能码列表（`b/f.java:76-80`），设备会退化成"只支持基础命令"。

---

## 3. V2 协议（新机型）

适用：Hey 3S / XH3 / Neo·Now3 / MyLink+ / X9K / COROS 全系

### 3.1 通道 UUID

`b/c/b.java:24-37`

| 用途 | Service | 写 (手机→设备) | 通知 (设备→手机) |
|------|---------|----------------|------------------|
| 命令通道 | `6E400001-B5A3-F393-E0A9-77656C6F6F70` | `…6E400002…` | `…6E400003…` |
| 数据通道 | `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` | `…6E400002…` | `…6E400003…` |
| Notification（可选） | `6E400001-B5A3-F393-E0A9-77757C7F7F70` | `…6E400002…` | — |

CCCD = `00002902-0000-1000-8000-00805f9b34fb`，写 `ENABLE_NOTIFICATION_VALUE` 后 `wait(1000)`（`b/c/b.java:130-152`）。
所有写特征都设为 `WRITE_TYPE_NO_RESPONSE`（`setWriteType(1)`，`b/c/b.java:105-114`）。

> 后缀 `77656C6F6F70` = ASCII **"weloop"**。

### 3.2 分包

**命令通道**（`b/c/b.java:201-225`），每包 20 字节：

```
[cmd:1][剩余包数:1][payload ≤18]
```

- 空 payload → `[cmd][0]`
- 总包数 = `ceil((len+18)/18)`；**byte[1] 是"还剩几包"**，最后一包为 0
- 发送顺序：从最后一包往前发（JADX 反编译出的循环是先 `putShort` 高位包）

**Notification 通道**（`b/c/b.java:227-262`）：

```
[cmd:1][剩余包数:1][序号:1][payload ≤17]
```

- byte[2]：首包 `序号|0x80`，后续包 `序号&0x7F`
- 序号自增，>127 归零

**字节序：小端。** `t.d(int)` 分配 buffer 时显式 `order(LITTLE_ENDIAN)`（`b/c/b/t.java:145-149`），因此所有 `putInt/putShort` 都是小端。

> ⚠️ 更正 `weloop-time-sync-master/PROTOCOL.md` 第 4 节的说法：源码里**不是**"ByteBuffer 默认大端、实测小端"，而是代码本身就是小端。结论（小端）是对的，推导过程是错的。

### 3.3 命令表（手机 → 设备）

来源：`b/c/b/r.java`（简单命令）+ 各专用事务类。混淆常量已还原。

| cmd | 用途 | payload 结构（小端） |
|-----|------|----------------------|
| **112** | 初始化 / 读取设备全量状态（configDevice） | 见 3.4，三种变体 |
| **113** | 推送音乐信息 | `[enable:1] 标题64B 0x00 艺人32B 0x00 专辑32B 0x00` |
| **114** | 切换表盘视图 | `[0]`=表盘 / `[1]`=相机 / `[2]`=录像 |
| **116** | 推送天气（城市） | `[type][code][count] short 城市名8B(gb2312/utf8)` |
| **117** | 天气开关 | 1B |
| **120** | 文件/资源传输（走数据通道） | 见 3.5 |
| **121** | 通知推送（走 Notification 通道） | 变长 |
| **124** | 请求数据同步 | `[dataIndex:1][0×4]` |
| **125** | 数据同步确认 | `[addr:4][crcOK:1]` |
| **126** | 背光模式 | 1B：`3`=auto `0`=night `1`=on `2`=off |
| **127** | 夜间时段 | 4B 起止（关闭时全 `0xFF`） |
| **128** | 抬手亮屏 | 1B |
| **129** | 跑步轨迹 | 1B |
| **130** | 骑行轨迹 | 1B |
| **132** | 重启 | 无 |
| **133** | 复位 | 无 / `[0]`=复位 COROS 表 / `[1]`=解绑 M1 |
| **134** | 勿扰模式 | 4B（关闭时全 `0xFF`）或 **5B**（COROS：`[开关][起h][起m][止h][止m]`） |
| **135** | 设置闹钟 | 4B，见下 |
| **136** | 震动 / 声音与振动 | 1B / 2B `[type][音量<<4 \| 振动]` |
| **137** | 防丢 | 1B |
| **139** | 用户资料 | 8B：`[身高][体重][年龄][步幅:2][性别][单位][?]` |
| **140** | **时间同步** | 见第 7 节 |
| **141** | （保留） | 1B |
| **142** | 白天高亮 | 1B |
| **143** | 久坐提醒 | 9B |
| **144** | 读取运动汇总 | 空；响应 ≥16B |
| **145** | 设置运动目标 | 18B（4 个 int + 可选 2B） |
| **146** | 显示颜色 | 1B：`0`=默认 `1`=黑白反色 `2`=黑白黄蓝反色 |
| **147** | 来电提醒 | `[type]` + `[idx?]` + 号码 ≤35B |
| **148** | OTA 开始 / 结束校验 | `[type:1][len:4][crc16:2]`，type `0`=开始 `1`=结束 |
| **149** | 佩戴方式（左右手） | 1B |
| **151** | 自动心率 | 1B |
| **152** | 自动同步开关 | 1B |
| **154** | 推送 GPS 位置 | 14B |
| **158** | 设置 SMB MAC / 读数据屏 | 6B MAC（`"AA:BB:CC:DD:EE:FF"` 逐段 hex）/ `[page]` |
| **159** | 数据屏设置 / 尾灯 | 22B 位打包 / 1B |
| **160** | 消息亮屏 / 音乐 EQ / 头盔喇叭 | `[5][value]` / `[4][n]` / `[value]` |
| **161** | ANT+ 控制（老固件为 **19**） | `[subcmd][b][a]`；响应 8B |
| **162** | 语言设置 | `[ordinal]` |
| **163** | 读充电状态 | 无 |
| **164** | 自动锁屏 | 1B |
| — | 读电量 | **不走命令通道**，直读 DIS `0x2A19` |
| — | 进入 DFU | **不支持**（`b/c/h.java:538-540` 直接返回 `errorUnsupportedCmd`） |

**cmd 135 闹钟**（`r.java:196-249`），两种编码：

- 通用 `r.b(z)`：`[idx][hour|0x80(单次)][minute][weekBitmap|0x80]`
- COROS `r.a(z)`：`[idx][short 位打包][week]`
  - `short = (enable&1)<<4 | (idx&15) | (hour&31)<<5 | (min&63)<<10`
  - 删除闹钟时 `hour=31, min=63`
  - week bitmap：bit0…bit6 = 周日…周六；无重复日时置 `0x80`

### 3.4 cmd 112（初始化）的三种变体 —— 最大的机型差异点

分发在 `b/c/h.java:411-464`。

**(A) 通用 WeLoop** — `b/c/b/m.java:31-50`，21 字节，小端

```
[0]      phoneType (android=1)
[1..4]   unix 秒 - 1388534400
[5]      时区 /30min
[6][7][8]   g.c() / g.d() / g.m()
[9..10]  g.e() short
[11][12] g.k() / g.l()
[13][14] 开关1 / 开关2
[15]     显示颜色
[16][17] g.q() / g.p()
[18]     单位体系 (0=公制 1=英制)
[19]     时区 /15min
[20]     g.r()
```

**(B) COROS 手表** — `b/c/b/e.java:31-76`，位/字节混合打包，上限 37 字节

用 `com/yf/lib/bluetooth/e/a` 位打包器（`a(bits,val)` 位域、`b(byte)`/`b(int)`/`a(short)` 字节对齐小端、`a(byte)` = **1 位**）：

```
b(byte)   phoneType
b(byte)   g.s()
b(int)    时间
b(byte)   时区/15min
b(byte)   g.r()
a(7)      g.m()
a(1)      g.q()
a(short)×3  g.e() / g.f() / g.g()
a(1)×3    开关 i / j / 单位体系
b(byte)×4 心率区间（不足 4 个补 0）
```

> **发这条命令前要 `Thread.sleep(500)`**（`e.java:17`，常量 `UIMsg.d_ResultType.SHORT_URL = 500`），COROS 手表需要延时。

**(C) COROS 头盔** — `b/c/b/l.java:25-33`，仅 7 字节

```
b(byte) phoneType   b(int) 时间   b(byte) 时区/15min
```

**响应是"渐进式"解析的**（`m.java:112-142`、`e.java:98-134`）：用 `hasRemaining()` 逐段判断，老固件返回短包、新固件返回长包 —— **这就是固件/机型差异的编码方式**。通用版响应大致为：

```
int    版本
byte   状态
4B     2 个时段对象
9B     3 个闹钟（每个 3B：[hour&0x7F][minute][week|0x80 单次位]）
4B     久坐时段
6B     byte[6]
1B     状态字节 c：bit0-1 抬手亮屏 / bit2-3 消息亮屏 / bit4-7 各开关
1B     状态字节 d：bit3 久坐 / bit4 左右手 / bit6 自动心率 / bit7 自动同步
1B     自动锁屏
1B     保留
4B     久坐参数
可选   7 个闹钟 × 3B（需功能码 T = onceAlarmSupport）
可选   int 步数目标
可选   int 低24位颜色 / 高8位版本
可选   4 字符型号 + 2 字节版本
```

COROS 手表响应额外包含：6B MAC、bit10 左右手、3 个时段、闹钟数量 + N×3B（bit7 = 单次闹钟）。

### 3.5 数据通道

**文件传输（手机→设备，cmd 120）** — `b/c/b/q.java:284-331`

头部 13 字节（小端）：

```
[dataIndex:1][文件内偏移:4][本块长度:2][剩余长度:4][CRC16:2]
```

- 块大小 4096，帧大小 1200
- dataIndex：`0`=font_bin `1`=unicode_bin `3`=名片/二维码 `7`=固件 `8/9/10`=表盘槽位1/2/3 `13`=星历(epo) `80`=24h天气；**任意自定义值**用于 COROS 多资源包
- 进度 ACK 仍回命令通道：`[120][x][x]` 三字节，`payload[0]!=0` 表示 CRC 通过（`q.java:182`）

**数据同步（设备→手机）** — `b/c/b/s.java`

- 请求 cmd **124**，payload 5B `[dataIndex][0×4]`
- 响应 cmd **124**：`[数据地址:4][块大小:2][剩余大小:4][块CRC:2][状态:1]`
  - 状态 `0`=正常 `1`/`3`=runningTraining `2`=ridingMode 其他=strengthTraining
- 确认 cmd **125**：`[数据地址:4][CRC是否OK:1]`
- dataIndex：`5`=运动 `6`=动态心率 `12`=设备日志 `14`=GPS

### 3.6 响应与被动事件（设备 → 手机）

`b/c/b/p.java:9-23`：

- `p.a(bArr)` = `bArr[0]` → **cmd**
- `p.b(bArr)` = `bArr[1]` → **剩余包数**（0 = 最后一包）
- `p.c(bArr)` = `bArr[2]` → **result / payload 首字节**

多包时接收方逐包去掉头 2 字节拼接（`b/c/b/m.java:54-69`）。

**简单命令的响应判定**（`b/c/b/r.java:473-507`）：

- `len == 3`：`bArr[2] == 1` → **成功**；`== 3` → **电量低**；其他 → 失败
- `len != 3` → 成功

**被动事件表**（`b/c/b/n.java`，注册见 `b/c/j.java:78-160`）

| cmd | 长度 | 事件 | payload |
|-----|------|------|---------|
| 0 | 3 | 音乐控制 | `[0][seq][op]`：`0`=请求信息 `1`=播放/暂停 `2`=下一首 `3`=上一首 |
| 1 | 3 | 相机控制 | bit0=拍照/录像模式 bit1=录像起停 bit2=拍照 bit3=闪光灯 |
| 2 | 3 | 电话控制 | `0`=号码 `1`=接听 `2`=拒接 |
| 3 | 2 | 请求翻页天气 | — |
| 4 | 2 | 查找手机 | — |
| 5 | 3 | 跑步状态 | `0`=开始 `1`=进行中 其他=结束 |
| 6 | 10 | **骑行/跑步实时数据（唯一需要手机回包）** | `[状态+类型:1][utc:4][距离:2]`；回包同为 10B |
| 9 | 2 | 自动同步 | — |
| 10 | 3 | 请求星历 | — |
| 16 | 4 | SOS / 起停 | `0`=SOS `1`=runOrPause |
| 19 | — | 老固件 ANT 控制 | 协议层重定向为 161（`b/c/b/v.java:180-184`） |
| 20 | 5 | 尾灯状态变化 | `bArr[2]` = 尾灯状态 |
| 21 | 1 | 充电状态变化 | `bArr[2]` = 充电状态 |

### 3.7 流控（包间隔）

`b/c/g.java` —— **取决于手机型号，不是设备机型**（参数 `gVar` 声明了但函数体没用到）：

| 手机 | 初始间隔 | 最小间隔 |
|------|----------|----------|
| Le X620（乐视） | 240 ms | 240 ms |
| OPPO R7sm | 60 ms | 35 ms |
| MI MAX / SM-N910C | 60 ms | 15 ms |
| SM705 / YQ601 | 60 ms | 10 ms |
| 其他（默认） | **60 ms** | **5 ms** |

自适应：连续成功 4 包则间隔 −5ms（不低于最小值）；写失败重试 ≥2 次则 +5ms（`b/c/b.java:284-337`）。

V1 的默认间隔是 **60ms**，最小由 `b/b/d/a()` 决定，写失败重试阈值 160ms（`b/b/d/x.java:218-227`）。

---

## 4. V1 协议（老机型）

适用：WeLoop Tommy (WN02B01)、Tommy2/B08 (WN08B08)、Now (BN01L02)、NowC (BN01SH02)、Now2

### 4.1 通道

只用一组 NUS：Service `6E400001-B5A3-F393-E0A9-77656C6F6F70`，写 `…0002`，通知 `…0003`（`b/b/i.java:25-31`）。
分包 20 字节（`b/b/d/d.java:70-72`），写类型 `WRITE_TYPE_NO_RESPONSE`。

### 4.2 两种包格式

#### 格式 A：`"H2DR"` 标签包，8 字节，带校验和

`b/b/g.java:34-42` + `b/b/f.java:19-22`

```
[0..3]  ASCII "H2DR"  (Host → Device Request)
[4]     cmd
[5]     arg1
[6]     arg2
[7]     checksum = sum(bytes[0..6]) & 0xFF
```

#### 格式 B：`0xA5` 二进制包，无校验和

```
[0]     0xA5
[1]     cmd
[2..]   payload（小端）
```

例：`ai.java:18-22`（天气开关）就是 `[0xA5][86][0/1]` 三字节。

**回包**：手机收到设备事件后，用 `d.b(byte[])` 把 payload 加上 `"D2HP"` 标签和校验和回给设备（`b/b/d/d.java:74-89`）。

### 4.3 两阶段事务（`aq`）

`b/b/d/aq.java`：

```
1. e()  → 发送 f()（8 字节命令头）
2. 设备回任意包 → 发送 v()（数据体）
3. 设备再回包   → l() 结束
```

`v()` 返回 `null` 的子类（`s.java`）是单阶段事务。
超时分三档：long 16s / middle 8s / short 4s（`b/b/d/d.java:12-29`），默认 10s。

**成功判定**：`bArr.length == 3 && bArr[2] == 1`（`ad.java:49-51`、`ak.java:69-71`）。
**回包路由**（`b/b/d/aw.java:223-255`）：`bArr[0] == 0xA5` → 按 `bArr[1]` 查被动表；否则按 `bArr[4]`（cmd）查主动表。
**退出码**：`0`=成功 `4`=正常结束 `-2`=超时 `-3`=忙 `-5`=GATT 为空。

### 4.4 命令表

**格式 A（"H2DR"）**

| cmd | arg1 | arg2 | 用途 | 文件 |
|-----|------|------|------|------|
| **144** | 11 | 0 | **时间同步**（第二阶段发 11B 时间数据） | `au.java:14` |
| **145** | 0 | 0 | 进入 DFU 模式 | `t.java:46` |
| **147** | 60 | 0 | 音乐信息 | `as.java:29` |
| **150** | 14 | 0 | 固件升级（头） | `ay.java:198` |
| **151** | len | 0 | 固件升级（数据块） | `ay.java:223` |
| **152** | 14 | 0 | 表盘推送（头） | `a/c.java:55` |
| **153** | len | 0 | 表盘推送（数据） | `a/c.java:282` |
| **154** | 9 | 0 | 读取表盘列表 | `j.java:98` |
| **155** | len | 0 | 同步通知过滤名单 | `at.java:24` |
| **156** | 0 | **0** | 退出相机模式 | `e.java:10` |
| **156** | 0 | **4** | 切换表盘视图 | `o.java:10` |
| **156** | 0 | **24** | 切换相机视图 | `m.java:10` / `n.java:10` |
| **156** | 0 | **26** | 切换录像视图 | `p.java:10` / `q.java:10` |
| **158** | len | ch | 设置闹钟（第二阶段发 4~5B 闹钟数据 + 校验和） | `z.java:130` |
| **159** | 0 | 0 | 读取运动汇总 | `l.java:134` |
| **172** | 0 | v | 消息亮屏 | `an.java:17` |
| **173** | 5 | 0 | 勿扰模式 | `aa.java:127` |
| **176** | 0 | v | 背光模式 | `ac.java:46` |
| **177** | 0 | v | 抬手亮屏（WN02B01 / WN08B08） | `ah.java:16` |
| **178** | 0 | v | 防丢 | `af.java:16` |
| **180** | v | 0 | 用户资料 | `am.java:97` |
| **181** | 15 | 0 | 运动目标 | `al.java:76` |
| **183** | 0 | v | 抬手亮屏（其他机型） | `ag.java:18` |
| **184** | 0 | v | 佩戴方式 | `aj.java:27` |
| **137** | 1 | v | 重启 / 复位（`v=0` 重启，`v=1` 复位） | `w.java:16` |

**格式 B（0xA5）**

| cmd | 长度 | 用途 | payload | 文件 |
|-----|------|------|---------|------|
| **64** | 14 | 字体修复 | — | `c.java:445` |
| **65** | 2 | 字体修复 | — | `c.java:301` |
| **66** | 11 | 字体修复 / 推送名片 | `[sub:1][offset:4][len:4]`（名片 sub=2） | `c.java:412` / `y.java:66` |
| **80** | 21 | **初始化 / 读取设备全量状态** | 见 4.5 | `i.java:83` |
| **85** | 7+len | 推送天气 | `[type][?][?] short 城市名(gb2312)` | `av.java:59` |
| **86** | 3 | 天气开关 | 1B | `ai.java:19` |
| **96** | 12 | 久坐提醒 | 9 个字段 | `ak.java:55` |
| **97** | 8 | 白天高亮 | `[开关][起h][起m][止h][止m]`… | `ad.java:39` |
| **98** | 8 | 心率设置 | — | `ae.java:72` |

**设备 → 手机 事件（被动）命令号**

| cmd | 用途 | 文件 |
|-----|------|------|
| 65 / 66 | 字体修复响应 | `c.java:422` |
| 68 | ? | `b.java:17` |
| 69 | ? | `f.java:24` |
| 70 | ? | `g.java:25` |
| 80 / 81 / 82 | 音乐控制 / 其他 | `r.java:32` |
| 83 | ? | `v.java:17` |
| 89 | ? | `a.java:26` |
| 90 / 91 | 相机控制（开始/停止录像） | `az.java:29` |
| 92 | ? | `ap.java:14` |
| **100** | 运动/实时数据（8 字节前缀） | `ao.java:59,76` |

另外，标准电量特征 `0x2A19` 的通知里，`bArr[1]` 被当作事件码复用（`b/b/d/x.java:127-152`）：
`1`=iOS 唤醒 `2`=开始跑步 `3`=跑步中 `4`=结束跑步 `5`=请求天气 `6`=自动同步。

### 4.5 初始化（cmd 80，21 字节）

`b/b/d/i.java:80-102`，小端：

```
[0]     0xA5
[1]     80
[2]     1
[3..8]  时间：年-2000、月、日、时、分、秒
[9]     身高/体重参数1
[10]    参数2
[11..12] 参数3 (short)
[13]    时区（偏移分钟 / 30）
[14][15] 参数4 / 参数5
[16][17] 开关1 / 开关2
[18]    单位体系（可选，仅 != -1 时附加）
```

响应：`[0xA5][80][len][data...]`，接收方用 `bArr[2]` 取长度再拼包（`i.java:53-59`）。

响应解析（`b/b/b.java:153-236`）按机型分两种布局：

- **WN02B01 / WN08B08**（`e()`）：偏移 7 的字节低 4 位=抬手亮屏、高 4 位=消息亮屏；偏移 10 的 `2`=竖屏左手 `1`=竖屏右手 `3`/`0`=横屏左手
- **BN01L02 / BN01SH02 / NOW2**（`d()`）：逐字节布局不同，且长度 ≥37 / 42 / 43 / 47 / 68 / 69 时逐段追加解析久坐、闹钟列表（最多 7 个）等

### 4.6 V1 机型差异

1. **功能码兜底（旧固件）** —— `b/e.java:294-309`，见第 6.2 节
2. **相机/录像视图** —— `b/b/e.java:274-291`
   - `WN08B08` → `m.java`(cmd 156 arg2=24) / `p.java`(cmd 156 arg2=26)
   - 其他 → `n.java`(156/24) / `q.java`(156/26)
3. **抬手亮屏** —— `b/b/e.java:323-351`
   - `WN02B01` → cmd 177，值 `1`/`2`
   - `WN08B08` → cmd 177，值 `(16|1)` / `(16|2)`（bit4 = 竖屏）
   - 其他 → cmd 183
4. **跑步轨迹** —— `b/b/e.java:548-556`：`WN02B01` 直接返回成功（不支持），其他发 `ar`
5. **闹钟条数** —— `b/b/e.java:802-805`：功能码含 `onceAlarmSupport`（'T'）时用 `z.b()`（支持单次+10 个），否则 `z.a()`
6. **Now2 心率** —— `b/b/b.java:60-86`：`NOW2` 且固件 ≥ "V 4.8" 时，初始化后额外下发一次心率设置（`ae`）

### 4.7 校验和 / CRC

`b/b/f.java:10-22`：逐字节相加取低 8 位，写入最后一个字节。

`b/b/g.java:78-93`：CRC16（初值由调用方给，常用 `0xFFFF`），逐字节：

```java
int v = crc & 0xFFFF;
for (i = start; i < start + len; i++) {
    int t = (((v & 0xFF) << 8) | ((0xFF00 & v) >> 8)) ^ (bArr[i] & 0xFF);
    int u = t ^ ((t & 0xFF) >> 4);
    int w = u ^ (((u << 8) << 4) & 0xFFFF);
    v = w ^ (((w & 0xFF) << 4) << 1);
}
```

---

## 5. V0 / DFU（固件升级）

### 5.1 V2 设备：cmd 148 + cmd 120

不支持 Nordic DFU 切换。流程（`b/c/e.java`）：

1. 计算整包 CRC16（`b/c/b/p.java:25-40`）
2. 发 cmd **148** type=0：`[0][len:4][crc:2]`；返回 `batteryLower` 则中止
3. `q.a(固件, 偏移)` → cmd **120**，dataIndex **7**，走数据通道
4. 失败一次整体重传（`e.java:112-120`）
5. 传完发 cmd **148** type=1 让设备校验
6. **XH3 + 固件 "V 1.7"** → 额外发 cmd **132** 重启（`e.java:190-193`）

COROS 多资源包（`b/c/f.java`）：主固件同样 148+120/7；之后遍历资源包用**自定义 dataIndex** 下发；资源包模式在 148 成功后 sleep 3000ms，最后发 cmd 132 重启。

### 5.2 V1 设备：cmd 145 → cmd 150/151

1. `t.java` 发 cmd **145** 进入 DFU
2. `ay.java` 发 cmd **150**（头，14B）+ cmd **151**（数据块）

### 5.3 Nordic Legacy DFU（`b/a/a.java` + `b/a/a/a.java`）

| 角色 | UUID |
|------|------|
| Service | `00001530-1212-efde-1523-785feabcd123` |
| 控制点（写 + 通知） | `00001531-1212-efde-1523-785feabcd123` |
| 数据包（write no response） | `00001532-1212-efde-1523-785feabcd123` |

流程（标准 Nordic Legacy DFU）：

| 步骤 | 写控制点 |
|------|----------|
| StartDFU | `[0x01, 0x11, 0x00]` |
| 写文件大小 | 写**数据包** `[size:4][0×16]` |
| 使能包回执通知 | `[0x08, 0x19, 0x00]`（每 25 包） |
| Receive FW Image | `[0x03, 0x11, 0x00]` |
| 发送固件 | 数据包，每包 20 字节，每 25 包 sleep 7ms |
| Validate | `[0x04, 0x11, 0x00]`；回执 `[0x10][0x04][0x01]` = 成功 |
| Activate & Reset | `[0x05, 0x11, 0x00]` |
| 取消 | `[0x06]` |

---

## 6. 功能码（Feature Code）

设备通过 **DIS `00002A27`（Hardware Revision String）上报一串字母**，每个字母代表一项能力。`com/yf/lib/bluetooth/c/c/o.java`。

| 字符 | 含义 | 字符 | 含义 |
|------|------|------|------|
| `A` | 新天气协议 | `S` | 24 小时天气预报 |
| `B` | 强震动 | `T` | 单次闹钟 + 10 个闹钟 |
| `C` | 横屏显示 | `U` | 长度单位体系设置 |
| `D` | 竖屏上下反转 | `V` | 运动轨迹开关 |
| `E` | 来电防火墙 | `W` | 内置 GPS |
| `F` | 静态心率 | `X` | 不显示抬手解锁 |
| `G` | 睡眠心率 | `Y` | 不显示消息过滤 |
| `H` | 久坐提醒 | `Z` | 不显示闹钟 |
| `I` | 白天高亮 | `a` | 不显示勿扰 |
| `J` | 动态心率 | `b` | 自定义运动页 |
| `K` | 表盘 | `c` | 24 小时时间格式 |
| `L` | 遥控相机 | `d` | 振动与蜂鸣器 |
| `M` | 二维码 | `e` | 设置首要追踪设备 |
| `N` | 背光设置 | `f` | 尾灯 |
| `O` | 消息亮屏 | `g` | 声音反馈 |
| `P` | 音乐控制 | `h` | 音乐均衡 |
| `Q` | 自动同步 | `i` | 遥控器连接 |
| `R` | 自动心率 | `j` | 用户指南 |
| — | — | `k` | 开启骑行 |
| — | — | `l` | 紧急联系人 |
| — | — | `m` | 语言设置 |
| — | — | `n` | 自动锁屏 |
| — | — | `o` | 国内设备 |

（`X`~`o` 中未列出的字符为保留；`unknown` = 字符 `'0'`）

### 6.1 解析

`b/e.java:310-323`：把 `2A27` 字符串逐字符 `o.from(c)`，非 unknown 的加入 `supportedFunctionCodeList`。读失败则列表清空。

### 6.2 旧固件兜底（关键机型差异）

`b/e.java:271-309`。先从**软件版本串**（如 `"V 4.3 02"`）用正则 `[0-9]+` 抽出主版本 `i3` 和次版本 `i2`（若第二段以 `0` 开头则取 0，否则 ≥10 时除以 10）：

| 机型 | 条件（固件版本） | 兜底写入的功能码串 |
|------|------------------|--------------------|
| **WN02B01** (Tommy) | `< 4.3` | `KLNP` — 表盘、相机、背光、音乐 |
| **WN08B08** (Tommy2/B08) | `< 4.3` | `ABKLMNP` — 新天气、强震动、表盘、相机、二维码、背光、音乐 |
| **BN01L02** (Now) | `< 4.7` | `DEO` — 竖屏反转、来电防火墙、消息亮屏 |
| **BN01SH02** (NowC) | `< 4.7` | `DEO` |
| **NOW2** | `< 4.9` | `ADEFIO` — 新天气、竖屏反转、来电防火墙、静态心率、白天高亮、消息亮屏 |
| 其余 9 个机型 | — | **不写任何兜底**，100% 依赖设备上报 |

> 即：新固件（≥ 各自阈值）被认为会自己在 `2A27` 里上报功能码串。COROS 全系、X9K、Neo/Now3、XH3、Hey 3S、MyLink+ 在 app 侧**没有任何硬编码功能码**。

协议栈里功能码只在 1 处生效：`b/c/h.java:411-415` —— 不支持 `U` 时强制公制。其余 20 余处判断都在 UI 层（`com/yf/smart/weloopx/device/setting/a/d.java`），即"功能码决定是否下发某条命令"，不在协议栈内分叉。

---

## 7. 时间同步速查表

### 7.1 公共算法

```java
// com/yf/lib/bluetooth/b/c/b/p.java:42-44
unixOffset = System.currentTimeMillis() / 1000 - 1388534400;   // 纪元 = 2014-01-01 00:00:00 UTC

// com/yf/lib/bluetooth/b/m.java
tz30 = tzOffsetMinutes / 30;   // 中国 +480 → 16
tz15 = tzOffsetMinutes / 15;   // 中国 +480 → 32
```

`m.a(int)`（单位选择）在代码中从未被调用，因此静态字段保持 0，`m.a()` 恒等于 `tz30`。

### 7.2 V2 机型（cmd 140）

`b/c/b/r.java:74-84`

```
包 = [140][seq][unixOffset:4 LE][tz30][tz15]           // 6 字节，默认
   = [140][seq][unixOffset:4 LE][tz30][tz15][extra]    // 7 字节，仅当 ag.a()==true
```

app 实际下发的是 `new ag(false, 0)`（`com/yf/smart/weloopx/device/setting/a/g.java:40`），所以**始终是 6 字节版本**。

示例（unix=1783392778，tz=+8）：`8c 00 8a 0f 89 17 10 20`

> ⚠️ 注意：cmd 140 的 payload 里**同时带时间戳和两个时区字节**。Hey 3S 额外有一个"时区修复"逻辑（`device/setting/a/g.java:29-36`）：仅当机型为 Hey 3S、固件 ≤ 2.12 且当前时区为负值（`m.b() < 0`）时，才会主动补发一次 syncTime（并在其后补发 sendUserInfo）。

### 7.3 V1 机型（cmd 144，两阶段）

`b/b/d/au.java`

```
阶段1（8 字节）: "H2DR" + [144][11][0][checksum]
阶段2（11 字节，设备回应后发）:
   [0..1] 年（LE，完整年份如 2026）
   [2]    月
   [3]    日
   [4]    时
   [5]    分
   [6]    秒
   [7]    时区（偏移分钟 / 30）
   [8][9] 0, 0
   [10]   checksum = sum(bytes[0..9]) & 0xFF
```

时间取 `now + 2000ms`（补偿传输延迟，`au.java:20`）。

### 7.4 V1 初始化里也带时间（cmd 80）

`b/b/d/i.java:72-77`，时间为 `[年-2000][月][日][时][分][秒]` 6 字节 + 时区 1 字节。

### 7.5 各机型时间同步方式汇总

| 机型 | 协议 | 命令 | 时间格式 |
|------|------|------|----------|
| Tommy (WN02B01) | V1 | 144（+11B 数据） | Y/M/D/H/M/S + tz/30 |
| Tommy2 (WN08B08) | V1 | 144 | 同上 |
| Now (BN01L02) | V1 | 144 | 同上 |
| NowC (BN01SH02) | V1 | 144 | 同上 |
| Now2 | V1 | 144 | 同上 |
| XH3 | V2 | 140 | unix-2014 纪元，4B 小端 + tz |
| **Hey 3S** | V2 | 140 | 同上（已实测） |
| Neo / Now 3 | V2 | 140 | 同上 |
| MyLink+ | V2 | 140 | 同上 |
| X9K | V2 | 140 | 同上 |
| COROS PACE / M1 / T1 | V2 | 140（也包含在 cmd 112 里） | 同上 |
| COROS OMNI | V2 | 140 | 同上 |
| COROS HELMET | V2 | 140（也包含在 cmd 112 里） | 同上 |

---

## 8. 代码定位索引

| 作用 | 文件 |
|------|------|
| 机型枚举 / 广播名 | `com/yf/lib/bluetooth/b/g.java` |
| 设备上下文、协议探测 | `com/yf/lib/bluetooth/b/e.java` |
| DIS 读取 | `com/yf/lib/bluetooth/b/f.java` |
| 时区计算 | `com/yf/lib/bluetooth/b/m.java` |
| 请求类型枚举（syncTime 等） | `com/yf/lib/bluetooth/c/b.java` |
| **V2** 协议入口 | `com/yf/lib/bluetooth/b/c/j.java` |
| **V2** GATT 回调 + UUID + 分包 | `com/yf/lib/bluetooth/b/c/b.java` |
| **V2** 任务分发（switch 全表） | `com/yf/lib/bluetooth/b/c/h.java` |
| **V2** 命令构造 | `com/yf/lib/bluetooth/b/c/b/r.java` |
| **V2** 事务基类 / 字节序 | `com/yf/lib/bluetooth/b/c/b/t.java`（`d()` 小端）、`d.java` |
| **V2** 响应字段读取 | `com/yf/lib/bluetooth/b/c/b/p.java` |
| **V2** 接收调度 | `com/yf/lib/bluetooth/b/c/b/v.java` |
| **V2** 被动事件 | `com/yf/lib/bluetooth/b/c/b/n.java` |
| **V2** 初始化（通用 / COROS 表 / 头盔） | `b/c/b/m.java` / `e.java` / `l.java` |
| **V2** 文件传输 / 数据同步 | `b/c/b/q.java` / `s.java` |
| **V2** OTA | `b/c/e.java` / `b/c/f.java` |
| **V1** 协议入口 | `com/yf/lib/bluetooth/b/b/i.java` |
| **V1** 任务分发 | `com/yf/lib/bluetooth/b/b/e.java` |
| **V1** GATT 回调 + 分包 | `com/yf/lib/bluetooth/b/b/d/x.java` |
| **V1** 组包 / 校验和 / CRC | `com/yf/lib/bluetooth/b/b/g.java` / `f.java` |
| **V1** 事务基类（两阶段） | `b/b/d/d.java` / `aq.java` / `s.java` |
| **V1** 事务调度 | `b/b/d/aw.java` |
| **V1** 初始化 + 响应解析 | `b/b/d/i.java` / `b/b/b.java` |
| **V1** 时间同步 | `b/b/d/au.java` |
| **DFU** | `com/yf/lib/bluetooth/b/a/a.java` / `b/a/a/a.java` |
| 功能码枚举 | `com/yf/lib/bluetooth/c/c/o.java` |
| 位打包器（COROS 用） | `com/yf/lib/bluetooth/e/a.java` |
| 包间隔（按手机型号） | `com/yf/lib/bluetooth/b/c/g.java` / `b/b.java` |

**混淆常量真值表**（反混淆必需）

| 常量 | 值 | 位置 |
|------|-----|------|
| `TinkerReport.KEY_APPLIED_INFO_CORRUPTED` | 124 | `TinkerReport.java:21` |
| `..._PACKAGE_CHECK_APK_TINKER_ID_NOT_FOUND` | 153 | :23 |
| `..._PACKAGE_CHECK_DEX_META` | 151 | :24 |
| `..._PACKAGE_CHECK_LIB_META` | 152 | :25 |
| `..._PACKAGE_CHECK_META_NOT_FOUND` | 155 | :26 |
| `..._PACKAGE_CHECK_PATCH_TINKER_ID_NOT_FOUND` | 154 | :27 |
| `..._PACKAGE_CHECK_SIGNATURE` | 150 | :29 |
| `..._PACKAGE_CHECK_TINKERFLAG_NOT_SUPPORT` | 158 | :30 |
| `..._PACKAGE_CHECK_TINKER_ID_NOT_EQUAL` | 156 | :31 |
| `..._PATCH_FILE_EXTRACT` | 181 | :32 |
| `..._DEX_EXTRACT` | 182 | :14 |
| `..._LIB_EXTRACT` | 183 | :22 |
| `..._RESOURCE_EXTRACT` | 184 | :33 |
| `..._VERSION_CHECK` | 180 | :42 |
| `W4SportDataType.YFSportDataTypeCadenceHeart` | 126 | `W4SportDataType.java:9` |
| `...CadenceHeartLevel` | 127 | :10 |
| `...CorosLapInfo` | 129 | :15 |
| `...CadencePace` | 130 | :11 |
| `...CadenceAltitude` | 132 | :7 |
| `BDLocation.TypeNetWorkLocation` | 161 | `BDLocation.java:54` |
| `BDLocation.TypeServerDecryptError` | 162 | :60 |
| `UIMsg.d_ResultType.SHORT_URL` | 500 | `UIMsg.java:168` |
| `q.LENGTH_UNIT_SYSTEM_NONE` | -1（即 `& 0xFF`） | `c/c/q.java:11` |

---

## 9. 尚未确认 / 需实测的点

1. **机型 → V1/V2 的归属是推断的**。协议选择实际由 GATT 服务决定，上表基于"协议层只对这 5 个机型做分支"推断。请用 nRF Connect 扫一下设备的 Service 列表确认：有 `E50E24DCCA9E` 的就是 V2。
2. **V1 格式 B（0xA5 包）没有校验和**，这是源码事实，但设备端是否真的不校验需实测。
3. **V1 响应包布局**：`bArr[0]==0xA5` 的走 `bArr[1]` 路由，非 0xA5 的走 `bArr[4]`（cmd）。非 0xA5 响应包的 0..3 字节内容源码里没体现，推测是 4 字节标签（可能是 `"D2HR"`），需抓包确认。
4. **`g.e(bArr)==2` 时丢弃响应**（`b/b/d/aw.java:231`）的语义未确定，推测是"设备忙/处理中"。
5. **cmd 112 响应各字段的确切含义**：源码只给出偏移和位，字段语义（哪个 bit 是哪个开关）多处理论推断，需要对照真机抓包校准。
6. **COROS 头盔（cmd 112 变体 C）** 的响应解析字段语义未完全还原。
7. **V2 时间同步对所有 V2 机型是否一致**：命令号 140 是共用的，但 COROS 手表/头盔的固件是否接受同样的 6 字节 payload 未验证。
8. **cmd 161 / 19（ANT+）** 的 payload 语义只还原了结构，子命令取值未确认。

---

## 附：最小可用实现（时间同步，V2 机型）

```javascript
// Web Bluetooth
const SERVICE = '6e400001-b5a3-f393-e0a9-77656c6f6f70';
const RX      = '6e400002-b5a3-f393-e0a9-77656c6f6f70';
const TX      = '6e400003-b5a3-f393-e0a9-77656c6f6f70';

const tzMin = -new Date().getTimezoneOffset();      // 东八区 = 480
const ts = Math.floor(Date.now() / 1000) - 1388534400;
const buf = new Uint8Array(8);
buf[0] = 140;                                        // cmd
buf[1] = 0;                                          // seq / 剩余包数
new DataView(buf.buffer).setUint32(2, ts >>> 0, true); // 小端
buf[6] = Math.floor(tzMin / 30) & 0xFF;
buf[7] = Math.floor(tzMin / 15) & 0xFF;

await rxCharacteristic.writeValueWithoutResponse(buf);
```

V1 机型需要两段握手（先发 `"H2DR"` + `[144][11][0][chk]`，等设备回应后再发 11 字节时间数据），无法用单次 write 完成。

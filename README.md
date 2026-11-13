# WeLoop 手表时间同步工具

> 停服也能对时间。一个 HTML 文件，用 Chrome 打开即可把手表的时间同步为当前时间，无需安装任何 app，不依赖云端服务器。

> 🔗 **在线直接用（无需下载）**：https://f00ladam.github.io/weloop-time/

WeLoop（唯乐）官方 app 早已停服，手表时间无法同步，云端登录走不通。
本项目支持多种weloop手表的同步时间协议。但本人只测试了小黑2。

## 文件说明

- README.md: 本说明文件，只有此文件是本人手打或Ctrl C+V的
- index.html: 由AI生成的同步时间用网页，基于weloop-time-sync的index.html，本人未编辑过(我让AI参考index.html生成了index2.html，并改名回index.html，我可能有删除过几行，但记不清了)
- PROTOCOL-ALL.md: 由AI整理的协议，供他人参考，本人未编辑过(原名如此，我可能有删除过几行，但记不清了)
- LICENSE: 许可证，复制于weloop-time-sync的LICENSE

## 使用方法

### 环境要求

| 平台 | 浏览器 | 支持 |
|------|--------|------|
| Windows / macOS / Linux | Chrome、Edge、Opera | ✅ |
| Android | Chrome（需 Android 6.0+，开启位置权限） | ✅ |
| iOS / iPadOS | 任何浏览器 | ❌（Apple 不开放 Web Bluetooth） |
| 任何平台 | Safari、Firefox | ❌ |

### 方式一：在线版（推荐，最省事）

直接在浏览器打开 GitHub Pages 托管的在线版，**无需下载、无需安装任何东西**：

> 🔗 **[https://f00ladam.github.io/weloop-time/](https://f00ladam.github.io/weloop-time/)**

打开后按下方「同步步骤」操作即可。建议收藏这个网址，以后随时对时间。

### 方式二：本地运行（离线 / 在线版打不开时备用）

1. **下载** [index.html](./index.html) 到本地
2. **用 Chrome或Edge等浏览器 打开** `index.html`
   - 直接双击在某些新版 Chrome 上可能被限制（Web Bluetooth 要求 https 或 localhost 安全来源），此时需要用本地 http 服务器托管：
     ```
     cd 到 index.html 所在目录
     python -m http.server 8000
     ```
     然后浏览器访问 `http://localhost:8000/`

### 同步步骤

1. **取消系统配对**：到系统蓝牙设置里，如果手表已配对，先**取消配对 / 忽略该设备**（Web Bluetooth 不能使用已配对设备，这是浏览器规范限制）
2. 打开上面任一方式的页面，**「选择设备前缀名」**，设置时区（中国填 `8`），点 **「连接并同步时间」**
3. 浏览器弹出蓝牙设备选择框，找到你的手表并 **「配对」**
4. 等待日志显示 `✅ 时间同步命令已发送完成！` 即可
5. 如果连上以后不能断开手表，可以尝试把电脑的蓝牙重新关闭再打开。


## 引用项目

已知github上有一些解决方案:
- 离线版APP: [weloop-offline](https://github.com/greatyingzi/weloop-offline)
但本人不想安装APP，只想简单的同步下手表时间。本人反编译了该APP，并由Codebuddy整理了协议。花费了免费的50积分。
- Hey 3S时间同步: [weloop-time-sync](https://github.com/micookie2/weloop-time-sync)
但该项目不支持本人手上的小黑2，故用AI重做了一遍。本项目基于weloop-time-sync，由Codebuddy基于上面整理出来的协议，修改了网页以支持多种协议。花费了免费的40积分。

本项目完全由Codebuddy制作，本人不承担任何责任，模型使用的是自动，本人甚至不知道它调用了哪个AI，用的也是免费的积分。

## 免责声明

- 本项目**仅供个人同步自己手表的时间使用**，不用于商业用途。
- 协议通过**逆向工程** WeLoop 非官方 app 所得，仅用于实现互操作。
- 本项目**不包含** 相关APP及其反编译源码，仅包含AI整理的协议以及由AI生成的网页。
- 本项目**由AI负责主要工作**，本人完全没有进行代码审查。
- 使用本工具的风险自负，本人以及AI不对任何设备损坏负责。

## 致谢

- 本项目基于[weloop-time-sync](https://github.com/micookie2/weloop-time-sync)
- 本项目反编译了APP[weloop-offline](https://github.com/greatyingzi/weloop-offline)
- 本项目使用了反编译工具[jadx](https://github.com/skylot/jadx/releases)
- 本项目使用了VSCode以及Tencent Cloud Codebuddy插件
- 感谢全人类的技术积累，感谢AI

## 📄 License

MIT
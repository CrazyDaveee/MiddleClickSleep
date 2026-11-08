# MiddleClickSleep

一个 Windows 10/11 托盘小工具：**双击鼠标中键**，让电脑进入系统睡眠，或只关闭显示器。

- 默认动作：**系统睡眠**（屏幕黑、系统挂起，网络通常断开）。
- 可选动作：**仅关闭显示器**（系统继续运行，下载/任务不中断）。
- 无需管理员权限，无需安装，不联网，无遥测。
- 单文件 `MiddleClickSleep.exe`；源码和构建脚本在项目目录，`release` 压缩包只包含运行所需文件。

## 先看这里：两种“黑屏”不一样

| 托盘里的动作 | 屏幕 | 系统/网络 | 怎么唤醒 |
| --- | --- | --- | --- |
| **系统睡眠**（默认） | 黑 | 系统挂起，网络通常断开 | 按电源键，或按键盘/鼠标（取决于设备设置） |
| **关闭显示器** | 黑 | 系统继续运行，下载继续 | 移动鼠标或按任意键，立即亮屏 |

如果你说的“黑屏睡眠”只是想让屏幕黑掉、电脑继续挂着下载，请在托盘菜单里改成 **关闭显示器**。
如果你要的是真正的睡眠，保持默认即可。

## 快速使用

1. 把 `MiddleClickSleep.exe` 放到一个固定目录，例如 `C:\Tools\MiddleClickSleep\`。
2. 双击运行。会打开一个 **“MiddleClickSleep - 运行中”状态窗口**，但**不会占用任务栏**；同时托盘出现鼠标+月亮图标。关闭窗口只会隐藏到托盘，不会退出。
3. 按一下中键：状态窗口会显示“收到第 1 次中键”；快速再按一次，默认会进入系统睡眠。如果当前是 UAC 过滤后的普通权限，第一次会询问是否以管理员身份重启；授权后会自动继续睡眠。
4. 右键托盘图标可以：
   - 暂停/启用双击手势；
   - 切换“系统睡眠 / 关闭显示器”；
   - 设置开机自动启动；
   - 显示状态窗口、查看日志、关于、退出。

状态窗口关闭后，双击托盘图标（或右键托盘 → “显示状态窗口”）可以重新打开。托盘图标可能藏在任务栏的 `^` 展开区里；开机自启动时只保留托盘图标，不显示窗口。

> 第一次中键点击会照常传给当前程序，第二次点击会被拦截。也就是说，在浏览器里双击中键可能先触发一次“中键打开新标签/自动滚动”，然后才睡眠。这是“单击不延迟”和“完全吞掉双击”之间无法同时满足的取舍。建议在桌面空白处使用这个手势。

## 安全测试（不会真的熄屏/睡眠）

先退出正在运行的实例，然后在命令行运行：

```powershell
.\MiddleClickSleep.exe --dry-run
```

再双击鼠标中键，状态窗口会显示“已识别双击 / 测试模式”，日志里会写 `Dry run: would ...`，但不会调用睡眠/熄屏 API。
确认手势识别正常后，退出测试实例，再正常启动即可。

如果按中键时状态窗口**没有任何变化**，说明程序没有收到这个中键。此时不是判定速度的问题，而是中键事件没有到达 Windows 低级鼠标钩子：可能是鼠标厂商软件改掉了中键功能，或远程桌面/虚拟机没有转发中键。

## 命令行参数

```text
MiddleClickSleep.exe                       正常启动，使用上次保存的设置
MiddleClickSleep.exe --dry-run             只检测手势，不执行电源动作
MiddleClickSleep.exe --action monitor      本次运行使用“关闭显示器”
MiddleClickSleep.exe --action sleep        本次运行使用“系统睡眠”
MiddleClickSleep.exe --startup             开机启动用：仅托盘，不显示窗口
MiddleClickSleep.exe --startup --quiet     开机启动用：仅托盘（显式安静）
MiddleClickSleep.exe --quiet               本次启动不显示状态窗口
MiddleClickSleep.exe --help                显示帮助
MiddleClickSleep.exe --version             显示版本
```

`--action` 只影响本次运行，不会改写托盘里保存的设置。

## 手势判定细节

- **时间阈值**：至少 **700 ms**，并取 Windows“双击速度”（默认 500 ms）中的较大值。中键按下去时鼠标容易被推动，所以这里比系统双击判定更宽松。
- **位置容差**：两次按下位置在 **32×32 像素**内（即横/纵位移都不超过 16 px）。
- **触发时机**：第二次中键**松开**后等待 450 ms 再执行动作，避免按键还没弹起就把屏幕关掉、随即又被唤醒。
- **三连击**：第二次触发后会重置，第三次重新算作单击。
- 只监听中键，不碰左右键和滚轮；其他鼠标事件原样放行。

如果按了两次但状态窗口显示“没有构成双击”，它会直接告诉你实际间隔/位移是多少，以及超了哪个上限。把窗口里显示的数字发给我，就能继续放宽或判断问题。

450 ms 的延迟可以在 `settings.ini` 里用 `delayMs` 调整（100–5000）。如果熄屏后马上又亮，先把延迟调大一些，或改用系统睡眠。

## 数据文件

优先存放在：

```text
%APPDATA%\MiddleClickSleep\
  settings.ini
  MiddleClickSleep.log
```

如果该目录不可写（便携环境、受限账户等），会自动退到临时目录或程序目录下的 `MiddleClickSleep-data\`。实际路径可以在托盘菜单的“关于”里看到。

`settings.ini` 示例：

```ini
# MiddleClickSleep settings
enabled=true
action=sleep
delayMs=450
```

- `action=sleep`：系统睡眠；`action=monitor`：仅关闭显示器。
- 直接编辑配置文件后，重启程序生效。
- 日志超过 1 MB 会自动轮转为 `MiddleClickSleep.log.old`。

## 开机自动启动

托盘菜单勾选“开机自动启动”即可。它只写当前用户的注册表：

```text
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
  MiddleClickSleep = "...\MiddleClickSleep.exe" --startup
```

不需要管理员权限。如果之后移动了 exe，请先取消勾选、再重新勾选，让注册表指向新路径。

## 卸载

1. 右键托盘图标 → 退出。
2. 如果勾过开机启动，先取消勾选（或删除上面的注册表值）。
3. 删除 `MiddleClickSleep.exe`。
4. 删除 `%APPDATA%\MiddleClickSleep\`（如果存在）。

## 从源码构建

要求：Windows 10/11，系统自带的 .NET Framework 4.x。**不需要**安装 .NET SDK、Visual Studio、NuGet 包或管理员权限。

```powershell
powershell -ExecutionPolicy Bypass -File .\build.ps1
```

输出：

```text
dist\MiddleClickSleep.exe
```

可选参数：

```powershell
# 跳过双击判定单元测试
powershell -ExecutionPolicy Bypass -File .\build.ps1 -SkipTests

# 构建并生成 release\MiddleClickSleep-1.3.0-win.zip
powershell -ExecutionPolicy Bypass -File .\build.ps1 -Package
```

构建脚本用 Windows 自带的 `csc.exe`（.NET Framework 4.x）编译，并自动运行双击判定和鼠标钩子的单元测试。`assets\app.ico` 和 `assets\app.manifest` 会嵌入 exe。

目录结构：

```text
middle-click-sleep\
  src\                        C# 源码
  tests\                      双击判定和鼠标钩子单元测试
  assets\                     图标、应用清单
  tools\make_icon.py          重新生成图标（可选，需要 Pillow）
  build.ps1                   构建/测试/打包
  dist\MiddleClickSleep.exe   构建产物
```

## 实现原理

1. 用 `SetWindowsHookEx(WH_MOUSE_LL, ...)` 安装全局低级鼠标钩子。
2. 只处理 `WM_MBUTTONDOWN` / `WM_MBUTTONUP`：
   - 记录上一次中键按下的时间和坐标；
   - 在至少 700 ms、32×32 像素范围内出现第二次按下，判定为双击；
   - 拦截第二次按下和抬起，第一次正常放行。
3. 第二次抬起后启动 450 ms 计时器，然后执行：
   - **关闭显示器**：`SendMessageTimeout(HWND_BROADCAST, WM_SYSCOMMAND, SC_MONITORPOWER, 2)`；失败时用 `PostMessage` 再试一次。
   - **系统睡眠**：启用 `SeShutdownPrivilege`，调用 `SetSuspendState(0, 0, 0)`（普通睡眠，不强制休眠）。UAC 过滤后的标准权限无法启用该特权时，会提示管理员重启，或退化为关闭显示器。
4. 电源动作失败会写日志，并在状态窗口/托盘气泡里提示。
5. `ApplicationContext` + `NotifyIcon` 提供状态窗口和托盘菜单；`Mutex` 保证只有一个实例。

不使用键盘记录、不读取鼠标位置历史、不联网。

## 现成工具的情况

- **Windows 本身没有**“双击中键 → 睡眠/熄屏”的设置。Windows 只有双击速度、鼠标键之类的基础设置。
- **部分鼠标厂商的驱动/配套软件**自带类似功能。例如 Microsoft Q&A 里有用户报告某些鼠标双击中键会让 PC 睡眠；回答认为这是鼠标或第三方软件的功能，不是 Windows 功能。这种功能取决于品牌/型号，不是通用方案。
- **AutoHotkey** 可以用十行左右的脚本实现“双击中键 → 关屏”，但需要自己处理双击判定、第二次点击拦截、熄屏延迟；还要安装 AutoHotkey 或自己编译脚本。它没有现成的、开箱即用的这个手势。
- **Snooze** 这类工具是“点一下图标关屏”，不是鼠标手势，也没有系统睡眠。

所以没有一个通用的、开箱即用的“中键双击睡眠”工具；要么碰运气看鼠标驱动，要么用 AutoHotkey 自己写，要么用这个小工具。

参考：

- Microsoft Q&A: <https://learn.microsoft.com/en-us/answers/questions/4137260/my-pc-goes-to-sleep-after-i-clicked-the-middle-mou>
- AutoHotkey 文档: <https://www.autohotkey.com/docs/v2/>
- Snooze: <https://ispoto.github.io/Snooze/>

## 睡眠权限与错误 1300/1314

真正睡眠需要进程令牌里可用的 `SeShutdownPrivilege`。如果程序运行在 **UAC 过滤后的管理员/标准权限**下，这个特权可能出现在 `whoami /priv` 里，但 `AdjustTokenPrivileges` 会返回 `ERROR_NOT_ALL_ASSIGNED (1300)`，随后 `SetSuspendState` 返回 `ERROR_PRIVILEGE_NOT_HELD (1314)`，睡眠就会失败。

新版的处理方式：

1. 第一次睡眠失败且错误码是 1300/1314 时，会弹窗询问是否 **以管理员身份重启**。
2. 点“是”会触发一次 UAC；授权后以管理员权限重新运行。
3. **授权后程序会自动继续睡眠**，不用再按一次中键。
4. 点“否”或取消 UAC，程序会自动退化为 **关闭显示器**，至少先黑屏。

也可以手动操作：右键 `MiddleClickSleep.exe` → **以管理员身份运行**。
如果你的账户不是管理员、无法通过 UAC，那么只能使用“关闭显示器”模式，或者请管理员在本地安全策略里给账户分配“关闭系统”权限。

> 开机自动启动使用的是普通权限的 Run 键；因此登录后的第一次睡眠仍可能弹 UAC。若希望开机也无提示地睡眠，需要用“任务计划程序”创建一个“使用最高权限运行”的登录任务，或者每次手动以管理员身份启动程序。

## 已知限制

- **UAC 安全桌面、锁屏界面、部分游戏/远程桌面**里，低级鼠标钩子可能收不到事件。
- **鼠标厂商软件**如果也拦截了中键，可能先于本工具处理；必要时在厂商软件里把中键设为默认。
- **多显示器**会一起关闭（`SC_MONITORPOWER` 是全局的）。
- **系统睡眠**取决于硬件和电源计划；有些 Modern Standby 机器睡眠时网络仍保持连接。如果 `SetSuspendState` 失败，日志会记录 Win32 错误码。
- 熄屏后如果鼠标/键盘有轻微输入，可能立即唤醒；可增大 `delayMs`，或改用系统睡眠。
- 这是**未签名**的小工具，Windows SmartScreen 可能提示“未知发布者”。可以核对 `release\SHA256.txt`，或直接用源码构建。
- 全局鼠标钩子偶尔会被杀毒软件误报；源码完全公开，可自行审查/编译。
- 不会主动锁屏；是否在睡眠唤醒后要求登录，由 Windows 的“唤醒时需要密码”设置决定。
- 第一次中键点击会传给前台程序，见上面的说明。

## 常见问题

**按中键时状态窗口没有任何变化？**
说明 Windows 低级鼠标钩子没有收到中键事件，不是双击速度的问题。优先检查：
- 菜单里“启用双击中键”是否勾选；
- 鼠标厂商软件是否把中键改成了别的功能；
- 是否在远程桌面/虚拟机里，宿主没有把中键转发进来；
- 换一个鼠标试试，排除中键硬件故障。

**状态窗口显示“收到第 2 次中键，但没有构成双击”？**
窗口会写出实际间隔、位移和上限。按提示按快一点或尽量别移动鼠标即可；如果数字只超了一点点，可以把窗口里的数字发给我，我再放宽。

**识别到双击但没有睡眠？**
查看日志里的 Win32 错误码：
- `1300` / `1314`：当前权限无法启用 `SeShutdownPrivilege`，见上面的“睡眠权限”一节；程序会提示管理员重启，或先退化为关闭显示器。
- 其他错误：可能是电源计划/驱动不支持 `SetSuspendState`。可先在托盘菜单里改成“关闭显示器”验证手势链路。

**熄屏后马上又亮？**
先把 `delayMs` 调到 800–1000；检查鼠标是否放在会抖动的表面、键盘/触摸板是否有输入。仍不行就改用系统睡眠。

**会损伤显示器吗？**
不会。这是 Windows 标准的显示器电源管理请求，和电源计划里的“关闭显示器”是同一类操作。

**能只关某一个显示器吗？**
不能。当前实现调用系统级 `SC_MONITORPOWER`，会关闭所有显示器。按显示器单独熄屏需要 DDC/CI，复杂很多，不在本工具范围内。

**需要管理员权限吗？**
关闭显示器不需要。真正睡眠需要进程令牌里可用的 `SeShutdownPrivilege`；UAC 过滤后的管理员/标准权限可能无法启用它，此时程序会提示以管理员身份重启。若无法提升，就只能用“关闭显示器”模式。

## License

MIT，见 [LICENSE](LICENSE)。

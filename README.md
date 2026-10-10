# TrafficMonitor · 网络流量统计器

一个单文件 PowerShell 脚本，持续统计本机**累计用了多少流量**（接收/发送），
账本存在磁盘上，**关机、重启、注销后都接着计数**，不丢也不重复。

```
                       接收(下载)      发送(上传)            合计
 本次开机                1.23 GB         456.7 MB        1.68 GB
 今天                    5.67 GB          1.20 GB        6.87 GB
 本月                  120.40 GB         30.20 GB      150.60 GB
 总计(自监测开始)        1.23 TB         345.6 GB        1.57 TB
```

---

## 1. 文件清单

| 文件 | 说明 |
| --- | --- |
| `TrafficMonitor.ps1` | 主程序（唯一必需的脚本，约 1200 行，含注释） |
| `TrafficMonitor.cmd` | 双击即看用量的启动器 |
| `README.md` | 本文档 |

运行期产生的文件（默认在 `C:\ProgramData\TrafficMonitor\`）：

| 文件 | 说明 |
| --- | --- |
| `state.json` | **账本**，所有累计数据都在这里，删掉等于清零 |
| `monitor.log` | 运行日志（超过 1MB 自动只留最后 1000 行） |
| `monitor.lock` | 单实例锁文件 |
| `TrafficMonitor.ps1` | `install` 时自动复制的脚本副本（计划任务执行的是它） |

---

## 2. 快速开始

```powershell
cd <本目录>

# ① 安装开机自启并立刻开始计数
powershell -ExecutionPolicy Bypass -File .\TrafficMonitor.ps1 -Action install

# ② 查看用了多少流量
powershell -ExecutionPolicy Bypass -File .\TrafficMonitor.ps1            # 等同 -Action status

# ③ 最近 30 天日报表
powershell -ExecutionPolicy Bypass -File .\TrafficMonitor.ps1 -Action report -Days 30
```

也可以直接双击 `TrafficMonitor.cmd` 看用量。

---

## 3. 为什么"关机再开机"也能接着计数

这是本工具的核心，值得花一分钟理解。

**问题**：Windows 把网卡的收发字节数记在网卡驱动里（`Get-NetAdapterStatistics` 的
`ReceivedBytes` / `SentBytes`）。这个计数器**每次开机都从 0 重新开始**，
所以它只能表示"本次开机以来用了多少"，直接读它永远得不到累计值。

**做法**：脚本自己维护一份持久化账本，每 5 秒采样一次：

```
每轮循环：
  读取所有网卡的原始计数器 current
  增量 delta = current - 上次记录的 last          （正常增长）
  ┌ 如果 current < last  → 计数器已归零（重启 / 驱动重载 / 网卡重新启用）
  │                       增量 delta = current     （即"本次开机以来的全部流量"）
  └ 如果是新网卡         → 增量 delta = current
  把 delta 累加进 state.json，并记下 last = current
```

- **不丢**：累加结果写在磁盘上，关机后依然在。
- **不重复**：每次只加"差值"，重启后按新计数器的值补计，绝不会把 1GB 算两次。
- **开机→登录的流量也算得到**：重启后计数器从 0 开始累计，登录后第一次采样读到
  的就是"本次开机到此刻"的总量，会被一次性补计进来。

由此推出两条边界（见第 6 节"准确性"）：最坏情况下丢失的是**进程被杀死前最后
一个采样周期**（≤5 秒）的流量。

---

## 4. 命令与参数

### 动作（`-Action`）

| 动作 | 说明 |
| --- | --- |
| `status` | 查看本次开机 / 今天 / 本月 / 总计用量 + 实时速率 + 各网卡累计（默认） |
| `report` | 最近 N 天的日报表 + 按月汇总 |
| `run` | 后台监测循环（由计划任务调用；`-Live` 可前台实时刷新，Ctrl+C 退出时自动保存） |
| `install` | 复制脚本到状态目录并注册开机自启计划任务 |
| `uninstall` | 移除计划任务（`-Purge` 连账本一起删） |
| `reset` | 清零账本，从当前时刻重新计数（会先停止监测进程，旧账本自动备份为 `.bak-时间戳`） |
| `selftest` | 运行内置自检（22 项断言，校验增量算法与跨重启逻辑） |
| `help` | 帮助 |

### 常用参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `-IntervalSeconds` | `5` | 采样间隔。越小越精确，5 秒足够；不要小于 1 |
| `-Scope` | `physical` | `physical` 只统计物理网卡；`all` 统计全部网卡 |
| `-IncludeAdapter` / `-ExcludeAdapter` | 无 | 按"网卡名 + 描述"做正则过滤，例如 `-ExcludeAdapter '蓝牙'` |
| `-StatePath` | `C:\ProgramData\TrafficMonitor\state.json` | 账本位置 |
| `-Provider` | `auto` | 采集方式：`netadapter`（默认首选）/ `perfcounter` / `auto` |
| `-InstallMode` | `auto` | `auto`＝管理员用开机启动，否则用登录时启动；也可强制 `startup` / `logon` |
| `-Json` | 关 | `status` / `report` 以 JSON 输出，便于其他程序读取 |
| `-Force` | 关 | `reset` / `uninstall` 跳过交互确认 |

### 示例

```powershell
# 前台实时看速率（Ctrl+C 退出）
.\TrafficMonitor.ps1 -Action run -Live

# 只统计以太网和 WLAN，忽略其它网卡
.\TrafficMonitor.ps1 -Scope all -IncludeAdapter '以太网','WLAN' -Action status

# 2 秒采样一次
.\TrafficMonitor.ps1 -Action install -IntervalSeconds 2

# 清零重来
.\TrafficMonitor.ps1 -Action reset

# 给监控面板/脚本读的 JSON
.\TrafficMonitor.ps1 -Action status -Json
```

---

## 5. 统计范围：为什么默认不统计所有网卡

如果把自己电脑上所有网卡的流量加起来，会**重复计数**，例如：

- `Tailscale` / WireGuard / OpenVPN 隧道网卡上的流量，本身就跑在物理网卡上；
- `VMware Network Adapter VMnet1/8`、`Hyper-V vEthernet`、`Wintun`、`TAP-Windows`
  是虚拟/本地网卡；
- `Microsoft Wi-Fi Direct Virtual Adapter`（"本地连接* 2"）是热点/直连虚拟网卡，
  数据实际从 `WLAN` 收发。

所以默认 `-Scope physical`：只统计 `HardwareInterface = $true` 的**物理网卡**
（本机即"以太网"和"WLAN"），并始终排除回环与隧道伪接口。
需要全都统计时用 `-Scope all`。

被排除的网卡其原始计数器**仍会被记录**（只是不计入总量），
所以你随时改 `-Scope` 都不会因为重新建立基线而重复计数。

---

## 6. 准确性说明与已知限制

| 情况 | 影响 |
| --- | --- |
| 采样间隔 | 进程被强杀/直接断电时，最多丢失最后一个周期的流量（默认 ≤5 秒） |
| 正常关机/注销 | 退出时会把账本落盘，几乎不丢 |
| 重启边界 | 若上次会话的计数器很小（A），而本次开机到第一次采样间已超过 A，则最多少计 A。此偏差**自限**：A 越小偏差越小 |
| 睡眠/休眠 | 计数器不清零，按差值照常统计，无影响 |
| 计数器回绕 | 64 位计数器不会回绕；万一变小一律按"重置"处理，不会算出天文数字 |
| 网卡热插拔 | 新网卡按"从 0 开始"计入，符合直觉 |
| 统计对象 | 只统计**总量**；"哪个程序用了多少"请看 Windows 设置 → 网络和 Internet → 数据使用量 |

**更精确的安装方式**：以管理员身份运行 `-Action install`，会注册为
**开机启动 + SYSTEM** 的计划任务，开机数秒后即开始采样，
不存在"登录前"的空档，也不受用户登录/注销影响。

---

## 7. 常见问题

**Q：怎么看它到底在不在跑？**
`-Action status` 会显示 `监测进程 : 运行中（N 秒前更新）`；也可以
`Get-ScheduledTask -TaskName TrafficMonitor`。

**Q：想换个位置放账本？**
所有命令加 `-StatePath "D:\流量\state.json"`；`install` 时会把这个路径写进计划任务。
（`-StatePath` 若指向目录，会自动追加 `state.json`。）

**Q：怎么彻底卸载？**
```powershell
.\TrafficMonitor.ps1 -Action uninstall -Purge    # 移除任务 + 删除账本和脚本副本
```

**Q：资源占用？**
实测稳定态 **CPU ≈ 1.5% 单核**（每 5 秒一次 `Get-NetAdapterStatistics`，单次约 130ms）、
内存约 80MB（PowerShell 进程本身的开销）。嫌高就把 `-IntervalSeconds` 调到 10~30。

**Q：会不会装了两个实例重复计数？**
不会。`monitor.lock` 以独占方式占用，第二个实例会直接退出（退出码 2）。
`reset` / `install` 也会先停掉正在运行的实例。

**Q：中文乱码？**
脚本是 UTF-8 with BOM，PowerShell 5.1 能正确识别。若你改过编码，请保持带 BOM。

---

## 8. 自检与验证结果

```
> .\TrafficMonitor.ps1 -Action selftest
 1) 增量算法 Get-TrafficDelta           6 项断言
 2) 跨重启累计（含虚拟网卡过滤）        13 项断言
 3) 序列化往返                          3 项断言
 全部自检通过 √
```

除了内置自检，还在本机做过真实链路验证：

- **重启模拟**：把账本里的计数器改成"重启前的大值"（107GB）后再采样，
  补计增量 = `146,611,057` 字节，当时网卡原始值 = `146,611,779` 字节，
  误差 722 字节（仅采样瞬间的正常流量）—— 说明"重启后按本次开机计数器补计"精确正确。
- **精度**：账本增量与网卡计数器增量一致，差异仅来自两次读取之间约 130ms 的时间差。
- **一致性**：`total == 各天之和 == 计入范围的网卡累计之和`。
- **单实例锁**：第二个 `run` 实例退出码为 2，不重复计数。

# 校园网连接与 MAC 变更（南岭宿舍有线）

> 2026-09-30 实测补充。适用范围：吉大南岭校区宿舍有线网接入。

## 一、🔴 核心规则：有线口认的是「登记在自助平台上的那个 MAC」

吉大校园网按 **MAC 地址授权**：网口后面有没有网，取决于**你账号绑定的 MAC 和当前网卡 MAC 是否一致**。

- 新设备插网线 → **改自助平台登记的 MAC**，不是改本机网卡
- ⚠️ 这两件事最容易搞混，用户说的"改 MAC"指的是**自助平台上的变更操作**

## 二、标准操作：无线网卡 → 有线网卡

**场景**：之前用 WiFi 上网，账号绑的是无线网卡 MAC；现在插网线，要把绑定改成有线网卡。

| 步骤 | 操作 |
|---|---|
| 1 | 查**有线**网卡 MAC：`Get-NetAdapter -Name '以太网 3' \| Select MacAddress`（或用 `ipconfig /all` 看"物理地址"） |
| 2 | 打开自助营业厅 `https://ip.jlu.edu.cn/pay/` |
| 3 | 登录：**一卡通卡号** + **身份证后 6 位**（⚠️ 不是学号、不是上网账号密码） |
| 4 | 找到「变更 MAC」→ 填入**有线网卡**的 MAC |
| 5 | 提交后**重新插拔网线 / 重启网卡**，让认证重新走一遍 |

⚠️ **一个账号同一时间只能绑一个 MAC** —— 改成有线后，原来那台绑 WiFi 的设备就失效了。想两头都用需要两个账号。

## 三、账号体系（最容易记混的三套）

| 用途 | 账号 | 密码 |
|---|---|---|
| **自助营业厅**（缴费 / 改 MAC / 销户） | **一卡通卡号** | 身份证后 6 位 |
| **上网认证**（客户端 / Portal 登录） | **吉大邮箱**（去掉 `@jlu.edu.cn`） | 邮箱密码 |
| 教务/办公系统 | 学号 | 统一身份认证密码 |

三套完全独立 —— 新生最常卡在这里。

## 四、诊断：怎么判断「是 MAC 没登记」还是「网口没开通」

跑 `校园网络\有线口体检.ps1`，看 **IP 段**：

| 拿到的 IP | 含义 | 处理 |
|---|---|---|
| `10.x.x.x` | ✅ 已授权，链路正常 | 去客户端/Portal 认证 |
| `169.254.x.x` | ❌ DHCP 无应答 = **没授权**或端口没开通 | 核对 MAC 登记 / 换口 / 报修 |

**关键判据（只读，不改任何配置）**：

```powershell
# 1) 拿到什么 IP
Get-NetIPAddress -InterfaceAlias '以太网 3' -AddressFamily IPv4 |
    Select IPAddress, PrefixOrigin

# 2) 二层有没有真邻居（网关 ARP 是否有 MAC）
Get-NetNeighbor -InterfaceAlias '以太网 3' -AddressFamily IPv4 |
    Where { $_.LinkLayerAddress -notin @('00-00-00-00-00-00','FF-FF-FF-FF-FF-FF') }

# 3) 物理层（LinkSpeed 有值只说明线协商上了，不代表通网）
Get-NetAdapter -Name '以太网 3' | Select LinkSpeed, MediaConnectionState
```

### 实测教训（2026-09-30）

- 网卡 `LinkSpeed 1Gbps / Connected` + 网关 ARP 有真 MAC + `ping 网关通`
  → **链路是通的**，问题在授权层，不在物理层
- 只拿到 `169.254.241.100`（APIPA）+ ARP 表里只有别的设备的 169.254 地址
  → 同一线路上的设备**全都没授权**，典型"插错口 / 未登记"
- ⚠️ **物理层"看着通"是最容易误判的地方** —— 一定要看 IP 段和 ARP

## 五、🔴 铁律：动网络配置前必须先问用户

**2026-09-30 事故**：看到用户是静态 IP（`<示例IP>/24`，非 `10.x` 内网段），
按"学校标准应 DHCP"擅自改掉 → **导致断网**，被用户明确批评。

**教训**：
- 那个静态配置是用户**刻意设的**，且本来能上网 —— **能上网 ≠ 配置正确，配置合规 ≠ 用户需要**
- 用户环境里的"非常规配置"往往是**有意为之**（多环境切换、遗留配置、特定软件要求）
- 正确姿势：**先只读诊断 → 报告发现 → 问用户要不要改 → 才动**
- 本机现状（作为对照基线）：
  - `以太网 3`（USB Realtek GbE，MAC `<有线网卡MAC>`）
  - 静态 `<示例IP>/24`，网关 `<示例IP>`，DNS `10.10.10.10` + `202.98.18.3`
  - Radmin VPN 虚拟网卡在启用状态

---

## 六、DRCOM 客户端频繁断线（"老是断链"）排查

> 2026-10-01 实测。吉大认证客户端 = **DRCOM**（`C:\Drcom\DrUpdateClient\`，进程 `DrClient`/`DrMain`/`DrUpdate`），认证服务器 `auth.jlu.edu.cn` → **`10.100.61.3`**。

### ⚠️ 第一原则：先分清「物理链路断」还是「认证会话断」

**90% 的"断链"是认证会话掉（超时离线），不是网卡断。** 别一上来就查驱动/网线。

只读分诊三连（都不改配置）：

```powershell
# 1) 到网关 + 到认证服务器 丢包率 —— 零丢包 = 物理链路好，问题在认证层
ping -n 30 <示例IP>          # 有线网关
ping -n 5  10.100.61.3             # DRCOM 认证服务器

# 2) 看是否多网卡抢路由（关键！）
Get-NetRoute -DestinationPrefix "0.0.0.0/0" | Sort-Object InterfaceMetric |
    Select InterfaceAlias, NextHop, InterfaceMetric, RouteMetric

# 3) 读客户端日志（真正的证据在这里）
```

**日志位置**：`C:\Drcom\DrUpdateClient\Log\`
- `auth_log.txt` — 认证主日志（**会涨到十几万行/几 MB**，别整读，用 `Get-Content -Tail 200`）
- `DrUpdate_Log<YYYYMMDD>.txt` / `DrClient_Log<YYYYMMDD>.txt` — 按天分的小日志

### 日志里的关键判据

| 日志行 | 含义 |
|---|---|
| `LGI STM <n> <bytes> ...` 每 20 秒一条、n 递增 | ✅ 心跳正常，会话活着 |
| `ERR:连接认证服务器超时，您已经离线！` | 🔴 **会话被踢** —— 这是"断链"的实录 |
| `Delete route for 10.100.61.3 ... faild! dwRetVal = 1168` | ⚠️ 客户端在删路由但**那条路由不存在**（1168 = ERROR_NOT_FOUND）→ 路由管理逻辑错乱 |
| `route DELETE 10.100.61.3 ... <WiFi网段> IF 5` | 🔴 **按 WiFi 网关(<WiFi网段>)/WLAN 接口(IF 5) 删路由**，但 `Client Auth IP Address` 是**有线 IP** → **客户端把"认证网卡"和"路由管理网卡"搞混了** |
| `ERR:pConfigRule is NULL!` | 噪声，每次都刷，**无害，别追** |

### 根因：多网卡（有线+WiFi+VPN）并存 → DRCOM 路由管理选错网卡

DRCOM 客户端检测到多张网卡时，会拿一张**过期快照**（上次登录时的网卡/网关）反复删路由，
而实际认证走的是另一张网卡 → 错位累积 → 触发认证会话重置。

**验证手法**：对比日志里 `Client Auth IP Address:`（实际认证网卡）和 `route DELETE ...` 行末尾的网关/接口号，**不一致即坐实**。
另：日志开头的 `Host Address:` 列表常**只列出 WiFi/VPN 的 IP，漏掉有线 IP** —— 也是同一问题的旁证。

### 处理顺序（从最小侵入开始，**每步都要用户点头**）

1. **抬高非主用网卡的 metric**（可回滚，先做这个）
   ```powershell
   Set-NetIPInterface -InterfaceAlias "Radmin VPN" -InterfaceMetric 100
   ```
2. **给认证服务器钉静态路由**（精准，不动全局配置）
   ```powershell
   route -p add 10.100.61.3 mask 255.255.255.255 <示例IP> metric 1
   ```
3. **关掉多余网卡**（最可能根治，但要用户同意）
   ```powershell
   Disable-NetAdapter -Name "WLAN" -Confirm:$false    # 有线已通时 WiFi 纯属多余
   ```
4. 不需要 VPN 时 → **退出/卸载 VPN**（本次实测：卸了 Radmin 后默认路由回归有线，metric 35 < WiFi 45，但**断线未根治** → 说明元凶是多网卡本身而非某个 VPN）

⚠️ 改网卡禁用/metric 都属于"动网络配置"，**必须先问用户**（见第五节铁律）。

### ⚠️ 本会话踩的坑

- **`Get-WinEvent -FilterHashtable @{...; ProviderName='...'}` 在部分机器上报 `The parameter is incorrect`** → 去掉 `ProviderName` 过滤，改为取回后 `Where-Object {$_.ProviderName -match ...}` 过滤
- **`Get-Process` 拿到 `Path` 为空 = 进程以 SYSTEM/管理员权限运行**（正常权限读不到路径），**不代表进程不存在**；要查路径用 `Get-CimInstance Win32_Process`（可能同样被拒，那就直接找安装目录）
- **终端里 `exit_code=1` 常常不是失败**：`Get-Process`/`Get-WinEvent` 过滤命中 0 条就返回非零，`ping` 成功也可能非零 —— **看输出内容判断，别只看 exit_code**（本次连续 6 次被误报"terminal has failed"，其实每条命令都成功执行了）
- DRCOM 客户端里**长期存在**的 `ERR:pConfigRule is NULL!` 是红鲱鱼，别当线索追

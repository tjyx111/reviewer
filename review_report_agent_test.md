# SDK 审查报告 —— `netfilter/session.go`（v1，只读）

| 项 | 内容 |
| --- | --- |
| 审查对象 | `D:\code\lbsprivate\lbs-sdk\netfilter\session.go`（工作区版本，596 行） |
| 版本基线 | Git HEAD `379c983`（2025-10-10，`fix: ipv4分片重组`）；本文件在工作区为已修改状态（`M netfilter/session.go`），未提交 |
| 版本约束 | v1（仓库内无 tag；历史上有 `cc68a2e`、`338e18c` 两个名为 `fix: v1` 的提交） |
| 审查方式 | 纯静态只读（Read/Grep）；未执行构建、测试、vet |
| 修改 SDK | 无。本轮未写入或修改任何 `lbs-sdk` 下的文件 |
| 证据获取说明 | 审查轮唯一的 Bash 调用被拒绝，未执行任何命令；下文行号均为静态阅读所得 |

---

## 1. 总体结论

**判定：当前工作区版本的 `session.go` 不适合作为 v1 交付或合入基线。**

它是“本机 IP 包 ↔ VPN 隧道”的核心适配层，承担 NAT 改写、校验和重算、分片/重组与注入，但目前存在：

- **3 个高严重性正确性缺陷**：出站分片处理会把首片当完整报文重写（其余片丢弃）、NAT 表无生命周期（65536 后永久失效）、NAT 复用只按源 IP 匹配（回包可投递到错误 socket）；
- **1 个半实现能力**：IPv6 出站被完整改写并分片，但传输层的 `Writer` 只接受 IPv4，实际必然被丢弃；入站 IPv6 直接 panic；
- **1 个版本治理问题**：被审文件未提交，结论无法绑定到可复核的 SHA。

报告分严重性汇总如下。

| 编号 | 严重性 | 摘要 | 位置 |
| --- | --- | --- | --- |
| H-1 | 高 | 出站 IPv4 分片：首片按完整报文重写、其余片丢弃 | session.go:312-321, 443-467 |
| H-2 | 高 | NAT 会话表只增不删，65536 后新流全部被丢 | session.go:367/387/406/425；nat.go:29-34, 92-125 |
| H-3 | 高 | NAT 复用只比对 `RealSrcIP`，可复用出错误 `RealSrcPort` | nat.go:69-89, 104-108（由 session.go 调用） |
| H-4 | 高 | IPv6 出站完整实现但被 `Writer` 校验拒绝；入站 IPv6 panic | session.go:330-352, 418-453；protocol/vpn.go:96-114 |
| H-5 | 高 | 非 TCP/UDP/ICMP 协议被静默黑洞，无计数、日志默认关闭 | session.go:439-442；filter.go:155-158 |
| M-1 | 中 | `SetFakeIP` 对 IPv6 配置直接 panic | session.go:236-243；main.go:118-125 |
| M-2 | 中 | `IpPkgToNetfilter` 显式 panic 路径 + 未校验头部长度 | session.go:471-517 |
| M-3 | 中 | 命名返回值 `verdict` 从未赋值，永远 `NF_DROP`，且调用方忽略返回值 | session.go:303, 468；callbacks.go:274 |
| M-4 | 中 | pcap 写路径自锁死（`writeToPcap` → `initPcapWriter` 重复加锁），整段为死代码 | session.go:46-52, 107-115 |
| M-5 | 中 | 注入失败/短写只打 warn，`NetfilterSendIpPkg` 错误被丢弃，无 SNMP 计数 | session.go:459-466, 509 |
| M-6 | 中 | 分片热路径里直接 `fmt.Printf` 到 stdout | session.go:490 |
| M-7 | 中 | `IpFamily`/`IpHeaderSize` 硬编码，且 C 侧根本不使用这两个参数 | session.go:506-507；netfilter.go:252-264 |
| M-8 | 中 | `Writer` 无同步；VPN 接收协程在驱动 Open 之前即启动 | session.go:245-247, 459；main.go:156-185 |
| M-9 | 中 | `FilterFnList` 全局可变、无锁 | session.go:142-183, 201；callbacks.go:270 |
| M-10 | 中 | 出站忽略 DF/路径 MTU，固定 1400 且多减 14 字节以太网头 | session.go:447-449；protocol/fragment.go:31-38 |
| M-11 | 中 | `SessionLog` 用 `log.Printf(fmt.Sprintf(...))`，新工具链 vet 会报非恒定格式串 | session.go:38-43 |
| M-12 | 中 | `DefaultNatTableIpv4/6` 为包级全局，多个 `ClientDriverNf` 实例共享 | session.go:188-215；nat.go:36-37 |
| L-1…L-8 | 低 | 死代码、死字段、静默溢出、注释失真等 | 见 §4 |

---

## 2. 高严重性发现

### H-1 出站分片处理错误（首片按完整报文重写，其余片丢弃）

**位置**：`netfilter/session.go:312-321`、`443-456`、`457-467`

**证据**：

```go
// session.go:318-321
if ipv4Hdr.FragmentOffset() != 0 {
    SessionLog("fragmented ipv4 packet")
    return                       // 非首片：直接 DROP（返回值是零值 NF_DROP）
}
...
// session.go:447
fragments = lp.FragmentPayload(ipPkg, utils.MTU)
```

该判断只排除了 `offset != 0` 的分片，**未检查 `More()` 标志**。因此：

1. `offset==0 && MF==1` 的首片会继续走 NAT 改写与 L4 校验和重算（`session.go:374-377`、`394-396`）。重算时伪头部长度取的是**当前片长**（`protocol/checksum.go:133`），而 UDP 头里的 Length 字段仍是原始完整报文长度；即使所有分片都能到达对端，重组后的校验和也必然不匹配。
2. 首片随后可能被 `FragmentPayload` 再次分片（`protocol/fragment.go:43-88`），把偏移/标志按“独立完整报文”重写，语义错误。
3. 其余分片全部丢弃，对端永远无法重组。

**分片确实会出现**：入站方向 `IpPkgToNetfilter` 专门实现了 IPv4 分片重组（`session.go:482-493`，调用 `lp.DefaultIPv4Reassembler`），说明作者已在真实链路观察到分片。

**影响**：本机发出的大于 MTU 且 DF=0 的 UDP/TCP 报文经 VPN 全部丢失；若首片先被转发，还会在对端产生带错误校验和的残余分片。

**建议**：
- 出站与入站对称：先按 `(src,dst,protocol,id)` 做重组，重组完成后再做 NAT 改写与（必要时）重新分片；或
- 明确策略：凡是 `FragmentOffset()!=0 || More()` 一律不重写 L4、不重新分片，按“透传/丢弃”二者之一处理，并加计数器与限流日志。
- 增加表驱动测试覆盖：首片、中片、末片、乱序、重复片。

### H-2 NAT 会话表无生命周期，65536 后新流全部失效

**位置**：`netfilter/session.go:367, 387, 406, 425`（`AddSession` 调用）；`netfilter/nat.go:29-34, 92-125`

**证据**：

- `FastNatTable` 是 `[65536]*IpSession` 的固定数组（`nat.go:29-34`），全文件没有任何删除/回收路径；`LastSeen` 只在 `nat.go:105/119/137` 被写入，从未被读取用于过期。
- `go_tcpClosed`（`callbacks.go:96-100`）只记录日志并调用 `nf_tcpClose`，不清理 NAT 项。
- `AddSession` 在找不到空槽时返回 `(nil,false)`，调用方随即丢弃数据包（`session.go:368-371`、`388-391`、`407-410`、`426-429`）。

**影响**：TCP 连接关闭后 fake 端口仍被永久占用。长时间运行的客户端在累计约 6.5 万个不同源端口/地址组合后，TCP、UDP、ICMP 依次进入“session full”，所有新连接被静默丢弃（`SessionLog` 默认关闭，见 `session.go:36-40`），只能重启进程恢复。

**建议**：
- 增加按 `LastSeen` 的 TTL 回收协程，并在 TCP 关闭事件中删除对应项；
- 增加“表占用率 / 丢弃数”指标（已有 `Ipv4Snmp` 风格的结构可复用）；
- 为此补充并发压测：持续建连/断连 10 万次，断言表不泄漏、吞吐不衰减。

### H-3 NAT 复用键只看源 IP，回包可能投递到错误 socket

**位置**：`netfilter/nat.go:69-89`、`104-108`，被 `session.go:367/387/406/425` 调用

**证据**：

```go
// nat.go:71-78
for port := startPort; port != 0; port++ {
    if table[port] == nil {
        return port, false
    } else {
        if table[port].RealSrcIP == realSrcIP {
            return port, true          // 只比较源 IP，realSrcPort 参数完全未使用
        }
    }
}
// nat.go:104-108：复用时不更新 RealSrcPort / DstIP / Options
```

`realSrcPort` 形参在 `findAvailableIdx` 中从未参与判断；`AddSession` 复用会话时也不校验/更新 `RealSrcPort`、`DstIP`、`Options`。

**可复现的失败序列**（同一主机 H，TCP）：
1. 会话 A：H:1000 → fake 1000（`table[1000].RealSrcPort=1000`）；
2. 1001 槽被其他主机的会话占用，H 的端口 1001 被分配 fake 1002（`table[1002].RealSrcPort=1001`）；
3. 新的 H:1002 连接扫描到 1002 槽，`RealSrcIP` 相同 → 复用，fakePort=1002，但该会话的 `RealSrcPort=1001`；
4. 对端回包 dest=fake:1002，`PerformDnatIpv4` 按 fake 端口查表（`nat.go:179-188`）改写回 H:1001 → 回包进入**错误的 socket**（或直接丢弃）。

端口压力越大（并发连接多、NAT 端口冲突频繁）越容易命中。

**建议**：复用条件改为完整四元组匹配 `(RealSrcIP, RealSrcPort, DstIP, DstPort, Protocol)`；不匹配就继续向后找空槽；同时把 `Options` 纳入一致性校验，避免注入时复用首条流的分区/接口标志。

### H-4 IPv6 出站是“完整实现但必然失败”，入站 IPv6 直接 panic

**位置**：`netfilter/session.go:330-352, 418-438, 445-453`；`protocol/vpn.go:96-114`；`app/vpnclient.go:80-102`

**证据**：

- 出站对 IPv6 做了源地址改写、L4 校验和重算、`FragmentPayloadIpv6` 分片，然后 `c.Writer.Write(frag)`（`session.go:459-460`）。
- 但 `c.Writer` 的实际类型是 `*app.VPNClient`（`main.go:156-168` + `connect.go:55`），其 `Write` 调用 `protocol.AppendLegacy` → `validateCompleteIPv4`：

```go
// protocol/vpn.go:102-103
if packet[0]>>4 != 4 {
    return invalidVPNPacket("inner IP version is %d, want 4", packet[0]>>4)
}
```

IPv6 报文（首字节高 4 位为 6）必然返回 `ErrInvalidVPNPacket`，`session.go:461-464` 仅打 `Warnf`，报文被丢弃。

- 入站方向 `IpPkgToNetfilter` 对 IPv6 直接 `panic("not implement ipv6 yet")`（`session.go:512-513`）。
- `FakeIpv6` 由 `IPv4ToIPv6` 生成 `0x0FC0::/…` 前缀（`session.go:233`），不在 ULA（`fc00::/7`）或任何标准伪造段内，语义可疑。

**影响**：IPv6 流量被静默丢弃且无计数；代码表面上“支持 v6”，会误导后续维护与测试；panic 路径见 M-2。

**建议**：短期在 `IpPkgFromNetfilter` 入口按策略显式丢弃 IPv6 并计数，删除或 `//go:build ignore` 掉 v6 分支；长期实现 v6 端到端（含 `validateCompleteIPv6`、注入头处理）后再启用。`FakeIpv6` 建议改为标准 ULA 段（如 `fd00::/64` 内映射）。

### H-5 非 TCP/UDP/ICMP 协议被静默黑洞

**位置**：`netfilter/session.go:439-442`；`netfilter/filter.go:155-158`；`netfilter/callbacks.go:270-275`

**证据**：

```go
// session.go:439-441
default:
    SessionLog("unknown protocol: %d", protocol)
    return
```

`PublicDstIp` 对非私网目标返回 `NF_STOLEN`（`filter.go:155-158`），随后 `go_ipSend` 把包交给 `IpPkgFromNetfilter`（`callbacks.go:273-275`）；协议不在 TCP/UDP/ICMP/ICMPv6 范围内（GRE、ESP、SCTP、大部分 UDP-based 私有协议等）时直接丢弃，而 `SessionLog` 默认关闭（`session.go:36-40`），`Ipv4Snmp` 也没有对应计数。

**影响**：启用“公网流量走 VPN”策略后，非上述四类协议全部不可用，用户表现为“部分游戏/语音/组播异常”，且无可观测证据。

**建议**：增加 `NextHeaderNotSupport` 类计数器 + fastlogger 限流日志；并明确策略——是 `NF_ACCEPT` 放行回协议栈，还是显式丢弃。

---

## 3. 中严重性发现

| 编号 | 位置 | 证据与风险 | 建议 |
| --- | --- | --- | --- |
| **M-1** | `session.go:236-243`；`main.go:118-125` | `SetFakeIP` 只判 `ParseIP != nil`，随后 `tcpip.AddrFrom4Slice(ip.To4())`；对 IPv6 配置 `To4()` 为 nil，gvisor `AddrFrom4Slice` 会 panic（`tcpip.go:172-175`），`[16]byte(IPv4ToIPv6(nil))` 同样 panic。main 用 `recover` 兜底退出，但属于配置错误触发崩溃 | 显式 `ip.To4() == nil` 时返回 error，`SetFakeIP` 改为返回 error 并在启动时处理 |
| **M-2** | `session.go:471-517` | ① `panic("not implement ipv6 yet")`、`panic("unknown ip version")` 位于数据面；VPN 接收协程（`connect.go:43`）没有 recover，panic 即进程退出。② `header.IPVersion` 对空包返回 `-1`，`header.IPv4(ipPkg).FragmentOffset()` 对短包/`TotalLength` 越界的包会切片 panic（gvisor `ipv4.go:324-325`、`385-386` 无长度保护）。当前 VPN 入口有 `validateCompleteIPv4` 兜底（`protocol/vpn.go:97-115`），但该方法是导出 API，契约不安全 | 数据面禁止 panic：返回 error + 计数；入口处校验 `len>=20`、`IHL>=20`、`IHL<=len`、`TotalLength<=len` |
| **M-3** | `session.go:303, 468`；`callbacks.go:274` | 命名返回值 `verdict` 在所有路径都是裸 `return`，零值即 `NF_DROP`（`netfilter.go:442-449`）；而调用方根本不使用返回值。API 语义与实际行为不一致，后续若有人依赖返回值会把已成功转发的包也判为 DROP | 删除返回值改为 `func (...)`，或显式返回语义化 verdict 并让调用方使用 |
| **M-4** | `session.go:46-52, 107-115` | `writeToPcap` 先 `pcapMutex.Lock()`，再在 `pcapWriter==nil` 时调用 `initPcapWriter()`，后者再次 `pcapMutex.Lock()` → 非重入互斥锁**自锁死**。当前 `writeToPcap`/`initPcapWriter`/`closePcapWriter`/`addEthernetHeader` 全仓无调用者，属死代码，但首次启用即挂 | 删除整段死代码（`utils.PcapDump` 已提供等价能力），或改为 `initPcapWriterLocked` 内部分层加锁 |
| **M-5** | `session.go:459-466, 509` | 写 VPN 失败/短写只 `Warnf`；`NetfilterSendIpPkg(frag, session.Options)` 的 error 被完全丢弃，且无任何计数器。注入失败在 SNMP 中不可见 | 记录限流日志并累加 `SenderNil`/新计数；返回值透传给 `IpPkgToNetfilter` 的调用方 |
| **M-6** | `session.go:490` | `fmt.Printf("waiting more fragments for ipv4\n")` 在每个分片上同步写 stdout，阻塞数据面且无上限 | 删除，或改用限流 fastlogger |
| **M-7** | `session.go:506-507`；`netfilter.go:252-264` | Go 侧硬编码 `IpFamily=2`、`IpHeaderSize=20`，忽略真实 IHL 与 IPv4 options；而 C 侧 `nf_sendpkg` 又无条件覆盖为 `AF_INET`/20，两个参数实际是“假接口” | 删除无效参数或在 C 侧真正使用；IPv4 带 options 的报文需按真实 IHL 注入 |
| **M-8** | `session.go:245-247, 459`；`connect.go:55-56`；`main.go:156-185` | `SetWriter` 在主协程写、driver 回调协程读，无同步；当前 `main` 的调用顺序（ConnectWithVPN → SetFakeIP → Open）避免了实际竞争，但没有代码约束。另：`VPNToNetfilter` 协程在 `netfilter.Open()` **之前**启动，期间收到的回包会调用未初始化的 `nf_ipPostSend`（`NetfilterSendIpPkg`），错误被忽略（见 M-5） | 用 `atomic.Value`/互斥保护 Writer；或在 Open 成功后再启动 VPN 接收循环，早到的包显式丢弃并计数 |
| **M-9** | `session.go:142-183, 201`；`callbacks.go:270` | `DefaultFilterFnListCache` 是包级可变全局，`AddFilterFnList` 无锁写 `Idx`，driver 回调无锁读；main 当前在 Open 前装配，一旦运行期增删过滤器即数据竞争 | 初始化后冻结（`sync.Once`/构建期不可变），或加 `RWMutex`/原子快照 |
| **M-10** | `session.go:447-449`；`protocol/fragment.go:31-38` | 固定 `utils.MTU=1400`，无条件重新分片，忽略 DF 与路径 MTU；`baseMaxData` 额外减去 14 字节以太网头，而此处并无以太网帧，分片偏小 | 分离“隧道载荷 MTU”与“链路 MTU”；尊重 DF；为 PMTUD 留出策略 |
| **M-11** | `session.go:38-43` | `log.Printf(fmt.Sprintf(...))` 使用非恒定格式串，Go 1.24+ 的 `go vet` printf 检查会报 `non-constant format string`；同时使用标准 `log`（stderr）而非 `fastlogger`，与项目日志体系不一致 | 改为 `fastlogger.Tracef("[session] "+fmtStr, args...)` 或 `fmt.Sprintf` 后 `log.Print` |
| **M-12** | `session.go:188-215`；`nat.go:36-37` | `ClientDriverNf.Sessions`（LRU 2048）从未被使用；真正的会话表是包级 `DefaultNatTableIpv4/6`。任何第二个 `ClientDriverNf` 实例都与单例共享 NAT 状态，测试或多实例部署会互相串扰 | 删除 `Sessions`；若要支持多实例，把 NAT 表移入 `ClientDriverNf` |

---

## 4. 低严重性 / 代码质量

| 编号 | 位置 | 说明 |
| --- | --- | --- |
| L-1 | `session.go:185-199, 217-234` | 死字段/死类型：`OutPut`、`OutPut` 字段、`VpnRemote`、`Tun5`（仅作未被使用的 LRU key）、`ProxyList`（硬编码公网 IP）、`ProxyAll`（`main.go:126` 赋值后全仓无读取） |
| L-2 | `session.go:519-557` | `USE_FIXED_FAKE_IP`、`FakeIp` 全局、`generateFakeIP` 全仓无调用者；`rand.Read` 错误被忽略（若将来启用）。`netfilter.FakeIp` 与 `SetFakeIP` 是两套入口，语义重叠 |
| L-3 | `session.go:560-595` | `parseTuple5` 无调用者；且 `tcp, _ := tcpLayer.(*layers.TCP)` 后未判空，类型断言失败会 nil panic |
| L-4 | `session.go:249-301` | `checksumUpdate2ByteAlignedAddress/Uint16` 与 gvisor 内部实现重复，仅被注释代码引用；`old.BitLen()%16` 对 32/128 位地址恒为 0，检查无效 |
| L-5 | `session.go:156-159, 178-180` | `AddFilterFnList` 溢出时静默忽略并返回 `NF_ACCEPT`；`Filter` 对未知 verdict（`NF_QUEUE/REPEAT/STOP`）一律当 `NF_DROP` |
| L-6 | `session.go:372, 392, 411, 438` | 格式串自带 `\n`，`log.Printf` 再加换行，日志出现空行 |
| L-7 | `session.go:477-478, 502` | `oldPort` 恒为 0，常量参数贯穿 `IpPkgToNetfilter → ProcessIpv4`，属未完成的重组/端口提取遗留 |
| L-8 | `session.go:204-209` | `lru.New` 对常量 2048 不可能失败，`panic` 分支为噪声；且该初始化与 H-2/M-12 的“会话表”概念冲突 |

---

## 5. 关联文件观察（超出 session.go 范围，供交叉验证）

这些不是本轮审查对象，但与 session.go 的行为直接耦合，建议同步处理：

1. **`netfilter/callbacks.go:270-286`（严重）**：`NF_ACCEPT` 分支内已调用一次 `C.nf_ipPostSend`（第 272 行），switch 结束后第 286 行又调用一次，**NF_ACCEPT 路径会重复投递同一数据包**。虽然不直接属于 session.go，但决定了 `IpPkgFromNetfilter` 是否被调用，需优先确认。
2. `netfilter/netfilter.go:560-568`：`NetfilterSendIpPkg` 用 `&packetData[0]` 取地址，空切片会 panic；建议入口校验 `len>0`。
3. `netfilter/nat.go:128-139`：`GetByFakePort` 在 `RLock` 下写 `session.LastSeen`，与 `AddSession` 的写构成数据竞争。
4. `netfilter/nat.go:55-66`：`getTableByProtocol` 对不支持协议返回 nil，随后 `findAvailableIdx(nil,...)`/`table[fakePort]` 会 nil 解引用；当前调用方只传四类协议，属潜在问题。
5. `protocol/fragment.go:391-424`：重组时硬编码 20 字节头（`header.IPv4MinimumSize`），忽略原始 IHL/options，带选项的分片重组结果头长与内容错位；`ReceivedLen` 只加不减、从未参与完整性判断。
6. `utils/utils.go:22-42`：`PcapDump` 的惰性 `pcapWriter` 无锁，从 driver 线程与 VPN 线程并发调用（session.go:306/458/472/508）时报文/文件可能损坏。
7. `main.go:118-125` 与 `session.go:236-243` 的 FakeIp 校验口径不一致（见 M-1）。

---

## 6. 建议的修复优先级

**P0（进入下一轮前必须完成）**
1. H-1 出站分片策略（重组后改写，或整体不处理并计数）。
2. H-2 NAT 生命周期与 H-3 四元组匹配。
3. H-4 IPv6：显式禁用并计数，删除半实现分支。
4. 确认并修复 `callbacks.go` 的 `NF_ACCEPT` 双投递。

**P1**
5. M-1/M-2：所有数据面 panic 改为 error + 计数；入口长度校验。
6. M-3/M-5/M-7：清理误导性 API 与静默失败。
7. M-8/M-9/M-12：Writer/Filter 同步与实例隔离。

**P2**
8. M-4/M-6/M-10/M-11 与 L-1…L-8：删除死代码、修正 MTU 语义、统一日志。

---

## 7. 本轮未验证项

- 未构建、未运行测试、未执行 `go vet`（只读约束 + Bash 调用被拒绝），M-11 的 vet 结论基于 Go 1.24 工具链行为推断，需实跑确认。
- 出站方向驱动是否确实会投递分片（H-1 的触发概率）需在目标 Windows 环境抓包确认；入站重组代码是间接证据。
- gvisor 版本以 `go.mod` 锁定的 `v0.0.0-20230927004350-cbd86285d259` 为准做源码核对；工作区模块缓存中另有 2026 版本，实际编译使用哪一个需以 `go list -m` 确认。

---

## 8. 最终判定

**`netfilter/session.go`（v1 工作区版本）：不予通过。** 构建层面的“能编译”不代表数据面正确：分片、NAT 生命周期、NAT 复用键三处缺陷会导致真实流量丢失或串流，IPv6 与异常协议路径则缺乏可观测性；同时该文件尚未提交，审查结论无法绑定到任何可复核的提交。建议按 §6 的 P0 修复并提交后，以具体 SHA 重新审查。

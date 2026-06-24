# 发-收-转链路时标流向分析

> ⚠ 状态（2026-06-10）：二进制 dspa1.json 派发模块已从运行期移除（protocolConvert 现为 TRACK 文本占位，m_dspConfig/reloadDspConfig 已删，dspa1.json 文件已删，协议引擎与设计器迁出另一应用）。以下分析按当时的二进制 dspa 实现撰写，仅作历史参考。

> 仅做现状记录与问题澄清，不涉及代码修改。
> 同伴文档：`timestamp_design_review.md`（设计稿）

## 0. 阅读约定

- **simMs**：基于文件源时标推算出的回放采样时间，由 `PlaybackController::simMsAtWall(wallNow)` 给出。
- **wallEpochMs**：本机系统时间 Unix epoch 毫秒（即 `QDateTime::currentMSecsSinceEpoch()`），目标方案中要写入协议字段 `globalMs`。
- **私有帧**：主从之间 UDP 通讯的文本帧 `FRAME_BEGIN ms=... / FRAME_TRACK ... / FRAME_END`。
- **dspa 帧**：每端按本地 `dspa1.json` 规约，由 `RoomManager::protocolConvert()` 组装的二进制下游帧，单路。

## 1. 当前代码定位（事实清单）

### 1.1 时标在主机侧产生

`mainwindow.cpp:3419 sendOneFrameNow()`

```
wallNow = m_playback->wallElapsed()           // 墙钟相对 t0
simNow  = m_playback->simMsAtWall(wallNow)    // 文件时标推算值
m_roomManager->sendFrame(m_tracks, simNow)    // 把 simNow 当作 globalMs 注入下游
```

> 此时**没有**对 `QDateTime::currentMSecsSinceEpoch()` 做任何采样。
> `simNow` 一个值同时承担"采样位置"和"协议时标"两重语义——文档第 3 节描述的漏洞由此处实化。

### 1.2 主机侧 `RoomManager::sendFrame()`（roommanager.cpp:533）

只做一件事：组装私有文本帧。

```
FRAME_BEGIN ms=<simNow>
FRAME_TRACK id=... lat=... lon=... fit=... color=... lw=... logname=... icon=...
...
FRAME_END
```

并按角色分发该字节流：

- **Host 模式 / 每个真实从机**：`sendToClient(c, frame)` → `m_hostSocket->writeDatagram(frame, slaveAddr, slaveRecvPort)`
- **Host 模式 / 主从一体**：`m_hostSocket->writeDatagram(frame, 127.0.0.1, SELF_SLAVE_PORT=8888)` 走环回到本进程的 `m_selfSlaveSocket`
- **Idle 模式（无房间）**：直接调用 `processFrameLocally(entries, simNow)`

主机在 `sendFrame()` 路径上**不会**调用 `protocolConvert()`。所有 dspa 帧的产生都委托给收端。

### 1.3 私有帧 `ms` 字段在网络上的形态

`roomprotocol.h:211 makeFrameBegin(qint64 ms)` →
`"FRAME_BEGIN ms=<simNow>\n"`

从机收到后只有这个十进制字符串可用，**没有携带主机的本机墙钟**。

### 1.4 收端解析与 `processIncomingFrame()`（roommanager.cpp:823）

每一端（任意从机、主机自从机）拿到 FRAME_BEGIN 报文，统一进入：

```
frameMs = parseFrameBeginMs(...)   // 即主机塞进来的 simNow
解析 entries
processFrameLocally(entries, frameMs)   // 在收端按本地 dspa 配置发 dspa 帧
emit frameReceived(entries, frameMs)    // 通知 MainWindow 更新 UI
```

### 1.5 收端 `processFrameLocally()`（roommanager.cpp:248）

每个收端独立执行，使用**收端自己**的 `m_dispatchChannel` 与 `m_dspConfig`：

```
对每个 entry:
    若 m_dispatchChannel 已配置:
        payload = protocolConvert(entry, frameMs)   // ← 写入字段 globalMs
        m_dispatchSender->sendRaw(payload)          // 出口 UDP
此外（若 m_downstreamEnabled）:
    向旧下游发文本: "TICK,<frameMs>\n" + "TRACK,..." 多行
```

`protocolConvert()` 中 `globalMs` 字段（roommanager.cpp:141-142）：

```cpp
if (n == "globalMs") {
    numValues[n] = quantize(static_cast<double>(globalMs), f.scale, f.offset);
}
```

`globalMs` 形参 = 解析自私有帧的 `frameMs` = 主机塞进来的 `simNow`。

### 1.6 主机自从机环回路径

`startSelfSlave()` 在 `127.0.0.1:8888` 绑 socket。`onSelfSlaveSocketReady()`（roommanager.cpp:617）和真实从机的 `onClientSocketReady()`（roommanager.cpp:771）走**完全相同**的 `processIncomingFrame` 路径。退化路径（self-slave 端口被占用）才直接调用 `processFrameLocally`，用于不丢失主机本地 dspa 输出能力。

### 1.7 从机 UI 端 `MainWindow::onRemoteFrameReceived()`（mainwindow.cpp:1808）

收端 UI 逻辑里 `globalMs` 用作：

- `setTrackDisplayTime(globalMs)`（mainwindow.cpp:1817）：从机回放地图游标定位
- `appendTrackPoint(tid, lat, lon, ft, globalMs)`（mainwindow.cpp:1868）：新增轨迹点的源时标
- `m_globalCurrentMs = globalMs`（mainwindow.cpp:1876）：从机全局当前时刻状态

⚠️ **这三处用的恰好是 simMs 语义**——它们需要的是"主机播放到了文件第几毫秒"，**不是**主机的本机墙钟。把这个值替换成 wallEpochMs 会把从机 UI 时标语义彻底破坏。

## 2. 实例：1 主机（参与者）+ 2 从机场景

设主机播放 `simNow = 250`（文件时标第 250 ms），主机本机墙钟 `wallEpochMs = 1746098400123`，墙钟字符串 `2026-05-01 12:30:00.123`。

### 2.1 主机一次 `sendOneFrameNow()` 的物理动作

| 编号 | 出口动作 | 目的地 | 报文内容（关键时标） |
|-----|----------|--------|---------------------|
| ① | `m_hostSocket->writeDatagram` | SlaveA `recvPort=8900` | 私有帧 `FRAME_BEGIN ms=250` |
| ② | `m_hostSocket->writeDatagram` | SlaveB `recvPort=8900` | 私有帧 `FRAME_BEGIN ms=250` |
| ③ | `m_hostSocket->writeDatagram` | `127.0.0.1:8888`（自从机） | 私有帧 `FRAME_BEGIN ms=250` |

主机此时**没有**任何 dspa 帧出口；主机进程内 `protocolConvert()` 一次都没跑。

### 2.2 三端 dspa 帧的产生（每端独立解析、独立组装）

每端把收到的私有帧送进 `processFrameLocally(entries, 250)`，按各自的 dspa1.json 配置发包。

| 收端 | dspa 出口（单路） | dspa 帧字段 `globalMs` 写入值 |
|------|------------------|--------------------------------|
| 主机自从机 | 按主机进程加载的 `m_dspConfig` 决定 | `250`（即主机的 simNow） |
| SlaveA | 按 SlaveA 进程加载的 `m_dspConfig` 决定 | `250`（同上，从私有帧解析得到） |
| SlaveB | 按 SlaveB 进程加载的 `m_dspConfig` 决定 | `250`（同上） |

**结论 1：dspa 帧 `globalMs` 字段值在三端是一致的，但都不是任何机器的本机墙钟，而是文件时标 simNow。**

**结论 2：dspa 帧的"再分发"是在每个收端独立进行的；主机/SlaveA/SlaveB 的 dspa1.json 可以各自不同，因此实际打到下游的物理报文集合和编码可能完全不同。**

### 2.3 此场景下的 UDP 报文总数（粗算）

设主机本机有 K_H 路启用、SlaveA 有 K_A 路、SlaveB 有 K_B 路、N 个对象。一帧的 UDP 报文数：

- 主机出口 3 个私有帧（① ② ③）
- 主机自从机环回到 `processFrameLocally` 后产生 `N × K_H` 个 dspa 包
- SlaveA 产生 `N × K_A` 个 dspa 包
- SlaveB 产生 `N × K_B` 个 dspa 包
- 若任一端 `m_downstreamEnabled`，再加一份 TRACK 文本帧

私有帧只 1 个（多行装一个 datagram），dspa 帧按对象 × 通道独立成包。

## 3. 当前设计的具体漏洞清单

### Bug-1：globalMs 一字两用

`sendOneFrameNow()` 把 simMs 直接当 globalMs 灌入私有帧；该字段又被 `protocolConvert()` 当协议时标用。结果是协议字段 `globalMs` 长期是**文件时标值，而非任何机器的本机墙钟**。

### Bug-2：协议时标在收端产生，时间同源约束无法成立

文档 §2.5 要求 `globalMs` 与 `UTC` 由**同一次**系统时间采样产生。但当前结构下：

- 主机端：从未采样墙钟。
- 收端（含主机自从机）：要做 dspa 转换时，确实可以采一次本机墙钟。但每个收端的本机时钟可能不同步（NTP 漂移、不同主机），三端各自采样会导致同一帧不同收端 `globalMs` / `UTC` **互不相等且差值可达毫秒级以上**。
- 私有帧只承载一个 `ms` 字段，无法把"主机的本机墙钟"和"主机 simMs"同时传到下游。

⇒ 文档建议的"每帧采样一次"必须明确"在哪一端采"。

### Bug-3：私有帧 `ms` 字段语义被多个语义共用

收端用同一个 `frameMs` 做：

- A) `setTrackDisplayTime(frameMs)`（UI 游标对齐，需要 simMs）
- B) `appendTrackPoint(..., frameMs)`（轨迹点入库源时标，需要 simMs）
- C) `processFrameLocally(..., frameMs)` → `protocolConvert` 写入字段 `globalMs`（协议时标，需要 wallEpochMs）

A、B 与 C 想要的是不同的东西。粗暴改一处会打坏另一处。

### Bug-4：`UTC` 字段链路缺失

- `protocolConvert` 当前识别字段名集合中没有 `UTC`；规约里即便配了 `UTC` 字段也只会保持 0。
- 私有帧 `FRAME_BEGIN` 没有对应的 UTC 字符串字段。
- `roomprotocol.h::FrameTrackEntry` 也没有 utc 文本承载位。

### Bug-5：旧文本下游 `"TICK,<ms>"` 也写的是 simMs

`processFrameLocally` 中 `lines << "TICK," + frameMs`（roommanager.cpp:254）。文档 §4.2 暂缓不改旧文本格式，记录在案。

## 4. 待确认的设计决策（与 timestamp_design_review.md 协同）

> **D1（采样位置）** ✅ 已确认 = 选项 a：**协议时标在主机采样一次**，把 `wallEpochMs + utcText` 随私有帧下发，所有收端（含真实从机、主机自从机）复用主机时标。三端 dspa 字段 `globalMs` / `UTC` 内容一致。
> 跨机时钟不一致问题由此规避（凡是 dspa 输出，用的都是主机时刻）。

> **D-Slave-Role（从机产品角色）** ✅ 已确认 = **P2 同步显示终端**：
> - 从机必须能在屏幕上看到飞机完整运动历史与拖尾线
> - 从机本地历史数据**仅内存保存，不落盘**
> - 从机轨迹回放显示需与主机一致（位置、Gap 段、loop 循环、时间轴游标）
> - **隐含约束**：`onRemoteFrameReceived` 的占位/镜像/`appendTrackPoint`/`setTrackDisplayTime` 等逻辑必须保留，时标重构不得破坏。

> **D3（私有帧 `ms` 字段语义）** ✅ 由 D-Slave-Role=P2 推导锁定 = 主建议：
> - `ms` 字段**保留为 simMs**（继续供 setTrackDisplayTime / appendTrackPoint / m_globalCurrentMs）
> - 私有帧**新增字段** `epoch=<wallEpochMs> utc=<text>`（供 protocolConvert 写入 dspa `globalMs` / `UTC`）
> - 三个语义彻底解耦，原有功能不破坏

> **D5（播放引擎是否仍按文件时间轴推进）** ✅ 已确认 = A 案：
> - 主机发送瞬间用 `simMsAtWall(wallNow)` 推算 simMs，`interpolateAt(simMs)` 取对象位置
> - 只修复"协议字段 globalMs 错填 simMs"这一处，**不动播放引擎**
> - **附注**：`playbackSpeed`（倍速）将来可能删除或仅在编辑界面使用，本次时标重构不依赖倍速行为，但也不主动删除——按现状保留，等后续产品决策

> **D2（私有帧 schema 扩展）** ✅ 已确认 = 解法 δ：
> - 私有帧 `FRAME_BEGIN` 只新增**一个**整数字段：`globalMs=<wallEpochMs>`
> - 不下发 utc 文本字段（避免空格 / 解析问题）
> - 最终私有帧形式：`FRAME_BEGIN ms=<simMs> globalMs=<wallEpochMs>\n`
> - UTC 字符串**由各收端在 dspa 出口现场格式化**：`QDateTime::fromMSecsSinceEpoch(globalMs).toString("yyyy-MM-dd HH:mm:ss.zzz")`
> - 严格满足 timestamp_design_review.md §2.5"时间同源"——utc 由 globalMs 唯一推导
> - 字段名 `globalMs` 和 dspa 协议字段 `globalMs` 重名，源到出口一致，无翻译层

> **D2（私有帧扩展）**：是否在 `FRAME_BEGIN` 加 `epoch=<wallEpochMs> utc=<text>` 两个字段？这是选项 a 的前置改动。

> **D3（私有帧原 `ms` 字段语义）**：保留为 simMs（用于 UI 游标 / 轨迹入库），还是改为 wallEpochMs？建议保留 simMs，新增独立字段承载墙钟。

> **D4（`UTC` 在 dspa 规约中的字段类型）**：固定按字符串走 rawBytes 旁路，长度由 `lengthBits` 决定，右补 `0x00`？格式是否允许在 dspa1.json 中自定义？

> **D5（"按 json 播放密度执行"的字面理解）**：用户口述 vs 文档 §2.1。是 A 案（保留文件时标做时间轴/插值，仅替换协议时标字段）还是 B 案（彻底丢弃文件时标，按 `preprocess.density_ms` 等间隔播）？文档明确选 A。

## 5. 待修改项：自从机端口绑定失败 + 右下角系统状态 LED 升级

> 状态：未实现，待编码。本节是产品/UX 决策记录，与时标重构主线并行。

### 5.1 术语对照（避免歧义）

| 内部代码 | 持久化字符串 | 用户可见 UI |
|----------|-------------|-------------|
| `RoomManager::HostRole::Observer` | `"observer"` | **数据主机** |
| `RoomManager::HostRole::Participant` | `"participant"` | **主从一体** |

枚举名 `Observer/Participant` 是早期设计遗留，目前**有意保留**，UI 已统一改为"数据主机/主从一体"。本节后续描述用 UI 术语。

### 5.2 当前实现（需要替换）

`roommanager.cpp::startSelfSlave()`：
- 8888 端口绑定失败时，仅 `qWarning` + `emit statusMessage("警告：自从机端口 8888 被占用…主从一体本机转发不可用")` + 把 `m_selfSlaveSocket` 置 nullptr。
- "主从一体"启动**仍被视为成功**（`HostRole::Participant` 设置已生效），后续 `sendFrame` 走"退化路径"——不发回环 UDP，直接同进程调用 `processFrameLocally`。
- 后果：dspa 下游分发不丢，但自镜像 UI（`emit frameReceived` → `onRemoteFrameReceived`）丢失，用户无明显感知。

### 5.3 行为决策（已锁）

**B1（启动语义）**：8888 绑定失败时，**拒绝进入"主从一体"，自动留在"数据主机"模式**。监听端口照常开，主机功能照常工作，仅"参与播放"这一面被关闭。
- 删除 `sendFrame` 中的退化分支（`if (m_selfSlaveSocket) ... else processFrameLocally(...)`）；不再存在"自从机失败但仍是 Participant"的中间状态。
- 即在写入 `m_hostRole = HostRole::Participant` 之前先尝试 `startSelfSlave()`；失败则把请求降级为 `HostRole::Observer` 并触发 B2 警告事件。

**B2（系统状态 LED——模型 ω：单灯优先级显示）** ✅
- 右下角**只有一个 LED 槽位**（统一指示器），颜色由当前最高严重度的活跃事件决定：

  | 状态 | LED | 含义 |
  |------|-----|------|
  | 灰/灭 | ⚫ | 系统正常 |
  | 黄 `#FFCC00`（待定 hex，按规则属"普通"色阶） | 🟡 | 系统级普通提示（如"配置变更，需重启生效"） |
  | 红 `#FF3B30` | 🔴 | 异常/错误（如"自从机端口被占用，主从一体不可用"） |

  红 > 黄，多事件并存时显示最高严重度对应的颜色与文字。
- 文字标签：跟随当前最高严重度事件的描述文本。

**B3（点击语义）** ✅
- LED 可点击。点击 = "一次性确认所有当前活跃的可确认事件"。
- "可确认 (ackable)" 标志位由事件源决定：
  - "需重启"事件：**不可确认**（只能通过把参数恢复到初值自动熄灭，参考 DESIGN §11 既有逻辑）
  - "自从机失败"事件：**可确认**（点击后从活跃集中移除）
- 点击后若仍有不可确认事件未消（如"需重启"还亮），LED 仍按剩余事件最高严重度显示；只有当前活跃集为空时才完全熄灭。
- **不弹模态对话框**；信息只在 LED 文字标签里展示。状态栏 toast 行为保留作即时反馈。
- 一期不实现"事件列表弹窗"；事件队列数据结构落地，但 UI 不暴露。

**B4（颜色一致性修复）** ✅ — 一并处理
- `mainwindow.cpp:242` 现有 `m_restartHintLed->setColorOn(QColor("#FF3B30"))`（红）改为黄 `#FFCC00`（hex 最终值待 UI 评审，此处先记为"黄色色阶"）。
- 文字标签字色 `#ffcc44` 是否同步调整待评审。
- 这是把现存代码与 B2 颜色分级规则对齐的必要修复，不修则模型 ω 退化为"单纯显示最近事件"，丧失分级意义。

### 5.4 实现骨架草案（仅作占位，等正式动手时再展开）

```text
SystemStatusLed (浮层组件，挂在 ui->mapContainer 右下角)
  - activeEvents: ordered list<EventEntry>
  - struct EventEntry {
        QString id;            // 事件唯一标识，如 "preprocess.restart_needed", "self_slave.bind_failed"
        Severity sev;          // Notice (黄) / Error (红)
        QString  text;         // 显示文本
        bool     ackable;      // 是否允许点击确认消除
    }
  - addOrUpdateEvent(EventEntry)
  - removeEvent(QString id)
  - onClicked() -> 移除所有 ackable=true 的事件，刷新显示
  - 显示规则：取 sev 最高的事件，颜色与 text 用其值；空集 → 隐藏
```

事件源接入：
- 现有"需重启" 逻辑（`setRestartHintVisible(bool, QString)`）改写为 `addOrUpdateEvent` / `removeEvent` 调用，事件 id = `"preprocess.restart_needed"`，sev = Notice，ackable = false。
- 新增"自从机绑定失败" 事件，事件 id = `"self_slave.bind_failed"`，sev = Error，ackable = true。在 `RoomManager::startHost(Participant)` 路径里、`startSelfSlave` 失败时通过新信号 `selfSlaveBindFailed(QString reason)` 向 `MainWindow` 报告。

### 5.5 暂仅记录、本期不实现

- **Q-LED-2（多事件源叠加 UI 暴露）**：当前事件队列已支持多事件，但 UI 不暴露列表，点击只能批量确认。后续若需要"展开列表 / 逐项确认 / 历史日志"再扩展。
- **Q-LED-3（持久化）**：LED 状态进程内有效，**不持久化到 `trackPath.json`**，重启后由各事件源重新检测决定亮灭。

### 5.6 与时标重构主线的耦合

- B1 删除退化分支后，`sendFrame` 分发逻辑只剩两条：
  - Host 模式：发真从机 + （Participant 时）回环自从机
  - Idle 模式：直接 `processFrameLocally`
- 三端 dspa 处理对称，**再无"自从机失败但仍跑 dspa"的特殊路径**——这对时标重构是利好，不必在退化路径里临时保留 `wallEpochMs` 逻辑。
- 建议落地顺序：**B1+B4 颜色修复优先**（清理路径分支与颜色规则），随后做时标重构主线（D2/D3/D4），最后接入 B2/B3 的"自从机失败"事件源。

## 6. 发-收-转模块职责重设计（新增）

> 本节由用户 §5 后追加的整体重设计驱动。整体管道是 **读 → 发 → 收 → 转**：读模块和转模块**不变**（转模块只是按收模块决策被有选择地调用），改动集中在**发**与**收**。

### 6.1 模块职责重新划分

| 模块 | 职责 | 是否变化 |
|------|------|----------|
| **读** | 文件加载、解析、预处理（density 插值） | 不变 |
| **发** | 数据传递 + 轨迹剪裁（时间段裁剪 / 编辑态单条筛选）+ 轨迹播放推进 + 主机墙钟采样 + **打"转发标志位"** | 职责扩大 |
| **收** | (1) **镜像显示**——所有帧都用于 UI 镜像，追求和主机播放一致的丝滑回放<br>(2) **按"转发标志位"筛选**——只把 `forward=1` 的帧交给转模块 | 职责重写 |
| **转** | dspa 协议编码 | 不变（被收模块按标志位有选择地触发） |

### 6.2 核心机制：发送频率 ≠ 转发频率

| 节拍 | 含义 | 配置参数 | 默认 / 推荐 |
|------|------|---------|------------|
| **发送频率** | 主机出 UDP 私有帧的节拍，决定从机镜像回放的画面流畅度 | `preprocess.refresh_interval_ms`（语义重定义为"发送频率"） | 50ms 或 100ms |
| **转发频率** | dspa 出口节拍，决定再下游设备的输入速率 | `preprocess.forward_interval_ms`（**新增**） | 视下游设备而定，常见 1000ms |

> **决策（Q1=a）**：复用 `refresh_interval_ms` 作为发送频率，新增 `forward_interval_ms` 作为转发频率。

> **决策（Q4 / 标志位计算）**：按发送序号计数为主：`if (frameSeq % ratio == 0) forward=1`，`ratio = forward_interval_ms / send_interval_ms`。
> **要求**：用墙钟做漂移修正——主机维护"上次 forward=1 的墙钟时刻 + 累计期望间隔"，发现偏差时调整下一帧的标志位时机。
> **UI 要求**：当前累计漂移值显示在**地图界面右上角**（与已有左上角时钟漂移标签 `m_driftLabel` 区别开，新增右上角浮层）。

> **决策（Q2 / Q3 标志位形态）**：标志位**整帧级**，不分对象。私有帧扩展为：
> ```
> FRAME_BEGIN ms=<simMs> globalMs=<wallEpochMs> forward=<0|1>\n
> ```
> 字段名 `forward`，0/1 整数。

> **决策（Q5 / UDP 乱序）**：不关心 UDP 乱序。收端只查本帧 `forward` 字段决定是否转发，跨帧不依赖。

> **决策（Q6 / 倍速）**：倍速将来仅在**编辑界面**使用。本次重构**不为倍速做特殊处理**，转发节拍按墙钟算。倍速对实时回放/转发链路无影响。

> **决策（Q8 / 主从一体）**：自从机走同一机制（也按 `forward=1` 才转）。**唯一区别**：本机播放轨迹回放**不需要遮蔽效果**（数据完整可见，无需做镜像层的"已收 vs 未收"差分），正常显示即可。

> **决策（Q9 / 轨迹剪裁含义）**：发模块的"轨迹剪裁"指**时间段裁剪**（loopRecord / `setTrackClipRange` 已实现的播放时间段过滤）。
> **附注**：`Track.visible` 的主要用途是**编辑界面单条轨迹专注剪裁的 UI 控制**（避免十几条线同时显示导致编辑混乱、占资源）——一次只显示一条做剪裁，与播放/转发链路本身无关。现有 `sendFrame` 中 `if (!t.visible) continue` 是顺带利用了同一字段。

### 6.3 收模块行为细化

> **决策（Q10 / UI 镜像方式）**：一期采用**简单方案**——从机收到一帧就刷新一次屏幕，UI 刷新率 = 发送频率（50/100ms）。
> **后期可选优化**（记录待办，本次不实现）：从机本地以 ~16ms（60fps）刷新，两帧之间靠 `interpolateAt(currentMs)` 在本地 buffer 上插值，达到比发送频率更细腻的视觉过渡。触发时机：观察到 50/100ms 仍卡顿时再启用。

收端伪代码：

```text
processIncomingFrame(lines):
  解析得到 simMs, wallEpochMs, forward, entries
  emit frameReceived(entries, simMs)              # UI 镜像不挑帧，仍用 simMs
  if (forward == 1):
      processFrameLocally(entries, wallEpochMs)   # 转发触发，转模块拿 wallEpochMs
```

### 6.4 旧文本 `TICK,<ms>` 处置

> **决策（Q7 / TICK 文本）** ✅ 已确认 = **删除**（用户确认无部署在用）。
> 已清除：
> - `roommanager.cpp` 中 `m_downstreamSender` 实例化、`setDownstreamUdp` 函数、`processFrameLocally` 内 `if (m_downstreamEnabled)` 整段
> - `roommanager.h` 中 `setDownstreamUdp` 声明、`m_downstreamIp/Port/Enabled` 与 `m_downstreamSender` 成员
> - `appconfig.cpp` / `appconfig.h` 中 `m_roomDownstreamIp/Port/Enabled` 成员、getter/setter、JSON 解析与保存
> - `mainwindow.cpp` 中 `setDownstreamUdp` 调用
> - `config.json` 中 `room.downstream_ip/port/enabled` 三个键
> - `SPEC.md` / `DESIGN.md` 中相应表格和示例 JSON 段落
>
> `UdpDataSender` 类保留（仍被 `m_dispatchSender` 使用）。
> 现有用户配置中的 `downstream_*` 字段会被静默忽略，下次保存自动消失，不做主动迁移。

### 6.5 与已锁定决策的兼容性检查

- **D1**（主机采样墙钟）：兼容——主机在每帧 `sendOneFrameNow` 入口处采样一次 `wallEpochMs`，所有出口报文（私有帧 + 后续转模块出的 dspa 帧）共用此值。
- **D2**（私有帧加 `globalMs` 字段）：兼容——再加一个 `forward` 字段。
- **D3**（私有帧 `ms` 字段保留 simMs）：兼容——`ms` 字段语义不变。
- **D4**（UTC 在 dspa 出口现场格式化）：兼容——只在 forward=1 触发的转模块路径上格式化。
- **D5**（A 案：播放引擎仍按文件时标）：兼容——发模块每次发都按文件时标插值。
- **§5 LED + 自从机绑定失败**：独立路径，不冲突。

### 6.6 实现分阶段建议（不动代码，仅规划）

阶段 A（最小可用闭环）：
1. 扩展配置项：新增 `preprocess.forward_interval_ms`，校验是 `refresh_interval_ms` 的整数倍（不整除时打 warning + 取最近整数倍）
2. `RoomManager::sendFrame` 入口采样 `wallEpochMs`、计算 `forward` 标志，扩展私有帧 `FRAME_BEGIN`
3. `roomprotocol.h` 解析新增字段（扩展 `parseFrameBegin*`）
4. `processIncomingFrame` 按 `forward` 决定是否调 `processFrameLocally`
5. `protocolConvert` 第三参换 `wallEpochMs`，`globalMs` 字段写 wallEpochMs，`UTC` 字段现场格式化

阶段 B（漂移监测 UI）：
6. 主机维护漂移累计、地图右上角新增浮层显示当前差值

阶段 C（清理 / 增强）：
7. TICK 文本去留（视 Q7 决定）
8. UI 镜像高级方案占位（仅文档记录，不做）

## 7. 后续分析路径建议（仍不动代码）

1. **跨场景验证用例**：列出 (Idle / Host-Observer / Host-Participant / Client) × (selfSlave OK / selfSlave fail) × (forward=0 / forward=1) 的所有路径，逐一标注 simMs / wallEpochMs / forward 来源与同源关系。
2. **收端三处用法对线**：`setTrackDisplayTime` / `appendTrackPoint` / `protocolConvert` 三处时标使用，确认 simMs vs wallEpochMs 不串味。
3. **配置整除校验位置**：在 `AppConfig` 加载时校验 `forward_interval_ms % refresh_interval_ms == 0`，不整除则取最近倍数 + 状态栏 toast 提示。
4. **dspa 转换接口签名**：`protocolConvert(const FrameTrackEntry &, qint64 wallEpochMs, qint64 simMs)`，第二参在文档中正名（不再叫 globalMs，避免歧义）。
5. **漂移修正算法落点**：阶段 B 需要主机内部维护一个"期望下一次 forward=1 的墙钟时刻"，每帧比较墙钟与期望值的差值，超过阈值时插入或跳过一次 forward=1，并把当前漂移值发到 UI 浮层。

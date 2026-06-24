# 播放 / UDP 发送时序解耦方案 (双 Timer)

## 1. 背景

当前 `appMoveTrack` 中存在两个间隔配置：

- `playback.interval_ms`（默认 100ms）：播放定时器物理 tick 间隔
  - 读：`appconfig.cpp:172`
  - 用：`mainwindow.cpp:755` 写入 `m_timerIntervalMs`
- `preprocess.refresh_interval_ms`（默认 100ms）：UDP 发送 / 显示刷新间隔
  - 读：`appconfig.cpp:203`
  - 用：`mainwindow.cpp:761` 写入 `m_refreshIntervalMs`

实际实现是**单 QTimer**：
1. `m_playbackTimer` 每 `playback.interval_ms` 墙钟 fire → `updatePlayback()`
2. `updatePlayback()` 末尾调用 `scheduleNextDataSend()`（mainwindow.cpp:3370）
3. `scheduleNextDataSend()` 计算 `(windowStart, m_globalCurrentMs]` 区间，按
   `m_refreshIntervalMs` 步长**循环补发**该区间内所有应发帧

由此带来三类问题：

- `playback=200, refresh=100`：每 200ms 墙钟突发 2 帧，平均频率对但不均匀
- `playback=50, refresh=100`：每个 tick 窗口跨 50ms，发 0 或 1 帧，发送时刻抖动
- `2× 速度`：sim 跨度 = `refresh × speed`，一个 tick 内连续补发多帧 → 突发

`refresh_interval_ms` 当前的实际语义是「仿真时间上的采样步长」，而非「UDP 物理发送
墙钟间隔」。本方案目标是把它改成后者。

## 2. 关键时间概念

- **墙钟时间 (wall-clock)**：`QElapsedTimer` 测量的真实墙上时间，与播放速度无关。
- **仿真时间 (simulation, `simMs` / `m_globalCurrentMs`)**：轨迹自身的时间轴。
- 二者关系（`PlaybackController` 内部锚点公式）：
  ```
  simMs(wallNow) = m_simBaseMs + (wallNow - m_wallClockBaseMs) × m_playbackSpeed
  ```
  seek / 变速 / 启停时调用 `anchorNow()` / `rebaseSpeed()` 重设两个 base。

## 3. 设计目标

| 配置 | 改后语义 | 影响 |
|---|---|---|
| `playback.interval_ms` | 墙钟 UI 刷新 / 光标推进节拍 | UI 平滑度，与 UDP 频率无关 |
| `preprocess.refresh_interval_ms` | 墙钟 UDP 物理发送间隔 | 真实 send rate；与播放速度无关 |
| `preprocess.density_ms` | 数据预处理密度 | 与上面两个独立，仅作用于 `preprocessTrack()` |

倍速影响：相邻发送帧之间的 sim 跨度变化（`Δsim = refresh × speed`），墙钟发送频率
不变。

业务状态机（无独立暂停）：
- **播放中**：playback timer 与 send timer 都跑
- **停止**：两个 timer 都停（send 不发心跳）
- **seek 过渡**：两个 timer 同步停 → 落点 anchor 完成 → 同步重启 + 立即补一帧

## 4. 实现步骤

### Step 1 — PlaybackController 暴露按墙钟取 sim 时刻的接口

**文件**：`playbackcontroller.h` / `playbackcontroller.cpp`

新增公开方法：
```cpp
qint64 simMsAtWall(qint64 wallNowMs) const;
```
实现公式必须与 `onTimerTick()` 内推进 `m_globalCurrentMs` 的公式逐字一致：
```cpp
qint64 dWall = wallNowMs - m_wallClockBaseMs;
qint64 sim   = m_simBaseMs + qint64(dWall * m_playbackSpeed);
if (sim < m_playbackClipStartMs) sim = m_playbackClipStartMs;
if (sim > m_playbackClipEndMs)   sim = m_playbackClipEndMs;
return sim;
```

### Step 2 — MainWindow 增加独立 send timer 成员

**文件**：`mainwindow.h` / `mainwindow.cpp` 构造函数

```cpp
// mainwindow.h
QTimer *m_sendTimer = nullptr;
void onSendTick();
void sendOneFrameNow();
```
```cpp
// 构造函数
m_sendTimer = new QTimer(this);
m_sendTimer->setTimerType(Qt::PreciseTimer);
connect(m_sendTimer, &QTimer::timeout, this, &MainWindow::onSendTick);
```

### Step 3 — applyProjectConfig 同步 send timer 间隔

**文件**：`mainwindow.cpp:761` 附近

```cpp
m_refreshIntervalMs = qMax(m_appConfig.refreshIntervalMs(), 20);  // 下限保护
m_sendTimer->setInterval(m_refreshIntervalMs);
```
去掉原 `qMax(refresh, density)` 强约束（refresh 现为墙钟语义，density 为数据语
义，二者独立）。

### Step 4 — playback timer 与 send timer 启停同步

凡是出现 `m_playbackTimer->start(...)` / `->stop()` 的位置，并排操作 `m_sendTimer`：

| 位置 | 当前操作 | 新增 |
|---|---|---|
| mainwindow.cpp:184 | `m_playbackTimer->start(m_timerIntervalMs)` | `m_sendTimer->start(m_refreshIntervalMs)` |
| mainwindow.cpp:1213 | `m_playbackTimer->stop()` | `m_sendTimer->stop()` |
| mainwindow.cpp:1258 | `m_playbackTimer->start(m_timerIntervalMs)` | `m_sendTimer->start(m_refreshIntervalMs)` |
| mainwindow.cpp:1356 | `m_playbackTimer->stop()` | `m_sendTimer->stop()` |
| mainwindow.cpp:2436 | `m_playbackTimer->stop()` | `m_sendTimer->stop()` |
| mainwindow.cpp:3108 | `m_playbackTimer->start(m_timerIntervalMs)` | `m_sendTimer->start(m_refreshIntervalMs)` |
| mainwindow.cpp:3123 | `m_playbackTimer->stop()` | `m_sendTimer->stop()` |

### Step 5 — 实现 onSendTick / sendOneFrameNow

**文件**：`mainwindow.cpp`（新增）

```cpp
void MainWindow::sendOneFrameNow() {
    if (!m_roomManager) return;
    qint64 wallNow = m_playback->wallElapsed();
    qint64 simNow  = m_playback->simMsAtWall(wallNow);
    m_roomManager->sendFrame(m_tracks, simNow);
    ++m_sentFrameCount;
    updateFrameCountLabel();
}

void MainWindow::onSendTick() { sendOneFrameNow(); }
```
关键：不再读 `m_globalCurrentMs`；每次按墙钟现采 sim 时刻。

### Step 6 — 删除 scheduleNextDataSend 旧补发逻辑

- 删除 `mainwindow.cpp:1482` 的 `scheduleNextDataSend();` 调用
- `scheduleNextDataSend()` 函数本体可整体删除（grep 确认无外部调用后再动）

### Step 7 — seek / 变速后立即补帧

- `seekAndPlay()`：anchor 完成、两 timer 重启之后 → `sendOneFrameNow()`
- `setPlaybackSpeed()` 内调用 `rebaseSpeed()` 之后 → `sendOneFrameNow()`

避免 slave 在 seek / 变速后等满一个 refresh 周期才看到新位置。

## 5. 影响文件清单

| 文件 | 变更摘要 |
|---|---|
| `playbackcontroller.h/.cpp` | 新增 `simMsAtWall()` |
| `mainwindow.h` | 新增 `m_sendTimer`、`onSendTick`、`sendOneFrameNow` 声明 |
| `mainwindow.cpp` | 构造 timer；applyProjectConfig 设 interval；7 处 start/stop 同步；删除 scheduleNextDataSend 调用；新增 onSendTick / sendOneFrameNow；seek/setSpeed 处补帧 |
| `displaysettingssection.cpp` (可选) | 调整 refresh 控件标签为「UDP 发送频率(墙钟)」 |
| `SPEC.md` / `DESIGN.md` | 同步语义说明（FR / 控制器章节） |

## 6. 验证用例

| 场景 | playback / refresh / speed | 期望结果 |
|---|---|---|
| 等值默认 | 100 / 100 / 1× | 墙钟 ~10Hz 发送，每帧 sim Δ≈100ms |
| tick 慢于发送 | 200 / 100 / 1× | 墙钟 ~10Hz 发送（不再突发 2 帧） |
| tick 快于发送 | 50 / 100 / 1× | 墙钟 ~10Hz 发送（不再过密） |
| 倍速 | 100 / 100 / 2× | 墙钟 ~10Hz 发送，每帧 sim Δ≈200ms |
| seek | 任意 | seek 落点立即收到一帧 |
| 变速切换 | 任意 | 新速度生效后立即收到一帧 |
| 停止 | — | send 完全停 |

---

## 7. 关联 Bug：发送数据相比文件源数据有微小差异

独立于上面方案，已识别的高嫌疑根因（建议在落 A 方案前先确认/修复）：

### 根因 #1（最可能）— sendFrame 永远走插值
`scheduleNextDataSend` 里的发送时间戳 `t = windowStart + N×refreshIntervalMs`，几
乎不会等于任何 Raw 点的 `timestamp_ms`。`interpolateAt()` 落入 `trackdata.h:121`
的线性插值分支：
```cpp
res.point.lat = p1.lat + ratio * (p2.lat - p1.lat);
```
即使 ratio≈0，浮点结果也已不再等于 `p1.lat`。

**修复方案**：在 `interpolateAt()` 入口加 snap-to-raw 容差：若
`|globalMs - p[k].timestamp_ms| ≤ snapMs`（建议 1ms），直接 `res.point = p[k]`。
A 方案改完后此问题仍存在，需独立修。

### 根因 #2 — 时间戳字符串截断到秒
`trackdata.h:148-159` 插值时回填 `timestamp` 字符串：
```cpp
qint64 secs = curAbsMs / 1000;   // 丢毫秒
QString("%1%2:%3:%4").arg(h).arg(m).arg(s);
```
原始数据若含毫秒分量，回填字符串会被截断到秒。

`trajectoryreader.cpp:794` 预处理插值点直接 `tp.timestamp = p1.timestamp`，但
`timestamp_ms = curMs` —— 文本字段与数值字段不一致。

### 根因 #3（次要）— 协议序列化精度
`RoomProtocol::makeFrameTrack` 中 lat/lon 序列化精度需复核。若用
`QString::number(d)` 默认 6 位有效数字，会出现 `39.123456789 → 39.1235` 的丢精度。
建议 ≥ 9 位。

### 调试切入点
1. `sendFrame` 入口打印 `globalMs` vs `p1.timestamp_ms` vs `r.point.lat - p1.lat`
   → 确认根因 #1
2. 检查 `roomprotocol.cpp` 内 lat/lon 的 `QString::number` / `setNum` 精度参数
   → 确认/排除根因 #3
3. 全文件二进制对比前后两次 send 的字节流，确认差异点

## 8. 开放问题（待确认）

1. seek 完成后是否补发一帧？默认：是
2. `refresh_interval_ms` 下限取多少？默认：20ms
3. 是否在本次同时修复「数据微小差异」bug（根因 #1 + #3）？默认：先合入 A 方案，
   bug 修复在独立 commit

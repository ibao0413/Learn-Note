# Qt 6.8.3 迁移与收口记录

日期：2026-06-23

## 目标

按老师要求，将相关程序收口到 Qt 6.8.3：

- 主程序使用 Qt 6.8.3 / C++17 / CMake / MinGW 11.2.0 x64。
- UDP 测试端改为 Qt 6.8.3/C++ 主用方案。
- Python 只保留为辅助或边角验证工具。

## 当前完成项

### 1. 构建配置收口

已确认当前本机可用工具链路径：

- Qt：`E:/work/Qt6.8.3/Qt6.8.3/6.8.3/mingw_64`
- MinGW：`E:/work/Qt6.8.3/Qt6.8.3/Tools/mingw1120_64`
- Ninja：`E:/work/Qt6.8.3/Qt6.8.3/Tools/Ninja`
- CMake：`E:/work/CLion/CLion 2024.3.6.1/bin/cmake/win/x64/bin/cmake.exe`

已调整：

- `CMakePresets.json` 改为本机实际 Qt 6.8.3 路径。
- `CMakeLists.txt` 增加 Qt 6.8.3 常见路径和环境变量兜底查找。
- `cmake/build.ps1` 改为自动查找 CMake，并处理真实输出文件名。

### 2. 新增 Qt/C++ UDP 测试端

新增文件：

- `tools/udp_test_endpoint_qt.cpp`

新增构建目标：

- `udp_test_endpoint_qt`

功能对齐原 Python 测试端：

- `selftest`：本机 UDP 自检。
- `listen`：监听 `TRACK,...` 转发数据。
- `slave`：模拟从机连接 Qt 软件主机。
- `host`：模拟主机供 Qt 软件从机加入。

### 3. Python 降级为辅助工具

保留：

- `tools/udp_test_endpoint.py`

定位：

- 辅助验证。
- 临时排查。
- 不作为主要演示和验收入口。

## Qt/C++ 测试端命令

自动回环：

```powershell
..\bin\udp_test_endpoint_qt.exe selftest
```

监听 TRACK 转发：

```powershell
..\bin\udp_test_endpoint_qt.exe listen --bind 127.0.0.1 --port 9001 --duration 120 --require-track
```

模拟从机连接软件主机：

```powershell
..\bin\udp_test_endpoint_qt.exe slave --host 127.0.0.1 --host-port 8899 --client-port 8900 --ready --duration 120
```

模拟主机供软件从机加入：

```powershell
..\bin\udp_test_endpoint_qt.exe host --host-port 8899
```

## 对老师可以这样说明

老师您好，我已经按您的要求把 UDP 测试端从 Python 主用方案调整为 Qt 6.8.3/C++ 方案。现在测试端作为 CMake 工程目标一起编译，支持 selftest 自检、listen 监听、slave 模拟从机和 host 模拟主机。Python 脚本保留为辅助备用，不作为主要验证入口。

## 本次验证结果

已完成验证：

- `cmake --preset debug` 配置通过，使用本机 Qt 6.8.3 / MinGW 11.2.0 / Ninja。
- `cmake --build --preset debug` 编译通过。
- 已生成 `E:/work/bin/appMoveTrack.exe`。
- 已生成 `E:/work/bin/udp_test_endpoint_qt.exe`。
- `udp_test_endpoint_qt.exe selftest` 通过，输出 `SELFTEST PASS`。
- 主程序 `appMoveTrack.exe` 启动冒烟通过，进程能保持运行。

## 2026-06-23 完整复测

复测内容：

- 清理旧 `appMoveTrack` / `udp_test_endpoint_qt` 进程，避免端口和 exe 锁定。
- `cmake --preset debug` 通过，确认 Qt 6.8.3 / MinGW / Ninja 路径有效。
- `cmake --build --preset debug` 通过。
- `cmake/build.ps1` 一键构建入口复测通过。
- `udp_test_endpoint_qt.exe selftest` 通过，覆盖 TRACK 本机回环和房间协议回环。
- `udp_test_endpoint_qt.exe listen` 单独测试通过，可收到 `TRACK,...` 数据。
- `udp_test_endpoint_qt.exe host` + `udp_test_endpoint_qt.exe slave` 对接测试通过，模拟从机收到 `ACCEPT`、`SET_READY`、`PLAY_START`、3 帧 `FRAME_TRACK` 和 `PLAY_STOP`。
- `appMoveTrack.exe` 启动冒烟通过，运行 5 秒未退出。

注意：

- CMake 配置阶段提示 `WrapVulkanHeaders` 未找到，不影响当前 Qt Widgets/QML/OpenGL 构建。
- Release 首次完整编译时有少量 `QDateTime` 弃用警告，不影响运行，可后续按 Qt 6 推荐写法继续清理。

## 控制台乱码处理

现象：

- `udp_test_endpoint_qt.exe selftest` 功能通过，但 PowerShell 中中文显示成乱码。

原因：

- 程序按 UTF-8 输出中文，而 Windows PowerShell 当前控制台编码可能不是 UTF-8。
- 这属于显示编码问题，不代表 UDP 测试失败；以 `SELFTEST PASS` 为功能通过标志。

处理：

- 已在 `tools/udp_test_endpoint_qt.cpp` 中为 Windows 控制台主动设置 UTF-8 输入/输出代码页。
- 如果旧 PowerShell 窗口仍显示异常，可以重新打开 PowerShell，或先执行 `chcp 65001` 后再运行测试端。

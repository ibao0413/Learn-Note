# moveTrack 运行与 UDP 回环验证记录

日期：2026-06-22

## 1. 程序功能理解

- 程序是 Qt 6.8.3 桌面轨迹可视化工具，前端为 QML，中心地图为原生 `MapWidget`。
- 主要流程：加载轨迹文件，按对象 ID 拆成多条轨迹，在地图和 3D 视图中显示，并通过回放页按时间轴播放。
- 联网层是 UDP 房间模型：主机监听 `8899`，发现端口 `8898`，从机默认接收 `8900`。
- 房间帧是文本 UDP 包：`FRAME_BEGIN`、一条或多条 `FRAME_TRACK`、`FRAME_END`。
- 转发出口当前是简化文本协议：`TRACK,objId,objType,lat,lon,alt,fit`。

## 2. 构建与启动复现

本机实际工具链路径：

- Qt：`E:\work\Qt6.8.3\Qt6.8.3\6.8.3\mingw_64`
- MinGW：`E:\work\Qt6.8.3\Qt6.8.3\Tools\mingw1120_64`
- Ninja：`E:\work\Qt6.8.3\Qt6.8.3\Tools\Ninja`
- CMake：`E:\work\CLion\CLion 2024.3.6.1\bin\cmake\win\x64\bin\cmake.exe`

标准构建流程会输出到 `E:\work\bin\appMoveTrack.exe`，Qt/C++ UDP 测试端会输出到 `E:\work\bin\udp_test_endpoint_qt.exe`。

验证启动命令需要带 Qt 运行库路径：

```powershell
$env:PATH = "E:\work\Qt6.8.3\Qt6.8.3\6.8.3\mingw_64\bin;E:\work\Qt6.8.3\Qt6.8.3\Tools\mingw1120_64\bin;$env:PATH"
Start-Process -FilePath "E:\work\appMoveTrack\local-bin\appMoveTrack.exe" -WorkingDirectory "E:\work\appMoveTrack\local-bin"
```

启动冒烟结果：程序保持运行，`local-bin\startup.log` 生成，内置自检输出总体 `PASS`。

## 3. 操作复现路径

可使用 `samples\demo_track.csv` 做演示数据。

1. 启动程序。
2. 进入数据/文件面板，选择工作区为本仓库目录。
3. 在 file1 加载 `samples\demo_track.csv`。
4. 字段映射按 `objID -> objId`、`objType -> objType`、`time -> time`、`lon -> lon`、`lat -> lat`、`alt -> alt`。
5. 切到回放页，确认轨迹参与回放，点击播放。
6. 打开 3D 视图，观察同一轨迹回放。
7. 进入节点配置页，可创建主机或加入主机，做 UDP 主从验证。

## 4. UDP 测试端

主测试端文件：`tools\udp_test_endpoint_qt.cpp`

说明：该测试端使用 Qt 6.8.3/C++ 编写，随 CMake 工程一起构建，作为主要验证入口。`tools\udp_test_endpoint.py` 仅作为辅助备用脚本。

自动回环：

```powershell
..\bin\udp_test_endpoint_qt.exe selftest
```

监听程序转发出口：

```powershell
..\bin\udp_test_endpoint_qt.exe listen --port 9001 --require-track
```

模拟从机连接 Qt 主机：

```powershell
..\bin\udp_test_endpoint_qt.exe slave --host 127.0.0.1 --host-port 8899 --client-port 8900 --ready
```

模拟主机供 Qt 从机加入：

```powershell
..\bin\udp_test_endpoint_qt.exe host --host-port 8899
```

## 5. 回环验证结论

`selftest` 覆盖两条链路：

- 本机 UDP 转发出口格式：发送并接收 `TRACK,T001,UAV,34.250000,108.940000,120.00,1`。
- 房间协议流程：`JOIN -> ACCEPT -> SET_READY -> PLAY_START -> FRAME_BEGIN/FRAME_TRACK/FRAME_END -> PLAY_STOP -> HB`。

验证通过即可证明 UDP 端口收发、文本协议封装、房间帧解析样式和转发包样式均可复现。

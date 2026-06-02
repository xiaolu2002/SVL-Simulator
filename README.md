# SVL-Simulator
# 仿真系统启动流程（命令行版）

本文档适用于完成环境搭建后，日常启动仿真系统的操作流程。所有步骤均在 **Windows 10/11** 环境下，使用 **WSL 2 + Docker Desktop** 运行。

***

## 一、启动前准备

### 1. 确认 Docker Desktop 已启动

- 系统托盘应看到 Docker 鲸鱼图标 🐳。
- 所有仿真相关容器已存在且镜像正确（可通过 `docker images` 查看）。

### 2. 确认网络配置文件正确

#### 数据显示与控制客户端（PythonAPI）

- 文件：`PythonAPI/examples/demo/config/network_config.json`
- **tcp/host** 必须为 `"0.0.0.0"`（监听所有网络接口）
- **testcase\_config\_file** 指向有效的场景文件，如 `"testcases/scenario1.json"`

#### 智能驾驶客户端（lanefollowing）

- 文件：`lanefollowing/scripts/network_config.json`
- **host** 必须为 `"host.docker.internal"`（容器通过此域名访问宿主机）

> 修改配置文件后无需重启 Docker，但需重启对应的 Python 程序。

***

## 二、启动所有 Docker 容器

使用 Docker Desktop 图形界面或命令行启动以下容器：

| 容器名称                 | 镜像名                   | 说明        |
| -------------------- | --------------------- | --------- |
| `sorasvl-mongo`      | `sora-svl-mongo`      | 数据库       |
| `sorasvl-server`     | `sora-svl-server`     | 本地云服务端    |
| `sorasvl-client`     | `sora-svl-client`     | 本地云客户端    |
| `sorasvl-router`     | `sora-svl-nginx`      | 路由 / 网关   |
| `inspiring_franklin` | `lgsvl/lanefollowing` | 智能驾驶客户端容器 |

> 容器名称可能不同，请使用 `docker ps -a` 查看实际名称。\
> 确保所有容器状态为 **Up (running)**。

***

## 三、启动各组件

### 3.1 启动 Simulator 模拟器

- 双击 `simulator-build/simulator.exe`
- 点击 **OPEN BROWSER** 按钮
- 在浏览器中依次点击：`Simulations` → `API Only`
- 确认状态变为 **Running**

### 3.2 启动数据显示与控制客户端（PythonAPI）

- 打开 PyCharm 或终端，进入目录：
  ```bash
  cd D:\Graduation_Project\ST-GraphAttention\Simulator\PythonAPI\examples\demo
  ```
- 运行主程序：
  ```bash
   python .\demo_window_with_forwarder.py
  ```
- 应看到以下输出：
  ```
  Tcp server 0.0.0.0:8888 started. Waiting for connection.
  Udp server 0.0.0.0:8880 started.
  ```
- **暂不点击 Start 按钮**，保持窗口打开。

### 3.3 进入 lanefollowing 容器并启动 bridge

- 打开 **第一个 WSL 终端**，执行：
  ```bash
  docker exec -it inspiring_franklin /bin/bash
  cd /lanefollowing/scripts
  ./my_lgsvl_bridge.sh
  ```
- 看到 `Listening on port 9090` 表示成功，保持终端运行。
- 看到 ROS2 bridge 启动成功的日志（无报错）。
- **此终端保持运行**，不要关闭。

### 3.4 启动 my\_drive（控制节点）

- 打开 **第二个 WSL 终端**，执行：
  ```bash
  docker exec -it inspiring_franklin /bin/bash
  cd /lanefollowing/scripts
  ./my_drive.sh
  ```
- 应看到大量 `[INFO] [my_autoware]: Sensor ...` 及最后一行：
  ```
  [INFO] [my_autoware]: Beginning client, shut down with CTRL-C
  ```
- **此终端保持运行**，不要关闭。

> 如果 `my_drive.sh` 报错连接失败，请检查 `network_config.json` 中的 `host` 是否为 `host.docker.internal`，并确认 `demo_Window.py` 已运行且 tcp/host = `0.0.0.0`。

***

## 四、开始仿真

按顺序执行以下操作：

1. **点击** **`demo_Window.py`** **界面中的绿色 Start 按钮**\
   （此时 `my_drive.sh` 终端应开始收到指令）
2. **立即点击 Simulator 网页中的 Start 按钮**（播放三角形图标）
3. 观察效果：
   - Simulator 窗口中的车辆开始移动
   - `my_drive.sh` 终端输出速度、控制指令等信息
   - `demo_Window.py` 画面实时更新

***

## 五、停止仿真

- 先点击 Simulator 网页中的 **Stop** 按钮
- 再点击 `demo_Window.py` 界面的 **Stop** 按钮
- 在三个终端窗口中分别按 `Ctrl + C` 停止 `my_drive.sh`、`my_lgsvl_bridge.sh`
- 最后关闭 `demo_Window.py` 窗口

***

## 六、常见问题及解决方法

| 问题                                          | 可能原因                        | 解决方法                                                                             |
| ------------------------------------------- | --------------------------- | -------------------------------------------------------------------------------- |
| `my_drive.sh` 报 `Failed to connect server`  | IP 配置错误或服务端未运行              | 检查 `network_config.json` 中 host 为 `host.docker.internal`，确保 `demo_Window.py` 已启动 |
| WSL 内 `telnet host.docker.internal 8888` 不通 | WSL DNS 配置问题                | 不影响容器内部，无需处理；若容器内也不通，重启 Docker Desktop 和 WSL                                     |
| Simulator 网页中点击 Start 后车辆不动                 | bridge 未运行                  | 确认 `my_lgsvl_bridge.sh` 已启动且无报错                                                  |
| 点击 Start 后 `my_drive.sh` 无反应                | TCP 连接未收到 Start 指令          | 重启所有组件，确保按顺序：bridge → my\_drive → Start (demo) → Start (simulator)               |
| 找不到场景文件                                     | `testcase_config_file` 路径错误 | 使用相对路径，例如 `testcases/scenario1.json`，确保文件存在                                      |

***

## 七、快捷启动建议

可将以下命令保存为 Windows 批处理文件（`.bat`），实现半自动化启动：

```batch
@echo off
echo Starting Docker containers...
docker start sorasvl-mongo sorasvl-server sorasvl-client sorasvl-router condescending_fermat

echo Starting bridge...
start wsl docker exec -it condescending_fermat /bin/bash -c "cd /lanefollowing/scripts && ./my_lgsvl_bridge.sh"

echo Starting my_drive...
start wsl docker exec -it condescending_fermat /bin/bash -c "cd /lanefollowing/scripts && ./my_drive.sh"

echo Please manually start simulator.exe and demo_Window.py
pause
```

> 注意：`demo_Window.py` 和 Simulator.exe 仍需手动启动，因为需要图形界面。

***

## 八、环境要求总结

- Windows 10/11 专业版或企业版（支持 Hyper-V 和 WSL2）
- Docker Desktop 20.10+ 并启用 WSL2 后端
- WSL2 内安装 Ubuntu 20.04
- Python 3.7+ 运行 `demo_Window.py`
- 所有配置文件严格按上述说明修改

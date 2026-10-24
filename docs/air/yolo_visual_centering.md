# YOLO 目标水平视觉居中

`openhutb_yolo_mlp_control` 提供基于 YOLO 检测框的无人机水平视觉居中控制。节点根据目标在图像中的水平偏差，发布横向速度，使目标逐步接近画面中心；原有八字轨迹和 MLP 控制模式不受影响。

## 功能范围

- 仅控制水平方向，不负责前进、距离保持、避障或垂直方向居中。
- 使用固定的 AirSim 机载相机，不自动调整相机朝向。
- 无人机需要先在仿真器中起飞并保持稳定；本模式不负责解锁、起飞或降落。
- 控制器根据里程计进行坐标转换和状态保护，目标丢失、输入过期或姿态及位移超限时停止发布速度。

目标类别默认为 `car`，也可以通过 `target_class:=person` 等参数切换。

## 构建

以下命令中的 `ROS_WS` 和 `REPO_DIR` 是示例变量，请替换为你的实际路径。

- `ROS_WS`：ROS 2 工作空间根目录；
- `REPO_DIR`：本仓库根目录，通常位于 `$ROS_WS/src/ros2`。

如果仓库放在其他位置，请将 `REPO_DIR` 设置为实际路径。

```bash
export ROS_WS=/绝对路径/到/ros2_ws
export REPO_DIR=/绝对路径/到/ros2
cd "$ROS_WS"
source /opt/ros/humble/setup.bash
# 首次使用时创建虚拟环境；已有兼容环境可直接 source
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
python -m pip install -r "$REPO_DIR/src/air/openhutb_yolo_mlp_control/requirements.txt"
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select openhutb_yolo_mlp_control
source install/setup.bash
```

如果尚未创建工作空间，可先将仓库克隆到 `$ROS_WS/src/ros2`，再执行上述构建命令。模型文件位于：

```text
$REPO_DIR/src/air/openhutb_yolo_mlp_control/openhutb_yolo_mlp_control/yolo11n.pt
```

同一工作空间不要同时放置两份同名功能包。Windows 端先启动 OpenHUTB AIR 模式，再在 WSL 中使用当前 Windows 地址，并确认 `41451` 端口可达：

```bash
export AIRSIM_IP=实际Windows地址
nc -vz "$AIRSIM_IP" 41451
export YOLO_MODEL="$REPO_DIR/src/air/openhutb_yolo_mlp_control/openhutb_yolo_mlp_control/yolo11n.pt"
test -f "$YOLO_MODEL"
```

## 预览模式

预览模式只运行检测和控制计算，不向飞行控制端发送运动指令：

```bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=car \
  dry_run:=true \
  show_window:=true
```

在另一个已加载 ROS 环境的终端查看状态：

```bash
ros2 topic echo /openhutb/visual_centering/status
ros2 topic echo /openhutb/visual_centering/preview_cmd_vel
```

状态消息包含当前状态、水平误差、速度和运行模式。正常状态为 `tracking` 或 `centered`；目标或里程计不可用时会进入保护状态并输出零速度。无图形桌面时使用 `show_window:=false`。

## YOLO 窗口置顶

启动参数 `start_topmost` 控制窗口是否置顶，默认值为 `true`。使用 `show_window:=true` 启动窗口后，可以按下面方式操作：

- 启动时置顶：`start_topmost:=true`；
- 启动时不置顶：`start_topmost:=false`；
- 运行中切换：先点击 YOLO 窗口使其获得焦点，再按 `T`。

在支持 WSLg 的 WSL2 环境中，程序优先调用 Windows 宿主窗口的置顶功能；不可用时使用 OpenCV 窗口接口。无图形界面或窗口管理器不支持置顶时，该选项不会影响检测和控制功能。

## 仿真控制

确认目标类别和相机方向正确、无人机已稳定悬停，并停止其他会发布飞行指令的节点后运行：

```bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=car \
  dry_run:=false \
  show_window:=true
```

也可在启动命令中显式指定类别：

```bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=car \
  dry_run:=false \
  start_topmost:=true
```

也可以直接调用仓库中的入口脚本，不要求当前目录固定为仓库根目录：

```bash
bash "$REPO_DIR/src/air/openhutb_yolo_mlp_control/main.sh" \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=car \
  dry_run:=false
```

退出 ROS 节点不会自动降落；需要由操作者按照仿真器流程接管无人机。

## 常用参数

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `dry_run` | `true` | 仅预览，不发送飞行控制指令 |
| `target_class` | `car` | 要跟踪的 YOLO 类别 |
| `gain` | `1.8` | 水平误差控制增益 |
| `max_speed` | `0.8` | 横向速度上限，单位 m/s |
| `max_displacement` | `3.0` | 相对初始位置的水平位移上限，单位 m |
| `start_topmost` | `true` | 是否启动时置顶 YOLO 窗口 |

通常只需调整 `target_class`、`gain`、`max_speed` 和 `max_displacement`。提高速度时应同时确认仿真场景有足够空间，并观察无人机状态。

## 运行状态

控制状态发布在 `/openhutb/visual_centering/status`，预览速度发布在 `/openhutb/visual_centering/preview_cmd_vel`。主要状态包括：

- `tracking`：已检测到目标并正在调整位置。
- `centered`：目标位于水平容差范围内。
- `target_lost`：暂未获得有效目标，输出零速度。
- `odom_stale`：里程计无效或已过期，输出零速度。
- `displacement_limit`、`heading_changed`：触发位移或航向保护，输出零速度。

该模式只修正目标的水平图像偏差；目标的纵向位置、距离和相机朝向需要通过场景布置或其他控制模块处理。

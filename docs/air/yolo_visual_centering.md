# YOLO 目标水平视觉居中

本功能扩展 `openhutb_yolo_mlp_control`：订阅 YOLO 检测框，选择指定类别，
用横向平移把目标保持在画面水平中央。原有八字 MLP 模式仍可单独启动。

## 范围与假设

- 只修正水平图像偏差，不提供前进、距离保持、避障或垂直画面居中。
- 相机必须固定前视、未镜像、无滚转且相对机体无偏航。可以使用固定的向下俯视角，
  但本功能不修改相机朝向；偏航云台和外部转动不在控制假设内。
- 第一次同时收到有效目标和里程计时记录高度与位置参考，限速修正高度漂移。
  原生飞控保持桥接连接时的航向，不代表高度和航向没有动态误差。
- 仿真无人机须先起飞并稳定。本模式不解锁、不起飞，也不自动降落。
- `control_backend:=auto` 延续原桥接的后备机制：飞控未就绪时可能直接更新位置。
  既有姿态位置后备会忽略碰撞，不能作为动力学、避障或实机验证证据。
  原生飞控实验须显式使用 `control_backend:=flight` 并确认飞控就绪。

误差定义为 `error_x = 2 * bbox_center_x / image_width - 1`。
正值表示目标在右侧，输出机体向右的速度，经里程计航向转换为世界 NED 速度。
误差绝对值不超过 `deadband` 时水平速度为零；否则使用比例增益并限速。
高度控制使用 NED 的 z 向下约定，不根据单目检测框估计目标距离。

## 构建与预览

通用的依赖安装及模型准备步骤见 [功能包 README](https://github.com/OpenHUTB/ros2/tree/master/src/air/openhutb_yolo_mlp_control)。以下使用已配置的运行工作空间。

现有独立运行工作空间为 `~/openhutb_ros_ws`，包在 `src/openhutb_yolo_mlp_control`。
官方仓库中的包位于 `src/air/openhutb_yolo_mlp_control`。
同一个工作空间不要放两份同名包。

```bash
cd ~/openhutb_ros_ws
source /opt/ros/humble/setup.bash
source .venv/bin/activate
colcon build --symlink-install --packages-select openhutb_yolo_mlp_control
source install/setup.bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py --show-args
```

Windows 启动 OpenHUTB AIR 模式后，在 WSL 使用已确认的当前 Windows 地址。
每次重启重新验证 `41451`；不要直接假设默认网关或历史地址可用。
以下第一行需替换为实际地址：

```bash
export AIRSIM_IP=实际Windows地址
nc -vz "$AIRSIM_IP" 41451
export YOLO_MODEL="$HOME/openhutb_ros_ws/src/openhutb_yolo_mlp_control/openhutb_yolo_mlp_control/yolo11n.pt"
test -f "$YOLO_MODEL"
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=person \
  dry_run:=true \
  show_window:=true
```

在另一已加载 ROS 环境的终端观察：

```bash
ros2 topic echo /openhutb/visual_centering/status
ros2 topic echo /openhutb/visual_centering/preview_cmd_vel
```

预览模式不接管飞行，仅向 `preview_cmd_vel` 发布计算出的控制量。
状态消息为 JSON，包含 `status`、`error_x`、`velocity_ned`、`dry_run`。
正常时状态为 `tracking` 或 `centered`；目标丢失为 `target_lost`；
里程计失效或未收到里程计为 `odom_stale`，这些异常状态均输出零速度。
无图形桌面时使用 `show_window:=false`。

窗口置顶可通过 `start_topmost:=true` 开启，或用 `start_topmost:=false` 关闭；
默认开启。在 YOLO 窗口获得键盘焦点后，按 `T` 可随时切换。
WSLg 优先通过 Windows 宿主窗口设置置顶，需要启用 Windows 可执行文件互操作；
不可用时尝试 OpenCV 后备并打印提示。具体支持取决于窗口管理器。

## 仿真控制

先停止预览。确认相机方向与目标类别、仿真无人机在空中稳定且周围有空间，
关闭八字控制、键盘控制和其他飞行指令发布者，再执行：

```bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" \
  yolo_model_path:="$YOLO_MODEL" \
  target_class:=person \
  dry_run:=false \
  max_speed:=0.3 \
  max_displacement:=3.0 \
  show_window:=true
```

也可以在官方仓库根目录使用 `bash src/air/openhutb_yolo_mlp_control/main.sh`，
后接相同的 `name:=value` 参数；该入口默认仍为预览。

launch 只启动桥接、YOLO 和视觉居中节点，不启动 MLP 或八字目标生成器。
活动模式向 `/openhutb/cmd_vel` 发消息；原生桥接通过非阻塞位置指令执行运动。
零指令会清除平滑和待执行位移，并请求保持当前位置。RPC 故障或服务端不响应时，
无法保证物理停止，应在仿真器接管。`Ctrl+C` 退出 ROS，不是自动降落指令。

## 参数与保护

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `dry_run` | `true` | 预览，不向飞行指令话题发布 |
| `start_topmost` | `true` | 启动时置顶 YOLO 窗口，运行中按 `T` 切换 |
| `target_class` | `person` | YOLO 类别名称，也可设置 `car` 等 |
| `confidence` | `0.5` | 最低置信度 |
| `gain` | `1.8` | 水平误差到机体横向速度的增益 |
| `deadband` | `0.08` | 水平容差，为半幅宽的 8% |
| `max_speed` | `0.8` | 横向速度上限，m/s |
| `max_acceleration` | `1.2` | 横向速度指令变化上限，m/s²；0 表示关闭斜坡 |
| `altitude_gain` | `0.8` | 初始高度保持增益 |
| `max_vertical_speed` | `0.3` | 垂直速度上限，m/s |
| `detection_timeout` | `0.8` | 自图像采集起的最大检测年龄，秒 |
| `odom_timeout` | `0.5` | 里程计最大年龄，秒 |
| `association_gate` | `0.25` | 相邻目标中心的归一化二维距离门限 |
| `lock_timeout` | `1.0` | 保留旧目标关联的时间，秒 |
| `max_displacement` | `3.0` | 相对初始参考位置的水平距离上限，米 |
| `max_yaw_error` | `0.35` | 相对初始参考航向的偏差上限，弧度 |
| `control_dt` | `0.05` | 控制周期，秒 |
| `camera_name` | `0` | AirSim 前视相机名称 |
| `max_dispatch_distance` | `0.2` | 桥接单次位移上限，米 |

### 看汽车与提高居中速度

使用模拟器已有的固定前视相机；不通过 ROS 修改相机朝向。

先停掉旧的控制进程，然后在已有已验证地址与模型变量的终端执行：

```bash
ros2 launch openhutb_yolo_mlp_control visual_centering.launch.py \
  airsim_ip:="$AIRSIM_IP" yolo_model_path:="$YOLO_MODEL" \
  target_class:=car dry_run:=false show_window:=true start_topmost:=true \
  gain:=1.8 max_speed:=0.8 max_acceleration:=1.2 max_dispatch_distance:=0.2
```

默认小误差下实际速度是 `gain * error_x`，仅提高 `max_speed` 不一定变快。
此 launch 默认增益为 1.8，速度上限为 0.8 m/s，并以 1.2 m/s² 限制横向速度指令变化；需要在场景中调试，
这不是已验证的最优参数。位移范围仍保持 3 米，到达边界时会停控。
姿态位置后备在视觉居中模式下消费累计位移，避免 RPC 较慢时每次只执行一个
控制周期的运动量；单次位移仍受上限约束。

首次选置信度最高的有效目标，之后选门限内最近的同类目标。
关联失败立即停控，超过 `lock_timeout` 才允许重新按置信度选取。
简单中心关联不保证跨遮挡身份一致，同类目标交叉时可能关联错误。

以下情况输出零速度：目标丢失、检测过期、里程计过期、非有限数值、无效四元数、
里程计 frame 不是 `airsim_ned`、横滚/俯仰超过 15 度、超过位移或航向限制。
最后两种分别对应 `displacement_limit` 和 `heading_changed`。
位移边界只是停控条件，不会主动返航，惯性或外部控制仍可能使无人机越界。

检测框必须按时间戳和 frame 匹配原图尺寸。未匹配、重复、倒序、未来或过期帧
不会刷新控制有效期。ROS 时钟检查采集年龄，单调时钟检查断流；时间回退后应重启节点。

## 无模拟器测试与仿真验收

在官方仓库根目录、已加载 Humble 和虚拟环境的终端运行：

```bash
export PYTHONPATH="$PWD/src/air/openhutb_yolo_mlp_control:$PYTHONPATH"
python -m unittest discover \
  -s src/air/openhutb_yolo_mlp_control/tests -p 'test_*.py' -v
ROS_DOMAIN_ID=89 python \
  src/air/openhutb_yolo_mlp_control/tests/ros_visual_centering_smoke.py -v
```

控制器使用解析针孔投影的合成目标验证闭环方向与收敛；ROS 测试通过真实 ROS
发布/订阅传送合成检测框和里程计；桥接测试使用模拟 RPC。
这些测试不连接模拟器，不替代 OpenHUTB 实测，也不提供实机可靠性结论。

仿真验收需依次验证：预览左右方向、静止目标的误差收敛、空检测立即停控、
检测流和里程计流断开后的停控，并观察模拟器实际是否停止。
记录初始误差、进入容差带时间、居中后误差、目标丢失至零指令延迟，以及实际后端。
可先用另一个终端记录：

```bash
ros2 bag record -o ~/visual_centering_run \
  /openhutb/visual_centering/status /openhutb/detections \
  /openhutb/odom /openhutb/cmd_vel /openhutb/yolo/image
```

本机操作者已在 OpenHUTB AIR 场景中确认汽车目标横向居中可运行；未记录量化居中耗时，
未提供本功能的飞行 GIF，尚需其他人在另一台机器复测。下文只读时序结果不等同于飞行性能指标。
代码、测试和文档使用 OpenAI Codex（ChatGPT）辅助编写和检查，提交者需审核最终内容并对其负责。

横向加速度参数 `max_acceleration` 的 launch 默认值为 1.2 m/s²；设为 0 可关闭速度斜坡。目标丢失、输入过期、进入中心死区或触及保护条件时直接输出零速度，不经斜坡延迟停控。该限制针对发布的速度指令，不能保证模拟器的实际物理加速度。

### 控制速度慢与走走停停

视觉居中 launch 启用 `async_image_capture`：图像抓取在独立 RPC 连接和线程执行，ROS 回调线程继续接收速度指令、发布里程计并执行看门狗。抓图请求合并，不随渲染延迟无限积压。原先同步 `simGetImages` 会阻塞这些回调，导致 `odom_stale` 和零速度反复交替，即使增大 `max_speed` 也可能每次只移动几厘米。

2026-10-24 在用户当前模拟器上以禁用 API 飞行控制的方式测试 10 秒，收到 200 条里程计，最大消息间隔约 0.09 秒；图像 RPC 同时返回。该验证未发送运动指令，不代表实际居中时间或飞行动力学测试。

`displacement_limit` 表示已经达到相对起点的水平位移上限，继续等待不会恢复运动。默认仍为 3 米。需要较大范围时，在确认仿真场景有足够空间后，可在启动命令中指定 `max_displacement:=10.0`。控制器仍只做水平居中；画面纵向位置不会由此控制器纠正。

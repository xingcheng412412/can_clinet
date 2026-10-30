# 五联割草机 CAN 控制项目

本仓库包含面向国内和韩国车辆的两套五联柴油割草机底盘控制程序。两者均基于 **ROS1（Noetic）和 Linux SocketCAN**，通过 CAN 总线连接车辆控制器（VCU）及语音报警灯，将上层运动指令转换为底盘控制帧，并向 ROS 发布车辆状态和故障信息。

## 两个项目

| 项目 | 国内版 `china` | 韩国版 `korea` |
| --- | --- | --- |
| 源码目录 | [china/can_client](china/can_client) | [korea/can_client](korea/can_client) |
| 主要用途 | 国内车辆的电缸行程控制与状态反馈 | 韩国车辆的速度控制、油门比例控制与状态反馈 |
| ROS 包名 / 节点 | `can_client` / `vehicle_core` | `can_client` / `vehicle_core` |
| 油门控制 | 速度模式，或通过 `/throttle_stroke_mode` 和 `/throttle_stroke_cmd` 切换并指定油门电缸行程 | 速度模式，或通过 `/throttle_cmd` 指定 −21～21 的整数比例档位 |
| 驻车控制 | 将驻车请求转换为电缸行程，行程参数由 YAML 配置 | 通过协议驻车位控制，`linear.z=0` 驻车、`linear.z=1` 允许行走 |
| 油门反馈 | 电缸行程，单位 mm | `throttle_out1_raw`、`throttle_out2_raw` 原始值 |
| 转向范围 | −47°～+34° | −44°～+33° |
| 发送频率 | 主控制帧 50 Hz，滚刀帧 20 Hz | 两帧当前均为 20 Hz，可分别在 YAML 中调整 |
| RTK 停车 | 支持非固定解延时停车及固定解延时恢复 | 同样支持，当前 YAML 中 `rtk/enabled: false` |
| 详细说明 | [国内版使用说明](china/can_client/README.md) | [韩国版使用说明](korea/can_client/README.md) |

**两个目录是同名 ROS 包的不同实现，构建时只选择其中一个。** 两版的 CAN 字段及 `VehicleStatus` 消息结构存在差异，切换版本时需要重新编译相关订阅程序，并使用对应车辆的配置。

## 共同功能

- **行驶控制**：接收目标速度、转向和驻车请求，结合软件使能及车辆模式下发控制指令。
- **滚刀控制**：支持完全上升、下降不运行、下降运行和半限位四种目标，包含动作状态机、反馈确认及超时处理。
- **车辆反馈**：解析 VCU 状态、液位、转向及故障数据，发布自定义 ROS 消息。
- **报警与提示**：控制 CAN 语音报警灯，处理车辆故障、安全触边、RTK 异常及障碍物提示；障碍物提示本身不触发停车。
- **定位联动**：使用 `/velocity_enu` 获取实测速度，使用 `/gnss_pose` 进行起步增益分级，并接收 `/rtk_log`。
- **调试工具**：提供基于 Python `curses` 的终端上位机，支持键盘控制和状态监视。

## 目录结构

```text
five_gang/
├── README.md
├── china/
│   └── can_client/             # 国内版 ROS 包
└── korea/
    └── can_client/             # 韩国版 ROS 包
```

两套 `can_client` 的主要目录如下：

```text
can_client/
├── config/can_client.yaml      # 运行参数，由 launch 加载
├── include/                   # 车辆接口、协议及控制状态机、告警逻辑
├── src/vehicle_core.cpp        # ROS 节点、SocketCAN 收发和控制流程
├── msg/                       # VehicleStatus、Alarm 消息
├── launch/can_client.launch    # 启动入口
├── upper_computer/             # 终端上位机 app.py、run.sh
├── CMakeLists.txt              # catkin 构建配置
├── package.xml                # ROS 包依赖
└── README.md                  # 对应版本的详细协议与使用说明
```

国内版另有 `include/knife_state_logger.hpp`；韩国版另有 `srv/ThrottleCommand.srv` 和 `tests/`。

## 环境与依赖

- Linux，支持 SocketCAN 的 CAN 适配器及驱动。
- ROS1 Noetic、catkin、C++ 编译工具和 Python 3。
- ROS 依赖：`roscpp`、`rospy`、`std_msgs`、`geometry_msgs`、`std_srvs`、`message_generation`、`message_runtime`。
- 外部消息包：`rtk_gps_driver_cpp` 和 `lidar_sam`，需要放入工作空间或提前构建并加载其环境；本仓库不包含这两个包。
- 调试 CAN 报文可使用 `can-utils` 中的 `candump`。

## 编译与启动

下面以国内版为例，从本仓库根目录执行，将选定版本复制到独立 catkin 工作空间。韩国版将 `china` 替换为 `korea`，并使用另一个工作空间目录。

```bash
source /opt/ros/noetic/setup.bash
mkdir -p ~/five_gang_china_ws/src
cp -r china/can_client ~/five_gang_china_ws/src/

# 编译前将 rtk_gps_driver_cpp、lidar_sam 放入 src，
# 或 source 已包含这两个包的工作空间环境。
cd ~/five_gang_china_ws
catkin_make
source devel/setup.bash
```

配置与车辆连接的 CAN 接口，默认接口为 `can0`，波特率为 **250 kbit/s**：

```bash
sudo ip link set can0 up type can bitrate 250000
ip link show can0
```

检查所选项目的 `config/can_client.yaml`，再启动节点：

```bash
roslaunch can_client can_client.launch
```

配置包含 CAN 接口、报警灯 ID、滚刀动作时间、RTK 延时和起步增益等内容；两版参数不完全相同。修改 YAML 后需要重新启动节点。

### 启动终端上位机

在另一个终端中运行：

```bash
cd ~/five_gang_china_ws/src/can_client/upper_computer
bash run.sh
```

`run.sh` 会加载 Noetic 和所在工作空间的 `install/setup.bash` 或 `devel/setup.bash`，然后启动终端界面。上位机需要车辆节点及相应服务已经启动，具体按键见各版本说明。

## 主要 ROS 接口

以下名称按默认 launch、无额外命名空间时列出。

| 接口 | 类型 | 用途 |
| --- | --- | --- |
| `/vehicle_cmd` | 话题，`geometry_msgs/Twist` | `linear.x` 为目标速度（m/s），`angular.z` 为转向角（度）；`linear.z` 为驻车请求，具体语义见对应版本 |
| `/state_cmd` | 服务，`std_srvs/SetBool` | 软件使能与禁用 |
| `/rod_mode` | 服务，`std_srvs/SetBool` | 滚刀目标状态请求 |
| `/fault_reset` | 话题，`std_msgs/UInt8` | 故障复位请求 |
| `/can_info` | 话题，`can_client/VehicleStatus` | 底盘状态反馈 |
| `/fault_raw` | 话题，`std_msgs/String` | 原始故障信息 |
| `/alarm` | 话题，`can_client/Alarm` | 底盘告警事件 |
| `/velocity_enu` | 话题，`geometry_msgs/TwistWithCovarianceStamped` | 实测速度输入 |
| `/gnss_pose` | 话题，`geometry_msgs/PoseStamped` | 起步增益所需的位置输入 |
| `/rtk_log` | 话题，`rtk_gps_driver_cpp/RTKGPSLog` | RTK 状态输入 |
| `/detection/lidar_detector/objects` | 话题，`lidar_sam/DetectedObjectArray` | 障碍物提示输入 |

国内版专用接口：`/throttle_stroke_mode`（`std_srvs/SetBool`）和 `/throttle_stroke_cmd`（`std_msgs/UInt16`）。韩国版专用接口：`/throttle_cmd`（`can_client/ThrottleCommand`）。

`/rod_mode` 使用 `SetBool` 的原始单字节值表达 0～3 四种目标，属于非标准用法；会把所有非零值转换为 `true` 的客户端无法区分目标 1、2、3。调用方式及动作完成判定请查看对应项目说明和实现。

## CAN 通信概览

| 方向 | CAN ID | 内容 |
| --- | --- | --- |
| 控制节点 → VCU | `0x110` | 行驶、油门、驻车及转向控制 |
| 控制节点 → VCU | `0x111` | 滚刀、实测速度及附加控制字段 |
| VCU → 控制节点 | `0x100` | 车辆主状态 |
| VCU → 控制节点 | `0x101` | 故障及液位等反馈 |
| 控制节点 → 报警灯 | `0x240`（默认） | 灯效及语音控制 |
| 报警灯 → 控制节点 | `0x241`（默认） | 控制确认 |

节点直接使用 SocketCAN，无需额外 CAN 桥接节点。两版相同 ID 的字段编码可能不同，接入前请按所选项目核对协议。

可通过以下命令查看运行状态：

```bash
rosnode list
rostopic echo /can_info
rostopic echo /alarm
candump can0
```

完整帧布局、控制流程、报警灯配置及故障排查分别见[国内版 README](china/can_client/README.md)和[韩国版 README](korea/can_client/README.md)。若详细说明中的参数默认值或接口名称与当前版本有出入，以所选版本的源码和 `config/can_client.yaml` 为准。

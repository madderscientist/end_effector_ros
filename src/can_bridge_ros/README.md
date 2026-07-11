# can_bridge_ros

通用 ROS 2 CAN 总线桥接：一个节点独占一个物理 USB-CAN 设备，将收到的全部帧发布为 `can_msgs/msg/Frame`，并订阅命令帧下发。支持 CANalyst-II 单设备多通道。

本包只负责 ROS 参数、消息转换、线程调度和话题分发；无 ROS 的总线创建、CANalyst-II `libusb` 准备及权限检查统一由 [`CAN-SDK`](../CAN-SDK/README.md) 提供。

`CAN-SDK` 是被 `COLCON_IGNORE` 排除的纯 Python 包。运行本节点前应 source 工作区的
`scripts/env.sh`，由它通过 `PYTHONPATH` 暴露 SDK 源码；无需安装本地 SDK。

## `can_msgs` 来源

`from can_msgs.msg import Frame` 来自上游 ROS 2 **`can_msgs`** 消息包，不是
`python-can`、`can_sdk` 或本仓库定义的类型。该包的源码位于
[`ros-industrial/ros_canopen`](https://github.com/ros-industrial/ros_canopen/tree/dashing-devel/can_msgs)，
用于定义 CAN 相关 ROS 消息；`Frame` 是由 ROS 接口生成器生成的 Python 消息类。

ROS 2 Foxy 安装命令：
```bash
sudo apt-get install -y ros-foxy-can-msgs
```

依赖同时在本包的 `package.xml` 中声明为 `<exec_depend>can_msgs</exec_depend>`。
其中 `python-can`/`can_sdk` 负责访问物理 CAN 总线，`can_msgs/Frame` 只负责 ROS 节点间传输一帧 CAN 数据。

```text
第1层 can_sdk        : python-can 后端与基础 I/O（无 ROS、无设备协议）
第2层 can_bridge_ros : 独占物理总线，发布 /canX/rx，订阅 /canX/tx
第3层设备节点        : 订阅 /canX/rx 并按自身 CAN ID 过滤
```

## 话题与参数

| 方向 | 话题 | 类型 | QoS |
|---|---|---|---|
| 发布 | `/<bus_name>/rx` | `can_msgs/Frame` | BEST_EFFORT, KEEP_LAST(depth) |
| 订阅 | `/<bus_name>/tx` | `can_msgs/Frame` | RELIABLE, KEEP_LAST |

| 参数 | 默认 | 说明 |
|---|---|---|
| `interface` | `canalystii` | python-can 后端 |
| `channel` | `"0"` | 单通道；多通道如 `"0,1"` |
| `bitrate` | `1000000` | 比特率 |
| `channel_ids` | `[0]` | `Message.channel` 值 |
| `bus_names` | `["can0"]` | 对应 ROS 总线名 |
| `rx_queue_depth` | `1000` | 接收发布队列深度 |
| `receive_own_messages` | `false` | 是否回显发送帧 |

## 启动

```bash
source ~/end_effector_ros/scripts/env.sh
ros2 launch can_bridge_ros can_bridge_ros.launch.py config:=single_bus.yaml
ros2 launch can_bridge_ros can_bridge_ros.launch.py config:=dual_bus.yaml
```

CANalyst-II 是一个 USB 设备。双通道必须用同一进程通过 `channel="0,1"` 打开，两个独立 Bus 可能产生 `Resource busy`。

## CANalyst-II Linux 权限
```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="04d8", ATTR{idProduct}=="0053", MODE="0666", GROUP="plugdev"' \
  | sudo tee /etc/udev/rules.d/99-canalystii.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## 设备节点约定
- 订阅 `/<bus_name>/rx`（BEST_EFFORT），按 `frame.id` 过滤。
- 发布命令到 `/<bus_name>/tx`（RELIABLE）。
- `can_msgs/Frame.data` 是定长 8 字节整数列表，`dlc` 表示有效长度。
- 不允许设备节点再次直接打开同一物理 CAN；直连模式仅用于 bridge 未运行时的台架调试。

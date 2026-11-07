---
layout: default
title: CAN 设备接入 ROS 2 - candump 的 ~ 不是区间
---

# CAN 设备接入 ROS 2：被全网用错的那个 `~`，和三种接法怎么选

> 实测环境：Ubuntu 24.04（内核 6.8.0-147-generic），加载 `vcan` 模块，`can-utils` + `python-can` + `ros2_socketcan`
> 一句话结论：**`candump` 里的 `~` 是「非」运算符，不是 ID 区间。** 用错的人比你想的多。

---

## 先说那个坑

如果你在搜"CAN 怎么过滤某一段 ID"，十有八九会看到这种写法：

```bash
candump can0,600~7FF:7FF
```

看起来像是"只看 0x600 到 0x7FF"。**它不是。**

`~` 在 candump 的过滤器语法里是**「非」**。上面这条的真实含义是：

> 放行除 0x600 以外的**所有**帧。

实测环境：`vcan0`，总线上有两帧 —— `0x601`（SDO 响应）和 `0x181`（TPDO1）。

```bash
candump vcan0,600~7FF:7FF
# 结果同时捕获 601 和 181
```

`0x181` 根本不在"600 到 7FF"里，但它被放行了——因为 `~` 只排除了 `0x600` 一个 ID。

反向复测，三种写法对照：

| 过滤器 | 实际效果（实测） |
|---|---|
| `candump vcan0,181:7FF` | 只捕获 `181` ✅ |
| `candump vcan0,601:7FF,581:7FF` | 只捕获 `601` / `581` ✅ |
| `candump vcan0,600:780` | 捕获 `601`，排除 `181` ✅（掩码） |
| `candump vcan0,600~7FF:7FF` | 捕获 `601` **和** `181` ❌ |

### 正确做法

**要"多个精确 ID"用逗号并列，按 OR 组合：**

```bash
# 只看节点 1 的 SDO 请求/响应
candump can0,601:7FF,581:7FF
```

**要"一个 ID 段"用掩码：**

```bash
# 0x600 掩码 0x780 → 捕获 0x600~0x67F 范围
candump can0,600:780
```

掩码的含义是"哪些位必须匹配"。`780` = `0111 1000 0000`，高 7 位固定，低 4 位任意。

这个错误在现场很隐蔽：**过滤器不会报错，只是悄悄多放行几百帧**。你以为是设备在乱发，其实是自己把无关帧放进了日志。

---

## 为什么要先在 vcan 上把过滤器跑一遍

真车上试过滤器的代价是：要有人在旁边看着设备，要担心误发控制帧，出了事还得查是谁发的。

`vcan` 让你在笔记本上凭空造一条 CAN 总线：

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
ip -details link show vcan0
```

发两帧进去：

```bash
cansend vcan0 601#2B40600001000000
cansend vcan0 181#0102030405060708
```

然后你就有了一条可以随便折腾的总线。上面那张对照表就是这么测出来的。

> ⚠️ WSL2 的内核**没有** `vcan` 模块（`modprobe vcan` 直接报 `Module vcan not found`）。要在 WSL 里做这件事，得换带 CAN 模块的常规内核，或者上虚拟机 / 实体机。Ubuntu 云镜像 + `linux-modules-extra-$(uname -r)` 就够。

---

## 三种接法，怎么选

CAN 设备要进 ROS 2，本质是把"总线上的一帧"变成"ROS 2 的一个话题消息"。三种做法，复杂度差很多。

### 方式一：`socketcan_bridge` / `ros2_socketcan`

最省事。装包、起节点、完事。

```bash
sudo apt install ros-$ROS_DISTRO-ros2-socketcan
ros2 run ros2_socketcan socket_can_receiver_node_exe \
  --ros-args -p interface:=can0
```

**优点**：零代码，`can0` 上所有帧进 `/from_can_bus`。
**缺点**：只做搬运，**不解协议**。拿到的是 `Frame` 原始 8 字节，位域、缩放、字节序全得你在下游自己拆。适合"先把数据接进来看看"，不适合做控制。

### 方式二：`ros2_canopen`

如果设备是标准 CANopen（伺服驱动器、比例阀、I/O 模块大量是），这条路能省掉你自己解 EDS、拼 SDO 的活。

```bash
sudo apt install ros-$ROS_DISTRO-ros2-canopen
```

你提供 `device.eds` 和总线配置，它把对象字典映射成 ROS 2 接口。

**优点**：协议层现成，PDO/SDO/NMT 都不用自己写。
**缺点**：对**非 CANopen 的私有协议无能为力**。国内很多工程机械控制器走的是私有报文格式，这条路走不通。

> **没有 EDS 就别猜。** 靠索引试出来的映射关系，一旦设备固件升级就全废，而且错了不会报错——只会表现为"数值不对"。

### 方式三：自己写桥接节点

最常见的现实选择。

```python
import can
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float64

class CanBridge(Node):
    def __init__(self):
        super().__init__('can_bridge')
        self.bus = can.interface.Bus(channel='can0', bustype='socketcan')
        self.pub = self.create_publisher(Float64, '/joint/speed', 10)

    def spin_once(self):
        msg = self.bus.recv(timeout=0.01)
        if msg and msg.arbitration_id == 0x181:
            raw = int.from_bytes(msg.data[0:2], 'little')  # 字节序要按 EDS/手册
            self.pub.publish(Float64(data=raw * 0.001))   # 缩放按手册

def main():
    rclpy.init()
    node = CanBridge()
    node.create_timer(0.001, node.spin_once)
    rclpy.spin(node)
```

**优点**：完全可控，能处理任意私有协议、能做超时看门狗、能做单位换算。
**缺点**：协议解析、错误处理、重连逻辑全要自己写。**而且 100Hz 控制环下最容易翻车。**

---

## 100Hz 控制环：最容易翻车的地方

三种方式在"能跑"层面差别不大，**差别在实时性和异常处理**。

几个必须处理的事：

**1. 心跳丢了要停输出。**

CANopen 的心跳周期上报，主站靠它判断节点是否掉线。如果心跳超时还继续发控制帧，就是**在给一个可能已经失联的驱动器下指令**——这是安全事故，不是 bug。

```python
if time.monotonic() - self.last_heartbeat > 0.5:
    self.publish_zero_velocity()   # 先停，再报警
    return
```

**2. 别用阻塞式 SDO 读周期数据。**

SDO 是请求-响应，阻塞式读写会卡住整个回调，直接毁掉 100Hz 的节拍。周期数据走 PDO，参数配置才用 SDO。

**3. 字节序和缩放，按手册来，别按猜的。**

同一个厂家的驱动器和变频器，字节序都可能不一样。`int.from_bytes(data, 'little')` 还是 `'big'`，必须查手册或 EDS。

**4. `recv` 一定要带超时。**

```python
msg = self.bus.recv(timeout=0.01)   # 不加超时，总线一静默就永久阻塞
```

---

## 自测清单

上线前在 vcan 上跑一遍：

- [ ] `vcan0` 能创建、能 up
- [ ] `cansend` 发出的帧能被 `candump` 捕获
- [ ] 过滤器用**精确 ID 并列**或**掩码**，没有用 `~` 当区间
- [ ] Python 节点发出的帧，`candump` 能看到
- [ ] 拔掉设备（`ip link set down`）后，节点能检测到并停输出
- [ ] 连续跑 30 分钟，节拍不漂

---

## 一句话总结

- **`~` 是「非」，不是区间**——这个错会让你的日志多出几百帧无关数据，而且不报错；
- 协议是标准 CANopen 就走 `ros2_canopen`，是私有报文就自己写桥接，别硬套；
- 先接进来（`ros2_socketcan`）再谈控制，顺序别反；
- 100Hz 之下，**心跳超时停输出**和**`recv` 带超时**是必做项。

---

*文中过滤语义、vcan 收发数据均来自 QEMU Ubuntu 24.04 + vcan0 实测。真实总线时序、终端电阻、错误计数仍需实车验证。*

---
layout: default
title: 国产平板跑 ROS 2 - 前摄没驱动、画面闪烁、RViz 起不来
---

# 国产平板跑 ROS 2：前摄没驱动、画面一闪一闪、RViz 起不来，我全踩了一遍

> 实测设备：小米 Pad 6 Pro（骁龙 8+ / arm64），Ubuntu 26.04，ROS 2 Lyrical，libcamera 0.7.0
> 结论先行：**能跑，后摄 30fps 稳定输出到 RViz；但前摄是内核层问题，应用层改不出来。**

---

## 为什么要在一台平板上跑 ROS 2

不是折腾。工业现场有个很实际的需求：**移动端低成本的视觉调试终端**。

现场的机器人底盘、机械臂、AGV，调试的时候经常需要一个人抱着笔记本蹲在设备边上，看 `/camera/image_raw`、看 TF 树、看 Nav2 的代价地图。笔记本沉、续航短、还得拖线。

平板天然合适：屏幕大、触控、能塞进工具包、续航顶一天。前提是——**内置摄像头能不能直接发布成 `sensor_msgs/Image`**。

这篇文章就是回答这一个问题，以及路上踩的五个坑。

---

## 坑一：前置摄像头，在设备树层面就不存在

第一个反直觉的结论：**不是你配置错了，是硬件通路没接。**

看设备树：

```bash
# 唯一的 camera 节点
/soc@0/cci@ac15000/i2c-bus@1/camera@10
# 另一路 CCI 是空的
/soc@0/cci@ac16000/i2c-bus@0    # 无 camera 子节点
/soc@0/cci@ac16000/i2c-bus@1    # 无 camera 子节点
```

再看内核模块：

```bash
ls /lib/modules/$(uname -r)/kernel/drivers/media/i2c/
# 只有 s5kjn1.ko，没有 imx、ov 系列的任何 sensor 驱动
```

再看供电 regulator：

```bash
# 只有 camera-wide-avdd / camera-wide-dvdd
# 没有前摄的独立供电 rail
```

三条证据指向同一个结论：**这台机器在当前内核移植里只有一颗 sensor —— 后置的 Samsung S5KJN1（i2c 地址 `7-0010`）。前摄既没有驱动、没有设备树节点、也没有上电序列。**

要启用前摄，得做这些事：

1. 新增 sensor 驱动（IMX / OV 系列，看具体型号）
2. 在设备树里补 CCI 总线 + CSIPHY 节点
3. 写完整的 regulator 上电序列
4. 准备 libcamera tuning 文件

**这是内核移植工作，不是应用层能解决的事。** 网上一堆"改个参数就能开前摄"的帖子，在这类移植板子上都是错的。

---

## 坑二：`v4l2_camera` 和 `usb_cam` 全部返回 EINVAL

后摄有驱动了，那接 ROS 2 是不是装个包就行？

不是。

```bash
sudo apt install ros-$ROS_DISTRO-v4l2-camera
ros2 run v4l2_camera v4l2_camera_node
# [ERROR] VIDIOC_S_FMT: Invalid argument
```

`usb_cam` 同样是 `EINVAL`。

原因在于这类 Qualcomm CAMSS 设备暴露的 `/dev/videoN` 是 **V4L2 多平面（MPLANE）节点**，只支持 `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE`。而 `v4l2_camera` / `usb_cam` 的 ioctl 走的是 `VIDEO_CAPTURE`，格式协商直接从 `VIDIOC_S_FMT` 就失败了。

验证一下节点类型：

```bash
v4l2-ctl -d /dev/video0 --all | grep -i "type"
# Type: Video Capture Multiplanar
```

**唯一现成的通路是 libcamera 的 simple pipeline handler**，它已经把 media controller 拓扑封装好了：`msm_csiphy3 → msm_csid0 → msm_vfe0_rdi0`。

所以数据链路必须是：

```
libcamerasrc (GStreamer) → videoscale → videoconvert → appsink → rclpy → /camera/image_raw
```

---

## 坑三：画面「一闪一闪」，但不是曝光问题

这是最费时间的一个。

现象：RViz 里的画面持续闪烁、跳帧，像老电视信号不良。

第一反应是自动曝光在震（AE 曝震），或者自动白平衡抖色。**先测，别猜。**

逐帧统计 50 帧的平均亮度：

```python
Y = 128.4 → 128.2   # 波动 < 0.2
```

**亮度恒定，直接排除 AE 曝震和白平衡抖色。**（顺带一提：这个移植版的 `ae-enable` 默认本来就是 `false`。）

那闪烁从哪来？测帧间隔：

```
旧脚本：~19 fps，间隔 27–73 ms
且存在 1–8 ms 的「连发帧」
```

找到根因了：**不是丢帧，是帧间隔严重不匀 + 连发帧。**

连发帧的危害在于：RViz 一帧的显示周期里，被刷新两次，人眼看到的就是"闪"。

修复分两步：

**第一步，把缩放挪到 GStreamer 端。**

Python 端缩放太慢，下游一卡，旧帧就堆在队列里，随后成串吐出。改成：

```
videoscale (GStreamer) → videoconvert → appsink
```

结果：

| 指标 | 改之前 | 改之后 |
|---|---|---|
| 帧间隔 min | 27 ms | 30 ms |
| 帧间隔 max | 73 ms | 68 ms |
| 间隔标准差 | 0.0124 | **0.0072** |
| 连发帧 | 有 | **消失** |

**第二步，QoS 改成 BEST_EFFORT + depth 1。**

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

qos = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,
    history=HistoryPolicy.KEEP_LAST,
    depth=1,
)
```

只要最新帧，旧帧不排队——这是视频流的正确做法。如果订阅端硬要 Reliable，才退回 `RELIABLE + depth 5`。

**第三步，把 RViz 的 Frame Rate 压到 10。**

RViz 默认 30 fps，在平板上跑 Xwayland 软件渲染根本跟不上，自己就成了卡顿源。

三步做完，画面稳定。

---

## 坑四：RViz 在 Wayland 会话下直接起不来

```bash
ros2 launch tablet_camera tablet_camera.launch.py
```

报错：

```
Invalid parentWindowHandle (wrong server or screen) in GLXWindow::create
```

原因：GTK 默认挂到 **Wayland** 上，而 Ogre 渲染后端用的是 **GLX**（X11 的东西）。协议对不上。

修复，启动前加两个环境变量：

```bash
export GDK_BACKEND=x11
export QT_QPA_PLATFORM=xcb
```

在 launch 文件里用 `additional_env` 固化：

```python
Node(
    package='rviz2',
    executable='rviz2',
    additional_env={'GDK_BACKEND': 'x11', 'QT_QPA_PLATFORM': 'xcb'},
)
```

这类问题在国产板子移植的 Ubuntu 上非常常见——**桌面会话是 Wayland，图形工具链还按 X11 走**，两套栈混在一起。

---

## 坑五：跑久了管线静默停帧，裸 while 把 CPU 吃满

测试连续运行的时候，节点彻底冻结了 **38 分钟**。

原因：libcamera 在长时间运行后可能不再产帧，`pull-sample()` 返回 `None`。而旧代码是：

```python
while True:
    sample = appsink.pull_sample()
    # sample 是 None 就空转
```

一个核直接被吃满，而且节点完全"看起来正常"——不报错、不退出。

两个修复：

1. **把轮询放进 `rclpy.timer`**，别用裸 `while`，节点还能响应回调。
2. **加看门狗**：超过 `watchdog_timeout`（默认 3 秒）没有新帧，就重建整条管线。

```python
if now - self.last_frame_time > self.watchdog_timeout:
    self.get_logger().warn('管线停帧，重建')
    self.rebuild_pipeline()
```

另外还有个硬件侧的坑：**相机 + RViz 连续跑会把平板电耗干**。RViz 走 Xwayland 软件渲染非常吃电，耗干后整机黑屏、SSH 全部无响应，插电约 20 分钟才自行恢复（数据没坏）。

**长时间采集一定要插着电跑。**

---

## 最终方案

整理成一个可以直接用的 ROS 2 包：`tablet_camera`（Apache-2.0）。

核心参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `width` / `height` | 960 / 600 | 缩放由 GStreamer 完成 |
| `fps` | 15.0 | 轮询上限，不是发布上限 |
| `image_best_effort` | true | 视频必开 |
| `watchdog_timeout` | 3.0 | 无帧重建管线 |
| `hfov_deg` | 60.0 | 未标定时按视场角估焦距 |

### 关于内参：这是估算值，不是标定值

`CameraInfo` 的 `K` / `P` 是按针孔模型反推的：

```
fx = fy = width / (2 * tan(hfov/2))
cx = width / 2,  cy = height / 2
D  = [0, 0, 0, 0, 0]
```

`D` 恒为 0。而且传感器的图像路径经过 libcamera 的 **uncalibrated IPA**（没有 tuning 文件），**几何畸变本身就没被校正**。

要跑 SLAM、测距、标定板对齐，必须自己跑一遍标定：

```bash
ros2 run camera_calibration cameracalibrator --size 8x6 --square 0.025 \
  image:=/camera/image_raw camera:=/camera
```

然后把 YAML 灌进下游节点。**别拿估算的内参去做量测，误差会毁掉整个标定链路。**

---

## 实测数据汇总

| 项目 | 结果 |
|---|---|
| 可用传感器 | 仅后置 S5KJN1 |
| 前置摄像头 | 不可用（无驱动/无设备树/无供电） |
| 发布分辨率 | 960 × 600 |
| 实际帧率 | 27.9 – 30 fps |
| 帧间隔 | 30 – 68 ms，std ≈ 0.007 s |
| 内参 | 估算 fx = fy = 831.384（未标定） |
| 通路 | libcamerasc → videoscale → appsink → rclpy |

---

## 一句话总结

国产 arm64 平板上跑 ROS 2 视觉链路，**能跑通，但前提是接受三条现实**：

1. 有驱动问题先看设备树，别在应用层试参数；
2. V4L2 多平面设备绕开 `v4l2_camera`，走 libcamera；
3. 画面异常先测数值——亮度恒定就说明不是 AE，去测帧间隔。

---

*文中所有数据来自真实设备实测，未做美化。遇到同类问题（Nav2、MoveIt2、CAN 设备接入、ROS 2 环境移植）可以留言，能远程看的我直接看。*

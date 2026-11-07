# 工业机器人 ROS 2 远程技术支持

> 一线交付工程师。做过工程机械远程操控改造、AGV 与机械臂部署。按问题收费，不卖课、不接代做。

**服务页：<https://bietou123.github.io/ros-support/>**

## 能解决的问题

| 方向 | 典型现象 |
|---|---|
| Nav2 / SLAM | 建图漂、定位丢、代价地图不对、参数调不通 |
| MoveIt2 | 配置生成、抓取抖动、规划失败、碰撞体不对 |
| URDF / xacro | 建模、TF 树错乱、RViz 显示异常 |
| CAN / 串口 / 网口 | 设备接进 ROS 2 没数据、时序不对、驱动报错 |
| Gazebo / Isaac Sim | 复现 demo 失败、模型加载报错、传感器不输出 |
| 国产板卡环境 | arm64 移植、平板 / 树莓派 / 工控机跑 ROS 2 |
| ros2_control | 硬件接口对接、控制器切换、状态发布 |
| 工程化 | Docker 化环境、CI 构建、多机通信配置 |

## 收费

| 服务 | 价格 | 说明 |
|---|---|---|
| 远程排障 | 200 元 / 小时 | 2 小时起，按实际时长结算 |
| 小单包干 | 800 – 2000 元 / 单 | 3–5 个工作日，开工前写死验收标准 |
| 月度陪跑 | 2000 元 / 月 | 4 次远程 + 随时答疑，限一个项目 |
| 现场调试 | 1500 元 / 天 | 差旅实报，需提前 3 天预约 |

先聊 10 分钟判断能不能做。做不了我直说，不收钱。

**不接**：毕设代做、整包 AGV / 巡检项目（需企业资质）。

## 技术文章

都是踩过的真实问题，带实测数据：

- [国产平板跑 ROS 2：前摄没驱动、画面一闪一闪、RViz 起不来](https://bietou123.github.io/ros-support/posts/tablet-ros2-camera.html)
  —— 小米 Pad 6 Pro / Ubuntu arm64 / libcamera。设备树判据、V4L2 多平面绕行、连发帧定位、Wayland 下 RViz 的 GLX 坑。
- [CAN 设备接入 ROS 2：被全网用错的那个 `~`](https://bietou123.github.io/ros-support/posts/can-ros2.html)
  —— `candump` 的 `~` 是「非」运算符不是 ID 区间，`600~7FF` 会放行除 0x600 外的所有帧。三种接法对比 + 100Hz 控制环清单。
- [Nav2 起不来？先看 `base_footprint`](https://bietou123.github.io/ros-support/posts/nav2-wont-start.html)
  —— 9 个 lifecycle 节点逐个查、`map→odom` 断链定位、`ros2 lifecycle get` 卡死、`default_server_timeout` 20 ms 在工控机上为什么太激进。

## 技术背书

[《工业机器人全栈学习系统》](https://github.com/bietou123/ros-fullstack-learning)：S0–S10 共 11 个阶段、76 节课，覆盖 ROS 2 基础、TF/URDF、Nav2、SLAM、MoveIt2、CAN 总线、ros2_control 与工程化部署。三端离线产物（Windows / Linux / Android），69 个课程实验全自动回归通过，命令与检查项全部真实可复现。

## 联系

把你的现象、报错、已经试过的命令发到 [Issues](https://github.com/bietou123/ros-support/issues/new)，我先判断能不能做。

---
layout: default
title: Nav2 起不来 - base_footprint 与 9 个 lifecycle 节点
---

# Nav2 起不来？先看 `base_footprint`，再看这 9 个 lifecycle 节点

> 实测环境：`ros:humble` + `ros-humble-nav2-bringup` 1.1.20，8×6 m 房间栅格图，合成差速底盘（20 Hz 积分 `/cmd_vel`，回发 `/odom` 与 `/scan`）
> 结论先行：**Nav2 十次起不来，有八次不是"配置写错了"，是 TF 对不上或者你根本没等它 active。**

---

## 症状：节点都起来了，就是不动

`ros2 launch nav2_bringup bringup_launch.py` 跑完，终端刷了一堆 `[INFO]`，看着挺正常。然后：

- RViz 里机器人是散的
- 发 `navigate_to_pose`，没反应，或者直接 `ABORTED`
- `/cmd_vel` 一条都不发

这时候别急着调 `params.yaml`。先按下面的顺序查。

---

## 第一步：`amcl.base_frame_id` —— 最常见的那个坑

`nav2_bringup` 的默认参数里：

```yaml
amcl:
  base_frame_id: base_footprint
  odom_frame_id: odom
  global_frame_id: map
  scan_topic: scan
```

但你自己 URDF 里的底盘 link 大概率叫 `base_link`，不是 `base_footprint`。

后果不是报个错就算了，而是：

```
[amcl]: Couldn't transform from laser to base_footprint
```

**然后 `map→odom` 永远不会发布。**

`map→odom` 不发 → `map→base_link` 断链 → 规划器拿不到机器人在地图里的位姿 → `planner_server` 一直等 TF → 你发目标它就一直等 → 最后超时。

整条链路静默卡死，唯一线索就是 AMCL 那一行 `Couldn't transform`。

**修复**：把 `base_frame_id` 改成你 TF 树里真实存在的那个：

```yaml
amcl:
  base_frame_id: base_link
```

改完确认：

```bash
ros2 run tf2_ros tf2_echo map base_link
```

有输出、数值在动，这一步才算过。

> **判断技巧**：`ros2 topic echo /amcl_pose` 没数据 + `tf2_echo map odom` 超时 = 100% 是这个问题。`odom→base_link` 有、`map→odom` 没有，也指向它。

---

## 第二步：9 个 lifecycle 节点，真的都 active 了吗

Nav2 是 lifecycle 架构，节点起来不等于能用。**没 active 的节点不会干活，而且不一定报错。**

实测一套标准的 Nav2 全栈应该有这 9 个：

```
map_server        amcl             bt_navigator
planner_server    controller_server behavior_server
smoother_server   waypoint_follower velocity_smoother
```

全部 `active [3]` 才算起来：

```bash
for n in map_server amcl bt_navigator planner_server controller_server \
         behavior_server smoother_server waypoint_follower velocity_smoother; do
  printf "%-20s " "$n"
  timeout 5 ros2 lifecycle get /$n || echo "TIMEOUT"
done
```

配套的 12 个动作服务器：

```bash
ros2 action list | grep -E "navigate_to_pose|compute_path|follow_path|spin|backup|wait|smooth"
```

预期能看到 `navigate_to_pose`、`compute_path_to_pose`、`compute_path_through_poses`、`follow_path`、`spin`、`backup`、`wait`、`smooth_path` 等。

### ⚠️ `ros2 lifecycle get` 会卡住

这一条是实测踩出来的：**命令行轮询 `ros2 lifecycle get` 可能长时间不返回。**

现象很迷惑——你明明在 RViz 里看到 Nav2 已经跑起来了，脚本却卡死在某一次查询上，看起来像是"节点没响应"。

原因不是节点的问题，是这条查询本身在某些状态下不返回。

**修法**：所有查询包一层 `timeout`：

```bash
timeout 5 ros2 lifecycle get /amcl
```

返回码非 0 就当超时处理，继续查下一个。

---

## 第三步：`bt_navigator` 的 `default_server_timeout` 太短

默认值是 **20 ms**。

在有 GPU、机器空闲的开发机上没问题。到了工控机、容器里 CPU 吃满的时候：

```
[bt_navigator]: Timed out while waiting for action server to acknowledge goal request
```

然后恢复行为（清代价地图、spin、wait、backup）跑到一半就中止，整个导航以 `ABORTED` 结束。

**这不是规划的错，是"电话打过去对方没接"。**

```yaml
bt_navigator:
  ros__parameters:
    default_server_timeout: 500   # 20 太激进，资源紧张时给到 500 ms
```

现场设备上我一般直接给 500 ms。

---

## 第四步：分清"规划必失败"和"恢复失败"

发一个**地图外**的目标，比如地图是 8×6 m，你发 `(6.0, 0.0)`：

```
[planner_server]: worldToMap failed: mx,my: 204,64, size_x,size_y: 168,128
[planner_server]: The goal sent to the planner is off the global costmap
```

然后行为树开始请求清除全局/局部代价地图，最后 `Goal failed` / `ABORTED`。

**这属于"规划阶段必失败"，不是行为树恢复失效。**

恢复行为（Fallback 分支）只有在**路径能算出来、执行阶段出错**时才真正展开。目标在地图外，第一步就失败了，后面的清除代价地图只是顺带试了一下。

分不清这两者，就会对着一个根本不存在的 bug 调半天。

---

## 一次成功导航长什么样

实测从 `(1.2, 0.0, 90°)` 到 `(1.0, -1.0)`：

```
Goal finished with status: SUCCEEDED
number_of_recoveries: 0
```

8.71 s 完成，最终位姿 `(0.923, -0.767)`，距目标 0.221 m —— 小于默认 `xy_goal_tolerance: 0.25`，所以判成功。

控制指令序列（2 s 采样）：

```
(0.00, 0.16)  →  先原地转向
(0.25, 1.00)  →  加速转向
(0.26, 0.26)  →  贴路径巡航（0.26 是默认 max_vel_x）
(0.00, 0.68)  →  接近终点转向
(0.00, 0.16)  →  对正航向
```

看到这个形状，说明规划、控制、TF、代价地图整条链路都是通的。

---

## 排查顺序（照着走）

1. `ros2 topic list | grep cmd_vel` —— 话题在不在
2. `tf2_echo odom base_link` —— 有没有
3. `tf2_echo map odom` —— **这个断，八成是 `base_frame_id`**
4. 9 个 lifecycle 节点全 active 吗（记得包 `timeout`）
5. `ros2 action list` 有 `navigate_to_pose` 吗
6. 目标点在地图内吗
7. 上面都过了还不行，才去调 `params.yaml`

---

## 一句话总结

- `base_footprint` vs `base_link` —— **Nav2 第一大坑，`map→odom` 不发布基本都是它**
- lifecycle 节点没 active ≠ 启动失败，要逐个查
- `ros2 lifecycle get` 会卡，必须包 `timeout`
- `default_server_timeout: 20` 在工控机上太激进
- 地图外目标失败是规划失败，别当成恢复失效

---

*文中节点状态、动作服务器数量、导航闭环时序与代价地图尺寸均来自 `ros:humble-nav2` 容器实测。真实底盘、真实传感器与现场参数仍需实车验证，调参结论不能直接照搬。*

---

<div style="margin-top:40px;padding:22px 24px;border:1px solid #d0d7de;border-radius:10px;background:#f6f8fa">
<strong>照着查还是不通？</strong><br>
可以把你的 <code>params.yaml</code>、<code>tf2_echo</code> 输出和 lifecycle 状态发过来，我做远程排查。Nav2 / SLAM 调参、MoveIt2 配置、CAN 接入、URDF/TF 排查、国产板卡 arm64 移植，按小时计费，先聊 10 分钟判断能不能做。<br><br>
<a href="https://bietou123.github.io/ros-support/">看服务与报价 →</a>
</div>

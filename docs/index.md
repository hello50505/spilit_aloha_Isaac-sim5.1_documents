---
hide:
  - navigation
  - toc
---

# Mobile ALOHA × Isaac Sim 开发笔记

> 在 **NVIDIA Isaac Sim 5.1** 中搭建 Mobile ALOHA 双臂移动机器人,
> 通过 **ROS2 Humble + MoveIt2 / TRAC-IK** 完成从 URDF 导入、相机
> 集成、双臂遥操作,到自动 **pick & place** 的全过程记录。
>
> 这本"开发日记"的目标读者:**两个月后回来重新跑这套东西的自己**,
> 以及任何想在 Isaac Sim 里复现 Mobile ALOHA 的同好。

---

## 这套笔记是什么

- **不是教科书**。每一篇都按"我当时遇到了什么问题 → 怎么定位 → 怎么改"的顺序写,带原始日志和命令。
- **可复现**。所有命令都可在 `dev_ws/scripts/` 下找到对应脚本,默认参数即可跑通。
- **演进可见**。早期章节走的是 cuRobo,后期切到 MoveIt2 + TRAC-IK,演化思路保留在文档里。

如果你只想跑起来,先看 [**快速启动**](overview/quickstart.md);如果想理解"为什么要这么搭",从 [**项目架构**](overview/architecture.md) 开始。

---

## 推荐阅读路径

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } **第一次接触**

    ---

    1. [项目架构](overview/architecture.md) — 宿主机 Isaac Sim + Docker ROS2 的分工
    2. [快速启动](overview/quickstart.md) — 两条命令把整个栈跑起来
    3. [ROS2 Humble Docker](setup/ros-humble-docker.md) — 第一次构建镜像

-   :material-tools:{ .lg .middle } **想理解某个模块**

    ---

    - 控制链路: [用 ROS2 控制](ros2-integration/control-via-ros2.md) → [末端位姿 IK](ros2-integration/ee-pose-ik.md) → [MoveIt2 + TRAC-IK](ros2-integration/moveit-trac-ik.md)
    - 感知链路: [导入相机](setup/import-camera.md) → [发布图像到 ROS2](ros2-integration/publish-camera-images.md)
    - 调试: [遥操作](debugging/debug-teleop.md) · [RViz 可视化](debugging/visualize-rviz.md)

-   :material-bug:{ .lg .middle } **遇到了问题**

    ---

    跳到 [快速启动](overview/quickstart.md) 第 4 节"常见踩坑"表格,按现象 grep。
    具体细节散在 `调试与可视化` 一章的各篇里。

-   :material-database:{ .lg .middle } **数采与自动化**

    ---

    [自动 pick & place](data-collection/auto-pick-place.md) — RoboTwin 风格
    任务 × 场景 × 机器人 自由组合 + seed 随机化,先打通自动控制再加录制。

</div>

---

## 软硬件版本

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| OS (Host) | Ubuntu 22.04 LTS | Isaac Sim 跑在宿主机 |
| GPU | NVIDIA, 推荐 RTX 30/40 系 | Vulkan + CUDA 必需 |
| NVIDIA Isaac Sim | **5.1.0** standalone | 文档示例针对 5.1.0 |
| ROS2 | **Humble** | 在 Docker 里跑 |
| MoveIt2 | Humble 源码编译 | 含 TRAC-IK kinematics plugin |
| Docker 镜像 | `mobile_aloha_ros2_humble:moveit` | 自构,含 MoveIt2 + TRAC-IK + cuRobo wheel |

> 其他 Ubuntu 版本(例如 24.04)可以通过 Docker 共用同一套 Humble 环境,
> 详见 [项目架构](overview/architecture.md)。

---

## 一句话备忘

```bash
# Terminal A — 宿主机起 Isaac Sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh

# Terminal B — Docker 起 ROS2 控制栈(MoveIt2 + TRAC-IK)
./dev_ws/scripts/run_ros2_humble_docker.sh --build --launch-control
```

剩下的所有参数都在 [快速启动](overview/quickstart.md) 第 3 节里。

---

## 反馈与贡献

发现错别字 / 路径过期 / 想补一篇新坑?欢迎提 Issue 或 PR(GitHub 仓库地址将在代码开源后补上)。

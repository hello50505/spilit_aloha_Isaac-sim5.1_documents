# Table of contents

* [简介](README.md)

## 概览

* [项目架构](docs/overview/architecture.md)
* [快速启动(必读)](docs/overview/quickstart.md)

## 环境与导入

* [ROS2 Humble Docker](docs/setup/ros-humble-docker.md)
* [导入 URDF 到 Isaac Sim](docs/setup/import-urdf.md)
* [导入真实相机模型](docs/setup/import-camera.md)

## ROS2 集成

* [用 ROS2 控制底盘与关节](docs/ros2-integration/control-via-ros2.md)
* [发布相机图像到 ROS2](docs/ros2-integration/publish-camera-images.md)
* [末端位姿 IK 控制](docs/ros2-integration/ee-pose-ik.md)
* [发布真实末端位姿](docs/ros2-integration/publish-real-ee-pose.md)
* [MoveIt2 + TRAC-IK 替换 cuRobo](docs/ros2-integration/moveit-trac-ik.md)

## 调试与可视化

* [红色方块调试 IK](docs/debugging/debug-ik-marker.md)
* [键盘 + SpaceMouse 遥操作](docs/debugging/debug-teleop.md)
* [键盘关节/底盘综合调试](docs/debugging/debug-joint-control.md)
* [在 RViz 中可视化机器人](docs/debugging/visualize-rviz.md)
* [右臂抓瓶碰撞调试](docs/debugging/debug-collision.md)
* [给瓶子和夹爪加摩擦](docs/debugging/add-friction.md)
* [升降台幽灵碰撞排查](docs/debugging/debug-ghost-collision.md)
* [cuRobo IK "无解" 排查](docs/debugging/debug-curobo-ik.md)

## 自动化与数采

* [自动 pick & place](docs/data-collection/auto-pick-place.md)

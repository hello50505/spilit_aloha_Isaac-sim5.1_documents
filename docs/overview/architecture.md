# Isaac Sim + ROS2 Humble 协作架构

## 目标

当前阶段只支持两件事：

- 在 Isaac Sim 中仿真机器人。
- 通过 ROS2 Humble 控制机器人，并接收相机图像。

自动数采、RL、VLA benchmark 先不提前铺目录，等控制链路稳定后再加模块。

## 总体方案

```text
isaac-sim/
  isaac-sim-standalone-5.1.0-linux-x86_64/   # Isaac Sim 本体，宿主机运行
  IsaacSim-ros_workspaces/                   # NVIDIA 官方 ROS workspace
    humble_ws/                               # Isaac Sim 官方 Humble 示例和消息包
  dev_ws/                                    # 团队自己的代码工作区
    ros2_ws/src/                             # 自己写的 ROS2 控制、消息、launch
    isaac/                                   # Isaac 场景、脚本、配置
    configs/                                 # 机器人、相机、ROS2 topic 配置
    source/                                  # 非 ROS package 的 Python 辅助代码
```

## 为什么宿主机跑 Isaac Sim，Docker 跑 ROS2

这是 Isaac Sim + ROS2 开发中很常见的方式，尤其适合你们这种 Ubuntu 22.04
和 Ubuntu 24.04 混合团队。

Isaac Sim GUI 对 NVIDIA 驱动、Vulkan、桌面显示、窗口系统比较敏感，直接在宿主机跑更稳定。
ROS2 的系统依赖、DDS、Python 包和 Ubuntu 版本差异更容易影响协作，放进 Docker 更容易统一。

NVIDIA 官方文档也提供了这种方式：使用 `osrf/ros:humble-desktop` 容器，并通过
`--net=host` 让容器内 ROS2 节点和宿主机 Isaac Sim ROS2 Bridge 通信。

## ROS2 版本

统一使用 ROS2 Humble。

- 你的宿主机是 Ubuntu 22.04，Humble 是原生支持版本。
- 队友即使是 Ubuntu 24.04，也可以通过 Docker 使用同一套 Humble 环境。
- Isaac Sim 5.1 的 ROS2 教程和官方 workspace 包含 `humble_ws`。

## 运行方式

以下命令假设当前仓库根目录是 `isaac-sim/`。如果放在其他位置，先进入自己的项目根目录：

```bash
cd <your-isaac-sim-project>
```

启动 Isaac Sim 推荐使用项目脚本：

```bash
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

这个脚本会先清理容易污染 Isaac Sim standalone 自带 Python/CUDA/PyTorch 的环境变量，
再只注入 Isaac ROS2 Bridge 需要的路径，并通过 `--exec` 自动启动底盘运动学控制器。
因此即使当前终端还带着 Conda 前缀，通常也不需要手动退出环境。

脚本内部会清理：

```bash
unset PYTHONPATH
unset PYTHONHOME
unset LD_LIBRARY_PATH
unset CUDA_HOME
unset CUDA_PATH
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV
```

如果不用项目脚本、而是手动启动 Isaac Sim，仍建议使用一次性干净环境：

```bash
env \
  -u LD_LIBRARY_PATH \
  -u PYTHONPATH \
  -u PYTHONHOME \
  -u CONDA_PREFIX \
  -u CONDA_DEFAULT_ENV \
  -u CUDA_HOME \
  -u CUDA_PATH \
  ./isaac-sim-standalone-5.1.0-linux-x86_64/isaac-sim.sh --no-ros-env
```

初始化 NVIDIA 官方 ROS workspace：

```bash
cd IsaacSim-ros_workspaces
git submodule update --init --recursive
```

进入 ROS2 Humble Docker。推荐先使用 `osrf/ros:humble-desktop` 配好环境，确认构建成功后再保存为自己的本地镜像：

```bash
xhost +

docker run -it --rm --net=host \
  --env="DISPLAY" \
  --env="ROS_DOMAIN_ID" \
  -v "$(pwd)/humble_ws:/humble_ws" \
  --name ros_ws_docker \
  osrf/ros:humble-desktop /bin/bash
```

容器里安装依赖并编译官方 `humble_ws`：

```bash
cd /humble_ws
apt update
apt install -y ros-humble-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
rosdep install --from-paths src --ignore-src --rosdistro=$ROS_DISTRO -y
source /opt/ros/$ROS_DISTRO/setup.sh
colcon build
source install/local_setup.bash
```

环境确认可用后，可以在宿主机另开一个终端保存当前容器：

```bash
docker commit ros_ws_docker isaac-ros-humble:ready
```

后续开发优先使用保存好的镜像：

```bash
cd <your-isaac-sim-project>/IsaacSim-ros_workspaces
xhost +

docker run -it --rm --net=host \
  --env="DISPLAY" \
  --env="ROS_DOMAIN_ID" \
  -v "$(pwd)/humble_ws:/humble_ws" \
  --name ros_ws_docker \
  isaac-ros-humble:ready /bin/bash
```

每次进入容器后初始化 ROS 环境：

```bash
cd /humble_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

## 通信约定

宿主机 Isaac Sim 和 Docker 内 ROS2 节点需要保持一致：

```bash
export ROS_DOMAIN_ID=0
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Docker 启动命令会把宿主机的 `ROS_DOMAIN_ID` 传入容器：

```text
--env="ROS_DOMAIN_ID"
```

因此如果需要隔离不同用户或不同实验，先在宿主机设置相同的 `ROS_DOMAIN_ID`，再启动 Isaac Sim 和 Docker 容器。

## 当前 dev_ws 最小结构

```text
dev_ws/
  README.md
  .gitignore
  docs/
  scripts/
    run_ros2_humble_docker.sh
  ros2_ws/
    src/
  isaac/
    scripts/
    scenes/
    usd/
    configs/
  configs/
    robots/
    sensors/
    ros2/
    tasks/
  source/
    sim/
    control/
    common/
```

## 后续扩展

等 ROS2 控制和相机图像链路稳定后，再按需要增加：

- `source/data_collection/`：自动数采、rosbag、HDF5、LeRobot 转换。
- `source/rl/`：RL 环境封装、训练和评估。
- `source/vla/`：VLA policy 接入、episode runner。
- `source/benchmark/`：统一 benchmark runner、指标统计和报告。

这些目录不提前创建，避免当前工作区过重。

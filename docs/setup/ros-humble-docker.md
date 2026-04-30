# ROS 2 Humble Docker 使用说明

本文记录在 Isaac Sim ROS 工作区中使用 `osrf/ros:humble-desktop` 容器的基本步骤。

以下命令默认在宿主机的 `IsaacSim-ros_workspaces` 目录下执行。

## 1. 在宿主机初始化子模块

`humble_ws` 是仓库中的子目录，不是 Git 仓库根目录。因此 `git submodule update` 应该在宿主机的 `IsaacSim-ros_workspaces` 目录下执行：

```bash
cd <IsaacSim-ros_workspaces>
git submodule update --init --recursive
```

## 2. 启动 Humble Docker 容器

先进入宿主机的 `IsaacSim-ros_workspaces` 目录：

```bash
cd <IsaacSim-ros_workspaces>
```

启动容器时，把当前目录下的 `humble_ws` 挂载到容器内的 `/humble_ws`：

```bash
xhost +

docker run -it --rm --net=host \
  --env="DISPLAY" \
  --env="ROS_DOMAIN_ID" \
  -v "$(pwd)/humble_ws:/humble_ws" \
  --name ros_ws_docker \
  osrf/ros:humble-desktop /bin/bash
```

注意不要使用下面这个路径：

```bash
~/IsaacSim-ros_workspaces/humble_ws
```

因为 `~` 只会展开为当前用户的 home 目录。如果仓库不在 home 目录直属位置，容器中的 `/humble_ws` 就会看不到正确的 `src` 目录。

## 3. 在容器内确认挂载成功

进入容器后执行：

```bash
cd /humble_ws
ls
ls src
```

如果 `src` 能正常列出，说明挂载路径正确。

## 4. 安装 Cyclone DDS RMW

在容器内执行：

```bash
apt update
apt install -y ros-humble-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

也可以使用环境变量写法：

```bash
apt install -y ros-${ROS_DISTRO}-rmw-cyclonedds-cpp
```

## 5. 安装依赖并构建工作区

在容器内继续执行：

```bash
cd /humble_ws
rosdep install --from-paths src --ignore-src --rosdistro=$ROS_DISTRO -y
source /opt/ros/$ROS_DISTRO/setup.sh
colcon build
source install/local_setup.bash
```

## 6. 保存当前已经配置好的容器

如果当前容器还在运行，并且容器名是 `ros_ws_docker`，可以在宿主机另开一个终端执行：

```bash
docker commit ros_ws_docker isaac-ros-humble:ready
```

这样会把当前容器中已经安装好的 ROS 依赖、Cyclone DDS 等环境保存成一个本地镜像：

```text
isaac-ros-humble:ready
```

之后就不用每次重新安装 `ros-humble-rmw-cyclonedds-cpp` 等依赖。

## 7. 后续使用保存好的镜像

以后使用保存好的镜像启动：

```bash
xhost +

docker run -it --rm --net=host \
  --env="DISPLAY" \
  --env="ROS_DOMAIN_ID" \
  -v "$(pwd)/humble_ws:/humble_ws" \
  --name ros_ws_docker \
  isaac-ros-humble:ready /bin/bash
```

这里的挂载关系是：

```text
宿主机: <IsaacSim-ros_workspaces>/humble_ws
容器内: /humble_ws
```

代码保存在宿主机，容器只提供 ROS 2 Humble 编译和运行环境。

## 8. 每次进入容器后的初始化

进入容器后执行：

```bash
cd /humble_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

之后就可以使用 ROS 2 命令：

```bash
ros2 pkg list
ros2 topic list
ros2 node list
```

## 9. 修改代码后的重新构建

如果修改了 `humble_ws/src` 下的代码，在容器内重新构建：

```bash
cd /humble_ws
source /opt/ros/humble/setup.bash
colcon build
source install/local_setup.bash
```

如果只想构建某一个包，可以使用：

```bash
colcon build --packages-select <package_name>
source install/local_setup.bash
```

## 10. 和 Isaac Sim 联调

推荐使用 Cyclone DDS：

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

如果需要固定 ROS Domain：

```bash
export ROS_DOMAIN_ID=<domain_id>
```

宿主机启动 Docker 时已经传入了：

```bash
--env="ROS_DOMAIN_ID"
```

所以如果宿主机设置了 `ROS_DOMAIN_ID`，容器内会继承同样的值。

## 11. 常用检查命令

检查工作区是否挂载正确：

```bash
cd /humble_ws
ls
ls src
```

检查 ROS 环境：

```bash
echo $ROS_DISTRO
echo $RMW_IMPLEMENTATION
ros2 pkg list | head
```

查看本地镜像：

```bash
docker images | grep isaac-ros-humble
```

查看正在运行的容器：

```bash
docker ps
```

## 常见问题

如果出现：

```text
given path 'src' does not exist
```

通常说明 Docker 挂载路径不对，容器里的 `/humble_ws` 不是宿主机项目中的真实 `humble_ws`。

如果出现：

```text
fatal: not a git repository
```

通常是因为在容器内的 `/humble_ws` 执行了 `git submodule update`。请回到宿主机的仓库根目录执行该命令。

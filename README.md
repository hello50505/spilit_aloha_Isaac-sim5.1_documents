# Mobile ALOHA × Isaac Sim 开发笔记

> 在 **NVIDIA Isaac Sim 5.1** 中搭建 Mobile ALOHA 双臂移动机器人,
> 通过 **ROS2 Humble + MoveIt2 / TRAC-IK** 完成从 URDF 导入、相机
> 集成、双臂遥操作,到自动 **pick & place** 的全过程记录。

这本"开发日记"的目标读者:**两个月后回来重新跑这套东西的自己**,
以及任何想在 Isaac Sim 里复现 Mobile ALOHA 的同好。

> 在线阅读(GitBook): _等仓库连接 GitBook 后填入访问 URL_
>
> GitHub 仓库: <https://github.com/hello50505/spilit_aloha_Isaac-sim5.1_documents>

---

## 这套笔记是什么

- **不是教科书**。每一篇都按"我当时遇到了什么问题 → 怎么定位 → 怎么改"的顺序写,带原始日志和命令。
- **可复现**。所有命令都可在项目主仓的 `dev_ws/scripts/` 下找到对应脚本,默认参数即可跑通。
- **演进可见**。早期章节走的是 cuRobo,后期切到 MoveIt2 + TRAC-IK,演化思路保留在文档里。

如果你只想跑起来,先看 [**快速启动**](docs/overview/quickstart.md);
如果想理解"为什么要这么搭",从 [**项目架构**](docs/overview/architecture.md) 开始。

---

## 推荐阅读路径

| 你的目标 | 起手三篇 |
| --- | --- |
| 第一次接触 | [项目架构](docs/overview/architecture.md) → [快速启动](docs/overview/quickstart.md) → [ROS2 Humble Docker](docs/setup/ros-humble-docker.md) |
| 搞懂控制链路 | [用 ROS2 控制](docs/ros2-integration/control-via-ros2.md) → [末端位姿 IK](docs/ros2-integration/ee-pose-ik.md) → [MoveIt2 + TRAC-IK](docs/ros2-integration/moveit-trac-ik.md) |
| 搞懂感知链路 | [导入相机](docs/setup/import-camera.md) → [发布图像到 ROS2](docs/ros2-integration/publish-camera-images.md) |
| 调试中遇到问题 | [快速启动 §4 常见踩坑](docs/overview/quickstart.md) → 调试章节按现象 grep |
| 数采与自动化 | [自动 pick & place](docs/data-collection/auto-pick-place.md) |

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
> 详见 [项目架构](docs/overview/architecture.md)。

---

## 一句话备忘

```bash
# Terminal A — 宿主机起 Isaac Sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh

# Terminal B — Docker 起 ROS2 控制栈(MoveIt2 + TRAC-IK)
./dev_ws/scripts/run_ros2_humble_docker.sh --build --launch-control
```

剩下的所有参数都在 [快速启动](docs/overview/quickstart.md) 第 3 节里。

---

## 仓库结构

```text
.
├── README.md                       本仓库 / GitBook 首页(就是你正在看的这一篇)
├── SUMMARY.md                      GitBook 目录(章节顺序 / 分组)
├── docs/                           文档源文件
│   ├── overview/                   概览(架构、快速启动)
│   ├── setup/                      环境与导入(Docker、URDF、相机)
│   ├── ros2-integration/           ROS2 集成(控制、图像、IK、MoveIt)
│   ├── debugging/                  调试与可视化
│   ├── data-collection/            自动化与数采
│   └── stylesheets/extra.css       MkDocs 模式下的中文排版优化
├── mkdocs.yml                      可选:本地用 MkDocs Material 预览
├── requirements.txt                可选:MkDocs 依赖
└── .gitignore
```

> 这个仓库**只装文档**,不包含项目源码。源码仓将在公开后另行发布,
> 文档里凡是 `dev_ws/...` 这种相对路径都是项目源码仓的内容。

---

## 在 GitBook 上托管(已选定方案)

主线方案是把这个仓库连接到 [GitBook](https://gitbook.com),它会自动:

1. 拉取仓库 markdown 内容。
2. 按 [`SUMMARY.md`](SUMMARY.md) 渲染左侧目录。
3. 提供搜索、暗色模式、版本管理、自定义域名等。

操作步骤:

1. 登录 GitBook → 新建一个 Space。
2. 在 Space 设置里选 **Sync with Git → GitHub**,授权 GitBook 读取
   `hello50505/spilit_aloha_Isaac-sim5.1_documents`。
3. 选择 `main` 分支,**Project directory 留空**(SUMMARY.md 在根目录)。
4. 等首次同步完成,GitBook 会发回一个公开链接。把它填到上面"在线阅读"那一行。

---

## 可选:本地用 MkDocs 预览

如果你不想等 GitBook 同步、就在本地看效果,仓库里也保留了一份完整的
**MkDocs Material** 配置(纯 Python,跟 GitBook 互不打扰)。

```bash
# 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate

# 装依赖
pip install -r requirements.txt

# 起本地服务,改文件会热重载
mkdocs serve              # http://127.0.0.1:8000

# 一次性构建静态站
mkdocs build --strict     # 产物在 ./site/
```

`mkdocs.yml` 里的导航跟 [`SUMMARY.md`](SUMMARY.md) 保持同步——以后加新文档,
两边都改一行即可。

---

## 反馈与贡献

发现错别字 / 路径过期 / 想补一篇新坑?直接提 [Issue](https://github.com/hello50505/spilit_aloha_Isaac-sim5.1_documents/issues)
或 PR。

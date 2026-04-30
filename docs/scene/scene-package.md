# Aloha in Blue Grid (ROS) — Isaac Sim 场景包

本压缩包包含一个可在 **NVIDIA Isaac Sim** 中直接打开的 Aloha 双臂移动机器人场景(蓝色 Grid 地板 + ROS 集成)。

---

## 1. 解压

将 `aloha_in_blue_grid_ROS_package.zip` 解压到任意目录,会得到如下结构(**请勿改动相对路径**,否则资产引用会断):

```
<解压目录>/
├── README.md
├── assets/
│   └── 6_aloha_in_blue_grid_ROS.usda          ← 主场景文件,从这个打开
└── mobile_aloha_sim/
    └── split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material/
        ├── split_aloha_rslidar_with_piper_isaac_material.usda
        └── configuration/
            ├── split_aloha_rslidar_with_piper_isaac_material_base.usda     (机器人 mesh/材质)
            ├── split_aloha_rslidar_with_piper_isaac_material_physics.usda  (碰撞/物理)
            ├── split_aloha_rslidar_with_piper_isaac_material_robot.usda    (关节驱动)
            └── split_aloha_rslidar_with_piper_isaac_material_sensor.usda   (相机等传感器)
```

---

## 2. 打开

1. 启动 Isaac Sim(推荐 **5.1 或更新版本**)。
2. `File → Open` → 选择 `assets/6_aloha_in_blue_grid_ROS.usda`。
3. 第一次打开会从 Omniverse 在线下载 Grid 地板环境(`default_environment.usd`),需要联网。
4. 加载完成后即可看到带双臂、升降柱、激光雷达和相机的 Aloha 机器人。

---

## 3. 外部依赖(已自动处理,无需打包)

- **`OmniPBR.mdl` / `OmniPBR_Opacity.mdl`** — Isaac Sim 自带材质,本地解析。
- **`default_environment.usd`** — Omniverse 公网资源,Isaac Sim 启动时自动下载:
  `https://omniverse-content-production.s3-us-west-2.amazonaws.com/Assets/Isaac/5.1/Isaac/Environments/Grid/default_environment.usd`

---

## 4. 注意事项

- 主场景使用相对路径 `../mobile_aloha_sim/...` 引用机器人 USD,**`assets/` 与 `mobile_aloha_sim/` 必须保持同级目录关系**。
- 包内 USD 全部为 ASCII 文本(`.usda`)以便可读和压缩;Isaac Sim 可直接加载。
- 若需启用 ROS2 桥接、关节控制、IK 等功能,请确认已安装 Isaac Sim ROS2 扩展(参考原仓库的 `1_ROS_HUMBLE_DOCKER_NOTES.md`、`4_control_ROS.md`)。

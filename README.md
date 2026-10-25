# vre

手机拍摄视频 → 单房间三维重建，本地运行，多方案对比评测。

## 状态

| 阶段 | 内容 | 状态 |
|---|---|---|
| 0 | 仓库初始化 | done |
| 1 | 环境搭建（Windows 裸机 RTX 5070） | todo |
| 2 | 数据准备（抽帧 + COLMAP） | todo |
| 3 | 方案 A：Nerfstudio `splatfacto` | todo |
| 4 | 方案 C/D：原版 3DGS、自写 gsplat | todo |
| 5 | 方案 E/F：`nerfacto`、Mesh | todo |
| 6 | 统一评测与结果汇总 | todo |

## 硬件与环境

| 项 | 值 |
|---|---|
| GPU | RTX 5070 (12 GB, sm_120 / Blackwell) |
| CPU / RAM | Intel Core Ultra 7 265 (20C) / 64 GB |
| OS | Windows 11 Pro（原生 + WSL2 Ubuntu 26.04） |
| Python | 3.12.10（Windows 原生） |

## 方案对比矩阵

详见 [`docs/research.md`](docs/research.md)。

| # | 方案 | 层级 | 说明 |
|---|---|---|---|
| A | Nerfstudio `splatfacto` | Tier 1 | GS，官方最佳实践封装，最先跑通 |
| B | Postshot (Jawset) | 参考 | 商业 GUI，闭源，仅作上限参考 |
| C | COLMAP + 原版 3DGS (Inria) | Tier 2 | 学术基线，可全参数调优 |
| D | COLMAP/GLOMAP + 自写 gsplat | Tier 2 | 最灵活，可做策略实验 |
| E | Nerfstudio `nerfacto` | Tier 3 | NeRF 对照组 |
| F | COLMAP + OpenMVS/Meshroom | Tier 3 | Mesh 对照组 |

## 目录结构

```
docs/       调研、环境搭建记录、评测结果
scripts/    各阶段可执行脚本（抽帧、COLMAP、训练、渲染、评测）
configs/    各方案的训练配置
data/raw/   原始视频（已 gitignore）
```

## 复现

```bash
# TODO: 阶段 1 完成后补充
```

## 输入素材

`data/raw/room.mp4` — 1920×1080, 30 fps, 67 s, 1966 帧, 163 MB（不入库）。

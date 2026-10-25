# 方案调研

## 1. 问题定义

输入：手机拍摄的单房间环视视频（1920×1080, 30 fps, 67 s, 1966 帧）。
输出：可自由漫游查看的 3D 场景。
约束：必须完全本地运行；最佳设备为 RTX 5070 (12 GB) + 64 GB RAM 的 Windows 主机。

用途定位为「仅 3D 可视化 / 漫游查看」，因此评价重点是渲染观感与实时性，而非可编辑网格或尺寸精度。

## 2. 技术原理

### 2.1 共同的第一步：SfM（Structure from Motion）

从视频帧提取特征点并做跨帧匹配，解算每帧相机位姿与稀疏点云。

- **COLMAP**：工业标准实现，增量式 SfM，支持 Windows 预编译包。
- **GLOMAP**：COLMAP 团队出的全局式 SfM，快一到两个数量级，作 COLMAP 的加速替代。

这一步是几乎所有后续方案的输入前提。

### 2.2 NeRF（隐式神经表示）

用 MLP 拟合「任意位置 + 任意视角方向 → 颜色 + 密度」，再沿光线积分渲染。

- 优点：视图一致性极好，遮挡/反射处理自然。
- 缺点：渲染慢、表面偏模糊、训练与推理都吃算力。
- 本项目的对照方案 E。

### 2.3 3D Gaussian Splatting（GS，高斯泼溅）

用几十万带各向异性协方差与球谐颜色系数的高斯球作为显式场景表示，走 CUDA 光栅化实时渲染。

- 优点：实时帧率、画面接近实拍、训练分钟级到小时级。
- 缺点：显存随高斯数量增长、大场景易出现漂浮伪影、导出体积大。
- 本项目主方案 A / C / D。

### 2.4 摄影测量 Mesh

稠密重建点云后做泊松/ Delaunay 网格化并贴纹理。

- 优点：几何准确、可编辑、易于下游建模。
- 缺点：玻璃/白墙/反光面重建失败，观感偏「硬」。
- 本项目对照方案 F。

## 3. 各方案优劣

### A. Nerfstudio `splatfacto`

| 维度 | 评价 |
|---|---|
| 质量 | 高。集成 absgrad densify、camera optimizer、可选 bilateral grid |
| 易用性 | 最高。`ns-process-data` → `ns-train` → `ns-viewer` → `ns-export` 一条龙 |
| 生态 | Apache-2.0，活跃；内置 web viewer (viset) 与多种导出 |
| 风险 | `pyproject.toml` 硬锁 `gsplat==1.4.0`；已知 TCNN 为可选加速项，与本方案无关 |

结论：**首选基线**。

### B. Postshot (Jawset)

| 维度 | 评价 |
|---|---|
| 质量 | 高，训练速度同类第一梯队 |
| 易用性 | 最高，拖拽视频即用，自带查看器 |
| 风险 | 闭源商业软件；免费版导出受限；无法代码化、无法纳入自动评测 |

结论：仅作「商业上限」参考，不进自动对比矩阵。

### C. 原版 3DGS (Inria `gaussian-splatting`)

| 维度 | 评价 |
|---|---|
| 质量 | 学术基准，参数全可调 |
| 易用性 | 低。依赖 submodule、手写编译、COLMAP 目录结构约定 |
| 风险 | 代码停更于 2024-10；Windows + sm_120 需手动设置 `TORCH_CUDA_ARCH_LIST=12.0` |

结论：Tier 2，作为「学术原版」对照。

### D. COLMAP/GLOMAP + 自写 gsplat 训练

| 维度 | 评价 |
|---|---|
| 质量 | 取决于调参，上限最高（可试 MCMC 策略、不同初始化） |
| 易用性 | 中。需要自己写训练循环与 densify 逻辑 |
| 风险 | 需自建评测与导出流程 |

结论：Tier 2，用于验证「官方封装是否已经够好」。

### E. Nerfstudio `nerfacto`

| 维度 | 评价 |
|---|---|
| 质量 | 中高，室内远景易糊 |
| 易用性 | 高，但 `HashEncoding` 默认走 TCNN，缺失时降级为纯 PyTorch，慢 5–10 倍 |
| 风险 | TCNN 在 Windows + sm_120 上编译困难；不装也能跑，仅速度退化 |

结论：Tier 3，仅作「非 GS」质量对照，不做耗时对比。

### F. COLMAP + OpenMVS / Meshroom

| 维度 | 评价 |
|---|---|
| 质量 | 纹理好，几何有大洞（玻璃、纯色墙） |
| 易用性 | 中，GUI 或 CLI |
| 风险 | 与漫游需求不完全匹配（非实时体积渲染） |

结论：Tier 3，用于说明「为什么不适合漫游」。

## 4. 已核实的关键依赖事实

| 事实 | 影响 |
|---|---|
| `splatfacto` 只依赖 `gsplat` / `torch` / `pytorch_msssim`，**不依赖 tiny-cuda-nn** | 裸机无需为 GS 编译 TCNN |
| `nerfstudio` 对 TCNN 是软导入（`nerfstudio/utils/external.py` 捕获 `ModuleNotFoundError`） | 缺 TCNN 不会导致安装/运行失败 |
| PyPI 版 `gsplat` 是 `py3-none-any` 通用包 | 首次安装会针对当前 torch 的 CUDA 架构现编，**天然支持 sm_120** |
| 预编译 wheel 索引 `docs.gsplat.studio/whl` 最高仅到 `cu124` | 不要走预编译路线，否则缺 sm_120 kernel |
| `torch` cu128 有 `cp312 win_amd64` 官方轮子（2.7.0 起） | 与机器上的 Python 3.12.10 匹配 |
| COLMAP 4.2.1 / GLOMAP 1.2.0 均提供 `x64-windows-cuda` 预编译包 | SfM 阶段无需自己编译 |

## 5. 排除的方案

| 方案 | 排除理由 |
|---|---|
| Luma / Polycam / RealityCapture 云 | 违反「必须全本地」 |
| Agisoft Metashape | 付费，代码化价值低 |
| NeRFStudio `splatfacto-w` | 面向大场景户外，本场景收益有限，后续可试 |
| 单目深度估计（DepthAnything 等）辅助 | 不属于重建方案本体，留作后续优化 |

## 6. 评测设计

为保证金对比有效，所有方案共享完全相同的输入与预算：

1. **统一帧集**：`ffmpeg -vf fps≈4.5` 抽约 300 帧，JPEG q95。
2. **统一位姿**：全部方案复用同一份 COLMAP 输出。
3. **统一训练预算**：30k iter，`downscale-factor=2`（OOM 再降到 4）。
4. **统一测试集**：约 1/8 帧留作 eval。
5. **统一指标**：
   - 质量：PSNR / SSIM / LPIPS
   - 性能：训练墙钟时间、显存峰值、输出 `.ply` 体积
   - 主观：viewer 漫游评分（1–5）
6. **统一相机轨迹**：同一路径渲染 10 s 漫游视频，便于肉眼对比。

## 7. 待确认

- 评测主环境：Windows 裸机 vs WSL2
- 是否补齐编译工具链（VS Build Tools + CUDA Toolkit 12.8，约 10 GB）
- 是否执行方案 B
- 最终对比矩阵规模（3 / 5 / 6 个方案）

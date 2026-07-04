# plaque-morph-radio

颅内动脉粥样硬化斑块**形态学特征**与**影像组学特征**提取工具集，用于复发/非复发等临床分组研究。

本仓库包含 3 个 Jupyter Notebook，分别对应：二维形态指标流水线、三维 PyRadiomics 特征提取、对侧参考层提取与 RR（Remodeling Ratio）指数计算。

> **图像类型说明**：本工具链处理的均为沿血管**中心线**重新采样/正交重建得到的图像（centerline-resliced images），即以血管中心线为基准、垂直于管腔走向的逐层切面。所有面积、直径、厚度、体积等二维/三维指标均基于这种中心线切面计算。

## 环境依赖

```bash
pip install -r requirements.txt
```

主要依赖：`SimpleITK`、`pyradiomics`、`opencv-python`、`scikit-learn`、`pandas`、`openpyxl`。

## 路径约定

为避免泄露本地绝对路径，Notebook 中所有数据/输出路径均以占位符 `PROJECT_ROOT` 表示，例如：

```python
lumen_root   = "PROJECT_ROOT/lumen"
wall_root    = "PROJECT_ROOT/wall"
output_excel = "PROJECT_ROOT/feature_excel/1.xlsx"
```

运行前请将 `PROJECT_ROOT` 替换为你本地的数据根目录（或定义 `PROJECT_ROOT = "/path/to/your/data"` 后用 `os.path.join` 拼接）。

## 数据目录结构（输入格式）

运行前请按下列约定组织数据，并将 `PROJECT_ROOT` 替换为本地实际路径。

### 通用约定

| 项目 | 说明 |
|------|------|
| 图像格式 | MetaImage（`.mhd` + 同名 `.raw`） |
| 体素间距 | 平面 0.1 × 0.1 mm，层间距 0.5 mm（部分脚本写死） |
| Mask 标签 | `1` = 管腔（lumen），`2` = 管壁（wall），`0` = 背景 |
| 图像类型 | 沿血管中心线正交重建的逐层切面（centerline-resliced） |
| 病例 ID | 推荐 `姓名_数字ID`（空格替换为 `_`） |

### Notebook 01 所需目录

```
project_root/
├── lumen/                          # 逐层管腔 mask
│   └── {case_id}/
│       └── *_slice_{N}_lumen.mhd
├── wall/                           # 逐层管壁 mask
│   └── {case_id}/
│       └── *_slice_{N}_wall.mhd
├── T1_slice/                       # 逐层 T1 原图
│   └── {case_id}/
│       └── *_slice_{N}_*.mhd
├── T1_MinLumen_slice/              # 最小管腔层面（由本 Notebook 生成）
├── reference/                      # 对侧参考层（远端/近端）
│   └── {case_id}/
│       ├── distal_lumen.mhd / distal_wall.mhd
│       └── proximal_lumen.mhd / proximal_wall.mhd
├── T1CE_slice/                     # T1CE 增强图像（强化比例模块）
├── T1_origin_slice/                # 与 T1CE 配对的 T1 原图
├── wall_t1ce/                      # T1CE 管壁 mask（需二值化）
└── feature_excel/                  # Excel 输出目录
```

**最小管腔切片选取逻辑**：遍历 `lumen/` 与 `wall/` 中所有切片，取管腔面积最小者；若面积相同则选管壁面积最大者。同步复制对应 T1 原图至 `T1_MinLumen_slice/`。

### Notebook 02 所需目录

```
project_root/
└── 复发分类/                        # 或自定义分组根目录
    ├── 复发/两端无狭窄/
    ├── 复发/一端狭窄/
    ├── 非复发/两端无狭窄/
    └── 非复发/一端狭窄/
        └── {姓名 ID (起始层-结束层)}/   # 例：zhangsan_1234567890 (10-45)
            ├── *_img.mhd               # T1 图像（不含 T1CE）
            └── *_contour_slice_prediction.mhd   # 分割 mask
```

文件夹名需匹配正则：`(.+?)\s*(\d{6,12})\s*[（(](\d+)\s*-\s*(\d+)[）)]`。

### Notebook 03 所需目录

```
project_root/
├── 对侧血管补充/                    # 对侧参考层原始 3D 数据
│   └── {姓名_ID (远端/近端参考层为N)}/  # 例：lisi_9876543210（远端参考层为12）
│       ├── *_img.mhd
│       └── *_contour_slice_prediction.mhd
├── reference/                      # 由 Cell 1 生成的单层参考 mask
├── lumen/                          # 全段逐层管腔 mask
├── wall/                           # 全段逐层管壁 mask
├── T1_MinLumen_slice/              # 病变处（最小管腔）单层 mask
└── feature_excel/
```

---

## Notebook 说明

| 文件 | 原文件名 | 功能 |
|------|----------|------|
| `notebooks/01_morphology_feature_extraction.ipynb` | `main.ipynb` | 斑块二维/逐层形态学特征全流程 |
| `notebooks/02_radiomics_3d_extraction.ipynb` | `main2.ipynb` | 3D PyRadiomics 影像组学特征 |
| `notebooks/03_reference_extraction_and_rr_index.ipynb` | `main3.ipynb` | 对侧参考层提取 + RR 指数 |

---

## 01 — 形态学特征提取

### 输出文件一览

| 输出文件 | 计算层面 | 主要指标 |
|----------|----------|----------|
| `T1_MinLumen_slice/{case_id}/` | 最小管腔单层 | 复制 `*_lumen.mhd`、`*_wall.mhd` 及对应 T1 原图 |
| `feature_excel/1.xlsx` | 最小管腔单层 | Case ID, Slice Index, Lumen/Wall/Vessel Area (mm²), Normalized Wall Index (NWI) |
| `feature_excel/2.xlsx` | 最小管腔单层 | ID, lumen_diameter_max/min/avg_slice (mm) |
| `feature_excel/3.xlsx` | 最小管腔单层 | ID, wall_diameter_max/min/avg_slice (mm) |
| `feature_excel/4.xlsx` | 最小管腔单层 | ID, wall_thickness_max/min/avg_slice (mm), eccentricity_index |
| `feature_excel/5.xlsx` | 最小管腔 vs 参考层 | ID, Remodeling_Index_distal, Remodeling_Index_proximal, Stenosis_Rate |
| `feature_excel/whole_remodeling_index.xlsx` | 全段 vs 参考层 | ID, Whole_Remodeling_Index_distal, Whole_Remodeling_Index_proximal |
| `feature_excel/6.xlsx` | 全段管腔 | ID, Slices, lumen_Diameter_max/min/mean_Total (mm) |
| `feature_excel/7.xlsx` | 全段管壁 | ID, Slices, wall_Diameter_max/min/mean_Total (mm) |
| `feature_excel/8.xlsx` | 全段管壁 | ID, Slices, wall_thickness_max/min/avg_total (mm) |
| `feature_excel/9.xlsx` | 全段逐层汇总 | Case ID, Total Slices, 各面积/NWI 的 Max/Min/Mean |
| `feature_excel/10.xlsx` | 全段体积 | ID, Slices, Lumen/Wall/Vessel Volume (mm³), Plaque_Length (mm) |
| `feature_excel/11.xlsx` | T1CE 强化 | patient_id, T1CE_value, T1_value, Enhancement_Ratio, Enhancement(%) |

### 指标定义摘要

- **NWI（Normalized Wall Index）**：`管壁面积 / (管腔面积 + 管壁面积)`
- **Remodeling Index (RI)**：`病变处血管总面积 / 参考层血管总面积`（远端、近端分别计算）
- **Stenosis Rate（狭窄率）**：`1 - 病变管腔面积 / 参考管腔面积`（优先使用近端+远端均值）
- **Eccentricity Index（偏心率）**：`(最大管壁厚度 - 最小管壁厚度) / 最大管壁厚度`
- **Enhancement Ratio**：`(T1CE_max - T1) / T1`（在 T1CE 最大强化切片上计算）

直径/厚度通过 mask 轮廓极坐标采样得到；体积 = 各层面积 × 0.5 mm 层厚。

---

## 02 — 3D 影像组学特征

### 输入

各分组文件夹下的 3D T1 图像与分割 mask（见上文目录结构）。

> **是否使用 Mask**：是。PyRadiomics 通过 `extractor.execute(imagePath, maskPath, label=...)` 同时读取图像与对应 mask，仅在与 mask 标签区域对应的体素上计算特征。Mask 标签优先取 `2`（管壁），否则回退到 `1`（管腔）；未标注区域（`0`）被排除。

### 提取设置

| 参数 | 值 |
|------|-----|
| 重采样间距 | 0.5 × 0.5 × 0.5 mm |
| 灰度 binWidth | 25 |
| LoG sigma | 1.0, 3.0 |
| 图像滤波 | Original, Wavelet, LoG, Square, Squareroot, Logarithm, Exponential, Gradient |
| 特征类 | firstorder, shape (17项), glcm, glrlm, glszm, gldm |
| Mask 标签 | 优先 label=2（管壁），否则 label=1 |

### 输出

`radiomics_features_3D_all_groups.xlsx`（路径可在 Notebook 内修改）

| Sheet | 内容 |
|-------|------|
| 影像组学特征 | patient_id, group, start/end_slice, total_slices, label_used + ~1427 个 PyRadiomics 特征 |
| 患者信息 | patient_id, group, original_folder, 切片范围, labels_found |

---

## 03 — 对侧参考层与 RR 指数

### Part A：参考层提取（Cell 1）

从 `对侧血管补充/` 中按文件夹名解析远端/近端参考层编号，提取对应单层 T1 与 lumen/wall mask，保存至 `reference/{case_id}/`：

```
distal_lumen.mhd, distal_wall.mhd, distal_t1.mhd
proximal_lumen.mhd, proximal_wall.mhd, proximal_t1.mhd
```

### Part B：RR 指数（Cell 2，论文标准）

| 指标 | 说明 |
|------|------|
| **RR** | `OWA_lesion / Expected_OWA`，其中 Expected_OWA = OWA_ref + S × D |
| **OWA** | 管腔面积 + 管壁面积（Outer Wall Area） |
| **S** | 全段管腔面积对 z 的线性回归斜率（锥度修正） |
| **D** | `(病变层索引 - 参考层索引) × 0.5` mm |
| **参考层** | 全段管壁面积最小的切片 |
| **病变层** | `T1_MinLumen_slice/` 中的最小管腔切片 |
| **Category** | RR > 1.05 → Positive；RR < 0.95 → Negative；否则 Intermediate |

### 输出

`feature_excel/RR_Paper_Standard.xlsx`

| 列名 | 含义 |
|------|------|
| ID | 病例 ID |
| RR | 重构比 |
| Category | Positive / Negative / Intermediate |
| Lesion_Slice | 病变层索引 |
| Ref_Slice(Thinnest_Wall) | 参考层索引 |
| WA_Ref | 参考层管壁面积 |
| S_Slope | 锥度斜率 |
| Distance_D(mm) | 病变-参考距离 |
| OWA_Lesion | 病变处 OWA |
| Expected_OWA | 预期 OWA |

---

## 推荐使用顺序

```
03 (Part A)  →  01  →  03 (Part B)
                  ↘
                   02（独立 3D 组学，可并行）
```

1. 运行 **03 Part A**，生成 `reference/` 对侧参考层；
2. 运行 **01**，从逐层 mask 提取形态特征并生成 `T1_MinLumen_slice/`；
3. 运行 **03 Part B**，计算 RR 指数；
4. 若有 3D 分组数据，运行 **02** 提取影像组学特征。

---

## 引用

形态与 RR 相关方法请参考团队发表论文；影像组学部分基于 [PyRadiomics](https://pyradiomics.readthedocs.io/)。

## 许可证

请联系 SIAT-NazhangGroup 获取数据使用与代码分发许可。

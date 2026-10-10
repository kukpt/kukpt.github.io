---
title: 基于 Landsat 多光谱影像的 MPDI 土壤水分监测：从 USGS 数据到 Java 实现  
date: 2026-10-10  
tags:  
  - Landsat  - 遥感  
  - 土壤水分  
  - MPDI  - Java  - Apache SIS  
---  
  
## 基于 Landsat 多光谱影像的 MPDI 土壤水分监测：从 USGS 数据到 Java 实现  
  
土壤水分是农业灌溉、旱情监测和生态环境分析中的关键指标。传统地面测量精度高，但难以持续覆盖大范围区域；卫星遥感则可以在较大的空间尺度上周期性观测地表，为土壤水分监测提供稳定的数据来源。  
  
本文以一个 Java 实现的 Landsat MPDI 项目为例，介绍如何从 USGS Landsat Collection 2 Level-2 单波段 GeoTIFF 出发，完成波段识别、地表反射率转换、质量掩膜、MPDI 计算、结果着色以及 Red–NIR 土壤线拟合。整个过程直接使用 USGS 解压后的场景目录，不需要预先合并多波段 TIFF。  


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-01-整体.jpg' | relative_url }}" alt="整体">


  
## 1. MPDI 能反映什么  
  
红光与近红外反射率会随植被覆盖和土壤含水状态发生变化。在 Red–NIR 特征空间中，裸土像元通常沿一条近似直线分布，这条线被称为土壤线。随着土壤变干，像元在特征空间中的位置会发生有规律的变化。  
  
PDI（Perpendicular Drought Index）利用像元与土壤线之间的几何关系描述干旱程度。MPDI（Modified Perpendicular Drought Index）进一步引入植被影响校正，使指数能够适用于存在部分植被覆盖的区域。  
  
一种常见的 MPDI 表达式为：  
  
```text  
       Red + M × NIR - fv × (RvRed + M × RvNIR)
MPDI = ------------------------------------------  
              (1 - fv) × √(M² + 1)
```  

  
其中：  
  
- `Red`：红光地表反射率；  
- `NIR`：近红外地表反射率；  
- `M`：研究区土壤线斜率；  
- `fv`：植被覆盖度；  
- `RvRed`：纯植被红光反射率；  
- `RvNIR`：纯植被近红外反射率。  
  
项目采用的经验参数为：  
  
```text  
M = 0.97  
RvRed = 0.05  
RvNIR = 0.50  
```  
  
需要注意，MPDI 是反映相对干湿状况的遥感指数，不直接等于体积含水量。一般而言，较高的 PDI 或 MPDI 对应较强的水分胁迫或较干燥的地表。若要得到百分比形式的土壤含水量，还需要使用研究区实测数据建立标定模型。  
  
## 2. 项目的输入数据  
  
项目面向 USGS Landsat Collection 2 Level-2 地表反射率产品。下载并解压一个场景后，会得到多个单波段 GeoTIFF 和元数据文件。  
  
以 Landsat 8 为例，计算所需的主要文件如下：  
  
```text  
LC08_..._SR_B4.TIF  
LC08_..._SR_B5.TIF  
LC08_..._QA_PIXEL.TIF  
LC08_..._QA_RADSAT.TIF  
LC08_..._MTL.txt  
```  
  
  
不同 Landsat 传感器的红光和近红外波段编号并不完全相同：  
  
| Landsat 卫星 | 红光 | 近红外 |  
| --- | --- | --- |  
| Landsat 8/9 | `SR_B4` | `SR_B5` |  
| Landsat 4/5/7 | `SR_B3` | `SR_B4` |  
  
程序通过产品编号自动识别属于同一场景的文件，并选择对应传感器的红光与近红外波段。一个输入目录只处理一个场景，避免不同日期、轨道或行列号的数据被错误组合。  
  
在开始逐像元计算前，程序还会检查各输入栅格的：  
  
- 行列数；  
- 栅格范围；  
- 坐标参考系统；  
- 网格到地理坐标的转换关系。  
  
只比较图片宽高是不够的。两张影像即使尺寸相同，也可能具有不同的地理起点或像元网格；如果忽略这一点，最终得到的将是空间错位后的指数。  
  
## 3. 从 DN 转换为地表反射率  
  
Landsat Collection 2 Level-2 的地表反射率以无符号 16 位整数存储。像元值是经过缩放的 DN，不能直接作为物理反射率使用。  
  
USGS Collection 2 Level-2 的常用转换关系为：  
  
```text  
Reflectance = DN × 0.0000275 - 0.2  
```  
  
项目优先从场景的 `MTL.txt` 中读取各波段的：  
  
```text  
REFLECTANCE_MULT_BAND_n  
REFLECTANCE_ADD_BAND_n  
```  
  
因此红光和近红外分别执行：  
  
```text  
Red = redDN × redMultiplier + redAddend  
NIR = nirDN × nirMultiplier + nirAddend  
```  
  
只有在元数据文件缺失时，程序才回退到 Collection 2 的默认缩放参数。  
  
这个处理顺序非常重要。因为转换公式包含 `-0.2` 的加性偏移，不能先用原始 DN 计算 NDVI，再把 NDVI 与地表反射率混合到同一个公式中。正确的 NDVI 应由转换后的反射率计算：  
  
```text  
NDVI = (NIR - Red) / (NIR + Red)  
```  
  
此外，Collection 2 地表反射率的填充值为 `DN = 0`。即使没有质量文件，项目也会把红光或近红外 DN 为 0 的像元视为无效值。  
  
## 4. QA_PIXEL 为什么不可忽略  
  
遥感指数对云、云影和积雪非常敏感。一片薄云可能同时改变红光和近红外反射率，使其在 MPDI 结果中表现为不真实的极值。因此，质量控制不是可选的“后处理”，而是计算流程的一部分。  
  
Landsat `QA_PIXEL.TIF` 使用位编码保存多个质量标志。一个十进制像元值需要按二进制位解析，而不是作为普通类别编号判断。  
  
项目默认排除以下标志：  
  
| Bit | 标志 | 处理方式 |  
| --- | --- | --- |  
| 0 | Fill | 排除 |  
| 1 | Dilated Cloud | 排除 |  
| 2 | Cirrus，Landsat 8/9 | 排除 |  
| 3 | Cloud | 排除 |  
| 4 | Cloud Shadow | 排除 |  
| 5 | Snow | 排除 |  
| 7 | Water | 可选排除 |  

  
代码中的核心判断可以概括为：  
  
```java  
boolean invalid = (qaPixel & badAtmosphereMask) != 0;  
```  
  
程序直接检查独立质量位，不只依赖 Clear 位。这种方式也能避免 Landsat 4–7 Collection 2 在部分情况下存在的 Clear 位不一致问题。  
  
水体是否排除取决于研究目标。对于普通土壤水分或耕地旱情分析，建议启用 `--mask-water`；如果研究对象包含湿地、水陆交界或季节性积水，则可以保留水体，再结合其他数据单独分析。  
  
## 5. QA_RADSAT 与传感器异常  
  
`QA_RADSAT.TIF` 描述传感器波段饱和及部分平台相关异常。即使像元没有被云覆盖，如果红光或近红外探测器已经饱和，该像元同样不能用于可靠的光谱指数计算。  
  
项目会排除：  
  
- 反射波段饱和；  
- Landsat 7 掉线像元；  
- Landsat 8/9 地形遮挡像元。  
  
通过 `QA_PIXEL` 与 `QA_RADSAT` 的组合，可以同时处理大气、地表覆盖和传感器层面的主要质量问题。被排除的像元在 Float32 MPDI 中保存为 `NaN`。  
  
## 6. Java 栅格处理流程  
  
项目使用 Apache SIS 读取和写入 GeoTIFF。核心流程可以分成五步：  
  
```text  
场景目录
   │
   ├── 自动识别 Red / NIR / QA / MTL
   │
   ├── 校验栅格网格与投影
   │
   ├── DN 转地表反射率
   │
   ├── QA 掩膜 + MPDI 逐像元计算
   │
   └── GeoTIFF、着色图和拟合结果输出
```  
  
程序为每个输入影像建立 `PixelIterator`，同步读取相同位置的红光、近红外和质量像元。简化后的伪代码如下：  
  
```java  
while (redIterator.next()) {
    nirIterator.next();
    qaIterator.next();

    if (isBadQuality(qaValue)) {
        output.setSample(x, y, 0, Double.NaN);
        continue;
    }

    double red = redConverter.convert(redDN);
    double nir = nirConverter.convert(nirDN);
    double mpdi = calculateMpdi(red, nir);
    output.setSample(x, y, 0, mpdi);
}
```  
  
结果使用单波段 Float32 栅格保存。与整型输出相比，浮点格式可以保留指数的小数精度，也可以直接表达 `NaN` 无效值。  
  
程序还会统计：  
  
- 有效像元数量；  
- 被 QA 排除的像元数量；  
- 由填充值、除零或非法计算产生的无效像元数量。  
  
这些统计值可以快速发现异常。例如，如果一个场景绝大多数像元都被 QA 排除，通常意味着云量过高，不适合继续用于土壤水分分析。  
  
## 7. MPDI 着色图  
  
Float32 GeoTIFF 适合定量分析，但不方便直接查看。项目会同步生成 MPDI 着色预览图：  
  
```text  
mpdi-color.png  
```  
  
色带定义为：  
```text
蓝色 → 青色 → 黄色 → 橙色 → 深红色
较湿                         较干
```



<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-02-mpdi-color.png' | relative_url }}" alt="mpdi-color">


  
为了避免极少数异常值压缩主体像元的颜色变化，色带上下限使用有效 MPDI 样本的第 2 百分位和第 98 百分位。超出范围的像元分别截断到色带两端，无效像元显示为灰色。  
  
百分位拉伸非常适合单景质量检查，但它属于相对着色。若要比较不同日期的旱情变化，应根据研究区的历史数据确定统一上下限，并对所有场景使用相同色带范围。  
  
着色文件是用于快速浏览的 PNG，定量分析和空间叠加仍应使用保留地理参考的 `mpdi.tif`。  
  
## 8. Red–NIR 散点图  
  
Red–NIR 散点图可以直观展示一个场景的光谱结构：  
  
```text  
横轴：Red surface reflectance  
纵轴：NIR surface reflectance  
```  
  
由于一个 Landsat 场景可能包含数千万个像元，把所有点绘制到 PNG 不但消耗大量内存，也不会增加可读性。项目使用固定随机种子的蓄水池抽样，最多保留 5 万个代表性像元用于绘图。  
  
固定随机种子有两个优点：  
  
1. 不需要预先知道有效像元总数；  
2. 相同输入可以得到稳定、可重复的散点图。  
  
散点图保存为：  
  
```text  
mpdi-red-nir-scatter.png  
```  
  
图中蓝色表示普通有效像元，橙色表示用于土壤线回归的候选像元。  


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-03-mpdi-red-nir-scatter.png' | relative_url }}" alt="mpdi-red-nir-scatter">

 
  
## 9. 土壤线拟合  
  
土壤线应该来自研究区内具有代表性的裸土样本。项目首先使用一个通用光谱规则筛选候选点：  
  
```text  
-0.05 ≤ NDVI ≤ 0.25  
0 ≤ Red ≤ 1  
0 ≤ NIR ≤ 1  
```  
  
随后使用普通最小二乘法拟合：  
  
```text  
NIR = slope × Red + intercept  
```  
  
回归过程累计全部候选像元的统计量，因此土壤线拟合不受散点图 5 万点上限影响。输出内容包括：  
  
- 斜率 `slope`；  
- 截距 `intercept`；  
- 决定系数 `R²`；  
- 裸土候选像元数量；  
- 候选筛选条件；  
- MPDI 着色范围。  
  
报告保存在：  
  
```text  
mpdi-soil-line.txt  
```  
  
NDVI 阈值只能得到光谱意义上的低植被候选，不能保证所有点都是真正的裸土。建筑、道路、采石场和部分浅水像元也可能进入候选集合。因此，自动拟合适合作为诊断工具；用于正式模型参数时，最好结合土地覆盖图、耕地边界或人工标注样本进一步筛选。  

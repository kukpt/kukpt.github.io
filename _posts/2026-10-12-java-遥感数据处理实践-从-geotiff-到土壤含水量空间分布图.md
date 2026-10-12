---
title: 基于 Landsat 多光谱影像的 MPDI 土壤水分监测：从 USGS 数据到 Java 实现  
date: 2026-10-10  
tags:  
  - Landsat  - 遥感  
  - 土壤水分  
  - MPDI  - Java  - Apache SIS  
---  
  
  
土壤水分是农业灌溉、旱情监测和生态环境分析中的关键指标。传统地面测量**精度高**，但**难以持续覆盖大范围区域**；卫星遥感则可以在**较大的空间尺度上周期性观测地表**，为土壤水分监测提供稳定的数据来源。 

> 卫星距离地面几百公里，既不能触摸土壤，也不能像传感器探针那样直接测量含水量。它为什么还能帮助我们判断一片土地是干是湿？

答案藏在**太阳光与地表的相互作用里**。

太阳光照到地面后，一部分被吸收，一部分被反射。土壤中的水分、植被叶片和地表粗糙度都会改变反射光的强弱。卫星记录不同波段的反射信号，再通过光谱指数和地面观测之间的关系，便能描绘大范围的相对干湿格局。

本文以 Landsat 红光—近红外光谱空间和 **MPDI**（Modified Perpendicular Drought Index，修正垂直干旱指数）为主线，解释从卫星像元到土壤干湿图的完整过程，并穿插 **Apache SIS、Tablesaw 和 Smile 的 Java 实现**。


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-01-卫星多光谱揭示土壤水分.png' | relative_url }}" alt="卫星多光谱揭示土壤水分">



## 1. 土壤含水量为什么会改变光谱
干土和湿土最直观的区别之一是颜色。浇水后，土壤通常会变暗，这是因为**水分改变了土壤颗粒表面的光学性质，使更多能量被吸收、较少能量被反射**。

但卫星看到的不只有人眼可见的颜色。Landsat 同时观测可见光、近红外、短波红外和热红外等多个波段。对土壤和植被分析而言，红光与近红外尤其重要：

- 叶绿素会强烈吸收红光；
- 健康叶片内部结构会强烈反射近红外；
- 裸土的红光和近红外反射率通常随亮度近似同步变化；
- 水分增加往往会降低土壤的整体反射率。

因此，把每个像元的红光反射率作为横坐标、近红外反射率作为纵坐标，就能得到一个 Red–NIR 光谱散点空间。水体、植被、干土和湿土会在这个空间中呈现不同的分布特征。
## 2. 一条重要的线：土壤线
对植被很少的像元来说，红光与近红外反射率通常近似满足线性关系：

`NIR = slope × Red + intercept`

这条经验直线被称为土壤线。它不是全世界固定不变的常数，而会受到以下因素影响：
- 土壤类型和矿物组成；
- 有机质含量；
- 表面粗糙度；
- 土壤水分；
- 传感器光谱响应；
- 大气校正质量。
这意味着，从某个地区得到的土壤线参数，不一定适合另一个地区。对于快速分析，可以使用文献或经验参数；对于定量研究，更稳妥的做法是从研究区内筛选裸土像元并重新拟合。


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-02-red–nir光谱空间与土壤线.png' | relative_url }}" alt="Red–NIR光谱空间与土壤线">



## 3. 从 PDI 到 MPDI

PDI 使用像元在 Red–NIR 空间中的位置描述土壤干旱程度。其常见形式为：

```
			Red + M × NIR
PDI = ---------------------------
               √(M² + 1)
```

其中 `M` 是土壤线相关参数。PDI 对裸土较为有效，但真实地表往往被作物或自然植被部分覆盖。植被会显著提高近红外反射率，如果不加区分，指数可能把植被变化误认为土壤水分变化。

MPDI 在 PDI 的基础上加入植被覆盖度 `fv`：

```
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
  
`RvRed` 和 `RvNIR` 表示纯植被在红光、近红外波段的反射率。常见经验值分别为 `0.05` 和 `0.50`。

植被覆盖度可以由 NDVI 估算：

```
NDVI = (NIR - Red) / (NIR + Red)

FVC = (NDVI - NDVIsoil) / (NDVIveg - NDVIsoil)
```

实际使用时还应把 FVC 限制在 `[0, 1]`。`NDVIsoil` 和 `NDVIveg` 可以来自研究区经验值，也可以用多时相影像或场景百分位估计。

需要强调的是，MPDI 是相对干湿或水分胁迫指数，不是土壤含水量百分比。一般情况下，较高的 PDI/MPDI 表示相对干燥；要得到真实体积含水量，还需要地面样本完成标定。
## 4. Landsat 下载目录里有什么

USGS Landsat Collection 2 Level-2 产品已经完成地表反射率处理，适合直接用于光谱指数分析。下载并解压一个场景后，会看到许多单波段 GeoTIFF：

```
LC08_..._SR_B4.TIF
LC08_..._SR_B5.TIF
LC08_..._QA_PIXEL.TIF
LC08_..._QA_RADSAT.TIF
LC08_..._SR_QA_AEROSOL.TIF
LC08_..._MTL.txt
```

Landsat 8/9 使用 `SR_B4` 作为红光、`SR_B5` 作为近红外；Landsat 4/5/7 则使用 `SR_B3` 和 `SR_B4`。

这些单波段文件本身已经包含投影、空间范围和像元大小。计算时可以同步读取红光与近红外。


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-03-screenshot_2026-10-12_094711_082.png' | relative_url }}" alt="ScreenShot_2026-10-12_094711_082">


## 5. 像元值为什么不能拿来直接计算

Landsat 地表反射率通常以 16 位整数保存。文件中的 DN 是缩放后的存储值，不是最终物理反射率。

Collection 2 Level-2 常用转换关系为：
`Reflectance = DN × 0.0000275 - 0.2`

更严谨的实现会从 `MTL.txt` 读取每个波段自己的乘数和加数：
`REFLECTANCE_MULT_BAND_4 REFLECTANCE_ADD_BAND_4 REFLECTANCE_MULT_BAND_5 REFLECTANCE_ADD_BAND_5`

由于公式包含 `-0.2` 的偏移，不能用原始 DN 计算 NDVI，再把这个 NDVI 与反射率混入 MPDI。正确顺序是
`DN → Red/NIR 反射率 → NDVI/FVC → MPDI`

Collection 2 地表反射率的填充值是 `DN = 0`，这类像元也应直接排除。

## 6. 真正容易被忽略的是 QA_PIXEL

一幅看起来“有数据”的卫星影像，不代表每个像元都可靠。云、薄云、云影、积雪、传感器饱和都会改变光谱值。

`QA_PIXEL.TIF` 使用二进制位保存质量信息：

|Bit|含义|常见处理|
|---|---|---|
|0|Fill|排除|
|1|Dilated Cloud|排除|
|2|Cirrus，Landsat 8/9|排除|
|3|Cloud|排除|
|4|Cloud Shadow|排除|
|5|Snow|排除|
|7|Water|按研究目标决定|

位运算判断既快速又直接：

```
private static final int BAD_PIXEL_MASK =
    (1 << 0) | // Fill
    (1 << 1) | // Dilated cloud
    (1 << 2) | // Cirrus
    (1 << 3) | // Cloud
    (1 << 4) | // Cloud shadow
    (1 << 5);  // Snow

boolean invalid = (qaPixel & BAD_PIXEL_MASK) != 0;
```


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-04-landsat-qa_pixel-位编码与掩膜示意.png' | relative_url }}" alt="Landsat QA_PIXEL 位编码与掩膜示意">



`QA_RADSAT.TIF` 用来识别波段饱和、Landsat 7 掉线数据和 Landsat 8/9 地形遮挡。Landsat 8/9 的 `SR_QA_AEROSOL.TIF` 还能表示气溶胶反演是否有效、是否经过插值以及气溶胶等级。

对于土壤水分分析，至少应使用 `QA_PIXEL`；需要更严格的地表反射率质量控制时，可以进一步结合 `QA_RADSAT` 和 `SR_QA_AEROSOL`。


## 7. Apache SIS：让 GeoTIFF 不只是普通图片

GeoTIFF 不只是带有像素的 TIFF。它还保存坐标参考系统、像元分辨率、地理范围和网格转换信息。Java 的普通图片 API 可以读取像素，却不了解这些空间含义。

Apache SIS 提供了面向地理空间数据的统一接口。打开 GeoTIFF 的核心代码并不复杂：

```java
import smile.data.formula.Formula;
import smile.regression.LinearModel;
import smile.regression.OLS;

LinearModel model = OLS.fit(
    Formula.lhs("nir"),
    samples.selectColumns("red", "nir")
        .smile()
        .toDataFrame()
);

double slope = model.coefficients()[1];
double intercept = model.intercept();
double rSquared = model.RSquared();
```

`GridGeometry` 是这里非常重要的对象。它描述栅格网格如何映射到真实地理空间。同步处理 Red、NIR 和 QA 前，应确认它们的网格几何一致，而不能只检查宽度和高度。

Apache SIS 的 `PixelIterator` 可以逐像元遍历影像：

```java
PixelIterator redPixels = PixelIterator.create(redImage);
PixelIterator nirPixels = PixelIterator.create(nirImage);

double[] redSample = null;
double[] nirSample = null;

while (redPixels.next() && nirPixels.next()) {
    redSample = redPixels.getPixel(redSample);
    nirSample = nirPixels.getPixel(nirSample);

    double redDN = redSample[0];
    double nirDN = nirSample[0];
    // 反射率转换和 MPDI 计算
}
```

这种方式不需要把像元重新组织成庞大的 Java 对象列表，适合处理完整 Landsat 场景。

## 8. Tablesaw：把光谱样本变成可分析的表

栅格计算适合流式执行，但回归、筛选和统计分析更适合表格结构。Tablesaw 提供了类似数据框的 Java API，可以把经过抽样的 Red–NIR 数据整理为两列：
![](http://localhost:63342/markdownPreview/1505951412/)
```java
DoubleColumn red = DoubleColumn.create("red");
DoubleColumn nir = DoubleColumn.create("nir");

for (SpectralPoint point : sampledPoints) {
    red.append(point.red());
    nir.append(point.nir());
}

Table samples = Table.create("Red-NIR samples")
    .addColumns(red, nir)
    .dropRowsWithMissingValues();
```

为什么只把抽样数据放进 Tablesaw，而不是把整个 Landsat 场景全部转换成表？

一景影像可能包含数千万个像元。如果每个像元都变成表格行，会产生大量对象和额外内存开销。更合理的分工是：

- Apache SIS 负责完整栅格的快速逐像元计算；
- 在线统计量负责累计全部裸土候选像元；
- Tablesaw 保存有限数量的代表性样本，用于筛选、查看和绘图；
- Smile 使用表格或累计统计量完成回归分析。

这也是栅格计算与数据科学工具结合时很实用的一条原则：不要为了使用表格工具而把所有像元都表格化。

## 9. Smile：拟合 Red–NIR 土壤线

获得裸土候选样本后，可以使用 Smile 的普通最小二乘回归拟合土壤线。Tablesaw 数据可以转换为 Smile DataFrame：

```java
import smile.data.formula.Formula;
import smile.regression.LinearModel;
import smile.regression.OLS;

LinearModel model = OLS.fit(
    Formula.lhs("nir"),
    samples.selectColumns("red", "nir")
        .smile()
        .toDataFrame()
);

double slope = model.coefficients()[1];
double intercept = model.intercept();
double rSquared = model.RSquared();
```

不同 Smile 版本的系数和统计量访问方法可能略有差异，但基本思路相同：以 `nir` 为因变量、`red` 为自变量进行线性回归。

为了初步排除植被，裸土候选可以使用较低 NDVI 范围，例如：

```
-0.05 ≤ NDVI ≤ 0.25
0 ≤ Red ≤ 1
0 ≤ NIR ≤ 1
```

自动阈值只能得到“可能是裸土”的光谱点。
城市建筑、道路和部分水体也可能进入这一范围，所以拟合结果最好与土地覆盖掩膜或人工选择的裸土样本交叉验证。



<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-05-mpdi-red-nir-scatter-1.png' | relative_url }}" alt="mpdi-red-nir-scatter 1">



  
  
## 10. MPDI 着色图  
  
Float32 GeoTIFF 适合定量分析，但不方便直接查看。项目会同步生成 MPDI 着色预览图：  
  
```text  
mpdi-color.png  
```  
  
色带定义为：  
```text
蓝色 → 青色 → 黄色 → 橙色 → 深红色
较湿                         较干
```



<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-06-mpdi-color-1.png' | relative_url }}" alt="mpdi-color 1">


  
为了避免极少数异常值压缩主体像元的颜色变化，色带上下限使用有效 MPDI 样本的第 2 百分位和第 98 百分位。超出范围的像元分别截断到色带两端，无效像元显示为灰色。  
  
百分位拉伸非常适合单景质量检查，但它属于相对着色。若要比较不同日期的旱情变化，应根据研究区的历史数据确定统一上下限，并对所有场景使用相同色带范围。  
  
着色文件是用于快速浏览的 PNG，定量分析和空间叠加仍应使用保留地理参考的 `mpdi.tif`。  

## 在地图上显示
**和程序输出png颜色不一样是因为着色色带不同**


<img src="{{ '/assets/posts/java-遥感数据处理实践-从-geotiff-到土壤含水量空间分布图/image-07-screenshot_2026-10-12_111903_121.png' | relative_url }}" alt="ScreenShot_2026-10-12_111903_121">



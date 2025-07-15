# Disney Principled BSDF 实现与分析
何旭   502024330015
## 1. 实验平台与环境

- **操作系统**：WSL - Ubuntu 22.04.5 LTS  
- **编译工具**：g++ (>= C++17)，CMake，Make  
- **渲染框架**：Moer  
- **实验目标**：实现 Disney Principled BSDF 材质模型  

---

## 2. 实验步骤概述

1. **代码获取与编译**
   - 拉取Moer渲染器代码。
   - 确保系统安装了支持 C++17 的编译器与构建工具链。
   - 配置并编译项目：  
     ```bash
     mkdir build
     cd build
     cmake ..
     make
     ```

2. **完成自己的DisneyBSDF代码**
   - 在 `src/FunctionLayer/Material` 目录中添加包含 `MyDisneyBSDF.h` 和 `MyDisneyBSDF.cpp` 
   - 根据论文指导参考指导实现Disney Principled BSDF材质模型

3. **运行渲染测试**
   - 执行渲染器，并指定场景文件：
     ```bash
     ./Moer_r ../../scenes/disney-bsdf
     ```

---


## 3. 代码实现思路

本实现基于 Disney 论文设计并参考了UCSD CSE272的lab指导，具体说明如下：

### 3.1 MyDisneyBSDF.h
1. DisneyBSDF：实现 BxDF 接口的 BSDF 模型

- 关键参数：包含完整的 Disney 材质参数集
- 关键方法：
```cpp
//根据随机数 sample 和出射方向 out，采样一个入射方向 in。
BxDFSampleResult sample(const Vec3d & out, const Point2d & sample) const override;
//计算从方向 in 入射、向方向 out 反射/透射时的 BSDF 值。
Spectrum f(const Vec3d & out, const Vec3d & in) const override;
//返回对 in 方向采样的概率密度函数值。
double pdf(const Vec3d & out, const Vec3d & in) const override;
//给定入射和出射方向，返回对应的折射率。
double eta(const Vec3d &out, const Vec3d & in) const override;
```


2. DisneyMaterial：材质类，负责纹理参数管理

### 3.2 MyDisneyBSDF.cpp
#### 五个材质模型的模块
- `DisneyDiffuse`: 包含 baseColor、roughness、subsurface，用于实现 Disney 中对漫反射的改进模型
- `DisneyMetal`: 包括baseColor、alpha、distrib，alpha是表面粗糙度参数，用于微表面分布建模，取决于 roughness 和 anisotropy
- `DisneyClearCoat`: 单一参数 clearCoatGloss，表示清漆层，单独模拟一层高亮反射。
- `DisneyGlass`: baseColor、eta 折射率、GGX 分布，与金属类似，多了 eta 折射率参数，支持双折射材质建模
- `DisneySheen`: 包含baseColor 与 sheenTint 参数，表示织物表面微小光泽

#### DisneyDiffuse的实现
1. f(out, in)的实现
```cpp
Spectrum baseColor = disneyBXDF.baseColor / M_PI;
```
- 表示漫反射项的 Lambertian 部分

```cpp
FD90 = 0.5 + 2 * roughness * dot^2(wh, in)
FD(w) = 1 + (FD90 - 1) * (1 - cosθ)^5
```
- 粗糙度高时，在光线靠近表面切线方向照射时会有更强的边缘亮度（Fresnel-like 增亮），FD90 表示在入射方向与出射方向非常接近 90° 时的反射增强因子
- FD(w)为Schlick 近似公式，$ F(\theta) = F_0 + (1 - F_0)(1 - \cos\theta)^5 $ 用来模拟 Fresnel 效应的简洁表达式。

```cpp
FSS90 = roughness * |dot(wh, in)|
FSS(w) = 1 + (FSS90 - 1) * (1 - cosθ)^5
scaleFactor = FSS(in) * FSS(out) * (1 / (cosθ_in + cosθ_out) - 0.5) + 0.5
subsurface = 1.25 * baseColor * scaleFactor
```
- Disney 对subsurface的做法通过修正 Diffuse BRDF 的公式，近似模拟
- FSS90 代表当视角接近 90° 时的能量增强量（边缘加亮），是一个基于粗糙度和方向关系的调节因子，模仿了“在粗糙表面光容易被散开”的现象。
- FSS(w)是基于 Schlick Approximation 的能量增强修正项
- scaleFactor是为了模拟出类似散射但不会太发亮的效果

2. pdf函数实现
- 使用 余弦加权采样
- $ p(\omega_i) = \frac{\cos\theta_i}{\pi}
\quad \text{where } \omega_i \in \text{upper hemisphere} $

3. sample函数实现
- 使用 [0,1]^2 采样点转换为余弦加权半球采样。
- Disney 将其归入 Glossy，因为它带有 roughness 调制和方向性调节（非标准 Lambertian）。

#### DisneyMetal的实现
1. f(out, in)的实现
```cpp
F * D * G / (4 * AbsCosTheta(in) * AbsCosTheta(out));
```
- 使用微表面模型公式$f_r(\omega_o, \omega_i) = \frac{F(\omega_i, \mathbf{h}) \cdot D(\mathbf{h}) \cdot G(\omega_i, \omega_o)}{4 (\omega_i \cdot \mathbf{n}) (\omega_o \cdot \mathbf{n})}$ 计算
- F同diffuse中的实现
- D为控制微面朝向的概率密度```double D = distrib.D(wh, alpha);```
- G使用 Smith’s 方法计算几何遮蔽项 ```double G = distrib.G1(in, alpha) * distrib.G1(out, alpha);```

2. pdf函数实现
```cpp
distrib.Pdf(out, wh, alpha) / (4 * dot(out, wh));
```
- 微表面模型的导出方向采样 PDF $\text{pdf}(\omega_i) = \frac{D(\mathbf{h}) \cdot \cos\theta_h}{4 \, |\omega_o \cdot \mathbf{h}|}$
- ```4 * dot(out, wh)``` 是从 half-vector 概率密度转换为入射方向密度。
- 这里 distrib.Pdf(out, wh, alpha) 已封装好了前面这部分

3. sample函数实现
```cpp
Vec3d reflected = normalize(-out + 2 * dot(out, wh) * wh);
```
- 根据法向量反射公式 $\omega_i = -\omega_o + 2 (\omega_o \cdot \mathbf{h}) \, \mathbf{h}$ 计算入射方向
- 采样同上

#### DisneyClearCoat的实现
1. f(out, in)的实现
```cpp
double R0 = pow((eta-1)/(eta+1),2);
double f = R0 + (1-R0) * pow((1 - absDot(wh,in)),5);
Spectrum F = Spectrum(f);
```



### 3. 分量核心逻辑概述

| 分量 | 描述 |
|------|------|
| **Diffuse** | 带次表面项的 Lambert 漫反射，使用 Schlick 菲涅耳项做能量调节 |
| **Metal** | 基于 GGX 分布的 Cook-Torrance 微面反射模型 |
| **ClearCoat** | 使用 GTR1 分布模拟清漆层镜面反射，菲涅耳固定 |
| **Glass** | 同时考虑反射与折射，使用 Fresnel 判定采样分支 |
| **Sheen** | 基于色调调制的浅角反射增强（用于丝绸/织物） |

### 4. 采样权重逻辑

- Diffuse: `(1 - metallic) * (1 - specularTransmission)`
- Metal: `1 - specularTransmission * (1 - metallic)`
- Glass: `(1 - metallic) * specularTransmission`
- ClearCoat: `0.25 * clearCoat`

这些用于控制混合分量在 `sample()` 和 `f()`、`pdf()` 中的加权行为。

---

## 算法理解与分析

### 优点：

- 支持多种复杂材质合成，参数语义清晰、艺术家友好；
- 多层次结构（如 clearCoat + metal + sheen）实现自然过渡；
- 使用 `std::variant` 实现高效运行时分量分派；
- 易于扩展其他分量（如透明塑料、织物、陶瓷等）

### 缺点与改进空间：

- 分量之间为线性叠加，非物理准确；
- 有能量守恒问题（需加入归一化因子或多重散射模型）；
- 清漆层使用简化分布模型（GTR1）精度较低；
- 不考虑各向异性 BRDF 的局部旋转处理；
- 可引入更通用的 microfacet 多散射模型（如 Heitz’s dual scattering）

---

## 实验结果图（请自行替换）

| 场景描述 | 渲染图像 |
|----------|----------|
| 漫反射球体（roughness=0.5） | ![](images/diffuse.png) |
| 金属材质球体（metallic=1.0） | ![](images/metal.png) |
| 透明玻璃球（transmission=1.0） | ![](images/glass.png) |
| 清漆层+金属组合 | ![](images/clearcoat_metal.png) |

---

## 验证材料

- **核心代码截图**：
  - `DisneyBSDF.cpp` 中 `evalDisneyBXDFOP::operator()` 各个实现；
  - `sampleDisneyBXDFOP::operator()` 的半球采样与分布采样逻辑；
  - `DisneyBSDF::sample()` 权重组合策略；

- **运行输出截图**：
  - 编译输出 `make -j` 的日志；
  - 渲染器终端输出（包括采样时间、图像大小等）；

- **图像输出文件**：
  - `.png` 或 `.ppm` 文件导出结果；
  - 可选提供 `.exr` 高动态范围图像用于质量评估。

---

## 参考资料

1. Disney BRDF 论文 – “Physically-Based Shading at Disney”, 2012  
2. https://cseweb.ucsd.edu/~tzli/cse272/homework1.pdf  
3. Burley, B. “Extending the Disney BRDF to a BSDF with Integrated Subsurface Scattering”, 2015  
4. Heitz, E. “Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs”, 2014  
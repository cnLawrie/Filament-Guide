# PBR
基于物理的渲染的范畴，由三部分组成：

- 基于物理的材质（Material）
- 基于物理的光照（Lighting）
- 基于物理适配的摄像机（Camera）

![pbr](../assets/pbr.png)

PBR 并不是“一项”技术，它是由一系列技术的集合，并不断改进的结果。

# 符号
下面是方程中将会使用到的符号  
符号 | 定义
---|:--:
v | 视角单位向量
l | 入射光单位向量
n | 表面法向单位向量
h | l 与 v之间的半单位向量
f | BRDF
f<sub>d</sub> | BRDF的漫反射分量
f<sub>r</sub> | BRDF的镜面反射分量
α | 粗糙度，使用输入perceptualRoughness重新映射
σ | 漫反射率
Ω | 球形域
f<sub>0</sub> | 掠角的反射率
f<sub>90</sub> | 入射余角的反射率
χ+(a) | Heaviside函数（如果a> 0则为1，否则为0）
n<sub>ior</sub> | 表面的折射率（IOR）
⟨n⋅l⟩ | 点分量夹逼到[0..1]
⟨a⟩ | 饱和值（夹逼到[0..1]）


[Moving Frostbite to PBR](https://www.ea.com/frostbite/news/moving-frostbite-to-pb)

[理解PBR：从原理到实现](https://neil3d.github.io/unreal/pbr-theory.html)

[Physically Based Rendering:From Theory To Implementation](http://www.pbr-book.org/)

[SIGGRAPH 2010 Course: Physically-Based Shading Models in Film and Game Production](http://renderwonk.com/publications/s2010-shading-course/)

[SIGGRAPH 2013 Course: Physically Based Shading in Theory and Practice](https://blog.selfshadow.com/publications/s2013-shading-course/)

[Microfacet 模型的反射与折射](https://segmentfault.com/a/1190000000436286)

[opengl-tutorial](http://www.opengl-tutorial.org/cn/)
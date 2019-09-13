
Filament 是一个基于物理的渲染(PBR)引擎。
本文的目的是解释Filament材质和照明模型背后的理论。它只关注算法，并将涉及数学及物理知识。如果你对这部分不感兴趣可以跳过，直接看例子了解Filament的Api，并进行相关开发。

# 基于物理的渲染
 与传统的实时模型相比，基于物理的渲染是一种渲染方法，可以更准确地表示材质以及它们与光的交互方式。 在PBR方法的核心是分离材料和照明，这让我们更容易创建在所有照明条件下看起来准确的真实模型。


## 入射角与掠射角
- 入射角(angle of incidence)
- 掠射角(grazing angle)
![](./assets/basis/grazingAndIncidenceAngles.png)

## diffuse lobe & specular lobe
![](./assets/basis/DiffuesLobe&SpecularLobe.jpg)

## 在图形学中什么是lobe
lobe是定义在直角坐标或极坐标中函数的一个峰值。如cos函数，在0或2pi可以取得峰值。
在照明中，lobe通常对应于反射光的方向，越高的lobe意味着更多的光量。

## 色差
色差是源于不同波长的光线在玻璃里的色散和折射系数的差异，从而导致不同波长的光线有不同的焦点。

# Diffuse BRDF

```
float Fd_Lambert() {
    return 1.0 / PI;
}

vec3 Fd = diffuseColor * Fd_Lambert();
```



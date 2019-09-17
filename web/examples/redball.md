这个教程创造一个红球，主要会介绍到材质和纹理的创建、使用。

创建redball.html, 内容与三角形的html一样，只是把引入的triangle.js改成redball.js。接下来我们会用两个可执行程序：matc和cmgen。在这里[Filament](https://github.com/google/filament/releases)下载对应的发行包，选择对应你机器的版本，而不是web版。

我使用的是Mac，下载到的就是filament-20190826-mac.tgz。解压压缩包，打开bin文件夹就可以看到我们需要的可执行程序matc和cmgen。
Mac和Linux用户可以将bin文件夹添加到PATH变量中，这里我是这样做的
```
vim ~/.bash_profile
添加export PATH=/Users/lawrie/Downloads/filament/bin
source 
```

## 定义塑料材质

matc工具会接受一个文本文件，文件中要包含对PBR材质的描述，输出一个二进制材质压缩包，压缩包中包含了shader代码和相关的元数据。查看[Filament材质系统](https://google.github.io/filament/Materials.md.html)获取详细信息。

我们一起来试试matc。创建一个名为plastic.mat的文件，文件包含以下文本：
```
material {
    name: Lit,
    shadingModel: lit,
    parameters: [
        { type: float3, name: baseColor },
        { type: float, name: roughness },
        { type: float, name: clearCoat },
        { type: float, name: clearCoatRoughness }
    ],
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        material.baseColor.rgb = materialParams.baseColor;
        material.roughness = materialParams.roughness;
        material.clearCoat = materialParams.clearCoat;
        material.clearCoatRoughness = materialParams.clearCoatRoughness;
    }
}
```

下一步，如下使用matc:
```
matc -a opengl -p mobile -o plastic.filamat plastic.mat
```
生成的plastic.filamat我们会在后面使用它

## 烘焙环境贴图
接下来，我们将使用Filament的cmgen工具以latlong格式使用HDR环境贴图，并生成两个cubemap贴图文件：一个mipmapped IBL和一个模糊天空盒。

下载[pillars_2k.hdr](https://github.com/google/filament/blob/master/third_party/environments/pillars_2k.hdr),并在命令行执行下面的指令：
```
cmgen -x . --format=ktx --size=256 --extract-blur=0.1 pillars_2k.hdr
```

执行完后悔生成一个pillars_2k文件夹，文件夹中含有一个IBL KTX文件，一个天空盒KTX文件，还有一个描述球谐函数系数的文本文件


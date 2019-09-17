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

## 新建JS

```
const environ = 'pillars_2k';
const ibl_url = `${environ}/${environ}_ibl.ktx`;
const sky_url = `${environ}/${environ}_skybox.ktx`;
const filamat_url = 'plastic.filamat'

Filament.init([ filamat_url, ibl_url, sky_url ], () => {
  // Create some global aliases to enums for convenience.
  window.VertexAttribute = Filament.VertexAttribute;
  window.AttributeType = Filament.VertexBuffer$AttributeType;
  window.PrimitiveType = Filament.RenderableManager$PrimitiveType;
  window.IndexType = Filament.IndexBuffer$IndexType;
  window.Fov = Filament.Camera$Fov;
  window.LightType = Filament.LightManager$Type;

  // Obtain the canvas DOM object and pass it to the App.
  const canvas = document.getElementsByTagName('canvas')[0];
  window.app = new App(canvas);
} );

class App {
  constructor(canvas) {
    this.canvas = canvas;
    const engine = this.engine = Filament.Engine.create(canvas);
    const scene = engine.createScene();

    // TODO: create material
    // TODO: create sphere
    // TODO: create lights
    // TODO: create IBL
    // TODO: create skybox

    this.swapChain = engine.createSwapChain();
    this.renderer = engine.createRenderer();
    this.camera = engine.createCamera();
    this.view = engine.createView();
    this.view.setCamera(this.camera);
    this.view.setScene(scene);
    this.resize();
    this.render = this.render.bind(this);
    this.resize = this.resize.bind(this);
    window.addEventListener('resize', this.resize);
    window.requestAnimationFrame(this.render);
  }

  render() {
    const eye = [0, 0, 4], center = [0, 0, 0], up = [0, 1, 0];
    const radians = Date.now() / 10000;
    vec3.rotateY(eye, eye, center, radians);
    this.camera.lookAt(eye, center, up);
    this.renderer.render(this.swapChain, this.view);
    window.requestAnimationFrame(this.render);
  }

  resize() {
    const dpr = window.devicePixelRatio;
    const width = this.canvas.width = window.innerWidth * dpr;
    const height = this.canvas.height = window.innerHeight * dpr;
    this.view.setViewport([0, 0, width, height]);
    this.camera.setProjectionFov(45, width / height, 1.0, 10.0, Fov.VERTICAL);
  }
}
```

上面的代码与上一个教程的大概一致，区别只是加载了新的资源、给相机加了动画。

下一步，我们由外部材质资源创建一个材质实例。在创建材质部分添加以下代码片段：
```
const material = engine.createMaterial(filamat_url);
const matinstance = material.createInstance();

const red = [0.8, 0.0, 0.0];
matinstance.setColor3Parameter('baseColor', Filament.RgbType.sRGB, red);
matinstance.setFloatParameter('roughness', 0.5);
matinstance.setFloatParameter('clearCoat', 1.0);
matinstance.setFloatParameter('clearCoatRoughness', 0.3);
```

接着，创建一个renderable球体对象。我们在过程中使用了**IcoSphere**工具类。这个工具类的作用是细分icosadedron，生成三个数组：
- icosphere.vertices Float32Array，XYZ坐标轴
- icosphere.tangents Uint16Array(half-floats)，将表面方向编码为四元数。
- icosphere.triangles Uint16Array，三角形索引

我们继续使用使用这些数组来构建顶点缓冲区和索引缓冲区。在创建球体部分添加以下代码片段：
```
const renderable = Filament.EntityManager.get().create();
scene.addEntity(renderable);

const icosphere = new Filament.IcoSphere(5);

const vb = Filament.VertexBuffer.Builder()
  .vertexCount(icosphere.vertices.length / 3)
  .bufferCount(2)
  .attribute(VertexAttribute.POSITION, 0, AttributeType.FLOAT3, 0, 0)
  .attribute(VertexAttribute.TANGENTS, 1, AttributeType.SHORT4, 0, 0)
  .normalized(VertexAttribute.TANGENTS)
  .build(engine);

const ib = Filament.IndexBuffer.Builder()
  .indexCount(icosphere.triangles.length)
  .bufferType(IndexType.USHORT)
  .build(engine);

vb.setBufferAt(engine, 0, icosphere.vertices);
vb.setBufferAt(engine, 1, icosphere.tangents);
ib.setBuffer(engine, icosphere.triangles);

Filament.RenderableManager.Builder(1)
  .boundingBox({ center: [-1, -1, -1], halfExtent: [1, 1, 1] })
  .material(0, matinstance)
  .geometry(0, PrimitiveType.TRIANGLES, vb, ib)
  .build(engine, renderable);
```

此时，已经正在渲染一个球体了，只是它是黑色的，你看不见。可以像第一教程中一样，通过setClearColor改变背景颜色就可以看见这个黑色的球了。

## 添加光照

在这个部分中，我们会创建一些平行光源和一个基于图像的光照(IBL)。我们在本文开头build出了两个ktx文件，pillars_2k_ibl.ktx和pillars_2k_skybox.ktx，该IBL就由pillars_2k_ibl.ktx定义。第一步，我们在创建光照部分添加以下代码片段：

```
const sunlight = Filament.EntityManager.get().create();
scene.addEntity(sunlight);
Filament.LightManager.Builder(LightType.SUN)
  .color([0.98, 0.92, 0.89])
  .intensity(110000.0)
  .direction([0.6, -1.0, -0.8])
  .sunAngularRadius(1.9)
  .sunHaloSize(10.0)
  .sunHaloFalloff(80.0)
  .build(engine, sunlight);

const backlight = Filament.EntityManager.get().create();
scene.addEntity(backlight);
Filament.LightManager.Builder(LightType.DIRECTIONAL)
        .direction([-1, 0, 1])
        .intensity(50000.0)
        .build(engine, backlight);
```

这里的SUN光源就类似于平行光源，但是多出一些额外的参数。

下面，我们需要由KTX IBL创建出一个IndirectLight对象。其中一种方法如下（先不要敲下面的代码）：
```
const format = Filament.PixelDataFormat.RGB;
const datatype = Filament.PixelDataType.UINT_10F_11F_11F_REV;

// Create a Texture object for the mipmapped cubemap.
const ibl_package = Filament.Buffer(Filament.assets[ibl_url]);
const iblktx = new Filament.KtxBundle(ibl_package);

const ibltex = Filament.Texture.Builder()
  .width(iblktx.info().pixelWidth)
  .height(iblktx.info().pixelHeight)
  .levels(iblktx.getNumMipLevels())
  .sampler(Filament.Texture$Sampler.SAMPLER_CUBEMAP)
  .format(Filament.Texture$InternalFormat.RGBA8)
  .build(engine);

for (let level = 0; level < iblktx.getNumMipLevels(); ++level) {
  const uint8array = iblktx.getCubeBlob(level).getBytes();
  const pixelbuffer = Filament.PixelBuffer(uint8array, format, datatype);
  ibltex.setImageCube(engine, level, pixelbuffer);
}

// Parse the spherical harmonics metadata.
const shstring = iblktx.getMetadata('sh');
const shfloats = shstring.split(/\s/, 9 * 3).map(parseFloat);

// Build the IBL object and insert it into the scene.
const indirectLight = Filament.IndirectLight.Builder()
  .reflections(ibltex)
  .irradianceSh(3, shfloats)
  .intensity(50000.0)
  .build(engine);

scene.setIndirectLight(indirectLight);
```

Filament提供了一个JS工具来简化创建IBL,我们在创建IBL部分添加以下代码片段：
```
const indirectLight = engine.createIblFromKtx(ibl_url);
indirectLight.setIntensity(50000);
scene.setIndirectLight(indirectLight);
```

## 添加背景
现在你已经可以看到一个红色塑料球在黑色背景下旋转。没有天空盒，球上的反射显得很奇怪。下面是一种创建天空盒纹理的办法：
```
const sky_package = Filament.Buffer(Filament.assets[sky_url]);
const skyktx = new Filament.KtxBundle(sky_package);
const skytex = Filament.Texture.Builder()
  .width(skyktx.info().pixelWidth)
  .height(skyktx.info().pixelHeight)
  .levels(1)
  .sampler(Filament.Texture$Sampler.SAMPLER_CUBEMAP)
  .format(Filament.Texture$InternalFormat.RGBA8)
  .build(engine);

const uint8array = skyktx.getCubeBlob(0).getBytes();
const pixelbuffer = Filament.PixelBuffer(uint8array, format, datatype);
skytex.setImageCube(engine, 0, pixelbuffer);
```

不巧，Filament又提供了一个JS工具来简化创建天空盒,我们在创建IBL部分添加以下代码片段：
```
const skybox = engine.createSkyFromKtx(sky_url);
scene.setSkybox(skybox);
```

现在，我们已经有一个漂浮着的闪亮红色塑料球。

下一个教程中，我们将关注与纹理和交互。
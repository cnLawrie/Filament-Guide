这个教程主要会介绍压缩纹理、mipmap的生成、纹理异步加载和操控模型旋转。

上个教程中，我们使用了matc和cmgen工具，这个教程中我们将使用另外两个工具：filamesh和mipgen。

## 创建filamesh文件

Filament没有资源加载系统，但是他提供了一个叫**filamesh**的二进制mesh格式用于简单的用例。让我们用这个[OBJ文件](https://github.com/google/filament/blob/master/assets/models/monkey/monkey.obj)创建一个压缩的filamesh文件。

```
filamesh --compress monkey.obj suzanne.filamesh
```

## 创造纹理映射过的纹理(mipmapped textures)

我们来用**mipgen**工具来创造一个经过纹理映射的KTX文件。我们将为每个纹理创建压缩和非压缩的版本，因为不是所有平台都支持一种压缩纹理格式。复制[猴子文件夹](https://github.com/google/filament/tree/master/assets/models/monkey)下的png文件，然后：

```
# Create mipmaps for base color and two compressed variants.
mipgen albedo.png albedo.ktx
mipgen --compression=astc_fast_ldr_4x4 albedo.png albedo_astc.ktx
mipgen --compression=s3tc_rgb_dxt1 albedo.png albedo_s3tc.ktx

# Create mipmaps for the normal map and a compressed variant.
mipgen --strip-alpha --kernel=NORMALS --linear normal.png normal.ktx
mipgen --strip-alpha --kernel=NORMALS --linear --compression=etc_rgb8_normalxyz_40 \
    normal.png normal_etc.ktx

# Create mipmaps for the single-component roughness map and a compressed variant.
mipgen --grayscale roughness.png roughness.ktx
mipgen --grayscale --compression=etc_r11_numeric_40 roughness.png roughness_etc.ktx

# Create mipmaps for the single-component metallic map and a compressed variant.
mipgen --grayscale metallic.png metallic.ktx
mipgen --grayscale --compression=etc_r11_numeric_40 metallic.png metallic_etc.ktx

# Create mipmaps for the single-component occlusion map and a compressed variant.
mipgen --grayscale ao.png ao.ktx
mipgen --grayscale --compression=etc_r11_numeric_40 ao.png ao_etc.ktx
```

**mipgen --help**可以看到mipgen的参数和支持的格式。

## 烘焙环境贴图

像上一个教程一样，我们用Filament的**cmgen**工具来生成cubemap,但这次不同的是我们会创建几个不同的版本。

下载[syferfontein_18d_clear_2k.hdr](https://github.com/google/filament/blob/master/third_party/environments/syferfontein_18d_clear_2k.hdr),然后在命令行输入以下命令：
```
# Create S3TC variant of the IBL, then rename it to have a _s3tc suffix.
cmgen -x . --format=ktx --size=256 --extract-blur=0.1 --compression=s3tc_rgba_dxt5 \
    syferfontein_18d_clear_2k.hdr
cd syfer* ; mv syfer*_ibl.ktx syferfontein_18d_clear_2k_ibl_s3tc.ktx ; cd -

# Create ETC variant of the IBL, then rename it to have a _s3tc suffix.
cmgen -x . --format=ktx --size=256 --extract-blur=0.1 --compression=etc_rgba8_rgba_40 \
    syferfontein_18d_clear_2k.hdr
cd syfer* ; mv syfer*_ibl.ktx syferfontein_18d_clear_2k_ibl_etc.ktx ; cd -

# Create small uncompressed Skybox variant, then rename it to have a _tiny suffix.
cmgen -x . --format=ktx --size=64 --extract-blur=0.1 syferfontein_18d_clear_2k.hdr
cd syfer* ; mv syfer*_ibl.ktx syferfontein_18d_clear_2k_skybox_tiny.ktx ; cd -

# Create full-size uncompressed Skybox and IBL
cmgen -x . --format=ktx --size=256 --extract-blur=0.1 syferfontein_18d_clear_2k.hdr
```

## 定义纹理材质

上个教程中，我们为红色塑料球生成了filament文件。在这个demo中，我们会用多个参数的纹理来创建材质。

创建一个textured.mat文件，添加以下文本。你可能会注意到现在材质定义中需要一个uv0属性。

```
material {
    name : textured,
    requires : [ uv0 ],
    shadingModel : lit,
    parameters : [
        { type : sampler2d, name : albedo },
        { type : sampler2d, name : roughness },
        { type : sampler2d, name : metallic },
        { type : float, name : clearCoat },
        { type : sampler2d, name : normal },
        { type : sampler2d, name : ao }
    ],
}

fragment {
    void material(inout MaterialInputs material) {
        material.normal = texture(materialParams_normal, getUV0()).xyz * 2.0 - 1.0;
        prepareMaterial(material);
        material.baseColor = texture(materialParams_albedo, getUV0());
        material.roughness = texture(materialParams_roughness, getUV0()).r;
        material.metallic = texture(materialParams_metallic, getUV0()).r;
        material.clearCoat = materialParams.clearCoat;
        material.ambientOcclusion = texture(materialParams_ao, getUV0()).r;
    }
}
```

下一步，调用matc工具：
```
matc -a opengl -p mobile -o textured.filamat textured.mat
```

现在你在目录下就会有一个材质文件了。对于suzanne材质，法线贴图会增加划痕，反射率贴图将眼睛涂成白色等等。要了解材质的更多信息，见[Filament Material System](https://google.github.io/filament/Materials.md.html)

## 创建程序框架

像之前一样，新建一个suzanne.html，suzanne.js。在suzanne.js中加入以下内容：

```
// TODO: 声明资源URL

Filament.init([ filamat_url, filamesh_url, sky_small_url, ibl_url ], () => {
    window.app = new App(document.getElementsByTagName('canvas')[0]);
});

class App {
    constructor(canvas) {
        this.canvas = canvas;
        this.engine = Filament.Engine.create(canvas);
        this.scene = this.engine.createScene();

        const material = this.engine.createMaterial(filamat_url);
        this.matinstance = material.createInstance();

        const filamesh = this.engine.loadFilamesh(filamesh_url, this.matinstance);
        this.suzanne = filamesh.renderable;

        // TODO: create sky box and IBL
        // TODO: initialize gltumble
        // TODO: fetch larger assets

        this.swapChain = this.engine.createSwapChain();
        this.renderer = this.engine.createRenderer();
        this.camera = this.engine.createCamera();
        this.view = this.engine.createView();
        this.view.setCamera(this.camera);
        this.view.setScene(this.scene);
        this.render = this.render.bind(this);
        this.resize = this.resize.bind(this);
        window.addEventListener('resize', this.resize);

        const eye = [0, 0, 4], center = [0, 0, 0], up = [0, 1, 0];
        this.camera.lookAt(eye, center, up);

        this.resize();
        window.requestAnimationFrame(this.render);
    }

    render() {
        // TODO: apply gltumble matrix
        this.renderer.render(this.swapChain, this.view);
        window.requestAnimationFrame(this.render);
    }

    resize() {
        const dpr = window.devicePixelRatio;
        const width = this.canvas.width = window.innerWidth * dpr;
        const height = this.canvas.height = window.innerHeight * dpr;
        this.view.setViewport([0, 0, width, height]);

        const aspect = width / height;
        const Fov = Filament.Camera$Fov, fov = aspect < 1 ? Fov.HORIZONTAL : Fov.VERTICAL;
        this.camera.setProjectionFov(45, aspect, 1.0, 10.0, fov);
    }
}
```

我们会使用10个下载好的资源，但App构造函数中只会用到4个。其他六个就在构造函数后下载。

下一步，因为不同的客户端对压缩纹理的兼容性不同，我们需要给url提供不同版本的资源。

我们只需要下载当前平台所需要的压缩纹理，Filament提供了**getSupportedFormatSuffix**方法。它的参数是一个需求格式的列表(etc,s3tc或astc)，开发者知道服务器有这些格式的资源。这个方法会在需求格式集合和支持格式集合取交集，然后返回一个正确的格式字符串，这个字符串可能为空字符串。

在这个demo中，我们知道web服务器IBL有etc和s3tc版本，反照率有astc和s3tc版本，以及用于其他纹理的其他版本。未压缩版本通常作为最后选择。继续使用以下代码段替换声明资源URLs。

```
const ibl_suffix = Filament.getSupportedFormatSuffix('etc s3tc');
const albedo_suffix = Filament.getSupportedFormatSuffix('astc s3tc');
const texture_suffix = Filament.getSupportedFormatSuffix('etc');

const environ = 'syferfontein_18d_clear_2k'
const ibl_url = `${environ}/${environ}_ibl${ibl_suffix}.ktx`;
const sky_small_url = `${environ}/${environ}_skybox_tiny.ktx`;
const sky_large_url = `${environ}/${environ}_skybox.ktx`;
const albedo_url = `albedo${albedo_suffix}.ktx`;
const ao_url = `ao${texture_suffix}.ktx`;
const metallic_url = `metallic${texture_suffix}.ktx`;
const normal_url = `normal${texture_suffix}.ktx`;
const roughness_url = `roughness${texture_suffix}.ktx`;
const filamat_url = 'textured.filamat';
const filamesh_url = 'suzanne.filamesh';
```


## 创建天空盒和IBL

下一步，让我们在App构造函数中新建低精度的天空盒与IBL。

```
this.skybox = this.engine.createSkyFromKtx(sky_small_url);
this.scene.setSkybox(this.skybox);
this.indirectLight = this.engine.createIblFromKtx(ibl_url);
this.indirectLight.setIntensity(100000);
this.scene.setIndirectLight(this.indirectLight);
```

## 异步获取资源

下一步，我们会在构造函数内调用Filament.fetch方法。这个方法跟Filament.init很像，接受一串资源URL还有一个在资源下载后调用的回调函数。

在回调函数内，材质实例会调用几个setTextureParameter,然后我们会用更高分辨率的材质重新创建太空盒。最后一步，我们会把之前在构造函数内创建的renderable显示出来。

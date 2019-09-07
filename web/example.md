# 一个三角形
1. 创建一个triangle.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Filament Tutorial</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,user-scalable=no,initial-scale=1">
    <style>
        body { margin: 0; overflow: hidden; }
        canvas { touch-action: none; width: 100%; height: 100%; }
    </style>
</head>
<body>
    <canvas></canvas>
    <script src="//unpkg.com/filament/filament.js"></script>
    <script src="//unpkg.com/gl-matrix@2.8.1/dist/gl-matrix-min.js"></script>
    <script src="triangle.js"></script>
</body>
</html>
```
其中
    - filament.js用于下载相关资源和编译Filament WASM模块，并且包含了一些工具，比如加载KTX纹理。
    - gl-matrix-min.js 是一个数学库，包含了对向量、矩阵的处理工具

2. 在目录下启动一个静态服务器
```
python3 -m http.server     # Python 3
python -m SimpleHTTPServer # Python 2.7
npx http-server -p 8000    # nodejs
```

3. 创建引擎和场景
现在我们已经有了一个基本框架，可以添加Filament对象了。
在App的构造函数中添加：
```
this.canvas = document.getElementsById('canvas')
const engine = this.engine = Filament.Engine.create(this.canvas)
```
引擎需要canvas Dom对象作为参数来创造一个WebGL 2.0的上下文对象。该engine变量是一个工厂，可以用来创建许多Filament个体，如场景。下面添加一个叫做triangle的空白个体。
```
this.scene = engine.createScene()
this.triangle = Filament.EntityManager.get().create()
this.scene.addEntity(this.triangle)
```

4. 构造类型数组
我们会构造两个类型数组：一个顶点数组包含了每个顶点的XY坐标，一个颜色数组包含了每个顶点的色值
```
const TRIANGLE_POSITIONS = new Float32Array([
    1, 0,
    Math.cos(Math.PI * 2 / 3), Math.sin(Math.PI * 2 / 3),
    Math.cos(Math.PI * 4 / 3), Math.sin(Math.PI * 4 / 3),
])
const TRIANGLE_COLORS = new Uint32Array([0xffff0000, 0xff00ff00, 0xff0000ff])
```
下一步，我们会用位置和颜色缓冲区来创建一个VertexBuffer对象
```
const VertexAttribute = Filament.VertexAttribute
const AttributeType = Filament.VertexBuffer$AttributeType
this.vb = Filament.VertexBuffer.Builder()
    .vertexCount(3)
    .bufferCount(2)
    .attribute(VertexAttribute.POSITION, 0, AttributeType.FLOAT2, 0, 8)
    .attribute(VertexAttribute.COLOR, 1, AttributeType.UBYTE4, 0, 4)
    .normalized(VertexAttribute.COLOR)
    .build(engine)

this.vb.setBufferAt(engine, 0, TRIANGLE_POSITIONS)
this.vb.setBufferAt(engine, 1, TRIANGLE_COLORS)
```
上面的代码片段先为两个枚举类型分别创建了别名，然后用Builder方法创建了顶点缓冲区, 最后用setBufferAt函数把两个缓冲区对象放在内存中的相应位置上。

我们在VertexBuffer中设置了两个缓冲区，每个缓冲区对应一个属性(Attribute)。另外我们也可以把这些属性插入或串联进一个缓冲。

接下来，我们会构造一个索引缓冲区。仅仅对我们这个demo来说，它是可有可无，它仅仅保存了整形0，1，2。
```
this.ib = Filament.IndexBuffer.Builder()
    .indexCount(3)
    .bufferType(Filament.IndexBuffer$IndexType.USHORT)
    .build(engine)
this.ib.setBuffer(engine, new Uint16Array([0, 1, 2]));
```
索引缓冲区只能保存两种类型的数据：USHORT或UINT

5. 结束初始化
下一步，我们会创建一个材质包中的真实材质（该材质是一个对象，材质包是一个二进制Blob），然后从材质对象中提取出默认的MaterialInstance。在取出材质实例化对象后，我们设置好边界框并传入顶点和索引缓冲区，我们终于可以把它渲染出来了。
```
const mat = engine.createMaterial('triangle.filamat')
const matinst = mat.getDefaultInstance()
Filament.RenderableManager.Builder(1)
    .boundingBox({ center: [-1, -1, -1], halfExtent: [1, 1, 1]})
    .material(0, matinst)
    .geometry(0, Filament.RenderableManager$PrimitiveType.TRIANGLES, this.vb, this.ib)
    .build(engine, this.triangle)
```

接下来让我们通过创建交换链，渲染器，相机和视图来结束初始化。
```
this.swapChain = engine.createSwapChain()
this.renderer = engine.createRenderer()
this.camera = engine.createCamera()
this.view = engine.createView()
this.view.setCamera(this.camera)
this.view.setScene(this.scene)
this.view.setClearColor([0.1, 0.2, 0.3, 1.0])
this.resize()
```
目前我们已经创造出了所有的Filament个体。然而，canvas依然是空白的。

6. 渲染和调整大小
render方法会每秒调用60次
```
render() {
    // TODO: render scene
    window.requestAnimationFrame(this.render);
}
```

让我们在render方法内渲染旋转着的三角形
```
// Rotate the triangle.
const radians = Date.now() / 1000;
const transform = mat4.fromRotation(mat4.create(), radians, [0, 0, 1]);
const tcm = this.engine.getTransformManager();
const inst = tcm.getInstance(this.triangle);
tcm.setTransform(inst, transform);
inst.delete();

// Render the frame.
this.renderer.render(this.swapChain, this.view);
```
render方法内的前一部分代码获取了要进行旋转的三角形个体，并使用gl-matrix库创建出了一个旋转矩阵。
后一部分代码在视图上启用了Filament的renderer（渲染器），并且让Filament引擎执行它内部缓冲区中的代码。

最后一步，把以下代码加入resize方法。当窗口尺寸变化时，会自动调整渲染层的分辨率,用*devicePixelRatio*将高分辨率屏幕考虑在内。同时会相应地调整摄像机的透视投影。
```
const dpr = window.devicePixelRatio
const width = this.canvas.width = window.innerWidth * dpr
const height = this.canvas.height = window.innerHeight * dpr
this.view.setViewport([0, 0, width, height]);

const aspect = width / height;
const Projection = Filament.Camera$Projection;
this.camera.setProjection(Projection.ORTHO, -aspect, aspect, -1, 1, 0, 1);
```
##WebAssembly
The core Filament library can be cross-compiled to WebAssembly from either macOS or Linux. To get started, follow the instructions for building Filament on your platform (macOS or linux), which will ensure you have the proper dependencies installed.
Next, you need to install the Emscripten SDK. The following instructions show how to install the same version that our continuous builds use.

```
cd <your chosen parent folder for the emscripten SDK>
curl -L https://github.com/emscripten-core/emsdk/archive/a77638d.zip > emsdk.zip
unzip emsdk.zip
mv emsdk-* emsdk
cd emsdk
./emsdk update
./emsdk install sdk-1.38.28-64bit
./emsdk activate sdk-1.38.28-64bit
After this you can invoke the easy build script as follows:
export EMSDK=<your chosen home for the emscripten SDK>
./build.sh -p webgl release
```

The EMSDK variable is required so that the build script can find the Emscripten SDK. The build creates a samples folder that can be used as the root of a simple static web server. Note that you cannot open the HTML directly from the filesystem due to CORS. One way to deal with this is to use Python to create a quick localhost server:
cd out/cmake-webgl-release/web/samples
python3 -m http.server     # Python 3
python -m SimpleHTTPServer # Python 2.7
You can then open http://localhost:8000/suzanne.html in your web browser.
Alternatively, if you have node installed you can use the live-server package, which automatically refreshes the web page when it detects a change.
Each sample app has its own handwritten html file. Additionally the server folder contains assets such as meshes, textures, and materials.

.. _building-for-the-web:

Building for the Web
====================

Overview
--------

Panda3D is able to run on the web using emscripten to compile the Panda3D and Python libraries to WebAssembly, a binary intermediate language which can be JIT-compiled by modern browsers and executed by their JavaScript virtual machine. The instructions assume you are on Linux or a UNIX OS. Windows will be added later.

You need to be prepared to deal with build systems, as well as have some basic knowledge of C to be able to adjust the code that bootstraps the application.

You will need to download and install the following software on the host system:

    Git
    Emscripten SDK
    Python (at least 3.8)
    Panda3D SDK (at least 1.10.9)

Compiling Panda3D
-----------------

Check out the WebGL branch from GitHub using this command:

git clone -b webgl-port https://github.com/panda3d/panda3d.git panda3d-webgl
cd panda3d-webgl

Now, get this zip file containing the precompiled dependencies and samples and extract it to the root of the Panda3D checkout:

wget https://rdb.name/webgl-editor-and-dependencies.zip
unzip webgl-editor-and-dependencies.zip

I am assuming you have set up and configured the Emscripten SDK at this point. Now you can go ahead and compile Panda3D:

source /home/rdb/local/src/emsdk/emsdk_env.sh
python3.8 makepanda/makepanda.py --nothing --use-python --use-vorbis --use-bullet --use-zlib --use-freetype --use-harfbuzz --use-openal --no-png --use-direct --use-gles2 --optimize 4 --static --target emscripten --threads 4

Before you move on, please check the output to make sure that the build ended successfully. If you run into errors, feel free to ask on the forums or on the Panda3D Discord server.

Compiling the demos

The zip file also contains two examples, the editor application and Roaming Ralph. Both include a freezify.py script that does the actual compilation for the web. Please study this file and modify it to correct the paths for your configuration.

Let's go ahead and compile it, which again needs to happen with Python 3.8:

cd editor
python3.8 -OO freezify.py

If it succeeded, we should now be able to view the demo in the browser. It needs to be hosted on a server for this. The simplest web server is bundled with Python (any version is fine), for example:

python -m http.server

…however, I do not recommend this! While it will work, the server that ships with Python lacks several features important for getting good results, including the ability to serve gzip-compressed files and serve the correct MIME type for WebAssembly files. If you have npm, this is an alternative web server that will work:

npm install -g http-server
http-server

Then, navigate to the server's address in your browser and locate the .html file in the demo's directory. This .html file is a container around the application (you can modify it however you wish) that loads the compiled application from the .js and .wasm files.
Troubleshooting

Panda3D prints error messages to the JavaScript console. In most browsers, press F12 to open it.

So far we've compiled everything in optimized mode, to get the most compact build and the best performance. If the application crashes, however, it is nearly impossible to find out what went wrong. Two things you can do to improve this are:

    Raise the ASSERTION level to 1 or 2 in freezify.py
    Compile Panda3D with --optimize 3 flag, or even lower.

There is a considerable performance and size penalty to doing this.

A common error looks something like this:

Aborted(stack overflow (Attempt to set SP to 0x00803e70, with stack limits [0x00804280 - 0x00814280]). If you require more stack space build with -sSTACK_SIZE=<bytes>)

If you get this, just do as it says - increase the stack size in the freezify.py file.

If you can't figure out the errors, let me know and I'll help.
Suggested Modifications

You will probably want to at least change these things to suit your own application:

    Modify the .html file to change the container webpage around the application.
    Modify the C code inside freezify.py to bootstrap the Python interpreter and extension modules however you need.
    Modify the Panda3D build to include and/or exclude features as needed by your application.

Best Practices

Here are some recommendations to improve performance and user experience when porting or designing an application for the web:
Enable gzip compression

It is important that you serve your application on a web server that supports serving files with gzip compression. This will significantly cut down on the time needed to start up the application and load resources. Check and double-check that the server is actually serving gzip-compressed files.
Limit HTTP requests

Any time that Panda3D can't find a file in the preload, it will try to load it from the server. That means it has to make an AJAX request to the server. You will want to eliminate any unnecessary HTTP requests. For that, heed the following recommendations:

Disable the default model extension, which will cause Panda3D to look for a file with the .egg extension when loading a file without an extension. In fact, you probably don't want to be using .egg files at all, and don't even build Panda3D with EGG support, instead using compiled .bam files.

Don't use automatic .pz detection, which causes Panda3D to look for a file with the .pz extension automatically. In fact, don't use pzip files at all, enable gzip compression on your web server instead.

Use as few directories on the model-path as possible. The default model path just includes the current directory, which should be adequate.

These PRC settings should reflect the above:

default-model-extension
vfs-implicit-pz false
model-path .

Furthermore, if you read from the web, it is recommended to bundle files together using multifiles (see the Panda3D manual) or zip files and mount those to the Panda3D VFS. Implicitly loading from multifiles should work, but is again not recommended because of the extra HTTP requests.
Loading Textures

When loading big textures, it is best to load them asynchronously using the browser. Panda3D supports this, but can't initiate this automatically yet. You can use the emscripten module (documented below) to initiate a texture load with image preloading:

import emscripten

def onload(file):
    texture = loader.load_texture(Filename('.', file))
    card.set_texture(texture)

def onerror(file):
    print(f"Download failed for {file}.")

url = "./big-texture.png"
handle = emscripten.async_wget(url, "target.png", onload, onerror)

This will create a file called target.png in emscripten's virtual file system (not Panda's!). Emscripten's VFS is mounted into Panda's VFS at the current directory by default, not as the filesystem root, hence the filename finagling.

To make sure the texture is decoded by the browser and not by Panda, you may need to add --use-preload-plugins to the flags in freezify.py.

WebGL is poorer at handling non-power-of-2 textures than plain OpenGL. Please ensure that your textures are sized appropriately.

Supplemental Modules

I wrote two additional Python extension modules specifically for interacting with the browser and the Emscripten runtime. They can be used to fill gaps in the Panda3D API, or to interact with the webpage hosting the application.
Including the modules

To enable these modules, download them from here. They are independent, so you can choose either or if you want.

    emscriptenmodule.c
    browsermodule.c

The easiest way to include them is to place them in the directory of freezify.py and add an #include line to the embedded C code:

#include "emscriptenmodule.c"
#include "browsermodule.c"

You also need to add them to sys.modules after initializing the Python interpreter, like so:

PyObject *sys_modules = PySys_GetObject("modules");
PyDict_SetItemString(sys_modules, "emscripten", PyInit_emscripten());
PyDict_SetItemString(sys_modules, "browser", PyInit_browser());

Lastly, you need to make sure that the --bind flag is passed to the link step of emcc.
Interacting with JavaScript APIs

The browser module allows you to write Python code that directly interfaces with browser APIs, as though you were writing JavaScript. You can write code that looks like this:

from browser import window, console, document

color = window.prompt("Pick your favorite HTML color:", "#ff0000")
if not color:
    color = "red"

header = document.getElementById("header")
header.style.backgroundColor = color

console.log("Writing to the console!")

print(window.eval("1 + 2"))

The support is quite comprehensive. Things that will work:

    Calling JavaScript functions
    Accessing and assigning properties of JS objects
    Passing primitive types to JS (ints, strings, floats, bools, None)
    Passing Python callback functions to JS functions (note: no reference is kept to the callback, so you need to retain it on the Python end)
    Catching JavaScript exceptions
    Rich comparison of JavaScript objects
    Iterating over JavaScript iterable objects
    Awaiting JavaScript promises inside Python coroutines

The major limitation is that you can't pass arbitrary Python objects to the JavaScript runtime. This is because there is no way to manage the reference count of a Python object from JavaScript.

Most mutable types, such as arrays and dictionaries, can't be passed directly from Python. Instead, you need to construct the JavaScript equivalent and fill it in, as follows:

from browser import Object, Array

o = Object()
o.key = value

a = Array(1, 2, 3)

Python doesn't have a new keyword. For many JavaScript objects, use of new is optional, but for some it's required. In those cases, you can use Reflect.construct() as an alternative. For example, if you want to create an XMLHTTPRequest object:

from browser import Reflect, Array, XMLHttpRequest

req = Reflect.construct(XMLHttpRequest, Array())

Asynchronous Loading

A major shortcoming is that loading resources asynchronously through the Panda3D API is not very easy right now. This will be remedied in the future. You can, however, use the JavaScript fetch or XHR APIs via the browser module, or the Emscripten wget API via the emscripten module.

I showed in an earlier section how to load textures asynchronously. For arbitrary data, there is the fetch API, which is best used in conjunction with Panda3D coroutines:

from direct.showbase.ShowBase import ShowBase
from browser import Object, fetch

async def fetchroutine(url):
    # optional options object
    options = Object()
    options.method = "GET"

    response = await fetch(url, options)
    if response.ok:
        print(await response.text())
    else:
        print("Error", response.status)

base = ShowBase()
base.taskMgr.add(fetchroutine("./test.txt"))
base.taskMgr.add(fetchroutine("./test2.txt"))
base.run()

Asyncify

You can experimentally use asyncify to improve responsiveness, but I don't recommend this for now. There is a considerable performance overhead and there is no easy way right now to get Emscripten to asyncify only the interesting parts of an application (such as the Panda3D loader and task manager).


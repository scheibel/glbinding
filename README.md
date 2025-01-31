![glbinding](glbinding-logo.png)

*glbinding* is a generated, cross-platform C++ binding for OpenGL which is solely based on the new xml-based OpenGL API specification ([gl.xml](https://cvs.khronos.org/svn/repos/ogl/trunk/doc/registry/public/api/gl.xml)). It is a fully fledged OpenGL API binding compatible with current code based on other C bindings, e.g., [GLEW](http://glew.sourceforge.net/). The binding is generated using python scripts and templates, that can be easily adapted to fit custom needs.
*glbinding* can be used as an alternative to GLEW and other projects, e.g., [glad](https://github.com/Dav1dde/glad), [gl3w](https://github.com/skaslev/gl3w), [glLoadGen](https://bitbucket.org/alfonse/glloadgen/overview), [glload](http://glsdk.sourceforge.net/docs/html/group__module__glload.html), and [flextGL](https://github.com/ginkgo/flextGL). *glbinding* is licenced under the [MIT license](http://opensource.org/licenses/MIT)

[//]: # (Comment)

The latest release is [glbinding-1.1.0](https://github.com/hpicgs/glbinding/releases/tag/v1.1.0).

*glbinding* leverages modern C++11 features like enum classes, lambdas, and variadic templates, instead of relying on macros (all OpenGL symbols are real functions and variables). It provides [type-safe parameters](#type-safe-parameters), [per feature API header](#per-feature-api-header), [lazy function resolution](#lazy-function-pointer-resolution), [multi-context](#multi-context-support) and [multi-thread](#multi-threading-support) support, [global](#function-callbacks) function callbacks, [meta information](#meta-information) about the generated OpenGL binding and the OpenGL runtime, as well as multiple [tools](https://github.com/hpicgs/glbinding/wiki/tools) and [examples](https://github.com/hpicgs/glbinding/wiki/examples) for quick-starting your projects. 

Current gl code, written using a typical C binding for OpenGL is fully compatible for the use with *glbinding*.
Just replace all includes to the old binding and use the appropriate api namespace, e.g., ```gl```: 

```cpp
#include <glbinding/gl/gl.h>
#include <glbinding/Binding.h>

using namespace gl;

int main()
{
  // create context, e.g. using GLFW, Qt, SDL, GLUT, ...

  glbinding::Binding::initialize();

  glBegin(GL_TRIANGLES);
  // ...
  glEnd();
}
```

## Features

##### Type-Safe Parameters

Every parameter of an OpenGL function expects a specific data type and *glbinding* enforces, if possible, this type in its interface. This results in the following behavior:

```cpp
glClear(GL_COLOR_BUFFER_BIT); // valid
glClear(GL_FRAMEBUFFER);      // compilation error: bitfield of group ClearBufferMask expected, got GLenum
```

For bitfields there are extensively specified groups that are additionally used to enforce type-safety (note: a bitfield value can be used by several groups). Combinations of bitfields that share no group results in a compilation error.

```cpp
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // valid
glClear(GL_COLOR_BUFFER_BIT | GL_MAP_READ_BIT);     // compile error: both bitfields share no group

glClear(GL_STENCIL_BUFFER_BIT | GL_LIGHTING_BIT);   // compile error: bitwise or operation is valid, 
                                                    //  the shared group is AttribMask, but the
                                                    //  resulting group does not match the expected.
```

The groups for enums are not yet as complete as we would like them to be to enable per function compile-time errors when trying to call functions with values from the wrong enum group. For example, ```GL_VERTEX_SHADER``` is in the group ```ShaderType``` and ```GL_COMPUTE_SHADER``` is not.

##### Per Feature API Header

Enums, bitfields, and functions can be included as usual in a combined ```gl.h``` header or individually via ```bitfield.h```, ```enum.h```, and ```functions.h``` respectively. Additionally, these headers are available for  feature-based API subsets, each using a specialized namespace, e.g.:

* ```gl32/functions.h``` provides all OpenGL commands available up to 3.2 in namespace ```gl32```.
* ```gl32core/functions.h``` provides all non-deprecated OpenGL commands available up to 3.2 in namespace ```gl32core```.
* ```gl32ext/functions.h``` provides all OpenGL commands specified either in 3.3 and above, or by extension in ```gl32ext```.

Depending on the intended use-case, this allows to limit coding to a specific OpenGL feature and reduces switching to other features to a replacement of includes and using directives. In both cases, non-featured symbols do not compile.

Furthermore, *glbinding* provides explicit, non-feature dependent headers for special values (```values.h```), booleans (```boolean.h```), and basic types (```types.h```). This allows for refined includes and can reduce compile time.


##### Lazy Function Pointer Resolution

By default, *glbinding* tries to resolve all OpenGL function pointers during initialization, which can consume some time:

```cpp
glbinding::Binding::initialize(); // immediate function pointer resolution
```

Alternatively, the user can decide that functions pointers are resolved only when used for the first time. This is achieved by:

```cpp
glbinding::Binding::initialize(false); // lazy function pointer resolution
```

##### Multi-Context Support

*glbinding* has built-in support for multiple contexts. The only requirement is, that the currently active context has to be specified. This feature mixes well with multi-threaded applications, but keep in mind that concurrent use of one context often result in non-meaningful communication with the OpenGL driver.

To use multiple contexts, use your favorite context creation library (e.g., glut, SDL, egl, glfw, Qt) to request as much contexts as required. The functions to make a context current should be provided by this library and is not part of *glbinding* (except that you can get the current context handle). When using multiple contexts, first, each has to be initialized when active: 

```cpp
// use context library to make current, e.g., glfwMakeCurrent(...)
glbinding::Binding::initialize();
```

Second, contexts switches are required to be communicated to *glinding* explicitly in order to have correctly dispatched function pointers:

```cpp
// use the current active context
glbinding::Binding::useCurrentContext();

// use another context, identified by platform-specific handle
glbinding::Binding::useContext(ContextHandle context); 
```

This feature is mainly intended for platforms where function pointers for different requested OpenGL features may vary.


##### Multi-Threading Support

Concurrent use of *glbinding* is mainly intended to the usage of multiple contexts in different threads (multiple threads operating on a single OpenGL context requires locking, which *glbinding* will not provide).
For this, *glbinding* supports multiple active contexts, one per thread. This necessitates that *glbinding* gets informed in each thread which context is currently active (see [multi-context](#multi-context-support)).


##### Function Callbacks

*glbinding* supports different types of callbacks that can be registered.
The main types are

 * Global before callbacks, that are called before the OpenGL function call
 * Per function before callbacks
 * Global after callbacks, that are called after the OpenGL function call
 * Per function after callbacks
 * Unresolved callbacks, that are called each time an unresolved OpenGL function should be called (instead of a segmentation fault)

The before callbacks are useful , e.g., for tracing or application-specific parameter checking.
The available informations in this callback are the wrapped OpenGL function (including its name and bound function address) and all parameters.
The after callbacks are useful, .e.g., for tracing, logging, or the obligatory error check.
Available informations are extended by the return value.
The unresolved callback provides information about the (unresolved) wrapped OpenGL function object.

Example for error checking:

```cpp
#include <iostream>

#include <glbinding/callbacks.h>

using namespace glbinding;

// ...
setCallbackMaskExcept(CallbackMask::After, { "glGetError" });
setAfterCallback([](const FunctionCall &)
{
  GLenum error = glGetError();
  if (error != GL_NO_ERROR)
    std::cout << "error: " << std::hex << error << std::endl;
});

// ... OpenGL code
```

Example for logging:

```cpp
#include <iostream>

#include <glbinding/callbacks.h>

using namespace glbinding;

// ...
setCallbackMask(CallbackMask::After | CallbackMask::ParametersAndReturnValue);
glbinding::setAfterCallback([](const glbinding::FunctionCall & call)
{
  std::cout << call.function->name() << "(";
  
  for (unsigned i = 0; i < call.parameters.size(); ++i)
  {
    std::cout << call.parameters[i]->asString();
    if (i < call.parameters.size() - 1)
      std::cout << ", ";
  }
  
  std::cout << ")";
  
  if (call.returnValue)
  {
    std::cout << " -> " << call.returnValue->asString();
  }
  
  std::cout << std::endl;
});

// ... OpenGL code
```

Example for per function callbacks:

```cpp
#include <iostream>

#include <glbinding/Binding.h>

using namespace glbinding;

// ...
Binding::ClearColor.setBeforeCallback([](GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha) {
    std::cout << "Switching clear color to ("
        << red << ", " << green << ", " << blue << ", " << alpha << ")" << std::endl;
});

// ...
glClearColor(0.5, 0.5, 0.5, 1.0);
// Output: Switching clear color to (0.5, 0.5, 0.5, 1.0)
```

It is also possible to call an OpenGL function without triggering registered callbacks (except unresolved callbacks), using the ```directCall()``` member function.

```cpp
#include <glbinding/Binding.h>

using namespace glbinding;

// ...
GLenum errorCode = Binding::GetError.directCall();
```

##### Context-Switch Callbacks

When switching between active contexts, not only *glbinding* may be interested in the current context, but your application as well (e.g., per context cached OpenGL state).
You may also not know when contexts will get changed (especially if you write a library) and propagating the current context may be troublesome.
Therefore, you can register one callback that is called when the current active context in *glbinding* is changed.

```cpp
#include <iostream>

#include <glbinding/Binding.h>

using namespace glbinding;

// ...
Binding::addContextSwitchCallback([](ContextHandle handle) {
    std::cout << "Switching to context " << handle << std::endl;
});
```

##### Meta Information

Besides an actual OpenGL binding, *glbinding* also supports queries for both compile time and run time information about the gl.xml and your OpenGL driver.
Typical use cases are querying the available OpenGL extensions or the associated extensions to an OpenGL feature and their functions and enums.

Example list of all available OpenGL versions/features (compile time):

```cpp
#include <iostream>

#include <glbinding/Meta.h>
#include <glbinding/Version.h>

using glbinding::Meta;

for (const Version & v : Meta::versions())
  std::cout << v.toString() << std::endl;
```

Example output of all enums (compile time):

```cpp
#include <iostream>
#include <iomanip>

#include <glbinding/Meta.h>

using glbinding::Meta;

if (Meta::extensive())
{
  std::cout << "# Enums: " << Meta::enums().size() << std::endl << std::endl;

  for (GLenum e : Meta::enums()) // c++ 14 ...
    std::cout << " (" << std::hex << std::showbase << std::internal << std::setfill('0') << std::setw(8) 
              << static_cast<std::underlying_type<GLenum>::type>(e) << ") " << Meta::getString(e) << std::dec << std::endl;

  std::cout << std::dec;
}
```

Example output of all available extensions (run time):

```cpp
#include <iostream>

#include <glbinding/Meta.h>

using glbinding::Meta;

if (Meta::stringsByGL())
{
  std::cout << " # Extensions: " << Meta::extensions().size() << std::endl << std::endl;

  for (GLextension e : Meta::extensions())
  {
    const Version v = Meta::getRequiringVersion(e);
    std::cout << " " << Meta::getString(e) << " " << (v.isNull() ? "" : v.toString()) << std::endl;
  }
}
```

##### Performance

*glbinding* causes no significant impact on runtime performance. The provided comparison example supports this statement. It compares the execution times of identical rendering code, dispatched once with *glbinding* and once with glew. Various results are provided in the [Examples](https://github.com/hpicgs/glbinding/wiki/examples) wiki.


##### Binding Generation

As a user of glbinding you are able to update the gl.xml by yourself and generate the glbinding code.
The necessary python scripts are provided in this repository. Since the ```gl.xml``` is not complete, a ```patch.xml``` is used to resolve possible conflicts or missing specifications. With ongoing development of the xml-based OpenGL API specification this could become obsolete in the future.
For ease-of-use, the update and generation can be triggered using the generated targets from cmake named *update* and *generate*.


## Context Creation Cheat Sheet

When requesting an OpenGL context of a specific version, the created context does not always match that version, but instead returns a context with "appropriate" capabilities. The mapping of requested and created version depends on various aspects, e.g., forward compatibility and core flags, context creation library, driver, graphics card, and operating system. To get some understanding of that mapping a [Context Creation Cheat Sheet](https://github.com/hpicgs/glbinding/wiki/Context-Creation-Cheat-Sheet) is provided, gathering the ouput of glbindings contexts example.


## Using glbinding


##### Project Health (master)

| Service | System | Compiler | Targets | Status |
| ------- | ------ | -------- | ------- | ------ |
|  [Drone](https://drone.io/github.com/cginternals/glbinding) | Ubuntu 12.04 | GCC 4.8 | all (no qt), test | [![Build Status](https://drone.io/github.com/cginternals/glbinding/status.png)](https://drone.io/github.com/cginternals/glbinding/latest) |
| Jenkins | Ubuntu 14.04 | GCC 4.8 | all, test | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-linux-gcc4.8)](http://jenkins.hpi3d.de/job/glbinding-linux-gcc4.8)|
| Jenkins | Ubuntu 14.04 | GCC 4.9 | all, test | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-linux-gcc4.9)](http://jenkins.hpi3d.de/job/glbinding-linux-gcc4.9)|
| Jenkins | Ubuntu 14.04 | GCC 5.3 | all, test | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-linux-gcc5.3)](http://jenkins.hpi3d.de/job/glbinding-linux-gcc5.3)|
| Jenkins | Ubuntu 14.04 | Clang 3.5 | all, test | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-linux-clang3.5)](http://jenkins.hpi3d.de/job/glbinding-linux-clang3.5) |
| Jenkins | OS X 10.10 | Clang 3.5 | - | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-osx-clang3.5)](http://jenkins.hpi3d.de/job/glbinding-osx-clang3.5) |
| Jenkins | Windows 8.1 | MSVC 2013 Update 5 | all, test, install | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-windows-msvc2013)](http://jenkins.hpi3d.de/job/glbinding-windows-msvc2013) |
| Jenkins | Windows 8.1 | MSVC 2015 Update 1 | all, test, install | [![Build Status](http://jenkins.hpi3d.de/buildStatus/icon?job=glbinding-windows-msvc2015)](http://jenkins.hpi3d.de/job/glbinding-windows-msvc2015) |
| [Coverity](https://scan.coverity.com/projects/6828?tab=overview) | Ubuntu | GCC 4.8 | all| [![Coverity Status](https://scan.coverity.com/projects/6828/badge.svg)](https://scan.coverity.com/projects/6828) |


##### Dependencies

The only run-time dependencies of glbinding are the STL of the used compiler and an OpenGL library, dynamically linked with your application.

Optional dependencies

* Python 2 or 3 to generate the binding
* Qt for some examples
* GLFW 3 for some examples
* GLEW for some examples

For building *glbinding* CMake 3.2 or newer and a C++11 compliant compiler (e.g. GCC 4.8, Clang 3.3, MSVC 2013 **Update 3**) are required.

When configuring *glbinding*, the options ```OPTION_BUILD_EXAMPLES``` (disabled by default) can be used to enable examples.
The tools of glbinding are build automatically if GLFW is found. Furthermore, ```OPTION_EXTENSIVE_META``` (enabled by default) allow string-to-symbol and symbol-to-string conversion (in ```glbinding::Meta```). Disable ```OPTION_EXTENSIVE_META``` to decrease build time and binary size.

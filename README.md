# cefpatches
# cef97源码版本信息
各部分代码的版本信息
- depot_tools版本
```
Revision: f7b8f8f3cd77ecbd16c932525f1328110b9a4ca5
Author: Vadim Shtayura <vadimsh@chromium.org>
Date: 2021/11/16 3:10:05
Message:
Fix bytes vs str error in DownloadGerritHook.

It fails with: `write() argument must be str, not bytes`.

R=sokcevic@google.com

Change-Id: If6787140ea863ce3761a5c3747eb0b857b22fa47
Reviewed-on: https://chromium-review.googlesource.com/c/chromium/tools/depot_tools/+/3276505
Reviewed-by: Josip Sokcevic <sokcevic@google.com>
Commit-Queue: Vadim Shtayura <vadimsh@chromium.org>
----
Modified: git_cl.py

```
- chromium版本97.0.4692.20
```
Revision: d60c10f0af397f6366c958dddb075d630e07c5d3
Author: Chrome Release Bot (LUCI) <chrome-official-brancher@chops-service-accounts.iam.gserviceaccount.com>
Date: 2021/11/16 9:31:53
Message:
Publish DEPS for 97.0.4692.20

----
Modified: DEPS

```
- cef版本4692
```
Revision: 444ce60075405dd75e82b39de7143e6b2d1e41e3
Author: Nicolas Dusart <dusartnicolas@gmail.com>
Date: 2021/11/24 6:14:59
Message:
Fix CefURLRequest crash with failing HEAD requests (fixes issue #3226)

----
Modified: libcef/browser/net_service/browser_urlrequest_impl.cc

```
# 0001_dx11sharetexture给cef97添加dx11纹理支持
原理参考:

cef开发文档，参考[https://www.chromium.org/developers/](https://www.chromium.org/developers/)

添加纹理支持的步骤 :

1. 添加javascript接口text
    - 原理参考[https://www.chromium.org/developers/web-idl-interfaces/](https://www.chromium.org/developers/web-idl-interfaces/)
    - 
2. 添加gpu command处理
    - 原理参考[https://www.chromium.org/developers/design-documents/gpu-command-buffer/](https://www.chromium.org/developers/design-documents/gpu-command-buffer/)
    - 例子参考[https://codereview.chromium.org/8772033/](https://codereview.chromium.org/8772033/)
3. angle添加接口实现


本次实现的功能描述：
能实现在web上播放自己生产的视频，可以支持多路视频。
实现：
将需要渲染的视频帧写到共享纹理，在Canvas对应的Webgl中绑定共享纹理，读取共享纹理进行渲染，添加web接口，给到页面绑定共享纹理。

## 1. 添加javascript接口texBindSharedHandle
公开为Javascript对象的Web接口，通常是由Web IDL(Interface Definition Language接口定义语言)指定。Web IDL是陈述性语言（有时没不够空间时，也写为WebIDL）。这是用在标准规范中的语言。Blink用IDL 文件来指定接口，并生成JavaScipt绑定（具体形式上，是V8 JavaScript虚拟机用来调用Blink本身的C++代码）。Blink中的Web IDL 接近标准，生成的绑定使用标准约定来调用Blink代码，但还有其他功能可以指定实现细节，主要是Blink IDL扩展属性。
在Blink中实现一个新的WebIDL接口：
- 接口：写一个IDL file: fool.idl
- 实现：写一个C++文件和头文件：foo.cc，foo.h
- 构建：加入将要构建的文件：编辑idl_in_core.gni/idl_in_modules.gni 和 generated_in_core.gni/generated_in_modules.gni
- 测试：在[web_tests](https://source.chromium.org/chromium/chromium/src/+/HEAD:third_party/blink/web_tests/)写单元测试(web tests)

这只我们只在现有接口Weg_render_context上新加一个方法，所以只需要修改对应的文件即可，不需要新增文件。涉及到需要修改的文件为
- src/third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.idl
- src/third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.h
- src/third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.cc
### webgl_rendering_context_base.idl
在该文件上添加web接口规范。该文件主要用于生成对应的Javascript绑定。
```C++
[CallWith=ExecutionContext, RaisesException] void texBindSharedHandle(GLintptr handle);
```
### webgl_rendering_context_base.h
在该文件添加方法texBindSharedHandle声明
```c++
void texBindSharedHandle(ExecutionContext*, int64_t handle, ExceptionState&);
```
### webgl_rendering_context_base.cc
在该文件添加方法texBindSharedHandle定义
```C++
void WebGLRenderingContextBase::texBindSharedHandle(ExecutionContext*,
                                                    int64_t handle,
                                                    ExceptionState&) {
  if (isContextLost())
    return;
  ContextGL()->TexBindSharedHandle((GLintptr)handle);
}
```
## 2. 添加gpu command处理
Gpu Command生成和处理，主要是在render进程中根据业务需要生成GPU命令，然后命令传到GPU进程中，在GPU进程进行具体的GPU操作。本功能中，是要将业务传的共享纹理句柄传入，以便在gpu进程中，能读取该共享纹理的数据进程渲染。

GPU Command的原理参考[https://www.chromium.org/developers/design-documents/gpu-command-buffer/](https://www.chromium.org/developers/design-documents/gpu-command-buffer/).

## GPU Command Buffer原理
GPU Command Buffer实现原理：
其基本实现是一个叫"command buffer"( 命令缓冲)。客户端（render进程、pepper plugin等）写命令到一些共享内存。其更新一个'put'指针，通过IPC告诉GPU进程 它已经写到缓冲区的多远的位置了。GPU进程 或者是服务会从buffer（缓冲区）中读取这些命令。对于每个命令，它都会验证命令、参数以及参数是否适合操作系统图形API的当前状态 ，然后才对操作系统进行实际调用。This means even a compromised renderer running native code, writing its own commands, can hopefully not get the GPU process to call the graphics system in such a way as to compromise the system.
在编写新的服务端代码时，请记住这一点。永远不要设计一个要求客户端表现良好的新命令。要假定客户端可以变得无赖。例如，确保无论客户做什么，服务的记账都不会出错。

API层的实现：

简要说：
gl2.h->gles2_c_lib.cc->GLES2Implemetation->GLES2CmdHelper...SharedMemory...->GLES2DecoderImpl->ui/gfx/gl/gl_bindings->OpenGL

有一个接口CommandBuffer，负责协调GLES2CmdHelpe和GLES2DecoderImpl之间的通信。它有创建和删除共享内存的方法，以及来回通信当前状态的方法。特别是通过AsyncFlush()或Flush()从客户端发送最新的'put'指针，并通过'Flush'的结果获取最新的"get"指针。

一个名为CommandBufferService的CommandBuffer实现直接与GLES2DecoderImpl对话。如果你有一个单线程单进程的chrome，你可以将CommandBufferService的一个实例传递给GLES2CmdHelper，该想法是一切都会正常运行。在真正的多进程chrome中，还有另一种实现，CommandBufferProx，它使用IPC通过GpuCommandBufferStub到GpuSchedler再到CommandBufferService从客户端到服务进程通信。

### 客户端代码（在render进程）
注意： 在src/gpu/command_buffer/client and src/gpu/command_buffer/common 中的所有代码都必须不依赖任何额外的库进行编译，因为他们是被用在不信任的 Pepper plugin中的。
以khronos为例（注意本dx11共享纹理patch项目实现用的是angle）

- 定义public OpenGL ES 2.0接口

src/third_party/khronos/GLES2/gl2.h

src/third_party/khronos/GLES2/gl2ext.h

- 以下定义C接口。大多数是自动生成的

src/gpu/command_buffer/client/gles2_c_lib.cc

src/gpu/command_buffer/client/gles2_c_lib_autogen.h

- 以下是命令到command buffer的实现客户端侧的实现。大部分是自动生成的。

src/gpu/command_buffer/client/gles2_implementation.cc

src/gpu/command_buffer/client/gles2_implementation_autogen.h

- 这是一个主要是自动生成的类，用于帮助格式化命令。

src/gpu/command_buffer/client/gles2_cmd_helper.h

src/gpu/command_buffer/client/gles2_cmd_helper_autogen.h

- 这些定义了命令的实际格式

src/gpu/command_buffer/common/cmd_buffer_common.h

src/gpu/command_buffer/common/gles2_cmd_format.h

src/gpu/command_buffer/common/gles2_cmd_format_autogen.h

### 服务端代码（在gpu进程）
- 这是读取命令、验证和调用OpenGL的代码。

src/gpu/command_buffer/service/gles2_cmd_decoder.cc

src/gpu/command_buffer/service/gles2_cmd_decoder_autogen.cc

## 3. angle添加接口实现_复制共享纹理

## global-metadata.dat文件加密
Unity游戏包体逆向会用到global-metadata.dat，因此通过对它进行加密操作，然后在il2cpp代码中解密，从而完成App加密
这涉及到对il2cpp静态库的编译，分iOS和Android两种情况：
##### Android
- 直接修改il2cpp源码，Unity打包
#### iOS
- 只能将il2cpp源码编译生成静态库libil2cpp.a文件，然后在Xcode工程替换libil2cpp.a

## Xcode编译libil2cpp.a
- 源码位置：/Applications/Unity/Hub/Editor/**2019.4.28f1c1（Your Unity Version）**/Unity.app/Contents/il2cpp
###自己动手
新建Xcode static library工程，Build Setting修改：
- Search headers
- other linker flags
- 宏定义
- 等

#### 自己动手

Unity2019版本，各种尝试设置，最终可以编译成功，Xcode静态库工程如下：
[unity il2cpp ios 构建xcode工程编译](https://git.youle.game/tkw/client/il2cppxcodeproject)

**参考链接：**
> [unity il2cpp ios 构建xcode工程编译](https://blog.csdn.net/leilonghao/article/details/121625414)
>[unity il2cpp 源码编译](https://blog.csdn.net/leilonghao/article/details/121625414) 
[Unity Global-Metadata Hiding](https://floe-ice.cn/archives/55) 
[Unity 分析global-metadata.dat 原理](https://zhuanlan.zhihu.com/p/622597028) 
[Unity保护之il2cpp](https://nszdhd1.github.io/2020/07/03/Unity%E4%BF%9D%E6%8A%A4%E4%B9%8Bil2cpp/) 

#### 换版本
当切换到Unity2021时，由于il2cpp代码结构的变化，导致原工程无法使用，仿照设置相关也不行，不管怎么尝试仍然有很多诡异的问题，比如：
- 好端端的代码他不识别了
- 明明可以找到但就是无法include进来

最终在网页搜索结果看到华佗hybridclr中包括自行编译libil2cpp.a的内容：
> 当 com.code-philosophy.hybridclr 版本 < v3.2.0[​](https://hybridclr.doc.code-philosophy.com/docs/basic/buildpipeline#%E5%BD%93-comcode-philosophyhybridclr-%E7%89%88%E6%9C%AC--v320-1 "当-comcode-philosophyhybridclr-版本--v320-1的直接链接")
>除了iOS以外平台都是根据libil2cpp源码编译出目标程序,iOS平台使用提前编译好libil2cpp.a文件。Unity导出的xcode工程引用了提前生成好的libil2cpp.a，而不包含libil2cpp源码， 直接打包无法支持热更新。因此编译iOS程序时需要自己单独编译libil2cpp.a，再**替换xcode工程的libil2cpp.a文件**，接着再打包。
>**替换xcode工程中的libil2cpp.a文件请自行完成**。


#### 利用华佗build_libil2cpp.sh脚本实现编译
华佗（HybridCLR）是一个Unity c#热更方案，其原理涉及到修改il2cpp并重新编译

- 找到< v3.2.0的低版本
- 找到编译il2cpp的模块
- 借助AI研究一下他的build_libil2cpp.sh脚本和cmakefilelist，避免加料，确保编出来的是原汁原味的
- 把我的il2cpp源码放到指定位置
- 跑一下脚本，饭做好了很顺利，成功构建出苦求两天的libil2cpp.a文件
- 放到Xcode工程，报错：
```
Undefined symbol: _SystemNative_ConvertErrorPalToPlatform
Undefined symbol: Il2CppCodeGenWriteBarrier(void**, void*)
Undefined symbol: Il2CppCodeGenWriteBarrierForType(Il2CppType const*, void**, void*)
Undefined symbol: Il2CppCodeGenWriteBarrierForClass(Il2CppClass*, void**, void*)
Undefined symbol: il2cpp_codegen_get_generic_virtual_method_internal(MethodInfo const*, MethodInfo const*)
Linker command failed with exit code 1 (use -v to see invocation)
```
又试了一下Unity2022，还是报错：
```
Undefined symbol: Il2CppCodeGenWriteBarrier(void**, void*)
Undefined symbol: Il2CppCodeGenWriteBarrierForType(Il2CppType const*, void**, void*)
Undefined symbol: Il2CppCodeGenWriteBarrierForClass(Il2CppClass*, void**, void*)
Undefined symbol: il2cpp_codegen_get_generic_virtual_method_internal(MethodInfo const*, MethodInfo const*)
Linker command failed with exit code 1 (use -v to see invocation)
```
看源码：
```
#if IL2CPP_ENABLE_WRITE_BARRIERS
void Il2CppCodeGenWriteBarrier(void** targetAddress, void* object);
void Il2CppCodeGenWriteBarrierForType(const Il2CppType* type, void** targetAddress, void* object);
void Il2CppCodeGenWriteBarrierForClass(Il2CppClass* klass, void** targetAddress, void* object);
#else
inline void Il2CppCodeGenWriteBarrier(void** targetAddress, void* object) {}
inline void Il2CppCodeGenWriteBarrierForType(const Il2CppType* type, void** targetAddress, void* object) {}
inline void Il2CppCodeGenWriteBarrierForClass(Il2CppClass* klass, void** targetAddress, void* object) {}
#endif
```
- 尝试编译脚本（3个CMakeList.txt）中加上**IL2CPP_ENABLE_WRITE_BARRIERS**这个宏
```
add_definitions(-DIL2CPP_ENABLE_WRITE_BARRIERS)
```

##### Unity2022的libil2cpp可以成功编译出来，出包测试没问题 ✓
##### 换Unity 2021，同样成功，也通过出包测试 ✓
##### 此方法应该可以稳定编译Unity各版本的libil2cpp.a
- il2cpp源码位置：CompileLibil2cpp/LocalIl2CppData-OSXEditor/il2cpp
- 编译脚本位置：CompileLibil2cpp/iOSBuild/build_libil2cpp.sh
- 最终编译出静态库位置：CompileLibil2cpp/iOSBuild/build/libil2cpp.a

>如果宏 IL2CPP_ENABLE_WRITE_BARRIERS 没有被定义，那么条件编译块 #if IL2CPP_ENABLE_WRITE_BARRIERS 之后的代码将不会被编译。这意味着在这种情况下，Il2WriteBarrier、Il2WriteBarrierForType 和 Il2WriteBarrierForClass 这三个函数的声明和定义（作为空函数）将会被包含在编译过程中。
>然而，您提到的错误信息表明，尽管宏没有被定义，链接器依然期望这些函数的存在。这可能是因为：
> - 其他部分的代码引用了这些函数：即使宏没有定义，如果代码中有其他部分引用了这些函数，链接器会尝试找到它们的定义。**由于这些函数在宏未定义的情况下没有定义（只有声明）**，因此会报链接错误。
> - IL2CPP内部依赖：IL2CPP可能在内部逻辑中依赖于这些函数，即使宏没有定义，它也期望这些函数以某种形式存在。
>- 链接器配置问题：可能是因为项目配置或链接器设置不正确，导致链接器无法正确处理这些条件编译的函数。








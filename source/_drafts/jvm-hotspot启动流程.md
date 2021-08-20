---
title: jvm-hotspot启动流程
tags:
---
jvm-HotSpot其创建流程。
   
 
入口 
```java
src/java.base/share/native/launcher/main.c
```

###main.c # main()
- 创建 createVm

```cpp
int
main(int argc, char **argv)
{
    int margc;
    char** margv;
    int jargc;
    char** jargv;
    const jboolean const_javaw = JNI_FALSE;

    ...

#endif /* WIN32 */
    return JLI_Launch(margc, margv,
                   jargc, (const char**) jargv,
                   0, NULL,
                   VERSION_STRING,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   jargc > 0,
                   const_cpwildcard, const_javaw, 0);
}
```

- 初始化 InitializeJVM()

- 创建vm  Threads::create_vm
```cpp
     //TLS初始化
      ThreadLocalStorage::init();
      
      // 输入输出流初始化
      ostream_init();
      
      //os模块初始化
      os::init();
      os::init_before_ergo();
      
      //VM初始化开始
      HOTSPOT_VM_INIT_BEGIN();
      
      //初始化全局数据结构并在堆中创建系统类
      vm_init_globals();
      
      //Attach线程
      JavaThread* main_thread = new JavaThread();
      main_thread->set_thread_state(_thread_in_vm);
      main_thread->initialize_thread_current();
      
      // 线程栈基和栈大小
      main_thread->record_stack_base_and_size();
      main_thread->set_active_handles(JNIHandleBlock::allocate_block());
      // Enable guard page *after* os::create_main_thread(), otherwise it would
      // crash Linux VM, see notes in os_linux.cpp.
      main_thread->create_stack_guard_pages();
      
      //初始化Java级同步子系统
      ObjectMonitor::Initialize();
      
      //初始化全局模块
      jint status = init_globals();
  
      // Should be done after the heap is fully created
      main_thread->cache_global_variables();
      //初始化java/lang下的类
      initialize_java_lang_classes(main_thread, CHECK_JNI_ERR);
      
      //标记初始化完成
      set_init_completed();
      
      //元空间初始化
      Metaspace::post_initialize();
      HOTSPOT_VM_INIT_END();
    
    
     // Signal Dispatcher needs to be started before VMInit event is posted
      os::initialize_jdk_signal_support(CHECK_JNI_ERR);
    
      // 预先初始化一些JSR292核心类，String,System,Class类
      initialize_jsr292_core_classes(CHECK_JNI_ERR);
    
      // 这将初始化模块系统。只有java.base类可以是加载到阶段2完成
      call_initPhase2(CHECK_JNI_ERR);
    
      // 即使还没有JVMTI环境，也要始终调用，因为环境可能会延迟连接，而且JVMTI必须跟踪VM执行的各个阶段
      JvmtiExport::enter_start_phase();
    
      // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
      JvmtiExport::post_vm_start();
    
      //最终系统初始化，包括安全管理器和系统类加载器
      call_initPhase3(CHECK_JNI_ERR);
    
      // 缓存系统类加载器
      SystemDictionary::compute_java_system_loader(CHECK_(JNI_ERR));
    
      //即使还没有JVMTI环境，也要始终调用，因为环境可能会延迟连接，而且JVMTI必须跟踪VM执行的各个阶段
      JvmtiExport::enter_live_phase();
    
      //通知初始化完成
      JvmtiExport::post_vm_initialized();
   ```

- 初始化VM全局数据结构及系统类   vm_init_globals()
```cpp
void vm_init_globals() {
  check_ThreadShadow();
  basic_types_init(); //heapOopSize与压缩指针
  eventlog_init(); //一些事件
  mutex_init();// 全局监视器和互斥体
  chunkpool_init();//四个ChunkPool
  perfMemory_init();//监控内存初始化
  SuspendibleThreadSet_init();
}

```
-  初始化全局模块 **init_globals()**
```cpp
    jint init_globals() {
      HandleMark hm;
      management_init(); //监控服务，线程服务，运行时服务，类加载服务初始化
      bytecodes_init(); // 字节码解释器初始化
      classLoader_init1(); //类加载器初始化第一阶段：zip,jimage入口，设置启动类路径
      compilationPolicy_init();//编译策略，根据编译等级确定从c1,c2编译器
      codeCache_init();//代码缓存初始化
      VM_Version_init();
      os_init_globals();
      stubRoutines_init1();//例程初始化第一阶段，例程：代码调用点
      //堆空间，元空间，AOTLoader,SymbolTable,StringTable，G1收集器初始化
      jint status = universe_init(); 
      if (status != JNI_OK)
        return status;
    
      interpreter_init();  // 模板解释器初始化
      invocationCounter_init();  //热点统计初始化
      marksweep_init();//标记清除GC初始化
      accessFlags_init();
      templateTable_init();//模板表初始化
      InterfaceSupport_init();
      SharedRuntime::generate_stubs();//生成部分例程入口
      universe2_init();  // vmSymbols，系统字典，预加载类，构建基本数据对象模型
      referenceProcessor_init();//对象引用策略
      jni_handles_init();//JNI引用句柄
    #if INCLUDE_VM_STRUCTS
      vmStructs_init(); //一些相关数据结构
    #endif // INCLUDE_VM_STRUCTS
    
      vtableStubs_init();//虚表例程初始化
      InlineCacheBuffer_init();//内联缓冲初始化
      compilerOracle_init();//Oracle相关的初始化
      dependencyContext_init();//监控统计相关
    
      if (!compileBroker_init()) {
        return JNI_EINVAL;
      }
      VMRegImpl::set_regName(); //虚拟机寄存器初始化
    
      if (!universe_post_init()) {//初始化必要的类，注册堆空间到内存服务
        return JNI_ERR;
      }
      javaClasses_init();   //java基础类相关计算
      stubRoutines_init2(); // 例程初始化第二阶段，模板解释器继承体系初始化
      MethodHandles::generate_adapters();//方法与代码缓存点适配
    
      return JNI_OK;
    }

```

-  解释器的初始化
```cpp
  JNI_CreateJavaVM
 |
 |--> Threads::create_vm
 |
 |--> init_globals
 |
 |-->interpreter_init
 |
 |-->AbstractInterpreter::initialize
 |
 |-->TemplateTable::initialize
 
  start_thread()                               pthread_create.c 
    JavaMain()                                 java.c
     InitializeJVM()                           java.c  
      JNI_CreateJavaVM()                       jni.cpp
       Threads::create_vm()                    thread.cpp
        init_globals()                         init.cpp
         interpreter_init()                    interpreter.cpp
          TemplateInterpreter::initialize()    templateInterpreter.cpp
 
```



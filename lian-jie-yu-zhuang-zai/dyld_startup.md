# dyld\_startup

dyld是苹果出品的动态链接器，是MacOS和iOS平台计算机体系的核心，它负责计算机体系中的装载和链接，本篇文章探索dyld在启动程序这一过程中的角色。这一过程中涉及到了dyld、libsystem、libdispatch、objc中的代码，文章中将相关代码简化后的伪代码贴出来，可读性非常高，降低了门槛，抓大放小，主旨在了解其原理。

启动程序的主要步骤：

1. kern创建进程并装载主程序和dyld，设置栈环境，最后将接力棒交给dyld。
2. 根据栈环境，dyld自举\(bootstrap\)，最后将接力棒交给dyld主函数。
3. dyld链接主程序，进行核心系统库、objc自举，最后将接力棒交给主程序主函数。
4. 主程序进入main函数，开始主程序的运行。

一切dyld源码的入口在`__dyld_start` 中，这段汇编结束就完成了启动程序。第一步交由kern引导，这段汇编会接收接力棒并进入`dyldbootstrap::start`，开始执行第二步。

## dyldbootstrap::start

dyld作为动态链接器在发挥作用前要进行自举，这段过程就是在kern留下的栈环境下，进行地址重定位，最后将接力棒交给dyld的主函数`dyld::_main`，伪代码如下：

```text
//  This is code to bootstrap dyld.  This work in normally done for a program by dyld and crt.
//  In dyld we have to do this manually.
//
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
        intptr_t slide, const struct macho_header* dyldsMachHeader,
        uintptr_t* startGlue)
{
  // if kernel had to slide dyld, we need to fix up load sensitive locations
  // we have to do this before using any global variables
  if ( slide != 0 ) {
    rebaseDyld(dyldsMachHeader, slide);
  }
​
  // allow dyld to use mach messaging
  mach_init();
​
  // kernel sets up env pointer to be just past end of agv array
  const char** envp = &argv[argc+1];
  
  // kernel sets up apple pointer to be just past end of envp array
  const char** apple = envp;
  while(*apple != NULL) { ++apple; }
  ++apple;
​
  // now that we are done bootstrapping dyld, call dyld's main
  uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
  return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

`rebaseDyld`会对所有地址有引用的地方进行基址重置。将重定位\(relocation\)表项和`__LINKEDIT`中的non lazy indirect symbol pointers相对地址调整为绝对地址，添加slide。伪代码如下：

```text
//
// If the kernel does not load dyld at its preferred address, we need to apply 
// fixups to various initialized parts of the __DATA segment
//
static void rebaseDyld(const struct macho_header* mh, intptr_t slide)
{
  // rebase non-lazy pointers (which all point internal to dyld, since dyld uses no shared libraries)
  // and get interesting pointers into dyld
  const uint32_t cmd_count = mh->ncmds;
  const struct load_command* const cmds = (struct load_command*)(((char*)mh)+sizeof(macho_header));
  const struct load_command* cmd = cmds;
  const struct macho_segment_command* linkEditSeg = NULL;
  const struct macho_segment_command* firstWritableSeg = NULL;
  const struct dysymtab_command* dynamicSymbolTable = NULL;
  for (uint32_t i = 0; i < cmd_count; ++i) {
    switch (cmd->cmd) {
      case LC_SEGMENT_COMMAND:
        {
          const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
          if ( strcmp(seg->segname, "__LINKEDIT") == 0 )
            linkEditSeg = seg;
          const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));
          const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];
          for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {
            const uint8_t type = sect->flags & SECTION_TYPE;
            if ( type == S_NON_LAZY_SYMBOL_POINTERS ) {
              // rebase non-lazy pointers (which all point internal to dyld, since dyld uses no shared libraries)
              const uint32_t pointerCount = (uint32_t)(sect->size / sizeof(uintptr_t));
              uintptr_t* const symbolPointers = (uintptr_t*)(sect->addr + slide);
              for (uint32_t j=0; j < pointerCount; ++j) {
                symbolPointers[j] += slide;
              }
            }
          }
          if ( (firstWritableSeg == NULL) && (seg->initprot & VM_PROT_WRITE) )
            firstWritableSeg = seg;
        }
        break;
      case LC_DYSYMTAB:
        dynamicSymbolTable = (struct dysymtab_command *)cmd;
        break;
    }
    cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
  }
  
  // use reloc's to rebase all random data pointers
  const uintptr_t relocBase = firstWritableSeg->vmaddr + slide;
  const relocation_info* const relocsStart = (struct relocation_info*)(linkEditSeg->vmaddr + slide + dynamicSymbolTable->locreloff - linkEditSeg->fileoff);
  const relocation_info* const relocsEnd = &relocsStart[dynamicSymbolTable->nlocrel];
  for (const relocation_info* reloc=relocsStart; reloc < relocsEnd; ++reloc) {
    // update pointer by amount dyld slid
    *((uintptr_t*)(reloc->r_address + relocBase)) += slide;
  }
}
```

## dyld::\_main

`dyld::_main`作为dyld的主函数，核心过程都发生在这里，`dyld::_main`主要步骤：

1. 对已装载的主程序链接。
2. 对插入的动态库\(`DYLD_INSERT_LIBRARIES`\)装载、链接。
3. 核心系统库\(lib...\)、objc自举\(`initializeMainExecutable`\)。
4. 获取主程序main函数地址并返回。

主程序的链接与动态库的链接步骤类似，`dyld::_main`拿到已经装载好的主程序，初始化主程序库并链接，其中`weakBind`这一步骤等所有依赖库都链接完毕再进行。

对于插入的动态库会在主程序链接前，先进行装载，以保证在符号flat namespace情况下，插入的动态库中的等同符号会覆盖主程序及主程序链接库中的等同符号。而后进行链接，保证插入的动态库所依赖库不会先于主程序所依赖库。

主程序从`LC_MAIN`或`LC_UNIXTHREAD`cmd中获取main函数地址。

伪代码如下：

```text
//
// Entry point for dyld.  The kernel loads dyld and jumps to __dyld_start which
// sets up some registers and call this function.
//
// Returns address of main() in target program which __dyld_start jumps to
//
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
    uintptr_t result = 0;
    sMainExecutableMachHeader = mainExecutableMH;
    
    try {
        // instantiate ImageLoader for main executable
        sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
        
        // load any inserted libraries
        if  ( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
            for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib)
                loadInsertedDylib(*lib);
        }
        
        // record count of inserted libraries so that a flat search will look at 
        // inserted libraries, then main, then others.
        sInsertedDylibCount = sAllImages.size()-1;
        
        // link main executable
        link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
        
        // link any inserted libraries
        // do this after linking main executable so that any dylibs pulled in by inserted 
        // dylibs (e.g. libSystem) will not be in front of dylibs the program uses
        if ( sInsertedDylibCount > 0 ) {
            for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
                ImageLoader* image = sAllImages[i+1];
                link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
                image->setNeverUnloadRecursive();
            }
            
            // only INSERTED libraries can interpose
            // register interposing info after all inserted libraries are bound so chaining works
            for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
                ImageLoader* image = sAllImages[i+1];
                image->registerInterposing();
            }
        }
        
        for (int i=sInsertedDylibCount+1; i < sAllImages.size(); ++i) {
            ImageLoader* image = sAllImages[i];
            if ( image->inSharedCache() )
                continue;
            image->registerInterposing();
        }
        
        sMainExecutable->weakBind(gLinkContext);
        
        // run all initializers
        initializeMainExecutable(); 
        
        // find entry point for main executable
        result = (uintptr_t)sMainExecutable->getThreadPC();
        if ( result != 0 ) {
            // main executable uses LC_MAIN, needs to return to glue in libdyld.dylib
            if ( (gLibSystemHelpers != NULL) && (gLibSystemHelpers->version >= 9) )
                *startGlue = (uintptr_t)gLibSystemHelpers->startGlueToCallExit;
        }
        else {
            // main executable uses LC_UNIXTHREAD, dyld needs to let "start" in program set up for main()
            result = (uintptr_t)sMainExecutable->getMain();
            *startGlue = 0;
        }
    }
    catch(const char* message) {
        ...
    }
       
    return result;
}
```

经过`initializeMainExecutable`这一步骤，核心系统库\(lib...\)、objc自举，自举后才生效。先进行初始化\(static initializers\)，后进行终止化\(static terminator\)。先初始化插入的动态库，再初始化主程序，最后以同样顺序终止化。伪代码如下：

```text
void initializeMainExecutable()
{
    // run initialzers for any inserted dylibs
    const size_t rootCount = sImageRoots.size();
    if ( rootCount > 1 ) {
        for(size_t i=1; i < rootCount; ++i) {
            sImageRoots[i]->recursiveInitialization(gLinkContext, mach_thread_self());
        }
    }
  
    // run initializers for main executable and everything it brings up 
    sMainExecutable->recursiveInitialization(gLinkContext, mach_thread_self());
    
    // register cxa_atexit() handler to run static terminators in all loaded images when this process exits
  if ( gLibSystemHelpers != NULL ) 
    (*gLibSystemHelpers->cxa_atexit)(&runAllStaticTerminators, NULL, NULL);
}
```

`recursiveInitialization`以拓扑排序的引用深度递归初始化依赖库，先初始化依赖库，后初始化被依赖库，并在过程中为终止化记录顺序。`runAllStaticTerminators`则以记录的顺序终止化。伪代码如下：

```text
void ImageLoader::recursiveInitialization(const LinkContext& context, mach_port_t this_thread)
{
    try {
        // initialize lower level libraries first
        for(unsigned int i=0; i < libraryCount(); ++i) {
            ImageLoader* dependentImage = libImage(i);
            if ( dependentImage != NULL ) {
                if ( dependentImage->fDepth >= fDepth ) {
                    dependentImage->recursiveInitialization(context, this_thread);
                }
            }
        }
        
        // record termination order
        if ( this->needsTermination() )
            context.terminationRecorder(this);
        
        this->doInitialization(context);
    }
    catch (const char* msg) {
        ...
    }    
}
​
static void runAllStaticTerminators(void* extra)
{
  try {
    const size_t imageCount = sImageFilesNeedingTermination.size();
    for(size_t i=imageCount; i > 0; --i){
      ImageLoader* image = sImageFilesNeedingTermination[i-1];
      image->doTermination(gLinkContext);
    }
    sImageFilesNeedingTermination.clear();
  }
  catch (const char* msg) {
        ...
  }
}
```

初始化最终收敛到`doInitialization`，`doInitialization`会运行库的Routine\(`doImageInit`\)和ModInitFunction\(`doModInitFunctions`\)。终止化最终收敛到`doTermination`，`doTermination`会运行库的ModTermFunction。这几个过程都是从segment中找出初始化函数的地址并调用，伪代码如下：

```text
void ImageLoaderMachO::doImageInit(const LinkContext& context)
{
    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;
    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
    const struct load_command* cmd = cmds;
    for (uint32_t i = 0; i < cmd_count; ++i) {
        switch (cmd->cmd) {
            case LC_ROUTINES_COMMAND:
                Initializer func = (Initializer)(((struct macho_routines_command*)cmd)->init_address + fSlide);
                func(context.argc, context.argv, context.envp, context.apple, &context.programVars);
                break;
        }
        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
    }
}
​
void ImageLoaderMachO::doModInitFunctions(const LinkContext& context)
{
    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;
    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
    const struct load_command* cmd = cmds;
    for (uint32_t i = 0; i < cmd_count; ++i) {
        if ( cmd->cmd == LC_SEGMENT_COMMAND ) {
            const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
            const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));
            const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];
            for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {
                const uint8_t type = sect->flags & SECTION_TYPE;
                if ( type == S_MOD_INIT_FUNC_POINTERS ) {
                    Initializer* inits = (Initializer*)(sect->addr + fSlide);
                    const size_t count = sect->size / sizeof(uintptr_t);
                    for (size_t i=0; i < count; ++i) {
                        Initializer func = inits[i];
                        func(context.argc, context.argv, context.envp, context.apple, &context.programVars);
                    }
                }
            }
        }
        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
    }
}
​
void ImageLoaderMachO::doTermination(const LinkContext& context)
{
    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;
    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
    const struct load_command* cmd = cmds;
    for (uint32_t i = 0; i < cmd_count; ++i) {
        if ( cmd->cmd == LC_SEGMENT_COMMAND ) {
            const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
            const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));
            const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];
            for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {
                const uint8_t type = sect->flags & SECTION_TYPE;
                if ( type == S_MOD_TERM_FUNC_POINTERS ) {
                    Terminator* terms = (Terminator*)(sect->addr + fSlide);
                    const size_t count = sect->size / sizeof(uintptr_t);
                    for (size_t i=count; i > 0; --i) {
                        Terminator func = terms[i-1];
                        func();
                    }
                }
            }
        }
        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
    }
}
```

经过`_attribute__((constructor))`修饰的函数，会被加入ModInitFunction中，而经过`_attribute__((destructor))`修饰的函数，会被加入ModTermFunction中，都会在此步被调用。

核心系统库就是用这种方式注入自举，代码在[libSystem](https://opensource.apple.com/source/Libsystem/)中，伪代码如下：

```text
// system library initialisers
extern void bootstrap_init(void);   // from liblaunch.dylib
extern void mach_init(void);      // from libsystem_mach.dylib
extern void pthread_init(void);     // from libc.a
extern void __libc_init(const struct ProgramVars *vars, void (*atfork_prepare)(void), void (*atfork_parent)(void), void (*atfork_child)(void), const char *apple[]);  // from libc.a
extern void __keymgr_initializer(void);   // from libkeymgr.a
extern void _dyld_initializer(void);    // from libdyld.a
extern void libdispatch_init(void);   // from libdispatch.a
extern void _libxpc_initializer(void);    // from libxpc
​
/*
 * libsyscall_initializer() initializes all of libSystem.dylib <rdar://problem/4892197>
 */
static __attribute__((constructor)) 
void libSystem_initializer(int argc, const char* argv[], const char* envp[], const char* apple[], const struct ProgramVars* vars)
{
  _libkernel_functions_t libkernel_funcs = {
    .get_reply_port = _mig_get_reply_port,
    .set_reply_port = _mig_set_reply_port,
    .get_errno = __error,
    .set_errno = cthread_set_errno_self,
    .dlsym = dlsym,
  };
​
  _libkernel_init(libkernel_funcs);
​
  bootstrap_init();
  mach_init();
  pthread_init();
  __libc_init(vars, libSystem_atfork_prepare, libSystem_atfork_parent, libSystem_atfork_child, apple);
  __keymgr_initializer();
  _dyld_initializer();
  libdispatch_init();
  _libxpc_initializer();
}
```

其中初始化的库包括：`libdispatch`、`libxpc`、`libsystem_mach`、`liblaunch`、`libc`、`libkeymgr`等。objc的自举在`libdispatch_init`中，代码在[libdispatch](https://opensource.apple.com/tarballs/libdispatch/)中，伪代码如下：

```text
libdispatch_init(void)
{
#if DISPATCH_USE_THREAD_LOCAL_STORAGE
    _dispatch_thread_key_create(&__dispatch_tsd_key, _libdispatch_tsd_cleanup);
#else
    _dispatch_thread_key_create(&dispatch_priority_key, NULL);
    _dispatch_thread_key_create(&dispatch_r2k_key, NULL);
    _dispatch_thread_key_create(&dispatch_queue_key, _dispatch_queue_cleanup);
    _dispatch_thread_key_create(&dispatch_frame_key, _dispatch_frame_cleanup);
    _dispatch_thread_key_create(&dispatch_cache_key, _dispatch_cache_cleanup);
    _dispatch_thread_key_create(&dispatch_context_key, _dispatch_context_cleanup);
    _dispatch_thread_key_create(&dispatch_pthread_root_queue_observer_hooks_key,
      NULL);
    _dispatch_thread_key_create(&dispatch_basepri_key, NULL);
#if DISPATCH_INTROSPECTION
    _dispatch_thread_key_create(&dispatch_introspection_key , NULL);
#elif DISPATCH_PERF_MON
    _dispatch_thread_key_create(&dispatch_bcounter_key, NULL);
#endif
    _dispatch_thread_key_create(&dispatch_wlh_key, _dispatch_wlh_cleanup);
    _dispatch_thread_key_create(&dispatch_voucher_key, _voucher_thread_cleanup);
    _dispatch_thread_key_create(&dispatch_deferred_items_key,
      _dispatch_deferred_items_cleanup);
    
    _dispatch_queue_set_current(&_dispatch_main_q);
    _dispatch_queue_set_bound_thread(&_dispatch_main_q);
    
    _dispatch_hw_config_init();
    _dispatch_time_init();
    _dispatch_vtable_init();
    _os_object_init();
    _voucher_init();
    _dispatch_introspection_init();
}
```

在茫茫初始化中，`_os_object_init`进行了objc的自举，`_os_object_init`主过程是在[objc4](https://opensource.apple.com/tarballs/objc4/)代码中的`_objc_init`，伪代码如下：

```text
/**********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();
​
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```

`_objc_init`在dyld中注册了`map_images`、`load_images` 、`unmap_image`，分别进行objc自举、调用Load方法、清理现场。

### map\_images

`map_images`会进入`_read_images`，`_read_images`会从库中相应segment读出class、protocol、category等信息并载入runtime，这就是objc自举的核心过程。class、protocol、category要向runtime注册结构，category还要依附到相应class结构。

此过程一直围绕着class结构，class结构中的data可能为只读结构`class_ro_t`或可读写结构`class_rw_t`，data为`class_ro_t`结构的class叫做unrealized class，data为`class_rw_t`结构的class叫做realized class，要使用class，`class_ro_t`都要经过resolve转化为`class_rw_t`，即对class进行realize。class结构在创建的时候就可能经过了resolve并初始化时要进行realize，这样的class学名叫future class。realize可以使用lazy技术，区分为lazy和non-lazy，non-lazy要求初始化时进行realize。

在此过程中，要对future class和non-lazy class进行realize，伪代码如下：

```text
/********************************************************************
* _read_images
* Perform initial processing of the headers in the linked 
* list beginning with headerList. 
*
* Called by: map_images_nolock
*
* Locking: runtimeLock acquired by map_images
**********************************************************************/
void _read_images(header_info hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
    header_info *hi;
    uint32_t hIndex;
    size_t count;
    size_t i;
    Class *resolvedFutureClasses = nil;
    size_t resolvedFutureClassCount = 0;
    static bool doneOnce;
    
#define EACH_HEADER \
    hIndex = 0;         \
    hIndex < hCount && (hi = hList[hIndex]); \
    hIndex++
​
    if (!doneOnce) {
        doneOnce = YES;
        
        // namedClasses
        // Preoptimized classes don't go in this table.
        // 4/3 is NXMapTable's load factor
        int namedClassesSize = 
            (isPreoptimized() ? unoptimizedTotalClasses : totalClasses) * 4 / 3;
        gdb_objc_realized_classes =
            NXCreateMapTable(NXStrValueMapPrototype, namedClassesSize);
    }
    
    // Discover classes. Fix up unresolved future classes. Mark bundle classes.
    for (EACH_HEADER) {
        classref_t *classlist = _getObjc2ClassList(hi, &count);
        for (i = 0; i < count; i++) {
            Class cls = (Class)classlist[i];
            Class newCls = readClass(cls);
​
            if (newCls != cls  &&  newCls) {
                // Class was moved but not deleted. Currently this occurs 
                // only when the new class resolved a future class.
                // Non-lazily realize the class below.
                resolvedFutureClasses = (Class *)
                    realloc(resolvedFutureClasses, 
                            (resolvedFutureClassCount+1) * sizeof(Class));
                resolvedFutureClasses[resolvedFutureClassCount++] = newCls;
            }
        }
    }
    
    // Discover protocols. Fix up protocol refs.
    for (EACH_HEADER) {
        extern objc_class OBJC_CLASS_$_Protocol;
        Class cls = (Class)&OBJC_CLASS_$_Protocol;
        assert(cls);
        NXMapTable *protocol_map = protocols();
        bool isPreoptimized = hi->isPreoptimized();
        bool isBundle = hi->isBundle();
​
        protocol_t **protolist = _getObjc2ProtocolList(hi, &count);
        for (i = 0; i < count; i++) {
            readProtocol(protolist[i], cls, protocol_map, 
                         isPreoptimized, isBundle);
        }
    }
    
    // Realize non-lazy classes (for +load methods and static instances)
    for (EACH_HEADER) {
        classref_t *classlist = 
            _getObjc2NonlazyClassList(hi, &count);
        for (i = 0; i < count; i++) {
            Class cls = remapClass(classlist[i]);
            if (!cls) continue;
            realizeClass(cls);
        }
    }
    
    // Realize newly-resolved future classes, in case CF manipulates them
    if (resolvedFutureClasses) {
        for (i = 0; i < resolvedFutureClassCount; i++) {
            realizeClass(resolvedFutureClasses[i]);
            resolvedFutureClasses[i]->setInstancesRequireRawIsa(false/*inherited*/);
        }
        free(resolvedFutureClasses);
    }
    
    // Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
​
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);
            
            if (!cls) {
                continue;
            }
            
            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                NXMapTable *cats = unattachedCategories();
                category_list *list = (category_list *)NXMapGet(cats, cls);
                NXMapInsert(cats, cls, list);
                
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                }
            }
​
            if (cat->classMethods  ||  cat->protocols) 
            {
                NXMapTable *cats = unattachedCategories();
                category_list *list = (category_list *)NXMapGet(cats, cls->ISA());
                NXMapInsert(cats, cls->ISA(), list);
                
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
            }
        }
}
```

其中`_getObjc2ClassList`、`_getObjc2ProtocolList`、`_getObjc2NonlazyClassList`、`_getObjc2CategoryList` 这几个过程都是从库中相应segement读出信息。

`readClass`、`readProtocol`这几个过程是向`gdb_objc_realized_classes`、`protocol_map`注册结构，伪代码如下：

```text
/***********************************************************************
* readClass
* Read a class and metaclass as written by a compiler.
* Returns the new class pointer. This could be: 
* - cls
* - nil  (cls has a missing weak-linked superclass)
* - something else (space for this class was reserved by a future class)
*
* Note that all work performed by this function is preflighted by 
* mustReadClasses(). Do not change this function without updating that one.
*
* Locking: runtimeLock acquired by map_images or objc_readClassPair
**********************************************************************/
Class readClass(Class cls)
{
    const char *mangledName = cls->mangledName();
    
    Class replacing = nil;
    if (Class newCls = popFutureNamedClass(mangledName)) {
        // This name was previously allocated as a future class.
        // Copy objc_class to future class's struct.
        // Preserve future's rw data block.
        
        class_rw_t *rw = newCls->data();
        const class_ro_t *old_ro = rw->ro;
        memcpy(newCls, cls, sizeof(objc_class));
        rw->ro = (class_ro_t *)newCls->data();
        newCls->setData(rw);
        freeIfMutable((char *)old_ro->name);
        free((void *)old_ro);
        
        addRemappedClass(cls, newCls);
        
        replacing = cls;
        cls = newCls;
    }
    
    Class old = getClass(mangledName);
    if (!old || old == replacing) {
        NXMapInsert(gdb_objc_realized_classes, mangledName, cls);
    }
    
    return cls;
}
​
/***********************************************************************
* readProtocol
* Read a protocol as written by a compiler.
**********************************************************************/
static void
readProtocol(protocol_t *newproto, Class protocol_class,
             NXMapTable *protocol_map)
{
    protocol_t *oldproto = (protocol_t *)getProtocol(newproto->mangledName);
    if (oldproto) {
        return;
    }
    
    newproto->initIsa(protocol_class);
    NXMapInsert(protocol_map, installedproto->mangledName, 
                 installedproto);
}
```

整体流程清晰后，再近距离观察一下对class结构处理的细节。对class进行realize，包含对class结构的一系列操作：

1. resolve class。
2. realize class的super class和meta class。
3. 重新设置class和super class、meta class的关系。
4. 重新计算instance variable layout。
5. 将class结构中的`class_ro_t`结构中的method、property、protocol安装到`class_rw_t`结构，并依附category。

`realizeClass`伪代码如下：

```text
/***********************************************************************
* realizeClass
* Performs first-time initialization on class cls, 
* including allocating its read-write data.
* Returns the real class structure for the class. 
* Locking: runtimeLock must be write-locked by the caller
**********************************************************************/
static Class realizeClass(Class cls)
{
    const class_ro_t *ro;
    class_rw_t *rw;
    Class supercls;
    Class metacls;
    bool isMeta;
​
    if (!cls) return nil;
    if (cls->isRealized()) return cls;
    
    // fixme verify class is not in an un-dlopened part of the shared cache?
​
    ro = (const class_ro_t *)cls->data();
    if (ro->flags & RO_FUTURE) {
        // This was a future class. rw data is already allocated.
        rw = cls->data();
        ro = cls->data()->ro;
        cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
    } else {
        // Normal class. Allocate writeable class data.
        rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
        rw->ro = ro;
        rw->flags = RW_REALIZED|RW_REALIZING;
        cls->setData(rw);
    }
​
    isMeta = ro->flags & RO_META;
    
    // Realize superclass and metaclass, if they aren't already.
    // This needs to be done after RW_REALIZED is set above, for root classes.
    // This needs to be done after class index is chosen, for root metaclasses.
    supercls = realizeClass(remapClass(cls->superclass));
    metacls = realizeClass(remapClass(cls->ISA()));
    
    // Update superclass and metaclass in case of remapping
    cls->superclass = supercls;
    cls->initClassIsa(metacls);
​
    // Reconcile instance variable offsets / layout.
    // This may reallocate class_ro_t, updating our ro variable.
    if (supercls  &&  !isMeta) reconcileInstanceVariables(cls, supercls, ro);
​
    // Set fastInstanceSize if it wasn't set already.
    cls->setInstanceSize(ro->instanceSize);
​
    // Copy some flags from ro to rw
    if (ro->flags & RO_HAS_CXX_STRUCTORS) {
        cls->setHasCxxDtor();
        if (! (ro->flags & RO_HAS_CXX_DTOR_ONLY)) {
            cls->setHasCxxCtor();
        }
    }
​
    // Connect this class to its superclass's subclass lists
    if (supercls) {
        addSubclass(supercls, cls);
    } else {
        addRootClass(cls);
    }
​
    // Attach categories
    methodizeClass(cls);
​
    return cls;
}
```

第四步在设置了super class后，如果super class空间与原class空间重叠，需要对原class的实例重新计算其位置并调整，伪代码如下：

```text
static void reconcileInstanceVariables(Class cls, Class supercls, const class_ro_t*& ro) 
{
    class_rw_t *rw = cls->data();
    // Non-fragile ivars - reconcile this class with its superclass
    const class_ro_t *super_ro = supercls->data()->ro;
    
    if (ro->instanceStart >= super_ro->instanceSize) {
        // Superclass has not overgrown its space. We're done here.
        return;
    }
​
    if (ro->instanceStart < super_ro->instanceSize) {
        // Superclass has changed size. This class's ivars must move.
        // Also slide layout bits in parallel.
        // This code is incapable of compacting the subclass to 
        //   compensate for a superclass that shrunk, so don't do that.
        class_ro_t *ro_w = make_ro_writeable(rw);
        ro = rw->ro;
        moveIvars(ro_w, super_ro->instanceSize);
    } 
}
​
/***********************************************************************
* moveIvars
* Slides a class's ivars to accommodate the given superclass size.
* Ivars are NOT compacted to compensate for a superclass that shrunk.
* Locking: runtimeLock must be held by the caller.
**********************************************************************/
static void moveIvars(class_ro_t *ro, uint32_t superSize)
{
    uint32_t diff;
​
    assert(superSize > ro->instanceStart);
    diff = superSize - ro->instanceStart;
​
    if (ro->ivars) {
        // Find maximum alignment in this class's ivars
        uint32_t maxAlignment = 1;
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield
​
            uint32_t alignment = ivar.alignment();
            if (alignment > maxAlignment) maxAlignment = alignment;
        }
​
        // Compute a slide value that preserves that alignment
        uint32_t alignMask = maxAlignment - 1;
        diff = (diff + alignMask) & ~alignMask;
​
        // Slide all of this class's ivars en masse
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield
​
            uint32_t oldOffset = (uint32_t)*ivar.offset;
            uint32_t newOffset = oldOffset + diff;
            *ivar.offset = newOffset;
        }
    }
​
    *(uint32_t *)&ro->instanceStart += diff;
    *(uint32_t *)&ro->instanceSize += diff;
}
```

最后一步的处理都在`methodizeClass`中，先将`class_ro_t`结构中的method、property、protocol安装到`class_rw_t`，后依附category，而`_read_images`中的`remethodizeClass`也是用同样的方式依附category，`methodizeClass`伪代码如下：

```text
/***********************************************************************
* methodizeClass
* Fixes up cls's method list, protocol list, and property list.
* Attaches any outstanding categories.
* Locking: runtimeLock must be held by the caller
**********************************************************************/
static void methodizeClass(Class cls)
{
    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;
    
    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }
​
    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }
​
    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }
    
    // Attach categories.
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);
}
```

依附category就是将category的method、property、protocol append到原class中，伪代码如下：

```text
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    bool isMeta = cls->isMetaClass();
​
    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));
    
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    
    while (i--) {
        auto& entry = cats->list[i];
​
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
        }
​
        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }
​
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }
    
    auto rw = cls->data();
​
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
​
    rw->properties.attachLists(proplists, propcount);
    free(proplists);
​
    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

### load\_images

`load_images`核心过程就是以super class &gt; class &gt; category的顺序调用class的load方法，先调整顺序再调用，伪代码如下：

```text
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;
​
    // Discover load methods
    prepare_load_methods((const headerType *)mh);
​
    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

`prepare_load_methods`过程以及其`schedule_class_load`子过程中，调整调用顺序，先加入super class，再加入原class，再加入category，伪代码如下：

```text
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        realizeClass(cls);
        add_category_to_loadable_list(cat);
    }
}

/***********************************************************************
* prepare_load_methods
* Schedule +load for classes in this image, any un-+load-ed 
* superclasses in other images, and any categories in this image.
**********************************************************************/
// Recursively schedule +load for cls and any un-+load-ed superclasses.
// cls must already be connected.
static void schedule_class_load(Class cls)
{
    if (!cls) return;

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
}
```

`call_load_methods`核心过程则是while循环调用load方法，伪代码如下：

```text
/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first. 
* Category +load methods are not called until after the parent class's +load.
* 
* This method must be RE-ENTRANT, because a +load could trigger 
* more image mapping. In addition, the superclass-first ordering 
* must be preserved in the face of re-entrant calls. Therefore, 
* only the OUTERMOST call of this function will do anything, and 
* that call will handle all loadable classes, even those generated 
* while it was running.
*
* The sequence below preserves +load ordering in the face of 
* image loading during a +load, and make sure that no 
* +load method is forgotten because it was added during 
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have 
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first" 
* ordering, even if a category +load triggers a new loadable class 
* and a new loadable category attached to that class. 
*
* Locking: loadMethodLock must be held by the caller 
*   All other locks must not be held.
**********************************************************************/
void call_load_methods(void)
{
    bool more_categories;

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);
}
```

## 引用

[https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html)


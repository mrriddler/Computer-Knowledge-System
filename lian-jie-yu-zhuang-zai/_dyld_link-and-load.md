# \_dyld\_link&load

dyld是苹果出品的动态链接器，是MacOS和iOS平台计算机体系的核心，它负责计算机体系中的装载和链接，本篇文章探索链接、装载在其源码中的奥秘。文章中贴了大量的经过简化的伪代码，可读性非常高，其深意跃然屏上。

下文收集了一些关键字，能帮助充分了解背景。

* preflight：预先执行装载来检查动态库是否可被装载成功，以确定运行时装载动态库不会出现差错。
* [preBind](https://opensource.apple.com/source/cctools/cctools-384/RelNotes/Prebinding.html)：将动态链接的重定位提前到装载之前，这要求进程要预留虚拟空间给动态库，动态链接就可以像静态链接第一阶段一样，在装载前确定和分配虚拟空间。这种技术也是为了节省动态库重定位耗费的时间。
* image：库的代指，库本质上是映射到进程虚拟空间，就像镜子照出来的影像一样。
* slide：[ASLR](https://zh.wikipedia.org/wiki/%E4%BD%8D%E5%9D%80%E7%A9%BA%E9%96%93%E9%85%8D%E7%BD%AE%E9%9A%A8%E6%A9%9F%E8%BC%89%E5%85%A5)\(地址空间配置随机加载\)技术随机出来的地址offset，对动态库绝对地址访问的时候要添加此offset。
* ordinal：序号的代指，指代可以用全称索引，而为了节省空间，改用号码来索引的方式。

## link & load

link是dyld的主要功能，也是高效阅读源码的入口，在了解dyld内部的过程中，真正能理解链接和装载是密不可分的。伪代码如下：

```text
void ImageLoader::link(const LinkContext& context, bool forceLazysBound, bool preflightOnly, bool neverUnload, const RPathChain& loaderRPaths){    this->recursiveLoadLibraries(context, preflightOnly, loaderRPaths);  this->recursiveUpdateDepth(context.imageCount());​  this->recursiveRebase(context);  this->recursiveBind(context, forceLazysBound, neverUnload);​  if ( !context.linkingMainExecutable )    this->weakBind(context);​  std::vector<DOFInfo> dofs;  this->recursiveGetDOFSections(context, dofs);  context.registerDOFs(dofs);​  // interpose any dynamically loaded images  if ( !context.linkingMainExecutable ) {    this->recursiveApplyInterposing(context);  }}
```

link一个库有以下几个步骤：

1. load dependency library：以广度优先找出所有依赖库并装载\(load\)。
2. update depth：记录深度遍历引用深度模拟拓扑排序，用于形成所有被依赖库在依赖库前序的排序。
3. rebase：对库及其依赖库进行rebase。
4. bind：对库及其依赖库进行bind。
5. weak bind：对库进行weak bind。
6. DOF：对库添加debugg工具DTrace\(Dynamic Trace\)支持。
7. interpose ：对动态库进行符号介入，可以将动态库符号地址替换成指定地址。

## load dependency library

链接库之前要装载所有依赖库，装载会以广度优先的顺序找出所有依赖库，伪代码如下：

```text
void ImageLoader::recursiveLoadLibraries(const LinkContext& context, bool preflightOnly, const RPathChain& loaderRPaths){    // get list of libraries this image needs  DependentLibraryInfo libraryInfos[fLibraryCount];   this->doGetDependentLibraries(libraryInfos);    for(unsigned int i=0; i < fLibraryCount; ++i){        ImageLoader* dependentLib;        DependentLibraryInfo& requiredLibInfo = libraryInfos[i];        try {            dependentLib = context.loadLibrary(requiredLibInfo.name, true, this->getPath(),                                     &thisRPaths);            LibraryInfo actualInfo = dependentLib->doGetLibraryInfo();            depLibReRequired = requiredLibInfo.required;            depLibCheckSumsMatch = ( actualInfo.checksum == requiredLibInfo.info.checksum );            depLibReExported = requiredLibInfo.reExported;        }      catch (const char* msg) {            ...        }        setLibImage(i, dependentLib, depLibReExported, requiredLibInfo.upward);    }    // tell each to load its dependents    for(unsigned int i=0; i < libraryCount(); ++i) {        ImageLoader* dependentImage = libImage(i);        if ( dependentImage != NULL ) {             dependentImage->recursiveLoadLibraries(context, preflightOnly, thisRPaths);        }    }}
```

其中，`doGetDependentLibraries`从`dylib_command` 查询出依赖库基本信息，伪代码如下：

```text
void ImageLoaderMachO::doGetDependentLibraries(DependentLibraryInfo libs[]){    uint32_t index = 0;    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];    const struct load_command* cmd = cmds;    for (uint32_t i = 0; i < cmd_count; ++i) {        switch (cmd->cmd) {            case LC_LOAD_DYLIB:            case LC_LOAD_WEAK_DYLIB:            case LC_REEXPORT_DYLIB:            case LC_LOAD_UPWARD_DYLIB:                {                    const struct dylib_command* dylib = (struct dylib_command*)cmd;                  DependentLibraryInfo* lib = &libs[index++];                  lib->name = (char*)cmd + dylib->dylib.name.offset;                  lib->info.checksum = dylib->dylib.timestamp;                  lib->info.minVersion = dylib->dylib.compatibility_version;                  lib->info.maxVersion = dylib->dylib.current_version;                  lib->required = (cmd->cmd != LC_LOAD_WEAK_DYLIB);                  lib->reExported = (cmd->cmd == LC_REEXPORT_DYLIB);                  lib->upward = (cmd->cmd == LC_LOAD_UPWARD_DYLIB);                }                break;        }        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);    }}
```

### load

真正装载库的动作在`loadLibrary`中，装载是由一系列的路径处理阶段和从路径初始化库组成。

### **load phase**

这一系列的路径处理阶段，就是求解所有可能的路径组合\(permutation\)。每一阶段都是增加可能的路径组合并调用下一阶段，在最后阶段尝试用可能路径初始化库。一系列阶段包含：

`loadPhase0`、`loadPhase1`、`loadPhase2`、`loadPhase3`、`loadPhase4`、`loadPhase5`、`loadPhase5check`、`loadPhase5load`、`loadPhase5open` 、`loadPhase6`。

1. `loadPhase0`：添加rootPath组合。
2. `loadPhase1`：添加LD\_LIBRARY\_PATH、DYLD\_FRAMEWORK\_PATH 、DYLD\_FALLBACK\_LIBRARY\_PATH组合。
3. `loadPhase2`：添加FrameworkPartialPath、LibraryLeaf组合。
4. `loadPhase3`：添加@variables组合\(@executable\_path、@loader\_path、@rpath\)。
5. `loadPhase4`：添加image suffix组合。
6. `loadPhase5`：检查是否有dylib override，有则进入`loadPhase5check`，没有则进入`loadPhase5load`。
7. `loadPhase5check` ：检查路径是否匹配已装载的库。
8. `loadPhase5load`：检查是否在缓存中有相应的库，没有则进入`loadPhase5open`。
9. `loadPhase5open`：检查路径是否可被打开，有则进入`loadPhase6`。
10. `loadPhase6`：读入文件首页\(page\)，并进入`ImageLoaderMachO`的`instantiateFromFile`

### **instantiateFromFile**

这一过程会初始化`ImageLoaderMachO`实例，整个过程先将库文件读入栈上空间，设置`ImageLoaderMachO`对数据的指针`fMachOData`指向栈上空间，对库segment进行解析，再mmap所有segment，重新设置对数据的指针`fMachOData`指向mmap出来的空间，最后释放栈上空间。

其中，`sniffLoadCommands`从库中获取code sign、encrypt、compress信息。如果compress了调用`ImageLoaderMachOCompressed`，反之调用`ImageLoaderMachOClassic`，伪代码如下：

```text
// create image by mapping in a mach-o fileImageLoader* ImageLoaderMachO::instantiateFromFile(const char* path, int fd, const uint8_t firstPage[4096], uint64_t offsetInFat, uint64_t lenInFat, const struct stat& info, const LinkContext& context){    // get load commands  const unsigned int dataSize = sizeof(macho_header) + ((macho_header*)firstPage)->sizeofcmds;  uint8_t buffer[dataSize];  const uint8_t* fileData = firstPage;  if ( dataSize > 4096 ) {    // only read more if cmds take up more space than first page    fileData = buffer;    memcpy(buffer, firstPage, 4096);    pread(fd, &buffer[4096], dataSize-4096, offsetInFat+4096);  }​  bool compressed;  unsigned int segCount;  unsigned int libCount;  const linkedit_data_command* codeSigCmd;  const encryption_info_command* encryptCmd;  sniffLoadCommands((const macho_header*)fileData, path, false, &compressed, &segCount, &libCount, context, &codeSigCmd, &encryptCmd);  // instantiate concrete class based on content of load commands  if ( compressed )     return ImageLoaderMachOCompressed::instantiateFromFile(path, fd, fileData, dataSize, offsetInFat, lenInFat, info, segCount, libCount, codeSigCmd, encryptCmd, context);  else    return ImageLoaderMachOClassic::instantiateFromFile(path, fd, fileData, dataSize, offsetInFat, lenInFat, info, segCount, libCount, codeSigCmd, context);}
```

`ImageLoaderMachOCompressed`和`ImageLoaderMachOClassic`都继承自\``ImageLoaderMachO`\`，`ImageLoaderMachOCompressed`是`ImageLoaderMachOClassic`的压缩版本，基本原理都一致，后文以`ImageLoaderMachOClassic`为例继续探索，`ImageLoaderMachOClassic`的`instantiateFromFile`伪代码如下：

```text
// create image by mapping in a mach-o fileImageLoaderMachOClassic* ImageLoaderMachOClassic::instantiateFromFile(const char* path, int fd, const uint8_t* fileData, size_t lenFileData,uint64_t offsetInFat, uint64_t lenInFat, const struct stat& info, unsigned int segCount, unsigned int libCount, const struct linkedit_data_command* codeSigCmd, const LinkContext& context){      ImageLoaderMachOClassic* image = ImageLoaderMachOClassic::instantiateStart((macho_header*)fileData, path, segCount, libCount);  try {    // record info about file      image->setFileInfo(info.st_dev, info.st_ino, info.st_mtime);​    // if this image is code signed, let kernel validate signature before mapping any pages from image    image->loadCodeSignature(codeSigCmd, fd, offsetInFat, context);    // Validate that first data we read with pread actually matches with code signature    image->validateFirstPages(codeSigCmd, fd, fileData, lenFileData, offsetInFat, context);​    // mmap segments    image->mapSegmentsClassic(fd, offsetInFat, lenInFat, info.st_size, context);​    // finish up    image->instantiateFinish(context);    }    catch (...) {    delete image;    throw;  }    return image;}
```

其中最重要的步骤是对代码进行签名验证\(`loadCodeSignature`\)、映射所有段\(`mapSegments`\)、解析segment\(`instantiateFinish`\)。

`loadCodeSignature`伪代码如下：

```text
void ImageLoaderMachO::loadCodeSignature(const struct linkedit_data_command* codeSigCmd, int fd,  uint64_t offsetInFatFile, const LinkContext& context){    if (codeSigCmd != NULL) {        fsignatures_t siginfo;        siginfo.fs_file_start=offsetInFatFile;        // start of mach-o slice in fat file         siginfo.fs_blob_start=(void*)(long)(codeSigCmd->dataoff); // start of CD in mach-o file        siginfo.fs_blob_size=codeSigCmd->datasize;      // size of CD        int result = fcntl(fd, F_ADDFILESIGS_RETURN, &siginfo);        if ( result == -1 ) {            ...    }    }}
```

`mapSegments`伪代码如下：

```text
void ImageLoaderMachO::mapSegments(int fd, uint64_t offsetInFat, uint64_t lenInFat, uint64_t fileLen, const LinkContext& context){    // find address range for image    intptr_t slide = this->assignSegmentAddresses(context);    // map in all segments    for(unsigned int i=0, e=segmentCount(); i < e; ++i) {        vm_offset_t fileOffset = segFileOffset(i) + offsetInFat;        vm_size_t size = segFileSize(i);        uintptr_t requestedLoadAddress = segPreferredLoadAddress(i) + slide;        int protection = 0;        if ( !segUnaccessible(i) ) {            // If has text-relocs, don't set x-bit initially.            // Instead set it later after text-relocs have been done.            if ( segExecutable(i) && !(segHasRebaseFixUps(i) && (slide != 0)) )                protection   |= PROT_EXEC;            if ( segReadable(i) )                protection   |= PROT_READ;            if ( segWriteable(i) )                protection   |= PROT_WRITE;        }        // wholly zero-fill segments have nothing to mmap() in        if ( size > 0 ) {                    void* loadAddress = xmmap((void*)requestedLoadAddress, size, protection, MAP_FIXED | MAP_PRIVATE, fd, fileOffset);        }    }}
```

`instantiateFinish`直接进入`parseLoadCmds`，`parseLoadCmds`伪代码如下：

```text
void ImageLoaderMachO::parseLoadCmds(const LinkContext& context){    // now that segments are mapped in, get real fMachOData, fLinkEditBase, and fSlide    for(unsigned int i=0; i < fSegmentsCount; ++i) {        // set up pointer to __LINKEDIT segment        if ( strcmp(segName(i),"__LINKEDIT") == 0 ) {            fLinkEditBase = (uint8_t*)(segActualLoadAddress(i) - segFileOffset(i));        }        // some segment always starts at beginning of file and contains mach_header and load commands        if ( (segFileOffset(i) == 0) && (segFileSize(i) != 0) ) {            fMachOData = (uint8_t*)(segActualLoadAddress(i));        }    }    const struct load_command* firstUnknownCmd = NULL;    const struct version_min_command* minOSVersionCmd = NULL;    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];    const struct load_command* cmd = cmds;    for (uint32_t i = 0; i < cmd_count; ++i) {        switch (cmd->cmd) {            case LC_SYMTAB:                {                    fSymbolTable = (struct symtab_command*)cmd;                    fStrings = (const char*)&fLinkEditBase[symtab->stroff];                    fDynamicInfo = (macho_nlist*)(&fLinkEditBase[symtab->symoff]);                }                break;            case LC_DYSYMTAB:                fDynamicInfo = (struct dysymtab_command*)cmd;                break;            case LC_SUB_UMBRELLA:                fHasSubUmbrella = true;                break;            case LC_SUB_FRAMEWORK:                fInUmbrella = true;                break;            case LC_SUB_LIBRARY:                fHasSubLibraries = true;                break;            case LC_ROUTINES_COMMAND:                fHasDashInit = true;                break;            case LC_DYLD_INFO:            case LC_DYLD_INFO_ONLY:                break;            case LC_SEGMENT_COMMAND:                {                    const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;                    const bool isTextSeg = (strcmp(seg->segname, "__TEXT") == 0);                    const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));                    const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];                    for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {                        const uint8_t type = sect->flags & SECTION_TYPE;                        if ( type == S_MOD_INIT_FUNC_POINTERS )                            fHasInitializers = true;                        else if ( type == S_MOD_TERM_FUNC_POINTERS )                            fHasTerminators = true;                        else if ( type == S_DTRACE_DOF )                            fHasDOFSections = true;                        else if ( isTextSeg && (strcmp(sect->sectname, "__eh_frame") == 0) )                            fEHFrameSectionOffset = (uint32_t)((uint8_t*)sect - fMachOData);                        else if ( isTextSeg && (strcmp(sect->sectname, "__unwind_info") == 0) )                            fUnwindInfoSectionOffset = (uint32_t)((uint8_t*)sect - fMachOData);                    }                }                break;            case LC_TWOLEVEL_HINTS:                // no longer supported                break;            case LC_ID_DYLIB:                {                    fDylibIDOffset = (uint32_t)((uint8_t*)cmd - fMachOData);                }                break;            case LC_RPATH:            case LC_LOAD_WEAK_DYLIB:            case LC_REEXPORT_DYLIB:            case LC_LOAD_UPWARD_DYLIB:            case LC_MAIN:                // do nothing, just prevent LC_REQ_DYLD exception from occuring                break;            case LC_VERSION_MIN_MACOSX:            case LC_VERSION_MIN_IPHONEOS:            case LC_VERSION_MIN_TVOS:            case LC_VERSION_MIN_WATCHOS:                minOSVersionCmd = (version_min_command*)cmd;                break;            default:                if ( (cmd->cmd & LC_REQ_DYLD) != 0 ) {                    if ( firstUnknownCmd == NULL )                        firstUnknownCmd = cmd;                }                break;         }        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);    }}
```

## rebase

对于重定位\(relocation\)表项，其指向需要重定位的地址是相对地址，是距第一个segement或第一个可写\(writable\)segment的offset，rebase这一步就是将相对地址调整为绝对地址，添加relocBase和slide。伪代码如下：

```text
void ImageLoaderMachOClassic::rebase(const LinkContext& context){    register const uintptr_t slide = this->fSlide;  const uintptr_t relocBase = this->getRelocBase();​  // loop through all local (internal) relocation records  const relocation_info* const relocsStart = (struct relocation_info*)(&fLinkEditBase[fDynamicInfo->locreloff]);  const relocation_info* const relocsEnd = &relocsStart[fDynamicInfo->nlocrel];  for (const relocation_info* reloc=relocsStart; reloc < relocsEnd; ++reloc) {    uintptr_t rebaseAddr = reloc->r_address + relocBase;    *((uintptr_t*)rebaseAddr) += slide;  }}
```

重定位表项结构relocation\_info在rebase和bind中至关重要，其定义如下：

```text
/* * Format of a relocation entry of a Mach-O file.  Modified from the 4.3BSD * format.  The modifications from the original format were changing the value * of the r_symbolnum field for "local" (r_extern == 0) relocation entries. * This modification is required to support symbols in an arbitrary number of * sections not just the three sections (text, data and bss) in a 4.3BSD file. * Also the last 4 bits have had the r_type tag added to them. */struct relocation_info {   int32_t  r_address;  /* offset in the section to what is being           relocated */   uint32_t     r_symbolnum:24, /* symbol index if r_extern == 1 or section           ordinal if r_extern == 0 */    r_pcrel:1,  /* was relocated pc relative already */    r_length:2, /* 0=byte, 1=word, 2=long, 3=quad */    r_extern:1, /* does not include value of sym referenced */    r_type:4; /* if not 0, machine specific relocation type */};
```

其中，`r_address`指向需要重定位的地址。对于被导出符号来说，`r_symbolnum`字段会指向符号在符号表中的位置。

## bind

bind有两个步骤：

1. 对被导出\(exported\)符号进行重定位。
2. 对Segment中的non lazy indirect symbol pointers进行绑定。
3. 对Segment中的lazy indirect symbol pointers完善stub机制。

### external relocation

对被导出符号重定位就是：依据被导出符号重定位表项，去符号表中找出符号的基本信息，再去其他库符号表中resolve符号，将resolve结果bind到需要重定位地址上。其中resolve undefined就是查找符号的过程。伪代码如下：

```text
void ImageLoaderMachOClassic::doBindExternalRelocations(const LinkContext& context){    const uintptr_t relocBase = this->getRelocBase();  const bool twoLevel = this->usesTwoLevelNameSpace();  // loop through all external relocation records and bind each  const relocation_info* const relocsStart = (struct relocation_info*)(&fLinkEditBase[fDynamicInfo->extreloff]);  const relocation_info* const relocsEnd = &relocsStart[fDynamicInfo->nextrel];  for (const relocation_info* reloc=relocsStart; reloc < relocsEnd; ++reloc) {    if (reloc->r_length == RELOC_SIZE) {      switch(reloc->r_type) {        case POINTER_RELOC:          {                        const struct macho_nlist* undefinedSymbol =                             &fSymbolTable[reloc->r_symbolnum];             uintptr_t* location = ((uintptr_t*)(reloc->r_address + relocBase));                         uintptr_t symbolAddr = this->resolveUndefined(context, undefinedSymbol, twoLevel,                      symbolIsWeakReference(undefinedSymbol), &image);                         uintptr_t value = *location;                         value += symbolAddr;​          }      }    }  }}
```

### **flat namespace vs two level namespace**

符号命名空间有两种方式：flat和two level，flat代表符号没有关联所在库，two level代表符号关联着所在库。这就导致了resolve undefined符号有两种方式：flat和two level，flat将在所有库中resolve符号，而two level会在符号的n\_desc标识符号所在的库，two level只要去所在库resolve符号。基础库符号都是two level方式。除此之外还有个重点就是弱引用符号可以找不到。伪码如下：

```text
uintptr_t ImageLoaderMachOClassic::resolveUndefined(const LinkContext& context, const struct    macho_nlist* undefinedSymbol, bool twoLevel, bool dontCoalesce, const ImageLoader` foundIn){  const char* symbolName = &fStrings[undefinedSymbol->n_un.n_strx];  if ( context.bindFlat || !twoLevel ) {        // flat lookup      const Symbol* sym;    if ( context.flatExportFinder(symbolName, &sym, foundIn) ) {      return (*foundIn)->getExportedSymbolAddress(sym, context, this);    }         if ( (undefinedSymbol->n_desc & N_WEAK_REF) != 0 ) {             // definition can't be found anywhere      // if reference is weak_import, then it is ok, just return 0      return 0;         }  } else {         // two level lookup    ImageLoader* target = NULL;    uint8_t ord = GET_LIBRARY_ORDINAL(undefinedSymbol->n_desc);    if ( ord == EXECUTABLE_ORDINAL ) {      target = context.mainExecutable;    }    else if ( ord == SELF_LIBRARY_ORDINAL ) {      target = this;        } else {            ...        }        const Symbol* sym = target->findExportedSymbol(symbolName, true, foundIn);        if ( sym!= NULL ) {      return (*foundIn)->getExportedSymbolAddress(sym, context, this);        } else if ( (undefinedSymbol->n_desc & N_WEAK_REF) != 0 ) {             // definition can't be found anywhere      // if reference is weak_import, then it is ok, just return 0      return 0;        } else {            ...        }  }}
```

flat lookup中的`flatExportFinder`会对所有库调用`findExportedSymbol` ，two level lookup也会调用`findExportedSymbol`，最终收敛到`binarySearch`，对符号表做二分搜索，其伪代码如下：

```text
const struct macho_nlist* ImageLoaderMachOClassic::binarySearch(const char* key, const char stringPool[], const struct macho_nlist symbols[], uint32_t symbolCount) const{    const struct macho_nlist* base = symbols;  for (uint32_t n = symbolCount; n > 0; n /= 2) {    const struct macho_nlist* pivot = &base[n/2];    const char* pivotStr = &stringPool[pivot->n_un.n_strx];​    int cmp = strcmp(key, pivotStr);    if ( cmp == 0 )      return pivot;    if ( cmp > 0 ) {      // key > pivot       // move base to symbol after pivot      base = &pivot[1];      --n;     }    else {      // key < pivot       // keep same base    }  }  return NULL;}
```

找到符号后，再获取符号地址并返回，伪代码如下：

```text
uintptr_t ImageLoaderMachOClassic::exportedSymbolAddress(const LinkContext& context, const Symbol* symbol, const ImageLoader* requestor, bool runResolver) const{  const struct macho_nlist* sym = (macho_nlist*)symbol;  uintptr_t result = sym->n_value + fSlide;  return result;}
```

伪代码中省略了对弱符号的处理，总之，对于弱符号无法resolve也可接受。

!\[symbol\_relocaiton\]\([https://raw.githubusercontent.com/mrriddler/Computer-Knowledge-System/master/image/link\_load/symbol\_stub.png](https://raw.githubusercontent.com/mrriddler/Computer-Knowledge-System/master/image/link_load/symbol_stub.png)\)

### bind non lazy indirection symbol pointers

`__DATA`中引用符号，以直接指向符号实际地址\(而不是符号表地址\)的指针形式存储，有non lazy symbol pointer和lazy symbol pointer两种，区别就是是否是使用了plt\(延迟绑定\)技术，而这些符号如果想要获得更多符号基本信息要通过间接跳转表，所以得名indirection symbol pointer。

这一步骤中，就是去bind non lazy symbol pointers，当然可以设置为强迫bind lazy symbol pointers。其中值得一提的动态库重定位中的.got是non lazy symbol pointers，这代表在这步骤中，.got也会被绑定。

遍历section，找到`S_NON_LAZY_SYMBOL_POINTERS`，并获得indirection table的index，来最终到达符号表，resolve undefined符号伪代码如下：

```text
void ImageLoaderMachOClassic::bindIndirectSymbolPointers(const LinkContext& context, bool bindNonLazys, bool bindLazys){    // scan for all non-lazy-pointer sections   const bool twoLevel = this->usesTwoLevelNameSpace();  const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;  const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];  const struct load_command* cmd = cmds;  const uint32_t* const indirectTable = (uint32_t*)&fLinkEditBase[fDynamicInfo->indirectsymoff];  for (uint32_t i = 0; i < cmd_count; ++i) {    switch (cmd->cmd) {      case LC_SEGMENT_COMMAND:        {          const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;          const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));          const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];          for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd;                                        ++sect) {            const uint8_t type = sect->flags & SECTION_TYPE;            uint32_t elementSize = sizeof(uintptr_t);            size_t elementCount = sect->size / elementSize;            if ( type == S_NON_LAZY_SYMBOL_POINTERS ) {              if ( ! bindNonLazys )                continue;            }            else if ( type == S_LAZY_SYMBOL_POINTERS ) {              // process each symbol pointer in this section              if ( ! bindLazys )                continue;            }            const uint32_t indirectTableOffset = sect->reserved1;            uint8_t* ptrToBind = (uint8_t*)(sect->addr + fSlide);            for (size_t j=0; j < elementCount; ++j, ptrToBind += elementSize) {              uint32_t symbolIndex = indirectTable[indirectTableOffset + j];                               const struct macho_nlist* sym = &fSymbolTable[symbolIndex];              const ImageLoader* image = NULL;              uintptr_t symbolAddr = resolveUndefined(context, sym, twoLevel,                             symbolIsWeakReference(sym), &image);              // update pointer                              *ptrToBind = symbolAddr;            }                    }                }        }        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);    }}
```

### setup lazy symbol pointers handler

对于lazy symbol pointer，这一步骤中，将为其设置stub机制的handler。具体位置在`__DATA`中的dyld section。伪代码如下：

```text
void ImageLoaderMachO::setupLazyPointerHandler(const LinkContext& context){  const macho_header* mh = (macho_header*)fMachOData;  const uint32_t cmd_count = mh->ncmds;  const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];  const struct load_command* cmd;    cmd = cmds;    for (uint32_t i = 0; i < cmd_count; ++i) {        if ( cmd->cmd == LC_SEGMENT_COMMAND ) {            const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;        if ( strncmp(seg->segname, "__DATA", 6) == 0 ) {                const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));         const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];         for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {                   if ( strcmp(sect->sectname, "__dyld" ) == 0 ) {                       struct DATAdyld* dd = (struct DATAdyld*)(sect->addr + fSlide);                       if ( sect->size > offsetof(DATAdyld, dyldLazyBinder) ) {                           if ( dd->dyldLazyBinder != (void*)&stub_binding_helper )                               dd->dyldLazyBinder = (void*)&stub_binding_helper;           }           if ( sect->size > offsetof(DATAdyld, dyldFuncLookup) ) {             if ( dd->dyldFuncLookup != (void*)&_dyld_func_lookup )               dd->dyldFuncLookup = (void*)&_dyld_func_lookup;           }                   }               }            }        }}
```

在延时绑定时，会进入`stub_binding_helper`汇编，而汇编最终交给`doBindLazySymbol`来完成延时绑定。 `doBindLazySymbol`会遍历section，根据地址范围找到所属section，以获得indirection table的index，来最终到达符号表，resolve undefined符号。伪代码如下：

```text
uintptr_t ImageLoaderMachOClassic::doBindLazySymbol(uintptr_t* lazyPointer, const LinkContext& context){  // scan for all lazy-pointer sections  const bool twoLevel = this->usesTwoLevelNameSpace();  const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;  const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];  const struct load_command* cmd = cmds;  const uint32_t* const indirectTable = (uint32_t*)&fLinkEditBase[fDynamicInfo->indirectsymoff];  for (uint32_t i = 0; i < cmd_count; ++i) {    switch (cmd->cmd) {      case LC_SEGMENT_COMMAND:        {          const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;          const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));          const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];          for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd;                                          ++sect) {            const uint8_t type = sect->flags & SECTION_TYPE;            uint32_t symbolIndex = INDIRECT_SYMBOL_LOCAL;            if ( type == S_LAZY_SYMBOL_POINTERS ) {              const size_t pointerCount = sect->size / sizeof(uintptr_t);              uintptr_t* const symbolPointers = (uintptr_t*)(sect->addr + fSlide);              if ( (lazyPointer >= symbolPointers) && (lazyPointer < &symbolPointers[pointerCount]) ) {                const uint32_t indirectTableOffset = sect->reserved1;                const size_t lazyIndex = lazyPointer - symbolPointers;                symbolIndex = indirectTable[indirectTableOffset + lazyIndex];              }            }            if ( symbolIndex != INDIRECT_SYMBOL_ABS && symbolIndex !=                                          INDIRECT_SYMBOL_LOCAL ) {              const char* symbolName =                          &fStrings[fSymbolTable[symbolIndex].n_un.n_strx];              const ImageLoader* image = NULL;              uintptr_t symbolAddr = this->resolveUndefined(context,                     &fSymbolTable[symbolIndex], twoLevel, false, &image);                               *lazyPointer = symbolAddr;              return symbolAddr;            }          }        }        break;    }    cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);  }}
```

### stub机制

起点在`__TEXT`代码中，`__TEXT`中对`_la_symbol_ptr`的引用是一条`call`命令，`call`命令会将下条地址留存到栈中，然后跳转到目的地址，执行完跳转过程后回到下条地址。`call`命令会跳转到`(__TEXT,__stubs)`中，其内容是一条`jmp`命令，再次进行跳转，目的地地址在`(__DATA,__la_symbol_ptr)`中，对于首次resolve的符号，其地址是`(__TEXT,__stub_helper)`，而经过resolve的符号，其地址便是符号的最终地址了。

stub机制需要经过两次跳转，相当于进行了一次间接\(indirection\)跳转，这种设计有两个要点：

* 间接跳转的本质是添加了次`jmp`，只能放在`__TEXT`中，这归结于其获得的结果可能是一段可执行代码。
* 目的地址要放在`__DATA`中，以在运行时将绑定过程地址更改为最终地址。

!\[symbol\_stub\]\(\image\link\_load\symbol\_stub.png\)

## weak bind

weak bind这一步是将相同的弱符号进行归并\(coalesce \)，如果有强符号则归并成强符号，否则归并成按装载顺序的首个弱符号，实现上通过符号字符串的冒泡排序，调整库的顺序以置前相同的符号和置后处理完的库，找到一个相同的符号后，如果是强符号则不需要再查找，否则从装载顺序找到首个弱符号地址，将所有库中的相同符号都覆盖\(ovveride\)为该地址，最后再增加\(increment\)首个库中的符号并重新排序。伪代码如下：

```text
void ImageLoader::weakBind(const LinkContext& context){    // get set of ImageLoaders that participate in coalecsing    ImageLoader* imagesNeedingCoalescing[fgImagesRequiringCoalescing];    int count = context.getCoalescedImages(imagesNeedingCoalescing);​    // make symbol iterators for each  ImageLoader::CoalIterator iterators[count];  ImageLoader::CoalIterator* sortedIts[count];  for(int i=0; i < count; ++i) {    imagesNeedingCoalescing[i]->initializeCoalIterator(iterators[i], i);    sortedIts[i] = &iterators[i];  }    // walk all symbols keeping iterators in sync by   // only ever incrementing the iterator with the lowest symbol   int doneCount = 0;  while ( doneCount != count ) {        // increment iterator with lowest symbol      if ( sortedIts[0]->image->incrementCoalIterator(*sortedIts[0]) )            ++doneCount;         // re-sort iterators    for(int i=1; i < count; ++i) {      int result = strcmp(sortedIts[i-1]->symbolName, sortedIts[i]->symbolName);      if ( result == 0 )        sortedIts[i-1]->symbolMatches = true;      if ( result > 0 ) {        // new one is bigger then next, so swap        ImageLoader::CoalIterator* temp = sortedIts[i-1];        sortedIts[i-1] = sortedIts[i];        sortedIts[i] = temp;      }      if ( result < 0 )        break;    }        // process all matching symbols just before incrementing the lowest one that matches    if ( sortedIts[0]->symbolMatches && !sortedIts[0]->done ) {      const char* nameToCoalesce = sortedIts[0]->symbolName;      // pick first symbol in load order (and non-weak overrides weak)      uintptr_t targetAddr = 0;      ImageLoader* targetImage = NULL;      for(int i=0; i < count; ++i) {        if ( strcmp(iterators[i].symbolName, nameToCoalesce) == 0 ) {          if ( iterators[i].weakSymbol ) {            if ( targetAddr == 0 ) {              targetAddr = iterators[i].image->getAddressCoalIterator(iterators[i], context);              if ( targetAddr != 0 )                targetImage = iterators[i].image;            }          }                    else {                        targetAddr = iterators[i].image->getAddressCoalIterator(iterators[i], context);                        if ( targetAddr != 0 ) {                            targetImage = iterators[i].image;                            // strong implementation found, stop searching                            break;                        }                    }                }            }        }        // tell each to bind to this symbol (unless already bound)    if ( targetAddr != 0 ) {            for(int i=0; i < count; ++i) {                if ( strcmp(iterators[i].symbolName, nameToCoalesce) == 0 ) {                    if ( ! iterators[i].image->fWeakSymbolsBound )                        iterators[i].image->updateUsesCoalIterator(iterators[i], targetAddr, targetImage, context);            iterators[i].symbolMatches = false;                 }      }    }     }}
```

`getCoalescedImages`将所有库中有被导出弱符号定义和导入了弱符号的库全部提取出来。

`CoalIterator`是为了排序而实现的迭代器，持有库和当前迭代符号index，结构定义如下：

```text
struct CoalIterator{    ImageLoader*  image;  const char*   symbolName;  unsigned int  loadOrder;  bool      weakSymbol;  bool      symbolMatches;  bool      done;  // the following are private to the ImageLoader subclass  uintptr_t   curIndex;  uintptr_t   endIndex;  uintptr_t   address;  uintptr_t   type;  uintptr_t   addend;};
```

在`initializeCoalIterator`中初始化，在`incrementCoalIterator`中迭代到下个符号，在`getAddressCoalIterator`中获取符号的地址，在`updateUsesCoalIterator`中更新符号地址。`updateUsesCoalIterator`与上文中的被导出符号进行重定位和绑定non lazy indirect symbol pointers类似，伪代码如下：

```text
void ImageLoaderMachOClassic::updateUsesCoalIterator(CoalIterator& it, uintptr_t value, ImageLoader* targetImage, const LinkContext& context){    uint32_t symbol_index = 0;    if ( fDynamicInfo->tocoff != 0 ) {        const dylib_table_of_contents* toc = (dylib_table_of_contents*)&fLinkEditBase[fDynamicInfo->tocoff];        symbol_index = toc[it.curIndex-1].symbol_index;    }    else {     symbol_index = fDynamicInfo->iextdefsym + (uint32_t)it.curIndex - 1;    }    // scan external relocations for uses of symbol_index    const uintptr_t relocBase = this->getRelocBase();    const relocation_info* const relocsStart = (struct relocation_info*)(&fLinkEditBase[fDynamicInfo->extreloff]);    const relocation_info* const relocsEnd = &relocsStart[fDynamicInfo->nextrel];    for (const relocation_info* reloc=relocsStart; reloc < relocsEnd; ++reloc) {        if ( reloc->r_symbolnum == symbol_index ) {            const struct macho_nlist* undefinedSymbol = &fSymbolTable[reloc->r_symbolnum];            const char* symbolName = &fStrings[undefinedSymbol->n_un.n_strx];            uintptr_t* location = ((uintptr_t*)(reloc->r_address + relocBase));            *location = value;        }    }    // scan lazy and non-lazy pointers for uses of symbol_index    const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];    const struct load_command* cmd = cmds;    const uint32_t* const indirectTable = (uint32_t*)&fLinkEditBase[fDynamicInfo->indirectsymoff];    for (uint32_t i = 0; i < cmd_count; ++i) {        if ( cmd->cmd == LC_SEGMENT_COMMAND ) {            const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;            const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));            const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];            for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {                uint32_t elementSize = sizeof(uintptr_t);                switch ( sect->flags & SECTION_TYPE ) {                    case S_NON_LAZY_SYMBOL_POINTERS:                    case S_LAZY_SYMBOL_POINTERS:                         {                            size_t elementCount = sect->size / elementSize;                            const uint32_t indirectTableOffset = sect->reserved1;                            uint8_t* ptrToBind = (uint8_t*)(sect->addr + fSlide);                            for (size_t j=0; j < elementCount; ++j, ptrToBind += elementSize) {                                if ( indirectTable[indirectTableOffset + j] == symbol_index ) {                // update pointer                                   *ptrToBind = value;                                }                            }                        }                        break;                }            }        }        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);    }}
```

## 总结

* 装载依赖\(load dependency\)：装载依赖以依赖关系形成有向图，可以广度优先或深度优先，一般以广度优先。装载库就是将库的段映射入进程空间。
* 基址重置\(rebasing\)：dyld链接步骤中的rebase，动态链接的第一阶段。
* 全局符号介入\(global symbol interpose\)：全局符号介入指的是，对于相同的符号，为了不造成决议成不同地址，要对相同符号介入处理，统一覆盖成全局首个加入的符号。dyld链接步骤中weak bind正是对弱符号进行全局符号介入，如果存在相同的强符号则覆盖成强符号，否则覆盖成首个加入的弱符号。
* 全局符号表\(global symbol table\)：dyld链接库的步骤中，没有合并符号表这一步，这表明，dyld不会生成全局符号表，符号分散到各自符号表中，决议符号根据flat或two level到所有符号表或关联符号表搜索。

## 引用

[https://glandium.org/blog/?p=2764](https://glandium.org/blog/?p=2764)

[http://newosxbook.com/articles/DYLD.html](http://newosxbook.com/articles/DYLD.html)


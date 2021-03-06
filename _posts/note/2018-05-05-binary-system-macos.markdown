---
layout:     post
title:      macOS binary system learning
author:     wooy0ung
tags:       macos
category:   note
---

- 目录
{:toc #markdown-toc}

>[index]  
>0x001 introduction of XNU's architecture  
>0x002 analysis of loading mach-o by dyld  
>0x003 introduce to macho format  
>0x004 macho dynamic link  
<!-- more -->


## 0x001 introduction of XNU's architecture

system's architecture
```
The User Experience layer       # ignore
The Application Frameworks layer
The Core Frameworks
Darwin                          # kernel
```

darwin's architecture
![](/assets/img/note/2018-05-05-binary-system-macos/0x001-001.png)  

XNU is darwin's kernel，a mixture of mach and bsd，mainly contains
```
Mach Micro kernel
BSD kernel
libKern
I/O Kit
```

function
```
abstract of process and thread
manager of virtual memory
adjust tasks
process communications
```

BSD's work
```
UNIX process model
POSIX process model(pthread)
manager of UNIX user and group
BSD Socket API
file system
instrucment system
```

libKern provides a C++ subset for I/O Kit，and I/O Kit is driver model framework with OOP features

what's info.plist? Every app or bundle would rely on it，it provide a property list containing: 
```
some info show to user directly
label your app or which types it can support
which system framework would use to load the app 
```

codesigned? As a iOS developer, you may have a certifica，public key and private key。In macOS, it is 
acquiescently closed, but you can open by "Gatekeeper".

Mandatory Access Control, firstly used in FreeBSD 5.x, is the basic of macOS's Sandboxing and iOS entitlement.
It can label users as well as other files with constant security property, and confirm whether a users can
access the files.

sandbox, you can see this picture with traslations
![](/assets/img/note/2018-05-05-binary-system-macos/0x001-002.png)  


## 0x002 analysis of loading mach-o by dyld

macho_header，open machostruc.h
```
struct MACHO_HEADER {
    byte    magic[4];
    uint32  cputype;
    uint32  cpusubtype;
    uint32  filetype;
    uint32  ncmds;
    uint32  sizeofcmds;
    uint32  flags;
} PACKED;

struct MACHO_HEADER64 {
    byte    magic[4];
    uint32  cputype;
    uint32  cpusubtype;
    uint32  filetype;
    uint32  ncmds;
    uint32  sizeofcmds;
    uint32  flags;
    uint32  reserved;
} PACKED;
```

and operations
![](/assets/img/note/2018-05-05-binary-system-macos/0x002-001.png)  

open ./src/ImageLoader.h
![](/assets/img/note/2018-05-05-binary-system-macos/0x002-002.png) 

iherition relations
```
ImageLoader --> ImageLoaderMachO --> ImageLoaderMachOClassic
                                 --> ImageLoaderMachOCompressed
```

open ./src/dyld.c，this part we pay attention to how to load macho file，and ignore other operations
```
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
  ... //对全局变量一通操作
    try {
        // add dyld itself to UUID list
        addDyldImageToUUIDList();
        CRSetCrashLogMessage(sLoadingCrashMessage);
        // instantiate ImageLoader for main executable
        sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);//加载MACHO到image
        ... //不关心了
    }
}
```

instantiateFromLoadedImage
```
// The kernel maps in main executable before dyld gets control.  We need to 
// make an ImageLoader* for the already mapped in main executable.
static ImageLoader* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
    // try mach-o loader
    if ( isCompatibleMachO((const uint8_t*)mh, path) ) {//检测是否合法
        ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext); //加载
        addImage(image);
        return image;
    }
    
    throw "main executable not a known format";
}
```

isCompatibleMachO
```
bool isCompatibleMachO(const uint8_t* firstPage, const char* path)
{
#if CPU_SUBTYPES_SUPPORTED
    // 支持检测CPU版本的情况
    // It is deemed compatible if any of the following are true:
    //  1) mach_header subtype is in list of compatible subtypes for running processor
    //  2) mach_header subtype is same as running processor subtype
    //  3) mach_header subtype runs on all processor variants
    const mach_header* mh = (mach_header*)firstPage;
    if ( mh->magic == sMainExecutableMachHeader->magic ) { 
        //传入的mach-o文件的magic是否和加载的主mach-o文件是否相同
        //这一次运行到这里的时候mh与sMainExecutableMacHeader应该是指向同一个mach-o的
        if ( mh->cputype == sMainExecutableMachHeader->cputype ) {
            if ( (mh->cputype & CPU_TYPE_MASK) == sHostCPU ) {
                //加载的mh是否在当前平台可以运行。
                // get preference ordered list of subtypes that this machine can use
                const cpu_subtype_t* subTypePreferenceList = findCPUSubtypeList(mh->cputype, sHostCPUsubtype);
                if ( subTypePreferenceList != NULL ) {
                      //如果该CPU的版本存在一个检测的列表，则进行检测
                    // if image's subtype is in the list, it is compatible
                    for (const cpu_subtype_t* p = subTypePreferenceList; *p != CPU_SUBTYPE_END_OF_LIST; ++p) {
                        if ( *p == mh->cpusubtype )
                            return true;
                    }
                    // have list and not in list, so not compatible
                    throwf("incompatible cpu-subtype: 0x%08X in %s", mh->cpusubtype, path);
                }
                // unknown cpu sub-type, but if exact match for current subtype then ok to use
                if ( mh->cpusubtype == sHostCPUsubtype ) 
                    //加载的mh与当前运行环境的CPU版本相同
                    return true;
            }
            
            // cpu type has no ordered list of subtypes
             // 这两种CPU支持所有版本的mach-o文件
            switch (mh->cputype) {
                case CPU_TYPE_I386:
                case CPU_TYPE_X86_64:
                    // subtypes are not used or these architectures
                    return true;
            }
        }
    }
#else
    // For architectures that don't support cpu-sub-types
    // this just check the cpu type.
    // 不支持检测CPU版本的时候，就只判断是mh的版本与CPU相同。
    const mach_header* mh = (mach_header*)firstPage;
    if ( mh->magic == sMainExecutableMachHeader->magic ) {
        if ( mh->cputype == sMainExecutableMachHeader->cputype ) {
            return true;
        }
    }
#endif
    return false;
}
```

CPU_TYPE & CPU_SUBTYPE are in ./src/machine.h
```
#define CPU_TYPE_ANY        ((cpu_type_t) -1)

#define CPU_TYPE_VAX        ((cpu_type_t) 1)
/* skip             ((cpu_type_t) 2)    */
/* skip             ((cpu_type_t) 3)    */
/* skip             ((cpu_type_t) 4)    */
/* skip             ((cpu_type_t) 5)    */
#define CPU_TYPE_MC680x0    ((cpu_type_t) 6)
#define CPU_TYPE_X86        ((cpu_type_t) 7)
#define CPU_TYPE_I386       CPU_TYPE_X86        /* compatibility */
#define CPU_TYPE_X86_64     (CPU_TYPE_X86 | CPU_ARCH_ABI64)

/* skip CPU_TYPE_MIPS       ((cpu_type_t) 8)    */
/* skip             ((cpu_type_t) 9)    */
#define CPU_TYPE_MC98000    ((cpu_type_t) 10)
#define CPU_TYPE_HPPA           ((cpu_type_t) 11)
#define CPU_TYPE_ARM        ((cpu_type_t) 12)
#define CPU_TYPE_ARM64          (CPU_TYPE_ARM | CPU_ARCH_ABI64)
#define CPU_TYPE_MC88000    ((cpu_type_t) 13)
#define CPU_TYPE_SPARC      ((cpu_type_t) 14)
#define CPU_TYPE_I860       ((cpu_type_t) 15)
/* skip CPU_TYPE_ALPHA      ((cpu_type_t) 16)   */
/* skip             ((cpu_type_t) 17)   */
#define CPU_TYPE_POWERPC        ((cpu_type_t) 18)
#define CPU_TYPE_POWERPC64      (CPU_TYPE_POWERPC | CPU_ARCH_ABI64)
/*
 *  Machine subtypes (these are defined here, instead of in a machine
 *  dependent directory, so that any program can get all definitions
 *  regardless of where is it compiled).
 */

/*
 * Capability bits used in the definition of cpu_subtype.
 */
#define CPU_SUBTYPE_MASK    0xff000000  /* mask for feature flags */
#define CPU_SUBTYPE_LIB64   0x80000000  /* 64 bit libraries */


/*
 *  Object files that are hand-crafted to run on any
 *  implementation of an architecture are tagged with
 *  CPU_SUBTYPE_MULTIPLE.  This functions essentially the same as
 *  the "ALL" subtype of an architecture except that it allows us
 *  to easily find object files that may need to be modified
 *  whenever a new implementation of an architecture comes out.
 *
 *  It is the responsibility of the implementor to make sure the
 *  software handles unsupported implementations elegantly.
 */
#define CPU_SUBTYPE_MULTIPLE        ((cpu_subtype_t) -1)
#define CPU_SUBTYPE_LITTLE_ENDIAN   ((cpu_subtype_t) 0)
#define CPU_SUBTYPE_BIG_ENDIAN      ((cpu_subtype_t) 1)

/*
 *     Machine threadtypes.
 *     This is none - not defined - for most machine types/subtypes.
 */
#define CPU_THREADTYPE_NONE     ((cpu_threadtype_t) 0)

/*
 *  VAX subtypes (these do *not* necessary conform to the actual cpu
 *  ID assigned by DEC available via the SID register).
 */

#define CPU_SUBTYPE_VAX_ALL ((cpu_subtype_t) 0) 
#define CPU_SUBTYPE_VAX780  ((cpu_subtype_t) 1)
#define CPU_SUBTYPE_VAX785  ((cpu_subtype_t) 2)
#define CPU_SUBTYPE_VAX750  ((cpu_subtype_t) 3)
#define CPU_SUBTYPE_VAX730  ((cpu_subtype_t) 4)
#define CPU_SUBTYPE_UVAXI   ((cpu_subtype_t) 5)
#define CPU_SUBTYPE_UVAXII  ((cpu_subtype_t) 6)
#define CPU_SUBTYPE_VAX8200 ((cpu_subtype_t) 7)
#define CPU_SUBTYPE_VAX8500 ((cpu_subtype_t) 8)
#define CPU_SUBTYPE_VAX8600 ((cpu_subtype_t) 9)
#define CPU_SUBTYPE_VAX8650 ((cpu_subtype_t) 10)
#define CPU_SUBTYPE_VAX8800 ((cpu_subtype_t) 11)
#define CPU_SUBTYPE_UVAXIII ((cpu_subtype_t) 12)
```

ImageLoaderMachO::instantiateMainExecutable
```
// create image for main executable
ImageLoader* ImageLoaderMachO::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path, const LinkContext& context)
{
    //dyld::log("ImageLoader=%ld, ImageLoaderMachO=%ld, ImageLoaderMachOClassic=%ld, ImageLoaderMachOCompressed=%ld\n",
    //  sizeof(ImageLoader), sizeof(ImageLoaderMachO), sizeof(ImageLoaderMachOClassic), sizeof(ImageLoaderMachOCompressed));
    bool compressed;
    unsigned int segCount;
    unsigned int libCount;
    const linkedit_data_command* codeSigCmd;
    const encryption_info_command* encryptCmd;
    sniffLoadCommands(mh, path, false, &compressed, &segCount, &libCount, context, &codeSigCmd, &encryptCmd); //判断macho是普通的还是压缩的
    // instantiate concrete class based on content of load commands
    if ( compressed ) 
        return ImageLoaderMachOCompressed::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
    else
#if SUPPORT_CLASSIC_MACHO
        return ImageLoaderMachOClassic::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
#else
        throw "missing LC_DYLD_INFO load command";
#endif
}
```

sniffLoadCommands
```
// determine if this mach-o file has classic or compressed LINKEDIT and number of segments it has
void ImageLoaderMachO::sniffLoadCommands(const macho_header* mh, const char* path, bool inCache, bool* compressed,
                                            unsigned int* segCount, unsigned int* libCount, const LinkContext& context,
                                            const linkedit_data_command** codeSigCmd,
                                            const encryption_info_command** encryptCmd)
{
    *compressed = false;
    *segCount = 0;
    *libCount = 0;
    *codeSigCmd = NULL;
    *encryptCmd = NULL;

    const uint32_t cmd_count = mh->ncmds;
    //获取cmds的个数,保存在mach-o文件的头部ncmds字段中
    const struct load_command* const startCmds    = (struct load_command*)(((uint8_t*)mh) + sizeof(macho_header));
    //获取command段开始的地址，startCmds = mach-o地址 + mach-o头部长度
    const struct load_command* const endCmds = (struct load_command*)(((uint8_t*)mh) + sizeof(macho_header) + mh->sizeofcmds);
    //获取command段结束的地址，endCmds = mach-o地址 + mach-o头部长度 + cmds所用的长度
    const struct load_command* cmd = startCmds;
    bool foundLoadCommandSegment = false;
    for (uint32_t i = 0; i < cmd_count; ++i) {
        //遍历每一个command
        uint32_t cmdLength = cmd->cmdsize;
        struct macho_segment_command* segCmd;
        if ( cmdLength < 8 ) {
            //格式检测：长度就不对抛出异常
            dyld::throwf("malformed mach-o image: load command #%d length (%u) too small in %s",
                                               i, cmdLength, path);
        }
        const struct load_command* const nextCmd = (const struct load_command*)(((char*)cmd)+cmdLength);
        if ( (nextCmd > endCmds) || (nextCmd < cmd) ) {
            //格式检测：通过当前command长度寻找nextcmd时，如果nextcmd指不合法的位置就抛出异常
            dyld::throwf("malformed mach-o image: load command #%d length (%u) would exceed sizeofcmds (%u) in %s",
                                               i, cmdLength, mh->sizeofcmds, path);
        }
        switch (cmd->cmd) {
            //针对每种类型的command做不同的操作
            case LC_DYLD_INFO:
            case LC_DYLD_INFO_ONLY:
                *compressed = true;
                //mach-o文件为压缩的mach-o文件
                break;
            case LC_SEGMENT_COMMAND:
                segCmd = (struct macho_segment_command*)cmd;
#if __MAC_OS_X_VERSION_MIN_REQUIRED
                // rdar://problem/19617624 allow unmapped segments on OSX (but not iOS)
                // 如果segCmd的文件长度大于segCmd的vmszie，抛出异常。
                // todo:结合mach-o文件加载内核部分再详细解释
                if ( (segCmd->filesize > segCmd->vmsize) && (segCmd->vmsize != 0) )
#else
                if ( segCmd->filesize > segCmd->vmsize )
#endif
                    dyld::throwf("malformed mach-o image: segment load command %s filesize is larger than vmsize", segCmd->segname);
                // ignore zero-sized segments
                // 忽略长度为0的segments，计算segments的个数
                if ( segCmd->vmsize != 0 )
                    *segCount += 1;
                if ( context.codeSigningEnforced ) {
                    //如果有强制代码签名，则需要更加严格的segments格式合法性检测。
                    uintptr_t vmStart   = segCmd->vmaddr;
                    uintptr_t vmSize    = segCmd->vmsize;
                    uintptr_t vmEnd     = vmStart + vmSize;
                    uintptr_t fileStart = segCmd->fileoff;
                    uintptr_t fileSize  = segCmd->filesize;
                    
                    //对参数做合法性检测，如果mach-o文件不合法则抛出异常
                    if ( (intptr_t)(vmEnd) < 0)
                        dyld::throwf("malformed mach-o image: segment load command %s vmsize too large", segCmd->segname);
                    if ( vmStart > vmEnd )
                        dyld::throwf("malformed mach-o image: segment load command %s wraps around address space", segCmd->segname);
                    if ( vmSize != fileSize ) {
                        if ( (segCmd->initprot == 0) && (fileSize != 0) )
                            dyld::throwf("malformed mach-o image: unaccessable segment %s has filesize != 0", segCmd->segname);
                        else if ( vmSize < fileSize )
                            dyld::throwf("malformed mach-o image: segment %s has vmsize < filesize", segCmd->segname);
                    }
                    if ( inCache ) {
                        if ( (fileSize != 0) && (segCmd->initprot == (VM_PROT_READ | VM_PROT_EXECUTE)) ) {
                            if ( foundLoadCommandSegment )
                                throw "load commands in multiple segments";
                            foundLoadCommandSegment = true;
                        }
                    }
                    else if ( (fileStart < mh->sizeofcmds) && (fileSize != 0) ) {
                        // <rdar://problem/7942521> all load commands must be in an executable segment
                        if ( (fileStart != 0) || (fileSize < (mh->sizeofcmds+sizeof(macho_header))) )
                            dyld::throwf("malformed mach-o image: segment %s does not span all load commands", segCmd->segname); 
                        if ( segCmd->initprot != (VM_PROT_READ | VM_PROT_EXECUTE) ) 
                            dyld::throwf("malformed mach-o image: load commands found in segment %s with wrong permissions", segCmd->segname); 
                        if ( foundLoadCommandSegment )
                            throw "load commands in multiple segments";
                        foundLoadCommandSegment = true;
                    }

                    const struct macho_section* const sectionsStart = (struct macho_section*)((char*)segCmd + sizeof(struct macho_segment_command));
                    const struct macho_section* const sectionsEnd = &sectionsStart[segCmd->nsects];
                    for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {
                        if (!inCache && sect->offset != 0 && ((sect->offset + sect->size) > (segCmd->fileoff + segCmd->filesize)))
                            dyld::throwf("malformed mach-o image: section %s,%s of '%s' exceeds segment %s booundary", sect->segname, sect->sectname, path, segCmd->segname);
                    }
                }
                break;
            case LC_SEGMENT_COMMAND_WRONG:
                dyld::throwf("malformed mach-o image: wrong LC_SEGMENT[_64] for architecture"); 
                break;
            case LC_LOAD_DYLIB:
            case LC_LOAD_WEAK_DYLIB:
            case LC_REEXPORT_DYLIB:
            case LC_LOAD_UPWARD_DYLIB:
                *libCount += 1;
                break;
            case LC_CODE_SIGNATURE:
                *codeSigCmd = (struct linkedit_data_command*)cmd; // only support one LC_CODE_SIGNATURE per image
                break;
            case LC_ENCRYPTION_INFO:
            case LC_ENCRYPTION_INFO_64:
                *encryptCmd = (struct encryption_info_command*)cmd; // only support one LC_ENCRYPTION_INFO[_64] per image
                break;
        }
        cmd = nextCmd;
    }

    if ( context.codeSigningEnforced && !foundLoadCommandSegment )
        throw "load commands not in a segment";

    // <rdar://problem/13145644> verify every segment does not overlap another segment
    if ( context.codeSigningEnforced ) {
        //如果设置了强制代码签名，则需要更加严格的检测，确认segments没有互相覆盖。
        uintptr_t lastFileStart = 0;
        uintptr_t linkeditFileStart = 0;
        const struct load_command* cmd1 = startCmds;
        for (uint32_t i = 0; i < cmd_count; ++i) {
            if ( cmd1->cmd == LC_SEGMENT_COMMAND ) {
                struct macho_segment_command* segCmd1 = (struct macho_segment_command*)cmd1;
                uintptr_t vmStart1   = segCmd1->vmaddr;
                uintptr_t vmEnd1     = segCmd1->vmaddr + segCmd1->vmsize;
                uintptr_t fileStart1 = segCmd1->fileoff;
                uintptr_t fileEnd1   = segCmd1->fileoff + segCmd1->filesize;

                if (fileStart1 > lastFileStart)
                    lastFileStart = fileStart1;

                if ( strcmp(&segCmd1->segname[0], "__LINKEDIT") == 0 ) {
                    linkeditFileStart = fileStart1;
                }

                const struct load_command* cmd2 = startCmds;
                for (uint32_t j = 0; j < cmd_count; ++j) {
                    if ( cmd2 == cmd1 )
                        continue;
                    if ( cmd2->cmd == LC_SEGMENT_COMMAND ) {
                        struct macho_segment_command* segCmd2 = (struct macho_segment_command*)cmd2;
                        uintptr_t vmStart2   = segCmd2->vmaddr;
                        uintptr_t vmEnd2     = segCmd2->vmaddr + segCmd2->vmsize;
                        uintptr_t fileStart2 = segCmd2->fileoff;
                        uintptr_t fileEnd2   = segCmd2->fileoff + segCmd2->filesize;
                        if ( ((vmStart2 <= vmStart1) && (vmEnd2 > vmStart1) && (vmEnd1 > vmStart1)) 
                        || ((vmStart2 >= vmStart1) && (vmStart2 < vmEnd1) && (vmEnd2 > vmStart2)) )
                            dyld::throwf("malformed mach-o image: segment %s vm overlaps segment %s", segCmd1->segname, segCmd2->segname);
                        if ( ((fileStart2 <= fileStart1) && (fileEnd2 > fileStart1) && (fileEnd1 > fileStart1))
                          || ((fileStart2 >= fileStart1) && (fileStart2 < fileEnd1) && (fileEnd2 > fileStart2)) )
                            dyld::throwf("malformed mach-o image: segment %s file content overlaps segment %s", segCmd1->segname, segCmd2->segname); 
                    }
                    cmd2 = (const struct load_command*)(((char*)cmd2)+cmd2->cmdsize);
                }
            }
            cmd1 = (const struct load_command*)(((char*)cmd1)+cmd1->cmdsize);
        }

        if (lastFileStart != linkeditFileStart)
            dyld::throwf("malformed mach-o image: __LINKEDIT must be last segment");
    }

    // fSegmentsArrayCount is only 8-bits
    if ( *segCount > 255 )
        dyld::throwf("malformed mach-o image: more than 255 segments in %s", path);

    // fSegmentsArrayCount is only 8-bits
    if ( *libCount > 4095 )
        dyld::throwf("malformed mach-o image: more than 4095 dependent libraries in %s", path);

    if ( needsAddedLibSystemDepency(*libCount, mh) )
        *libCount = 1;
}
```

ImageLoaderMachOClassic::instantiateMainExecutable
```
// create image for main executable
ImageLoaderMachOClassic* ImageLoaderMachOClassic::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path, 
                                                                        unsigned int segCount, unsigned int libCount, const LinkContext& context)
{
    ImageLoaderMachOClassic* image = ImageLoaderMachOClassic::instantiateStart(mh, path, segCount, libCount);
    //实例化image
    
    //为PIE设置所需的参数，Position Independent Executables
    //todo:分析了解PIE
    // set slide for PIE programs
    image->setSlide(slide);

    // for PIE record end of program, to know where to start loading dylibs
    if ( slide != 0 )
        fgNextPIEDylibAddress = (uintptr_t)image->getEnd();

    //设置一堆参数
    image->disableCoverageCheck();
    image->instantiateFinish(context);
    image->setMapped(context);

#if __i386__
    // kernel may have mapped in __IMPORT segment read-only, we need it read/write to do binding
    if ( image->fReadOnlyImportSegment ) {
        for(unsigned int i=0; i < image->fSegmentsCount; ++i) {
            if ( image->segIsReadOnlyImport(i) )
                image->segMakeWritable(i, context);
        }
    }
#endif
    
    //如果设置了context.verboseMapping，打印详细的LOG
    if ( context.verboseMapping ) {
        dyld::log("dyld: Main executable mapped %s\n", path);
        for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
            const char* name = image->segName(i);
            if ( (strcmp(name, "__PAGEZERO") == 0) || (strcmp(name, "__UNIXSTACK") == 0)  )
                dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segPreferredLoadAddress(i), image->segPreferredLoadAddress(i)+image->segSize(i));
            else
                dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segActualLoadAddress(i), image->segActualEndAddress(i));
        }
    }

    return image;
}
```

instantiateStart
```
// construct ImageLoaderMachOClassic using "placement new" with SegmentMachO objects array at end
ImageLoaderMachOClassic* ImageLoaderMachOClassic::instantiateStart(const macho_header* mh, const char* path,
                                                                        unsigned int segCount, unsigned int libCount)
{
    size_t size = sizeof(ImageLoaderMachOClassic) + segCount * sizeof(uint32_t) + libCount * sizeof(ImageLoader*);
    ImageLoaderMachOClassic* allocatedSpace = static_cast<ImageLoaderMachOClassic*>(malloc(size));
    if ( allocatedSpace == NULL )
        throw "malloc failed";
    uint32_t* segOffsets = ((uint32_t*)(((uint8_t*)allocatedSpace) + sizeof(ImageLoaderMachOClassic)));
    bzero(&segOffsets[segCount], libCount*sizeof(void*));   // zero out lib array
    return new (allocatedSpace) ImageLoaderMachOClassic(mh, path, segCount, segOffsets, libCount);
}
```

ImageLoaderMachO
```
ImageLoaderMachOClassic::ImageLoaderMachOClassic(const macho_header* mh, const char* path, 
                                                    unsigned int segCount, uint32_t segOffsets[], unsigned int libCount)
 : ImageLoaderMachO(mh, path, segCount, segOffsets, libCount), fStrings(NULL), fSymbolTable(NULL), fDynamicInfo(NULL)
{
}

ImageLoaderMachO::ImageLoaderMachO(const macho_header* mh, const char* path, unsigned int segCount, 
                                                                uint32_t segOffsets[], unsigned int libCount)
 : ImageLoader(path, libCount), fCoveredCodeLength(0), fMachOData((uint8_t*)mh), fLinkEditBase(NULL), fSlide(0),
    fEHFrameSectionOffset(0), fUnwindInfoSectionOffset(0), fDylibIDOffset(0), 
fSegmentsCount(segCount), fIsSplitSeg(false), fInSharedCache(false),
#if TEXT_RELOC_SUPPORT
    fTextSegmentRebases(false),
    fTextSegmentBinds(false),
#endif
#if __i386__
    fReadOnlyImportSegment(false),
#endif
    fHasSubLibraries(false), fHasSubUmbrella(false), fInUmbrella(false), fHasDOFSections(false), fHasDashInit(false),
    fHasInitializers(false), fHasTerminators(false), fRegisteredAsRequiresCoalescing(false)
{
    fIsSplitSeg = ((mh->flags & MH_SPLIT_SEGS) != 0);        

    // construct SegmentMachO object for each LC_SEGMENT cmd using "placement new" to put 
    // each SegmentMachO object in array at end of ImageLoaderMachO object
    const uint32_t cmd_count = mh->ncmds;
    const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
    const struct load_command* cmd = cmds;
    for (uint32_t i = 0, segIndex=0; i < cmd_count; ++i) {
        if ( cmd->cmd == LC_SEGMENT_COMMAND ) {
            const struct macho_segment_command* segCmd = (struct macho_segment_command*)cmd;
            // ignore zero-sized segments
            if ( segCmd->vmsize != 0 ) {
                // record offset of load command
                segOffsets[segIndex++] = (uint32_t)((uint8_t*)segCmd - fMachOData);
            }
        }
        cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
    }

}
```

addimage
```
static void addImage(ImageLoader* image)
{
    // add to master list
    // 对所有images的容器原子添加image
    allImagesLock();
        sAllImages.push_back(image);
    allImagesUnlock();
    
    // update mapped ranges
    // 更新内存分布的数据
    uintptr_t lastSegStart = 0;
    uintptr_t lastSegEnd = 0;
    for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
        if ( image->segUnaccessible(i) ) 
            continue;
        uintptr_t start = image->segActualLoadAddress(i);
        uintptr_t end = image->segActualEndAddress(i);
        if ( start == lastSegEnd ) {
            // two segments are contiguous, just record combined segments
            lastSegEnd = end;
        }
        else {
            // non-contiguous segments, record last (if any)
            if ( lastSegEnd != 0 )
                addMappedRange(image, lastSegStart, lastSegEnd);
            lastSegStart = start;
            lastSegEnd = end;
        }       
    }
    if ( lastSegEnd != 0 )
        addMappedRange(image, lastSegStart, lastSegEnd);

    
    if ( sEnv.DYLD_PRINT_LIBRARIES || (sEnv.DYLD_PRINT_LIBRARIES_POST_LAUNCH && (sMainExecutable!=NULL) && sMainExecutable->isLinked()) ) {
        dyld::log("dyld: loaded: %s\n", image->getPath());
    }
    
}
```

0x003 introduce to macho format

Mach-O structure
![](/assets/img/note/2018-05-05-binary-system-macos/0x003-001.png)  

Mach-O header
```
/*
 * The 32-bit mach header appears at the very beginning of the object file for
 * 32-bit architectures.
 */
struct mach_header {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu specifier */
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* type of file */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
};

/* Constant for the magic field of the mach_header (32-bit architectures) */
#define MH_MAGIC    0xfeedface  /* the mach magic number */
#define MH_CIGAM    0xcefaedfe  /* NXSwapInt(MH_MAGIC) */

/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu specifier */
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* type of file */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
    uint32_t    reserved;   /* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */
```

check Mach-O headers on a file by otool
```
$ otool -h ht
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777223          3  0x80           2    18       2080 0x00218085
```

filetype can label exec、lib、coredump and so on...
````
#define MH_OBJECT   0x1     /* relocatable object file */
#define MH_EXECUTE  0x2     /* demand paged executable file */
#define MH_FVMLIB   0x3     /* fixed VM shared library file */
#define MH_CORE     0x4     /* core file */
#define MH_PRELOAD  0x5     /* preloaded executable file */
#define MH_DYLIB    0x6     /* dynamically bound shared library */
#define MH_DYLINKER 0x7     /* dynamic link editor */
#define MH_BUNDLE   0x8     /* dynamically bound bundle file */
#define MH_DYLIB_STUB   0x9     /* shared library stub for static */
                    /*  linking only, no section contents */
#define MH_DSYM     0xa     /* companion file with only debug */
                    /*  sections */
#define MH_KEXT_BUNDLE  0xb     /* x86_64 kexts */
```

flags label dyld loading parameter
```
// EXTERNAL_HEADERS/mach-o/x86_64/loader.h
#define MH_INCRLINK 0x2     /* the object file is the output of an
                       incremental link against a base file
                       and can't be link edited again */
#define MH_DYLDLINK 0x4     /* the object file is input for the
                       dynamic linker and can't be staticly
                       link edited again */
#define MH_BINDATLOAD   0x8     /* the object file's undefined
                       references are bound by the dynamic
                       linker when loaded. */
#define MH_PREBOUND 0x10        /* the file has its dynamic undefined
                       references prebound. */
#define MH_SPLIT_SEGS   0x20        /* the file has its read-only and
                       read-write segments split */
#define MH_LAZY_INIT    0x40        /* the shared library init routine is
                       to be run lazily via catching memory
                       faults to its writeable segments
                       (obsolete) */
#define MH_TWOLEVEL 0x80        /* the image is using two-level name
                       space bindings */
...
```

load_command structure
```
struct load_command {
    uint32_t cmd;       /* type of load command */
    uint32_t cmdsize;   /* total size of command in bytes */
};

after loading header，load_command would reslove to load macho data by kernel
```
static
load_return_t
parse_machfile(
    struct vnode        *vp,       
    vm_map_t        map,
    thread_t        thread,
    struct mach_header  *header,
    off_t           file_offset,
    off_t           macho_size,
    int         depth,
    int64_t         aslr_offset,
    int64_t         dyld_aslr_offset,
    load_result_t       *result
)
{
    [...] //此处省略大量初始化与检测

        /*
         * Loop through each of the load_commands indicated by the
         * Mach-O header; if an absurd value is provided, we just
         * run off the end of the reserved section by incrementing
         * the offset too far, so we are implicitly fail-safe.
         */
        offset = mach_header_sz;
        ncmds = header->ncmds;

        while (ncmds--) {
            /*
             *  Get a pointer to the command.
             */
            lcp = (struct load_command *)(addr + offset);
            //lcp设为当前要解析的cmd的地址
            oldoffset = offset;
            //oldoffset是从macho文件内存开始的地方偏移到当前command的偏移量
            offset += lcp->cmdsize;
            //重新计算offset，再加上当前command的长度，offset的值为文件内存起始地址到下一个command的偏移量
            /*
             * Perform prevalidation of the struct load_command
             * before we attempt to use its contents.  Invalid
             * values are ones which result in an overflow, or
             * which can not possibly be valid commands, or which
             * straddle or exist past the reserved section at the
             * start of the image.
             */
            if (oldoffset > offset ||
                lcp->cmdsize < sizeof(struct load_command) ||
                offset > header->sizeofcmds + mach_header_sz) {
                ret = LOAD_BADMACHO;
                break;
            }
            //做了一个检测，与如何加载进入内存无关

            /*
             * Act on struct load_command's for which kernel
             * intervention is required.
             */
            switch(lcp->cmd) {
            case LC_SEGMENT:
                [...]
                ret = load_segment(lcp,
                                   header->filetype,
                                   control,
                                   file_offset,
                                   macho_size,
                                   vp,
                                   map,
                                   slide,
                                   result);
                break;
            case LC_SEGMENT_64:
                [...]
                ret = load_segment(lcp,
                                   header->filetype,
                                   control,
                                   file_offset,
                                   macho_size,
                                   vp,
                                   map,
                                   slide,
                                   result);
                break;
            case LC_UNIXTHREAD:
                if (pass != 1)
                    break;
                ret = load_unixthread(
                         (struct thread_command *) lcp,
                         thread,
                         slide,
                         result);
                break;
            case LC_MAIN:
                if (pass != 1)
                    break;
                if (depth != 1)
                    break;
                ret = load_main(
                         (struct entry_point_command *) lcp,
                         thread,
                         slide,
                         result);
                break;
            case LC_LOAD_DYLINKER:
                if (pass != 3)
                    break;
                if ((depth == 1) && (dlp == 0)) {
                    dlp = (struct dylinker_command *)lcp;
                    dlarchbits = (header->cputype & CPU_ARCH_MASK);
                } else {
                    ret = LOAD_FAILURE;
                }
                break;
            case LC_UUID:
                if (pass == 1 && depth == 1) {
                    ret = load_uuid((struct uuid_command *) lcp,
                            (char *)addr + mach_header_sz + header->sizeofcmds,
                            result);
                }
                break;
            case LC_CODE_SIGNATURE:
                [...]
                ret = load_code_signature(
                    (struct linkedit_data_command *) lcp,
                    vp,
                    file_offset,
                    macho_size,
                    header->cputype,
                    result);
                [...]
                break;
#if CONFIG_CODE_DECRYPTION
            case LC_ENCRYPTION_INFO:
            case LC_ENCRYPTION_INFO_64:
                if (pass != 3)
                    break;
                ret = set_code_unprotect(
                    (struct encryption_info_command *) lcp,
                    addr, map, slide, vp, file_offset,
                    header->cputype, header->cpusubtype);
                if (ret != LOAD_SUCCESS) {
                    printf("proc %d: set_code_unprotect() error %d "
                           "for file \"%s\"\n",
                           p->p_pid, ret, vp->v_name);
                    /* 
                     * Don't let the app run if it's 
                     * encrypted but we failed to set up the
                     * decrypter. If the keys are missing it will
                     * return LOAD_DECRYPTFAIL.
                     */
                     if (ret == LOAD_DECRYPTFAIL) {
                        /* failed to load due to missing FP keys */
                        proc_lock(p);
                        p->p_lflag |= P_LTERM_DECRYPTFAIL;
                        proc_unlock(p);
                     }
                     psignal(p, SIGKILL);
                }
                break;
#endif
            default:
                /* Other commands are ignored by the kernel */
                ret = LOAD_SUCCESS;
                break;
            }
            if (ret != LOAD_SUCCESS)
                break;
        }
        if (ret != LOAD_SUCCESS)
            break;
    }

    [...] //此处略去加载之后的处理代码
}
```

cmdsize segment
```
...
lcp = (struct load_command *)(addr + offset);
//lcp设为当前要解析的cmd的地址
oldoffset = offset;
//oldoffset是从macho文件内存开始的地方偏移到当前command的偏移量
offset += lcp->cmdsize;
//重新计算offset，再加上当前command的长度，offset的值为文件内存起始地址到下一个command的偏移量
...
```

cmd segment
```
switch(lcp->cmd) {
            case LC_SEGMENT:
                [...]
                ret = load_segment(lcp,
                                   header->filetype,
                                   control,
                                   file_offset,
                                   macho_size,
                                   vp,
                                   map,
                                   slide,
                                   result);
                break;
            case LC_SEGMENT_64:
                [...]
                ret = load_segment(lcp,
                                   header->filetype,
                                   control,
                                   file_offset,
                                   macho_size,
                                   vp,
                                   map,
                                   slide,
                                   result);
                break;
            case LC_UNIXTHREAD:
                if (pass != 1)
                    break;
                ret = load_unixthread(
                         (struct thread_command *) lcp,
                         thread,
                         slide,
                         result);
                break;
            case LC_MAIN:
                if (pass != 1)
                    break;
                if (depth != 1)
                    break;
                ret = load_main(
                         (struct entry_point_command *) lcp,
                         thread,
                         slide,
                         result);
                break;
            case LC_LOAD_DYLINKER:
                if (pass != 3)
                    break;
                if ((depth == 1) && (dlp == 0)) {
                    dlp = (struct dylinker_command *)lcp;
                    dlarchbits = (header->cputype & CPU_ARCH_MASK);
                } else {
                    ret = LOAD_FAILURE;
                }
                break;
            case LC_UUID:
                if (pass == 1 && depth == 1) {
                    ret = load_uuid((struct uuid_command *) lcp,
                            (char *)addr + mach_header_sz + header->sizeofcmds,
                            result);
                }
                break;
            case LC_CODE_SIGNATURE:
                [...]
                ret = load_code_signature(
                    (struct linkedit_data_command *) lcp,
                    vp,
                    file_offset,
                    macho_size,
                    header->cputype,
                    result);
                [...]
                break;
#if CONFIG_CODE_DECRYPTION
            case LC_ENCRYPTION_INFO:
            case LC_ENCRYPTION_INFO_64:
                if (pass != 3)
                    break;
                ret = set_code_unprotect(
                    (struct encryption_info_command *) lcp,
                    addr, map, slide, vp, file_offset,
                    header->cputype, header->cpusubtype);
                if (ret != LOAD_SUCCESS) {
                    printf("proc %d: set_code_unprotect() error %d "
                           "for file \"%s\"\n",
                           p->p_pid, ret, vp->v_name);
                    /* 
                     * Don't let the app run if it's 
                     * encrypted but we failed to set up the
                     * decrypter. If the keys are missing it will
                     * return LOAD_DECRYPTFAIL.
                     */
                     if (ret == LOAD_DECRYPTFAIL) {
                        /* failed to load due to missing FP keys */
                        proc_lock(p);
                        p->p_lflag |= P_LTERM_DECRYPTFAIL;
                        proc_unlock(p);
                     }
                     psignal(p, SIGKILL);
                }
                break;
#endif
            default:
                /* Other commands are ignored by the kernel */
                ret = LOAD_SUCCESS;
                break;
            }
```

segment
```
struct segment_command { /* for 32-bit architectures */
    uint32_t    cmd;        /* LC_SEGMENT */
    uint32_t    cmdsize;    /* includes sizeof section structs */
    char        segname[16];    /* segment name */
    uint32_t    vmaddr;     /* memory address of this segment */
    uint32_t    vmsize;     /* memory size of this segment */
    uint32_t    fileoff;    /* file offset of this segment */
    uint32_t    filesize;   /* amount to map from the file */
    vm_prot_t   maxprot;    /* maximum VM protection */
    vm_prot_t   initprot;   /* initial VM protection */
    uint32_t    nsects;     /* number of sections in segment */
    uint32_t    flags;      /* flags */
};


struct segment_command_64 { /* for 64-bit architectures */
    uint32_t    cmd;        /* LC_SEGMENT_64 */
    uint32_t    cmdsize;    /* includes sizeof section_64 structs */
    char        segname[16];    /* segment name */
    uint64_t    vmaddr;     /* memory address of this segment */
    uint64_t    vmsize;     /* memory size of this segment */
    uint64_t    fileoff;    /* file offset of this segment */
    uint64_t    filesize;   /* amount to map from the file */
    vm_prot_t   maxprot;    /* maximum VM protection */
    vm_prot_t   initprot;   /* initial VM protection */
    uint32_t    nsects;     /* number of sections in segment */
    uint32_t    flags;      /* flags */
};
```

section
```
struct section { /* for 32-bit architectures */
    char        sectname[16];   /* name of this section */
    char        segname[16];    /* segment this section goes in */
    uint32_t    addr;       /* memory address of this section */
    uint32_t    size;       /* size in bytes of this section */
    uint32_t    offset;     /* file offset of this section */
    uint32_t    align;      /* section alignment (power of 2) */
    uint32_t    reloff;     /* file offset of relocation entries */
    uint32_t    nreloc;     /* number of relocation entries */
    uint32_t    flags;      /* flags (section type and attributes)*/
    uint32_t    reserved1;  /* reserved (for offset or index) */
    uint32_t    reserved2;  /* reserved (for count or sizeof) */
};

struct section_64 { /* for 64-bit architectures */
    char        sectname[16];   /* name of this section */
    char        segname[16];    /* segment this section goes in */
    uint64_t    addr;       /* memory address of this section */
    uint64_t    size;       /* size in bytes of this section */
    uint32_t    offset;     /* file offset of this section */
    uint32_t    align;      /* section alignment (power of 2) */
    uint32_t    reloff;     /* file offset of relocation entries */
    uint32_t    nreloc;     /* number of relocation entries */
    uint32_t    flags;      /* flags (section type and attributes)*/
    uint32_t    reserved1;  /* reserved (for offset or index) */
    uint32_t    reserved2;  /* reserved (for count or sizeof) */
    uint32_t    reserved3;  /* reserved */
};
```

section examples
```
Section         function
__text          code
__cstring       constant string
__const const   variable
__DATA.__bss    bss segment
```


## 0x004 macho dynamic link

level1.c
```
#include <stdio.h>

int main(int argc, const char * argv[]) {
    // insert code here...
    printf("Hello, World!\n");
    printf("2Hello, World!\n");
    return 0;
}
```

first printf
```
level1`main:                                                                                                        │
->  0x100000f52 <+34>: callq  0x100000f76               ; symbol stub for: printf                                   │
    0x100000f57 <+39>: leaq   0x47(%rip), %rdi          ; "2Hello, World!\n"                                        │
    0x100000f5e <+46>: movl   %eax, -0x14(%rbp)                                                                     │
    0x100000f61 <+49>: movb   $0x0, %al                                                                             │
Target 0: (level1) stopped.
```

step into
```
level1`printf:                                                                                                      │
->  0x100000f76 <+0>: jmpq   *0x94(%rip)               ; (void *)0x0000000100000f8c                                 │
    0x100000f7c:      leaq   0x85(%rip), %r11          ; (void *)0x0000000000000000                                 │
    0x100000f83:      pushq  %r11                                                                                   │
    0x100000f85:      jmpq   *0x75(%rip)               ; (void *)0x00007fffb4666168: dyld_stub_binder               │
Target 0: (level1) stopped.
```

get 0x0000000100000f8c by Lazy Symbol Pointers
![](/assets/img/note/2018-05-05-binary-system-macos/0x004-001.png) 

jmp to 0x100000f8c
```
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step into                                   │
    frame #0: 0x0000000100000f8c level1                                                                             │
->  0x100000f8c: pushq  $0x0                                                                                        │
    0x100000f91: jmp    0x100000f7c                                                                                 │
    0x100000f96: gs                                                                                                 │
    0x100000f98: insb   %dx, %es:(%rdi)                                                                             │
Target 0: (level1) stopped.
```

push 0x0, the symbol num，you can get it at Symbol Stub
![](/assets/img/note/2018-05-05-binary-system-macos/0x004-002.png) 

jump to __stub__helper, calculate prtinf_addr by calling dyld_stubbinder
```
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over                                   │
    frame #0: 0x0000000100000f7c level1                                                                             │
->  0x100000f7c: leaq   0x85(%rip), %r11          ; (void *)0x0000000000000000                                      │
    0x100000f83: pushq  %r11                                                                                        │
    0x100000f85: jmpq   *0x75(%rip)               ; (void *)0x00007fffb4666168: dyld_stub_binder                    │
    0x100000f8b: nop                                                                                                │
Target 0: (level1) stopped.
```
![](/assets/img/note/2018-05-05-binary-system-macos/0x004-003.png) 

goto second printf，now we get printf_addr at 0x0000000100000f76(Lazy Symbol Pointers)
```
level1`printf:                                                                                                      │
->  0x100000f76 <+0>: jmpq   *0x94(%rip)               ; (void *)0x00007fffb46e3180: printf                         │
    0x100000f7c:      leaq   0x85(%rip), %r11          ; (void *)0x0000000100045b80: initialPoolContent + 592       │
    0x100000f83:      pushq  %r11                                                                                   │
    0x100000f85:      jmpq   *0x75(%rip)               ; (void *)0x00007fffb4666168: dyld_stub_binder               │
Target 0: (level1) stopped.
```

That's all, but we need to learn other data structure

LC_SYMTAB，provide some info
![](/assets/img/note/2018-05-05-binary-system-macos/0x004-004.png) 
```
Symbol Table offset and num # include all calling funtions info
String Table offset and len
```

LC_DYSYMTAB，provide dynamic link table offset and num
![](/assets/img/note/2018-05-05-binary-system-macos/0x004-005.png) 


## 0x005 load analysis

after loading exec file, other dylib would be loaded as well
```
// load any inserted libraries
// 类似于linux里面的LD_PRELOAD
if  ( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
    for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
        loadInsertedDylib(*lib); //!!!动态加载dylib
}
// record count of inserted libraries so that a flat search will look at 
// inserted libraries, then main, then others.
sInsertedDylibCount = sAllImages.size()-1;
```

these functions may call load eventually
![](/assets/img/note/2018-05-05-binary-system-macos/0x005-001.png) 

load
```
//处理suffix字段。
//通过loadPhase0函数从share_cache中加载image。
//如果share_cache中不存在image，则再使用不同的参数调用loadPhase0函数，通过open函数读取文件并加载image到内存中。
//函数调用结束后的内存管理
//
//根据所有的环境变量生成路径，去加载一个ImageLoader
ImageLoader* load(const char* path, const LoadContext& context)
{
    CRSetCrashLogMessage2(path);
    const char* orgPath = path;
    
    //dyld::log("%s(%s)\n", __func__ , path);
    char realPath[PATH_MAX];
    // when DYLD_IMAGE_SUFFIX is in used, do a realpath(), otherwise a load of "Foo.framework/Foo" will not match
    // 当设置了DYLD_IMAGE_SUFFIX字段，需要使用realpath来加载
    if ( context.useSearchPaths && ( gLinkContext.imageSuffix != NULL) ) {
        if ( realpath(path, realPath) != NULL )
            path = realPath;
    }
    
    // try all path permutations and check against existing loaded images
    // 尝试各种路径组合去加载image
    ImageLoader* image = loadPhase0(path, orgPath, context, NULL);
    if ( image != NULL ) {
        CRSetCrashLogMessage2(NULL);
        return image;
    }

    // try all path permutations and try open() until first success
    std::vector<const char*> exceptions;
    image = loadPhase0(path, orgPath, context, &exceptions);
    
    /*...*/

    CRSetCrashLogMessage2(NULL);
    if ( image != NULL ) {
        /* 加载成功内存处理*/
        return image;
    }
    else if ( exceptions.size() == 0 ) {
        /* 出错处理 */
    }
    else {
        /* 出错处理 */
    }
}
```

loadPhase0
```
//遍历DYLD_ROOT_PATH环境变量，生成加载路径，调用loadPhase1。
//如果不存在DYLD_ROOT_PATH环境变量，则使用原始的路径
//
// try root substitutions
// 主要处理DYLD_ROOT_PATH环境变量的功能，修饰Loadimage时候的path
// 运行完结之后执行loadPhase1
static ImageLoader* loadPhase0(const char* path, const char* orgPath, const LoadContext& context, std::vector<const char*>* exceptions)
{
    //dyld::log("%s(%s, %p)\n", __func__ , path, exceptions);
    
    // handle DYLD_ROOT_PATH which forces absolute paths to use a new root
    if ( (gLinkContext.rootPaths != NULL) && (path[0] == '/') ) {
        for(const char* const* rootPath = gLinkContext.rootPaths ; *rootPath != NULL; ++rootPath) {
            char newPath[strlen(*rootPath) + strlen(path)+2];
            strcpy(newPath, *rootPath);
            strcat(newPath, path);
            ImageLoader* image = loadPhase1(newloadPhase1Path, orgPath, context, exceptions);
            if ( image != NULL )
                return image;
        }
    }

    // try raw path
    return loadPhase1(path, orgPath, context, exceptions);
}
```

loadPhase1
```
//通过LD_LIBRARY_PATH参数组成的所有路径，通过loadPhase2尝试加载image。
//当无法通过LD_LIBRARY_PATH获取image时，则通过DYLD_FRAMEWORK_PATH与DYLD_LIBRARY_PATH组成的路径，通过loadPhase2尝试加载image。
//如果上面两个流程都无法加载到image则通过原始路径通过loadPhase3尝试加载image。
//如果依然无法加载到image则通过DYLD_FALLBACK_FRAMEWORK_PATH环境变量，组成路径最后尝试加载image。
//
static ImageLoader* loadPhase1(const char* path, const char* orgPath, const LoadContext& context, std::vector<const char*>* exceptions)
{
    //dyld::log("%s(%s, %p)\n", __func__ , path, exceptions);
    ImageLoader* image = NULL;

    // handle LD_LIBRARY_PATH environment variables that force searching
    // 如果存在LD_LIBRARY_PATH变量，优先通过LD_LIBRARY_PATH中设置的路径进行搜索和加载
    if ( context.useLdLibraryPath && (sEnv.LD_LIBRARY_PATH != NULL) ) {
        image = loadPhase2(path, orgPath, context, NULL, sEnv.LD_LIBRARY_PATH, exceptions);
        if ( image != NULL )
            return image;
    }

    // handle DYLD_ environment variables that force searching
    // 如果使用了DYLD_FRAMEWORK_PATH或者sEnv.DYLD_LIBRARY_PATH，则使用这两个环境变量去加载
    if ( context.useSearchPaths && ((sEnv.DYLD_FRAMEWORK_PATH != NULL) || (sEnv.DYLD_LIBRARY_PATH != NULL)) ) {
        image = loadPhase2(path, orgPath, context, sEnv.DYLD_FRAMEWORK_PATH, sEnv.DYLD_LIBRARY_PATH, exceptions);
        if ( image != NULL )
            return image;
    }
    
    // try raw path
    // 如果上面的环境变量都没有设置，就使用原始地址去加载
    // 函数是loadphase3
    image = loadPhase3(path, orgPath, context, exceptions);
    if ( image != NULL )
        return image;
    
    // try fallback paths during second time (will open file)
    const char* const* fallbackLibraryPaths = sEnv.DYLD_FALLBACK_LIBRARY_PATH;
    if ( (fallbackLibraryPaths != NULL) && !context.useFallbackPaths )
        fallbackLibraryPaths = NULL;
    if ( !context.dontLoad  && (exceptions != NULL) && ((sEnv.DYLD_FALLBACK_FRAMEWORK_PATH != NULL) || (fallbackLibraryPaths != NULL)) ) {
        image = loadPhase2(path, orgPath, context, sEnv.DYLD_FALLBACK_FRAMEWORK_PATH, fallbackLibraryPaths, exceptions);
        if ( image != NULL )
            return image;
    }
        
    return NULL;
}
```

loadPhase2 to loadPhase4 reference
[loadPhase2](https://github.com/turingH/dyld_soucecode_analysis/blob/master/src/dyld.cpp#L2933)
[loadPhase3](https://github.com/turingH/dyld_soucecode_analysis/blob/master/src/dyld.cpp#L2820)
[loadPhase4](https://github.com/turingH/dyld_soucecode_analysis/blob/master/src/dyld.cpp#L2797)

loadPhase5
```
//loadPhase5根据参数exceptions的不同形成了两个不同的分支。
//loadPhase5load：通过读取文件，加载文件到内存中，实例化ImageLoader。
//loadPhase5check: 通过遍历已经加载的ImageLoader的容器，获取已经加载的ImageLoader。
//
// open or check existing
// 检测是否有覆盖的，修正path，最后调用loadPhase5Load或者check
static ImageLoader* loadPhase5(const char* path, const char* orgPath, const LoadContext& context, std::vector<const char*>* exceptions)
{
    //dyld::log("%s(%s, %p)\n", __func__ , path, exceptions);
    
    // check for specific dylib overrides
    for (std::vector<DylibOverride>::iterator it = sDylibOverrides.begin(); it != sDylibOverrides.end(); ++it) {
        if ( strcmp(it->installName, path) == 0 ) {
            path = it->override;
            break;
        }
    }
    
    if ( exceptions != NULL ) 
        return loadPhase5load(path, orgPath, context, exceptions);
    else
        return loadPhase5check(path, orgPath, context);
}
```

loadPhase5load
```
//防止Image改名，在Image的容器里面遍历，检查是否已经加载
//在SharedCache寻找是否存在Image的缓存，如果存在的使用ImageLoaderMachO::instantiateFromCache来实例化ImageLoader。
//如果上面两个都没有找到的话，就通过loadPhase5open打开文件，并读取到内存。
//
static ImageLoader* loadPhase5load(const char* path, const char* orgPath, const LoadContext& context, std::vector<const char*>* exceptions)
{
    //dyld::log("%s(%s, %p)\n", __func__ , path, exceptions);
    ImageLoader* image = NULL;

    // just return NULL if file not found, but record any other errors
    struct stat stat_buf;
    if ( my_stat(path, &stat_buf) == -1 ) {
        int err = errno;
        if ( err != ENOENT ) {
            exceptions->push_back(dyld::mkstringf("%s: stat() failed with errno=%d", path, err));
        }
        return NULL;
    }
    
    // in case image was renamed or found via symlinks, check for inode match
    image = findLoadedImage(stat_buf);
    if ( image != NULL )
        return image;
    
    // do nothing if not already loaded and if RTLD_NOLOAD or NSADDIMAGE_OPTION_RETURN_ONLY_IF_LOADED
    //RTLD_NOLOAD或者NSADDIMAGE_OPTION_RETURN_ONLY_IF_LOADED字段设置了则不进行加载
    if ( context.dontLoad )
        return NULL;

#if DYLD_SHARED_CACHE_SUPPORT
    // see if this image is in shared cache
    const macho_header* mhInCache;
    const char*         pathInCache;
    long                slideInCache;
    // 如果在sharedCacheImage中找到了，则通过cache来加载
    if ( findInSharedCacheImage(path, false, &stat_buf, &mhInCache, &pathInCache, &slideInCache) ) {
        image = ImageLoaderMachO::instantiateFromCache(mhInCache, pathInCache, slideInCache, stat_buf, gLinkContext);
        return checkandAddImage(image, context);
    }
#endif
    // file exists and is not in dyld shared cache, so open it
    // shared_cache中不存在image，则通过LoadPhase5open来加载image
    return loadPhase5open(path, context, stat_buf, exceptions);
}
```

loadPhase5open
```
//根据路径打开文件
//调用loadPhase6
//
static ImageLoader* loadPhase5open(const char* path, const LoadContext& context, const struct stat& stat_buf, std::vector<const char*>* exceptions)
{
    //dyld::log("%s(%s, %p)\n", __func__ , path, exceptions);

    // open file (automagically closed when this function exits)
    FileOpener file(path);
        
    // just return NULL if file not found, but record any other errors
    if ( file.getFileDescriptor() == -1 ) {
        int err = errno;
        if ( err != ENOENT ) {
            const char* newMsg = dyld::mkstringf("%s: open() failed with errno=%d", path, err);
            exceptions->push_back(newMsg);
        }
        return NULL;
    }

    try {
        return loadPhase6(file.getFileDescriptor(), stat_buf, path, context);
    }
    catch (const char* msg) {
        const char* newMsg = dyld::mkstringf("%s: %s", path, msg);
        exceptions->push_back(newMsg);
        free((void*)msg);
        return NULL;
    }
}
```

loadPhase6
```
//做了Fat格式的检测，子类型文件提取。
//检测Mach-O类型，只有MH_EXECUTE，MH_DYLIB，MH_BUNDLE三种文件才可被动态加载。
//通过ImageLoaderMachO::instantiateFromFile生成ImageLoader的实例。
//
static ImageLoader* loadPhase6(int fd, const struct stat& stat_buf, const char* path, const LoadContext& context)
{
    //dyld::log("%s(%s)\n", __func__ , path);
    uint64_t fileOffset = 0;
    uint64_t fileLength = stat_buf.st_size;

    // validate it is a file (not directory)
    if ( (stat_buf.st_mode & S_IFMT) != S_IFREG ) 
        throw "not a file";

    uint8_t firstPage[4096];
    bool shortPage = false;
    
    // min mach-o file is 4K
    if ( fileLength < 4096 ) {
        if ( pread(fd, firstPage, fileLength, 0) != (ssize_t)fileLength )
            throwf("pread of short file failed: %d", errno);
        shortPage = true;
    } 
    else {
        if ( pread(fd, firstPage, 4096,0) != 4096 )
            throwf("pread of first 4K failed: %d", errno);
    }
    
    // if fat wrapper, find usable sub-file
    // 如果是一个fat格式的文件，找到对应的子文件
    // 从fat文件中找到对应子文件的代码。
    const fat_header* fileStartAsFat = (fat_header*)firstPage;
    if ( fileStartAsFat->magic == OSSwapBigToHostInt32(FAT_MAGIC) ) {
        if ( fatFindBest(fileStartAsFat, &fileOffset, &fileLength) ) {
            if ( (fileOffset+fileLength) > (uint64_t)(stat_buf.st_size) )
                throwf("truncated fat file.  file length=%llu, but needed slice goes to %llu", stat_buf.st_size, fileOffset+fileLength);
            if (pread(fd, firstPage, 4096, fileOffset) != 4096)
                throwf("pread of fat file failed: %d", errno);
        }
        else {
            throw "no matching architecture in universal wrapper";
        }
    }
    
    // try mach-o loader
    if ( shortPage ) 
        throw "file too short";
    // 检测运行平台是否正确
    if ( isCompatibleMachO(firstPage, path) ) {
    
        // only MH_BUNDLE, MH_DYLIB, and some MH_EXECUTE can be dynamically loaded
        // 只有MH_EXECUTE，MH_DYLIB，MH_BUNDLE三种文件才可被动态加载
        switch ( ((mach_header*)firstPage)->filetype ) {
            case MH_EXECUTE:
            case MH_DYLIB:
            case MH_BUNDLE:
                break;
            default:
                throw "mach-o, but wrong filetype";
        }

#if TARGET_IPHONE_SIMULATOR 
    #if TARGET_OS_WATCH || TARGET_OS_TV
        // disable error during bring up of these simulators
    #else
        // <rdar://problem/14168872> dyld_sim should restrict loading osx binaries
        if ( !isSimulatorBinary(firstPage, path) ) {
            throw "mach-o, but not built for iOS simulator";
        }
    #endif
#endif

        // instantiate an image
        ImageLoader* image = ImageLoaderMachO::instantiateFromFile(path, fd, firstPage, fileOffset, fileLength, stat_buf, gLinkContext);
        
        // validate
        return checkandAddImage(image, context);
    }
    
    // try other file formats here...
    
    
    // throw error about what was found
    switch (*(uint32_t*)firstPage) {
        case MH_MAGIC:
        case MH_CIGAM:
        case MH_MAGIC_64:
        case MH_CIGAM_64:
            throw "mach-o, but wrong architecture";
        default:
        throwf("unknown file type, first eight bytes: 0x%02X 0x%02X 0x%02X 0x%02X 0x%02X 0x%02X 0x%02X 0x%02X", 
            firstPage[0], firstPage[1], firstPage[2], firstPage[3], firstPage[4], firstPage[5], firstPage[6],firstPage[7]);
    }
}
```

## 0x006 fishhook analysis

```
// Copyright (c) 2013, Facebook, Inc.
// All rights reserved.
// Redistribution and use in source and binary forms, with or without
// modification, are permitted provided that the following conditions are met:
//   * Redistributions of source code must retain the above copyright notice,
//     this list of conditions and the following disclaimer.
//   * Redistributions in binary form must reproduce the above copyright notice,
//     this list of conditions and the following disclaimer in the documentation
//     and/or other materials provided with the distribution.
//   * Neither the name Facebook nor the names of its contributors may be used to
//     endorse or promote products derived from this software without specific
//     prior written permission.
// THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
// AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
// IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
// DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
// FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
// DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
// SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
// CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
// OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
// OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

#import "fishhook.h"

#import <dlfcn.h>
#import <stdlib.h>
#import <string.h>
#import <sys/types.h>
#import <mach-o/dyld.h>
#import <mach-o/loader.h>
#import <mach-o/nlist.h>

#ifdef __LP64__
typedef struct mach_header_64 mach_header_t;
typedef struct segment_command_64 segment_command_t;
typedef struct section_64 section_t;
typedef struct nlist_64 nlist_t;
#define LC_SEGMENT_ARCH_DEPENDENT LC_SEGMENT_64
#else
typedef struct mach_header mach_header_t;
typedef struct segment_command segment_command_t;
typedef struct section section_t;
typedef struct nlist nlist_t;
#define LC_SEGMENT_ARCH_DEPENDENT LC_SEGMENT
#endif

#ifndef SEG_DATA_CONST
#define SEG_DATA_CONST  "__DATA_CONST"
#endif

struct rebindings_entry {
  struct rebinding *rebindings;
  size_t rebindings_nel;
  struct rebindings_entry *next;
};

static struct rebindings_entry *_rebindings_head;

static int prepend_rebindings(struct rebindings_entry **rebindings_head,
                              struct rebinding rebindings[],
                              size_t nel) {
  struct rebindings_entry *new_entry = malloc(sizeof(struct rebindings_entry));
  if (!new_entry) {
    return -1;
  }
  new_entry->rebindings = malloc(sizeof(struct rebinding) * nel);
  if (!new_entry->rebindings) {
    free(new_entry);
    return -1;
  }
  memcpy(new_entry->rebindings, rebindings, sizeof(struct rebinding) * nel);
  new_entry->rebindings_nel = nel;
  new_entry->next = *rebindings_head;
  *rebindings_head = new_entry;
  return 0;
}

static void perform_rebinding_with_section(struct rebindings_entry *rebindings,
                                           section_t *section,
                                           intptr_t slide,
                                           nlist_t *symtab,
                                           char *strtab,
                                           uint32_t *indirect_symtab) {
  //__la_symbol_ptr的reserved1字段标识了section描述的符号在符号表中开始的index
  //动态符号表中第一个需要解析的符号 开始地址
  uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1;
  
  void **indirect_symbol_bindings = (void **)((uintptr_t)slide + section->addr);
  
  for (uint i = 0; i < section->size / sizeof(void *); i++) {
    uint32_t symtab_index = indirect_symbol_indices[i];
    if (symtab_index == INDIRECT_SYMBOL_ABS || symtab_index == INDIRECT_SYMBOL_LOCAL ||
        symtab_index == (INDIRECT_SYMBOL_LOCAL   | INDIRECT_SYMBOL_ABS)) {
      continue;
    }
    //获取每一个需要动态解析的符号在符号表中的地址
    uint32_t strtab_offset = symtab[symtab_index].n_un.n_strx;
    
    //通过符号表中每一个导入符号的字符串表偏移量获取符号对应的字符串（符号的名字）
    char *symbol_name = strtab + strtab_offset;
    struct rebindings_entry *cur = rebindings;
    while (cur) {
      for (uint j = 0; j < cur->rebindings_nel; j++) {
        if (strlen(symbol_name) > 1 &&
            strcmp(&symbol_name[1], cur->rebindings[j].name) == 0) {
          //找到相同的函数替换指针
          if (cur->rebindings[j].replaced != NULL &&
              indirect_symbol_bindings[i] != cur->rebindings[j].replacement) {
            *(cur->rebindings[j].replaced) = indirect_symbol_bindings[i];
          }
          indirect_symbol_bindings[i] = cur->rebindings[j].replacement;
          goto symbol_loop;
        }
      }
      cur = cur->next;
    }
  symbol_loop:;
  }
}


//typedef struct {
//    const char *dli_fname;  /* Pathname of shared object that
//                               contains address */
//    void       *dli_fbase;  /* Address at which shared object
//                               is loaded */
//    const char *dli_sname;  /* Name of nearest symbol with address
//                               lower than addr */
//    void       *dli_saddr;  /* Exact address of symbol named
//                               in dli_sname */
//} Dl_info;

static void rebind_symbols_for_image(struct rebindings_entry *rebindings,
                                     const struct mach_header *header,
                                     intptr_t slide) {
  //获取Dl_info
  Dl_info info;
  if (dladdr(header, &info) == 0) {
    return;
  }
    
  
  segment_command_t *cur_seg_cmd;
  segment_command_t *linkedit_segment = NULL;
  struct symtab_command* symtab_cmd = NULL;
  struct dysymtab_command* dysymtab_cmd = NULL;

  uintptr_t cur = (uintptr_t)header + sizeof(mach_header_t);
  for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
    cur_seg_cmd = (segment_command_t *)cur;
    if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
      if (strcmp(cur_seg_cmd->segname, SEG_LINKEDIT) == 0) {
        //在lc_segment中遍历寻找__LINKEDIT的section
        linkedit_segment = cur_seg_cmd;
      }
    } else if (cur_seg_cmd->cmd == LC_SYMTAB) {
      //遍历寻找lc_symtab
      symtab_cmd = (struct symtab_command*)cur_seg_cmd;
    } else if (cur_seg_cmd->cmd == LC_DYSYMTAB) {
      //遍历寻找lc_dysymtab
      dysymtab_cmd = (struct dysymtab_command*)cur_seg_cmd;
    }
  }

  //检测必要的数据结构是否都存在
  /*
  LC_SYMTAB这个LoadCommand主要提供了两个信息
    Symbol Table的偏移量与Symbol Table中元素的个数
    String Table的偏移量与String Table的长度
  LC_DYSYMTAB
    提供了动态符号表的位移和元素个数，还有一些其他的表格索引
  LC_SEGMENT.__LINKEDIT
    含有为动态链接库使用的原始数据
  */
  if (!symtab_cmd || !dysymtab_cmd || !linkedit_segment ||
      !dysymtab_cmd->nindirectsyms) {
    return;
  }

  // Find base symbol/string table addresses
  // 获取链接时程序的基址
  // 基址 = __LINKEDIT.VM_Address - __LINK.File_Offset + silde的改变值
  // machoview随便看一个程序的__LINKEDIT
  // offset     | data                  | Description       | value
  // ...
  // 0x0000390    0x0000000100002000      VM Address          4294975488
  // 0x0000398    0x0000000000003000      VM Size             12288
  // 0x00003A0    0x0000000000002000      File Offset         8192
  // 0x00003A8    0x0000000000002690      File Size           9872
  
  // base = 100002000-2000 + slide = 0x0000000100000000 + slide
  // 这应该是一个公式
  uintptr_t linkedit_base = (uintptr_t)slide + linkedit_segment->vmaddr - linkedit_segment->fileoff;
  
  // 符号表的地址 = 基址 + 符号表偏移量
  nlist_t *symtab = (nlist_t *)(linkedit_base + symtab_cmd->symoff);
  // 字符串表的地址 = 基址 + 字符串表偏移量
  char *strtab = (char *)(linkedit_base + symtab_cmd->stroff);

  // Get indirect symbol table (array of uint32_t indices into symbol table)
  // 动态符号表地址 = 基址 + 动态符号表偏移量
  uint32_t *indirect_symtab = (uint32_t *)(linkedit_base + dysymtab_cmd->indirectsymoff);

  //再一次遍历loadcommands
  cur = (uintptr_t)header + sizeof(mach_header_t);
  for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
    cur_seg_cmd = (segment_command_t *)cur;
    if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
      if (strcmp(cur_seg_cmd->segname, SEG_DATA) != 0 &&
          strcmp(cur_seg_cmd->segname, SEG_DATA_CONST) != 0) {
        continue;
      }
      //找到__DATA和__DATA_CONST的section
      //__DATA section里面保存的是符号跳转的函数指针表
      // 对__nl_symbol_ptr以及__la_symbol_ptr进行rebind
      for (uint j = 0; j < cur_seg_cmd->nsects; j++) {
        section_t *sect =
          (section_t *)(cur + sizeof(segment_command_t)) + j;
        if ((sect->flags & SECTION_TYPE) == S_LAZY_SYMBOL_POINTERS) {
          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
        }
        if ((sect->flags & SECTION_TYPE) == S_NON_LAZY_SYMBOL_POINTERS) {
          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
        }
      }
    }
  }
}

static void _rebind_symbols_for_image(const struct mach_header *header,
                                      intptr_t slide) {
    rebind_symbols_for_image(_rebindings_head, header, slide);
}

int rebind_symbols_image(void *header,
                         intptr_t slide,
                         struct rebinding rebindings[],
                         size_t rebindings_nel) {
    struct rebindings_entry *rebindings_head = NULL;
    int retval = prepend_rebindings(&rebindings_head, rebindings, rebindings_nel);
    rebind_symbols_for_image(rebindings_head, header, slide);
    free(rebindings_head);
    return retval;
}

int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
  int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
  if (retval < 0) {
    return retval;
  }
  // If this was the first call, register callback for image additions (which is also invoked for
  // existing images, otherwise, just run on existing images
  if (!_rebindings_head->next) {
    _dyld_register_func_for_add_image(_rebind_symbols_for_image);
  } else {
    uint32_t c = _dyld_image_count();
    for (uint32_t i = 0; i < c; i++) {
      _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
    }
  }
  return retval;
}
```
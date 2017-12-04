import os
import sys
import glob
import re
import time
import datetime
import atexit
import platform

import SCons.Action
import SCons.Script.Main

def CreateNewEnv():

    SetupOptions()

    env = Environment(
        DEBUG_BUILD     = GetOption('option_debug'),
        ZPREFIX         = GetOption('option_zprefix'),
        SOLO            = GetOption('option_solo'),   
        COVER           = GetOption('option_cover'),   
        GCC_CLASSIC     = GetOption('option_gcc_classic'),
        VERBOSE_COMPILE = GetOption('option_verbose'),
    )

    num_cpus = get_cpu_nums()
    ColorPrinter().InfoPrint("Building with " + str(num_cpus) + " parallel jobs")
    env.SetOption( "num_jobs", num_cpus )

    env.baseProjectDir = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    env.VariantDir(Dir('build'),      Dir('.'), duplicate=0)
    env.VariantDir(Dir('build/test'), Dir('./test'), duplicate=0)

    #if(MSVC)
    #    set(CMAKE_DEBUG_POSTFIX "d")
    #    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    #    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
    #    include_directories(${CMAKE_CURRENT_SOURCE_DIR})
    #endif()

    source_files = [
        'build/adler32.c',
        'build/compress.c',
        'build/crc32.c',
        'build/deflate.c',
        'build/gzclose.c',
        'build/gzlib.c',
        'build/gzread.c',
        'build/gzwrite.c',
        'build/inflate.c',
        'build/infback.c',
        'build/inftrees.c',
        'build/inffast.c',
        'build/trees.c',
        'build/uncompr.c',
        'build/zutil.c',
    ]

    headerFiles = []

    libType = "ReleaseBuild"
    if(env['DEBUG_BUILD']):
        libType = 'DebugBuild'
        env.Append(CPPDEFINES=[
        ])

    env.Append(CPPDEFINES=[
    ])

    env.Append(CPPPATH=[

    ])    
        
    env.Append(LIBPATH=[

    ])

    libs = [
        
    ]
    if(env['DEBUG_BUILD']):
        libs = [ 
        ]
    
    env.Append(LIBS=libs)
    env.Append(LIBS=[

    ])
    
    resourceFiles = [
        
    ]

    progCounter = ProgressCounter()
    reset_callback = SCons.Action.ActionFactory( progCounter.ResetProgress,
                                            lambda source_files, target_bins: 'Reseting Progress Counter for ' + str(target_bins))
    progCounter.ResetProgress(source_files, ['libzlib.so'])

    env = ConfigureEnv(env)

    shared_env, shared_lib = SetupBuild(
        env, 
        'shared', 
        'zlib', 
        source_files
    )

    static_env, static_lib = SetupBuild(
        env, 
        'static', 
        'zlib',
        source_files, 
        reset_callback, 
        shared_lib
    )

    example_env, example_bin = SetupBuild(
        env, 
        'exec', 
        'example', 
        ['build/test/example.c'], 
        reset_callback, 
        static_lib
    )
    example_env.Append(LIBPATH=[Dir('.')])
    example_env.Append(LIBS=['libzlib.a'])

    minigzip_env, minigzip_bin  = SetupBuild(
        env, 
        'exec', 
        'minigzip', 
        ['build/test/minigzip.c'], 
        reset_callback, 
        example_bin
    )
    minigzip_env.Append(LIBPATH=[Dir('.')])
    minigzip_env.Append(LIBS=['libzlib.a'])

    #env = SetupInstalls(env)
    #env = ConfigPlatformIDE(env, source_files, headerFiles, resourceFiles, prog)

    Progress(progCounter, interval=1)
    
def ConfigureEnv(env):

    def CheckLargeFile64(context):
        context.Message('Checking for off64_t... ')

        prevDefines = ""
        if('CPPDEFINES' in context.env):
            prevDefines = context.env['CPPDEFINES']

        context.env.Append(CPPDEFINES=['_LARGEFILE64_SOURCE=1'])
        result = context.TryCompile("""
        #include <sys/types.h>
        off64_t dummy = 0;
        """, 
        '.c')
        if not result:
            context.env.Replace(CPPDEFINES = prevDefines)
        context.Result(result)
        return result

    def CheckFseeko(context):
        context.Message('Checking for fseeko... ')
        result = context.TryCompile("""
        #include <stdio.h>
        int main(void) {
            fseeko(NULL, 0, 0);
            return 0;
        }
        """, 
        '.c')
        if not result:
            context.env.Append(CPPDEFINES=['NO_FSEEKO'])
        context.Result(result)
        return result

    def CheckSizeT(context):
        context.Message('Checking for size_t... ')
        result = context.TryCompile("""
        #include <stdio.h>
        #include <stdlib.h>
        size_t dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result
    
    def CheckSizeTLongLong(context):
        context.Message('Checking for long long... ')
        result = context.TryCompile("""
        long long dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result

    def CheckSizeTPointerSize(context, longlong_result):
        result = []
        context.Message('Checking for a pointer-size integer type... ')
        if longlong_result:
            result = context.TryRun("""
            #include <stdio.h>
            int main(void) {
                if (sizeof(void *) <= sizeof(int)) puts("int");
                else if (sizeof(void *) <= sizeof(long)) puts("long");
                else puts("z_longlong");
                return 0;
            }
            """,
            '.c')
        else:
            result = context.TryRun("""
            #include <stdio.h>
            int main(void) {
                if (sizeof(void *) <= sizeof(int)) puts("int");
                else puts("long");
                return 0;
            }
            """, 
            '.c')
        
        if result[1] == "":
            context.Result("Failed.");
            return False
        else:
            context.env.Append(CPPDEFINES=['NO_SIZE_T='+size_t_type])
            context.Result(size_t_type + ".")
            return True

    def CheckSharedLibrary(context):
        context.Message('Checking for shared library support... ')
        result = context.TryBuild(context.env.SharedLibrary, """
        extern int getchar();
        int hello() {return getchar();}
        """,
        '.c')

        context.Result(result)
        return result

    def CheckUnistdH(context):
        context.Message('Checking for unistd.h... ')
        result = context.TryCompile("""
        #include <unistd.h>
        int main() { return 0; }
        """, 
        '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_UNISTD_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

   

    def CheckStrerror(context):
        context.Message('Checking for strerror... ')
        result = context.TryCompile("""
        #include <string.h>
        #include <errno.h>
        int main() { return strlen(strerror(errno)); }
        """, 
        '.c')
        if not result:
            context.env.Append(CPPDEFINES=['NO_STRERROR'])
        context.Result(result)
        return result
        
    def CheckStdargH(context):
        context.Message('Checking for stdarg.h... ')
        result = context.TryCompile("""
        #include <stdarg.h>
        int main() { return 0; }
        """, 
        '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_STDARG_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddZPrefix(context):
        context.Message('Using z_ prefix on all symbols... ')
        result = context.env['ZPREFIX'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sZ_PREFIX(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddSolo(context):
        context.Message('Using Z_SOLO to build... ')
        result = context.env['SOLO'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"\#define\sZCONF_H", r"#define ZCONF_H\n#define Z_SOLO", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddCover(context):
        context.Message('Using code coverage flags... ')
        result = context.env['COVER'] 
        if result:
            context.env.Append(CCFLAGS=[
                '-fprofile-arcs', 
                '-ftest-coverage',
            ])
        if not context.env['GCC_CLASSIC'] == "":
            context.env.Replace(CC = context.env['GCC_CLASSIC'])
        context.Result(result)
        return result
    
    def CheckVsnprintf(context):
        context.Message("Checking whether to use vs[n]printf() or s[n]printf()... ")
        result = context.TryCompile("""
        #include <stdio.h>
        #include <stdarg.h>
        #include "zconf.h"
        int main()
        {
        #ifndef STDC
            choke me
        #endif
            return 0;
        }
        """, 
        '.c')
        if result:
            context.Result("using vs[n]printf().")
        else:
            context.Result("using s[n]printf().")
        return result

    def CheckVsnStdio(context):
        context.Message("Checking for vsnprintf() in stdio.h... ")
        result = context.TryCompile("""
        #include <stdio.h>
        #include <stdarg.h>
        int mytest(const char *fmt, ...)
        {
            char buf[20];
            va_list ap;
            va_start(ap, fmt);
            vsnprintf(buf, sizeof(buf), fmt, ap);
            va_end(ap);
            return 0;
        }
        int main()
        {
            return (mytest("Hello%d\\n", 1));
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["NO_vsnprintf"])
            print("  WARNING: vsnprintf() not found, falling back to vsprintf(). zlib")
            print("  can build but will be open to possible buffer-overflow security")
            print("  vulnerabilities.")
        return result

    def CheckVsnprintfReturn(context):
        context.Message("Checking for return value of vsnprintf()... ")
        result = context.TryCompile("""
        #include <stdio.h>
        #include <stdarg.h>
        int mytest(const char *fmt, ...)
        {
            int n;
            char buf[20];
            va_list ap;
            va_start(ap, fmt);
            n = vsnprintf(buf, sizeof(buf), fmt, ap);
            va_end(ap);
            return n;
        }
        int main()
        {
            return (mytest("Hello%d\\n", 1));
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_vsnprintf_void"])
            print("  WARNING: apparently vsnprintf() does not return a value. zlib")
            print("  can build but will be open to possible string-format security")
            print("  vulnerabilities.")
        return result

    def CheckVsprintfReturn(context):
        context.Message("Checking for return value of vsnprintf()... ")
        result = context.TryCompile("""
        #include <stdio.h>
        #include <stdarg.h>
        int mytest(const char *fmt, ...)
        {
            int n;
            char buf[20];
            va_list ap;
            va_start(ap, fmt);
            n = vsprintf(buf, fmt, ap);
            va_end(ap);
            return n;
        }
        int main()
        {
            return (mytest("Hello%d\\n", 1));
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_vsprintf_void"])
            print("  WARNING: apparently vsprintf() does not return a value. zlib")
            print("  can build but will be open to possible string-format security")
            print("  vulnerabilities.")
        return result

    def CheckSnStdio(context):
        context.Message("Checking for snprintf() in stdio.h... ")
        result = context.TryCompile("""
        #include <stdio.h>
        int mytest()
        {
            char buf[20];
            snprintf(buf, sizeof(buf), "%s", "foo");
            return 0;
        }
        int main()
        {
            return (mytest());
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["NO_snprintf"])
            print("  WARNING: snprintf() not found, falling back to sprintf(). zlib")
            print("  can build but will be open to possible buffer-overflow security")
            print("  vulnerabilities.")
        return result

    def CheckSnprintfReturn(context):
        context.Message("Checking for return value of snprintf()... ")
        result = context.TryCompile("""
        #include <stdio.h>
        int mytest()
        {
            char buf[20];
            return snprintf(buf, sizeof(buf), "%s", "foo");
        }
        int main()
        {
            return (mytest());
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_snprintf_void"])
            print("  WARNING: apparently snprintf() does not return a value. zlib")
            print("  can build but will be open to possible string-format security")
            print("  vulnerabilities.")
        return result

    def CheckSprintfReturn(context):
        context.Message("Checking for return value of sprintf()... ")
        result = context.TryCompile("""
        #include <stdio.h>
        int mytest()
        {
            char buf[20];
            return sprintf(buf, "%s", "foo");
        }
        int main()
        {
            return (mytest());
        }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_sprintf_void"])
            print("  WARNING: apparently sprintf() does not return a value. zlib")
            print("  can build but will be open to possible string-format security")
            print("  vulnerabilities.")
        return result

    def CheckHidden(context):
        context.Message("Checking for attribute(visibility) support... ")
        result = context.TryCompile("""
        #define ZLIB_INTERNAL __attribute__((visibility ("hidden")))
        int ZLIB_INTERNAL foo;
        int main()
        {
            return 0;
        }
        """, 
        '.c')
        context.Result(result)
        if result:
            context.env.Append(CPPDEFINES=["HAVE_HIDDEN"])
        return result

    if not env.GetOption('clean'):
    
        if(GetOption('option_reconfigure')):
            os.remove('build.conf')

        vars = Variables('build.conf')
        
        vars.Add('ZPREFIX', '' )
        vars.Add('SOLO', '')
        vars.Add('COVER', '')
        vars.Update(env)

        if('--zprefix' in sys.argv): env['ZPREFIX'] = GetOption('option_zprefix')
        if('--solo' in sys.argv):    env['SOLO'] = GetOption('option_solo')
        if('--cover' in sys.argv):   env['COVER'] = GetOption('option_cover')

        vars.Save('build.conf', env)

        configureString = "Configuring with "
        if env['ZPREFIX']: configureString += "--zprefix "
        if env['SOLO']:    configureString += "--solo "
        if env['COVER']:   configureString += "--cover "

        ColorPrinter().InfoPrint(configureString)

        conf = Configure(env,
            custom_tests = {
                'CheckLargeFile64'     : CheckLargeFile64, 
                'CheckFseeko'          : CheckFseeko,
                'CheckSizeT'           : CheckSizeT,
                'CheckSizeTLongLong'   : CheckSizeTLongLong,
                'CheckSizeTPointerSize': CheckSizeTPointerSize,
                'CheckSharedLibrary'   : CheckSharedLibrary,
                'CheckUnistdH'         : CheckUnistdH,
                'CheckStrerror'        : CheckStrerror,
                'CheckStdargH'         : CheckStdargH,
                'CheckVsnprintf'       : CheckVsnprintf,
                'CheckVsnStdio'        : CheckVsnStdio,
                'CheckVsnprintfReturn' : CheckVsnprintfReturn,
                'CheckVsprintfReturn'  : CheckVsprintfReturn,
                'CheckSnStdio'         : CheckSnStdio,
                'CheckSnprintfReturn'  : CheckSnprintfReturn,
                'CheckSprintfReturn'   : CheckSprintfReturn,
                'CheckHidden'          : CheckHidden,
                'AddZPrefix'           : AddZPrefix,
                'AddSolo'              : AddSolo,
                'AddCover'             : AddCover,
                })

        with open('zconf.h.in', 'r') as content_file:
            #print("Reading zconf.h.in... ")
            conf.env["ZCONFH"] = str(content_file.read())  

        conf.CheckCC()
        conf.CheckSharedLibrary()
        #conf.CheckExternalNames()
        if not conf.CheckSizeT():
            conf.CheckSizeTPointerSize(conf.CheckSizeTLongLong())
        if not conf.CheckLargeFile64():
            conf.CheckFseeko()
        
        conf.CheckStrerror()
        conf.CheckUnistdH()
        conf.CheckStdargH()

        if conf.env['ZPREFIX']: conf.AddZPrefix()
        if conf.env['SOLO']:    conf.AddSolo()
        if conf.env['COVER']:   conf.AddCover()

        with open('zconf.h', 'w') as content_file:
            #print("Writing zconf.h...")
            content_file.write(conf.env["ZCONFH"])

        if conf.CheckVsnprintf():
            if conf.CheckVsnStdio():
                conf.CheckVsnprintfReturn()
            else:
                conf.CheckVsprintfReturn()
        else:
            if conf.CheckSnStdio():
                conf.CheckSnprintfReturn()
            else:
                conf.CheckSprintfReturn()

        conf.CheckHidden()

        env = conf.Finish()
    
    if("linux" in platform.system().lower() ):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
            '-O3',
            #'-fPIC',
            #"-rdynamic",
        ])

        env.Append(CXXFLAGS= [
            #"-std=c++11",
        ])

        env.Append(LDFLAGS= [
            debugFlag,
            #"-rdynamic",
        ])

        env.Append(LIBPATH=[
            #"/usr/lib/gnome-settings-daemon-3.0/",
        ])

        env.Append(LIBS=[
            #"GL",
        ])

    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower() ):

        degugDefine = 'NDEBUG'
        debugFlag = "/O2"
        degug = '/DEBUG:NONE'
        debugRuntime = "/MD"
        libType = "Release"
        if(env['DEBUG_BUILD']):
            degugDefine = 'DEBUG'
            debugFlag = "/Od"
            degug = '/DEBUG:FULL'
            debugRuntime = "/MDd"
            libType = 'Debug'

        env.Append(CPPDEFINES=[
            'NOMINMAX',
            "WIN32",
            degugDefine,
        ])

        env.Append(CCFLAGS= [
            "/analyze-",
            "/GS",
            "/Zc:wchar_t",
            "/W1",
            "/Z7",
            "/Gm-",
            debugFlag,
            "/WX-",
            '/FC',
            "/Zc:forScope",             # Force Conformance in for Loop Scope
            "/GR",                      # Enable Run-Time Type Information
            "/Oy-",                     # Disable Frame-Pointer Omission
            debugRuntime,               # Use Multithread DLL version of the runt-time library
            "/EHsc",
            "/nologo",
        ])

        env.Append(LINKFLAGS=[
            degug,
            "/SUBSYSTEM:CONSOLE",
            "/ENTRY:mainCRTStartup",
            #"/SAFESEH:NO",
        ])

        env.Append(LIBS=[
            'OpenGL32',
            'user32',
        ])
       
    return env

def ConfigPlatformIDE(env, sourceFiles, headerFiles, resources, program):
    if platform == "linux" or platform == "linux2":
        print("Eclipse C++ project not implemented yet")
    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower() ):
        variantSourceFiles = []
        for file in sourceFiles:
            variantSourceFiles.append(re.sub("^build", "../Source", file))
        variantHeaderFiles = []
        for file in headerFiles:
                variantHeaderFiles.append(re.sub("^Source", "../Source", file))
        buildSettings = {
            'LocalDebuggerCommand':os.path.abspath(env.baseProjectDir).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory':os.path.abspath(env.baseProjectDir).replace('\\', '/') + "/build/",
            
        }
        buildVariant = 'Release|x64'
        cmdargs = ''
        if(env['DEBUG_BUILD']):
            buildVariant = 'Debug|x64'
            cmdargs = '--debug_build'
        env.MSVSProject(target = 'VisualStudio/MyLifeApp' + env['MSVSPROJECTSUFFIX'],
                    srcs = variantSourceFiles,
                    localincs = variantHeaderFiles,
                    resources = resources,
                    buildtarget = program,
                    DebugSettings = buildSettings,
                    variant = buildVariant,
                    cmdargs = cmdargs)
    return env

def get_cpu_nums():
    # Linux, Unix and MacOS:
    if hasattr( os, "sysconf" ):
        if "SC_NPROCESSORS_ONLN" in os.sysconf_names:
            # Linux & Unix:
            ncpus = os.sysconf( "SC_NPROCESSORS_ONLN" )
        if isinstance(ncpus, int) and ncpus > 0:
            return ncpus
        else: # OSX:
            return int( os.popen2( "sysctl -n hw.ncpu")[1].read() )
    # Windows:
    if "NUMBER_OF_PROCESSORS" in os.environ:
        ncpus = int( os.environ[ "NUMBER_OF_PROCESSORS" ] );
    if ncpus > 0:
        return ncpus
    return 1 # Default

def SetupBuild(env, type, progName, sourceFiles, resetCallback = None, previousBuild = None):

    buildEnv = env.Clone()

    windowsRedirect = ""
    linuxRedirect = "2>&1"
    if("Windows" in platform.system()):
        windowsRedirect = "2>&1"
        linuxRedirect = ""

    soureFileObjs = []
    for file in sourceFiles:
        if(type == "shared"):
            buildObj = buildEnv.SharedObject(file, 
                                CCCOM=env['CCCOM'] + " " + windowsRedirect + " > \"" + buildEnv.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect,
                                CXXCOM=env['CXXCOM'] + " " + windowsRedirect + " > \"" + buildEnv.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect)
            soureFileObjs.append(buildObj)
        elif(type == "static" or type == 'exec'):
            buildObj = buildEnv.Object(file, 
                                CCCOM=env['CCCOM'] + " " + windowsRedirect + " > \"" + buildEnv.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect,
                                CXXCOM=env['CXXCOM'] + " " + windowsRedirect + " > \"" + buildEnv.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect)
            soureFileObjs.append(buildObj)

    if("Windows" in platform.system()):
        env['LINKCOM'].list[0].cmd_list = env['LINKCOM'].list[0].cmd_list.replace('",'," 2>&1 > \\\"" + buildEnv.baseProjectDir + "/build/build_logs/MyLifeApp_link.txt\\\"\",") 
    else:
        env['LINKCOM'] = env['LINKCOM'].replace('",'," > \\\"" + buildEnv.baseProjectDir + "/build/build_logs/MyLifeApp_link.txt\\\"\" 2>&1 ,") 
   
    if(type == "shared"):
        prog = buildEnv.SharedLibrary(progName, soureFileObjs)
    elif(type == "static"):
        prog = buildEnv.StaticLibrary(progName, soureFileObjs)
    elif(type == 'exec'):
        prog = buildEnv.Program(progName, soureFileObjs)

    ###################################################
    # setup build output
    if not os.path.exists(buildEnv.baseProjectDir + "/build/build_logs"):
        os.makedirs(buildEnv.baseProjectDir + "/build/build_logs")

    if ARGUMENTS.get('fail', 0):
        Command('target', 'source', ['/bin/false'])

    atexit.register(display_build_status)

    def print_cmd_line(s, targets, sources, env):
        with open(buildEnv.baseProjectDir + "/build/build_logs/build_" + env['BUILD_LOG_TIME'] + ".log", "a") as f:
            f.write(s + "\n")

    env['BUILD_LOG_TIME'] = datetime.datetime.fromtimestamp(time.time()).strftime('%Y_%m_%d__%H_%M_%S')
    if(not env['VERBOSE_COMPILE']):
        env['PRINT_CMD_LINE_FUNC'] = print_cmd_line
    
    builtBins = []
    if("Windows" in platform.system()):
        builtBins.append("build/" + progName + ".exe")
    else:
        if(type == 'shared'):
            builtBins.append('lib' + progName + ".so")
        elif(type == 'static'):
            builtBins.append('lib' + progName + ".a")
        elif(type == 'exec'):
            builtBins.append( progName )

    if(not resetCallback == None and not previousBuild == None):
        reset = buildEnv.Command( None, sourceFiles, resetCallback(sourceFiles, builtBins))
        Depends(reset, previousBuild)
        for node in soureFileObjs:
            Depends(node, reset)

    return [buildEnv, prog]

def SetupInstalls(env):

    return env
    
def SetupOptions():

    AddOption(
        '--debug-build',
        dest='option_debug',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Build debug libraries'
    )

    AddOption(
        '--ASM686',
        dest='option_asm686',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable building i686 assembly implementation'
    )

    AddOption(
        '--AMD64',
        dest='option_amd64',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable building amd64 assembly implementation'
    )

    AddOption(
        '--zprefix',
        dest='option_zprefix',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Precedes all external symbols with z_ to reduce probability of a symbol name collision'
    )

    AddOption(
        '--solo',
        dest='option_solo',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Used to compile and use zlib without the use of any external libraries.'
    )

    AddOption(
        '--cover',
        dest='option_cover',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='code coverage testing was requested'
    )

    AddOption(
        '--gcc-classic',
        dest='option_gcc_classic',
        action='store',
        type='string',
        metavar='STRING',
        default='',
        help='An alternate GCC to use in some cases such as Code Coverage testing.'
    )

    AddOption(
        '--verbose',
        dest='option_verbose',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='View compiler output.'
    )

    AddOption(
        '--reconfigure',
        dest='option_reconfigure',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Clear out configuration and reconfigure with new options.'
    )

    if(not GetOption('option_verbose')):
        scons_ver = SCons.__version__
        if int(scons_ver[0]) >= 3:
            SetOption('silent', 1)
        SCons.Script.Main.progress_display.set_mode(0)

class ProgressCounter(object):

    def __init__(self):
        self.count = 0.0
        self.printer = ColorPrinter()
        self.max_count = 1.0
        self.progress_sources = []
        self.target_binaries = []

    def __call__(self, node, *args, **kw):
        #print(str(node))

        slashed_node = str(node).replace("\\", "/")
        if(slashed_node in self.target_binaries):
            filename = os.path.basename(slashed_node)
            if(node.get_state() == 2): self.printer.LinkPrint("Linking " + filename)
            else:                      self.printer.LinkPrint("Skipping, already built " + filename)

        # TODO: make hanlding this file extensions better
        if(   slashed_node.endswith(".obj") 
           or slashed_node.endswith(".o"  ) 
           or slashed_node.endswith(".os" ) ):

            slashed_node_file = os.path.splitext(slashed_node)[0] + ".c"
            if(slashed_node_file in self.progress_sources ):

                if(self.count == 0):
                    start_build_string = "Building "
                    for bin in self.target_binaries:
                        start_build_string += bin + ", "
                    start_build_string = start_build_string[:-2]
                    self.printer.InfoPrint(start_build_string)

                self.count += 1
                percent = self.count / self.max_count * 100.00
                filename = os.path.basename(slashed_node)
                
                if(node.get_state() == 2): 
                    self.printer.CompilePrint( percent, "Compiling " + filename )
                else:                      
                    self.printer.CompilePrint( percent, "Skipping, already built " + filename )

    def ResetProgress(self, source_files, target_binaries):
        self.count = 0.0
        self.max_count = float(len(source_files))
        self.progress_sources = source_files
        self.target_binaries = target_binaries

def bf_to_str(bf):
    """Convert an element of GetBuildFailures() to a string
    in a useful way."""
    import SCons.Errors
    if bf is None: # unknown targets product None in list
        return '(unknown tgt)'
    elif isinstance(bf, SCons.Errors.StopError):
        return str(bf)
    elif bf.node:
        return str(bf.node) + ': ' + bf.errstr
    elif bf.filename:
        return bf.filename + ': ' + bf.errstr
    return 'unknown failure: ' + bf.errstr


def build_status():
    """Convert the build status to a 2-tuple, (status, msg)."""
    from SCons.Script import GetBuildFailures
    bf = GetBuildFailures()
    if bf:
        # bf is normally a list of build failures; if an element is None,
        # it's because of a target that scons doesn't know anything about.
        status = 'failed'
        failures_message = "\n".join(["%s" % bf_to_str(x)
                           for x in bf if x is not None])
    else:
        # if bf is None, the build completed successfully.
        status = 'ok'
        failures_message = ''
    return (status, failures_message)

def highlight_word(line, word, color):
    ENDC = '\033[0m'
    return line.replace(word, color + word + ENDC )
    
    
def display_build_status():
    """Display the build status.  Called by atexit.
    Here you could do all kinds of complicated things."""
    status, failures_message = build_status()
    
    HEADER = '\033[95m'
    OKBLUE = '\033[94m'
    OKGREEN = '\033[92m'
    WARNING = '\033[93m'
    FAIL = '\033[91m'
    ENDC = '\033[0m'
    
    compileOutput = []
    for filename in glob.glob('build/build_logs/*_compile.txt'):
        f = open(filename, "r")
        tempList = f.read().splitlines()
        if(len(tempList) > 0):
            compileOutput += [OKBLUE + os.path.basename(filename).replace("_compile.txt", "") + ":" + ENDC]
            compileOutput += tempList
    for line in compileOutput:
        line = highlight_word(line, "error", FAIL)
        line = highlight_word(line, "warning", WARNING)
        print(line)
    linkOutput = []
    buildLink = ""
    for filename in glob.glob('build/build_logs/*_link.txt'):
        f = open(filename, "r")
        tempList = f.read().splitlines()
        if(len(tempList) > 0):
            buildLink = os.path.basename(filename).replace("_link.txt", "")
            linkOutput += [OKBLUE + buildLink + ":" + ENDC]
            linkOutput += tempList
    for line in linkOutput:
        line = highlight_word(line, "error", FAIL)
        line = highlight_word(line, "warning", WARNING)
        print(line)
        
    #if status == 'failed':
    #    print(FAIL + "Build of " + buildLink + " failed." + ENDC)
    #elif status == 'ok':
    #    print (OKGREEN + "Build of " + buildLink + " succeeded." + ENDC)

class ColorPrinter():

    def __init__(self):
        self.HEADER = '\033[95m'
        self.OKBLUE = '\033[94m'
        self.OKGREEN = '\033[92m'
        self.WARNING = '\033[93m'
        self.FAIL = '\033[91m'
        self.ENDC = '\033[0m'

    def InfoPrint(self, message):
        print(self.HEADER + "[   INFO] " + self.ENDC + message)

    def CompilePrint(self, percent, message):
        percent_string = "{0:.2f}".format(percent)
        if(percent < 100): percent_string = " " + percent_string
        if(percent < 10):  percent_string = " " + percent_string
        print(self.OKGREEN + "[" + percent_string + "%] " + self.ENDC + message )

    def LinkPrint(self, message):
        print(self.OKGREEN + "[   LINK] " + self.ENDC + message)


CreateNewEnv()

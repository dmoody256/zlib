import os
import json
import sys
import logging
import glob
import re
import time
import datetime
import atexit
import platform
import xml.etree.cElementTree as ET
from cStringIO import StringIO
import SCons.Action

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
    context.Result(result)
    context.Message('Checking for fseeko... ')
    if not result:
        context.env.Replace(CPPDEFINES = prevDefines)
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
    if not result:
        context.Message('Checking for long long... ')
        result = context.TryCompile("""
        long long dummy = 0;
        """, 
        '.c')
        context.Result(result)
        context.Message('Checking for a pointer-size integer type... ')
        size_t_type = ""
        if result:
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
            size_t_type = result[1]
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
            size_t_type = result[1]
        if size_t_type == "":
            context.Result(False);
            context.Message("Failed to find a pointer-size integer type.")
            return False
        else:
            context.env.Append(CPPDEFINES=['NO_SIZE_T='+size_t_type])
            context.Result(size_t_type + ".")
            return True

    return result

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

def ConfigureHeader(context):
    context.Message("Writing configuration header zconf.h... ")
    with open('zconf.h', 'w') as content_file:
        content_file.write(context.env["ZCONFH"])
        context.Result(True)
        return True
    context.Result(False)
    return False
    


def CreateNewEnv():

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

    env = Environment(DEBUG_BUILD = GetOption('option_debug'))
    env.baseProjectDir = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    env.VariantDir(Dir('build'), Dir('.'), duplicate=0)

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
   
    env, prog = SetupBuildOutput(env, 'zlib', source_files)
    env = SetupInstalls(env)
    env = ConfigureEnv(env)
    #env = ConfigPlatformIDE(env, source_files, headerFiles, resourceFiles, prog)
    

def ConfigureEnv(env):
    if not env.GetOption('clean'):
    
        with open('zconf.h.in', 'r') as content_file:
            print("Reading zconf.h... ")
            env["ZCONFH"] = str(content_file.read())

        conf = Configure(env,
            custom_tests = {
                'CheckLargeFile64'  : CheckLargeFile64, 
                'CheckSizeT'        : CheckSizeT,
                'CheckSharedLibrary': CheckSharedLibrary,
                'CheckUnistdH'      : CheckUnistdH,
                'ConfigureHeader'   : ConfigureHeader,
                })

        conf.CheckCC()
        conf.CheckSharedLibrary()
        conf.CheckSizeT()
        conf.CheckLargeFile64()
        conf.CheckCHeader('sys/types.h')
        conf.CheckUnistdH()
        conf.ConfigureHeader()
        env = conf.Finish()
    
    if("linux" in platform.system().lower() ):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
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

    elif platform == "darwin":
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
    elif platform == "darwin":
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

def SetupBuildOutput(env, progName, sourceFiles):

    ###################################################
    # Determine number of CPUs
    num_cpus = get_cpu_nums()
    print ("Building with %d parallel jobs" % num_cpus)
    env.SetOption( "num_jobs", num_cpus )

    windowsRedirect = ""
    linuxRedirect = "2>&1"
    if("Windows" in platform.system()):
        windowsRedirect = "2>&1"
        linuxRedirect = ""

    soureFileObjs = []
    for file in sourceFiles:
        buildObj = env.SharedObject(file, 
                             CCCOM=env['CCCOM'] + " " + windowsRedirect + " > \"" + env.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect,
                             CXXCOM=env['CXXCOM'] + " " + windowsRedirect + " > \"" + env.baseProjectDir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linuxRedirect)
        soureFileObjs.append(buildObj)

    if("Windows" in platform.system()):
        env['LINKCOM'].list[0].cmd_list = env['LINKCOM'].list[0].cmd_list.replace('",'," 2>&1 > \\\"" + env.baseProjectDir + "/build/build_logs/MyLifeApp_link.txt\\\"\",") 
    else:
        env['LINKCOM'] = env['LINKCOM'].replace('",'," > \\\"" + env.baseProjectDir + "/build/build_logs/MyLifeApp_link.txt\\\"\" 2>&1 ,") 
   
    prog = env.SharedLibrary(progName, soureFileObjs)

    ###################################################
    # setup build output
    if not os.path.exists(env.baseProjectDir + "/build/build_logs"):
        os.makedirs(env.baseProjectDir + "/build/build_logs")

    if ARGUMENTS.get('fail', 0):
        Command('target', 'source', ['/bin/false'])

    atexit.register(display_build_status)

    def print_cmd_line(s, targets, sources, env):
        with open(env.baseProjectDir + "/build/build_logs/build_" + env['BUILD_LOG_TIME'] + ".log", "a") as f:
            f.write(s + "\n")

    env['BUILD_LOG_TIME'] = datetime.datetime.fromtimestamp(time.time()).strftime('%Y_%m_%d__%H_%M_%S')
    if("VERBOSE" not in ARGUMENTS):
        env['PRINT_CMD_LINE_FUNC'] = print_cmd_line
    
    builtBins = []
    if("Windows" in platform.system()):
        builtBins.append("build/" + progName + ".exe")
    else:
        builtBins.append('libzlib.so')
        
    Progress(ProgressCounter(sourceFiles, builtBins), interval=1)

    return [env, prog]

def SetupInstalls(env):

    return env
    
class ProgressCounter(object):

    HEADER = '\033[95m'
    OKBLUE = '\033[94m'
    OKGREEN = '\033[92m'
    WARNING = '\033[93m'
    FAIL = '\033[91m'
    ENDC = '\033[0m'

    def __init__(self, sourceFiles, targetBinaries):
        self.count = 0.0
        self.maxCount = float(len(sourceFiles))
        self.progressSources = sourceFiles
        self.targetBinaries = targetBinaries

    def disable(self):
        self.HEADER = ''
        self.OKBLUE = ''
        self.OKGREEN = ''
        self.WARNING = ''
        self.FAIL = ''
        self.ENDC = ''

    def __call__(self, node, *args, **kw):
        #print(str(node))
        slashedNode = str(node).replace("\\", "/")
        if(slashedNode in self.targetBinaries):
            filename = os.path.basename(str(node))
            if(node.get_state() == 2):
                print(self.OKGREEN + "[   LINK] " + self.ENDC + "Linking " +  filename )
            else:
                print(self.OKGREEN + "[   LINK] " + self.ENDC + "Skipping, already built " +  filename )
        
        if(str(node).endswith(".obj") or str(node).endswith(".o") or str(node).endswith(".os")):
            slashedNodeObj = os.path.splitext(slashedNode)[0] + ".c"
            if(slashedNodeObj in self.progressSources ):
                self.count += 1
                percent = self.count / self.maxCount * 100.00
                filename = os.path.basename(str(node))
                percentString = "{0:.2f}".format(percent)
                if(percent < 100):
                    percentString = " " + percentString
                if(percent < 10):
                    percentString = " " + percentString
                    
                if(node.get_state() == 2):
                    print(self.OKGREEN + "[" + percentString + "%] " + self.ENDC + "Compiling " + filename )
                else:
                    print(self.OKGREEN + "[" + percentString + "%] " + self.ENDC + "Skipping, already built " + filename )




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
    for filename in glob.glob('build/build_logs/*_link.txt'):
        f = open(filename, "r")
        tempList = f.read().splitlines()
        if(len(tempList) > 0):
            linkOutput += [OKBLUE + os.path.basename(filename).replace("_link.txt", "") + ":" + ENDC]
            linkOutput += tempList
    for line in linkOutput:
        line = highlight_word(line, "error", FAIL)
        line = highlight_word(line, "warning", WARNING)
        print(line)
        
    if status == 'failed':
        print(FAIL + "Build failed." + ENDC)
    elif status == 'ok':
        print (OKGREEN + "Build succeeded." + ENDC)
    

build = CreateNewEnv()
Return('build')

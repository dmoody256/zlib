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

    env.project_dir = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    env.build_dir = 'build'

    num_cpus = get_cpu_nums()
    ColorPrinter().InfoPrint("Building with " + str(num_cpus) + " parallel jobs")
    env.SetOption( "num_jobs", num_cpus )

    env.VariantDir(env.build_dir,           Dir('.'), duplicate=0)
    env.VariantDir(env.build_dir + '/test', Dir('./test'), duplicate=0)

    base_source_files = [
        env.build_dir + '/adler32.c',
        env.build_dir + '/compress.c',
        env.build_dir + '/crc32.c',
        env.build_dir + '/deflate.c',
        env.build_dir + '/gzclose.c',
        env.build_dir + '/gzlib.c',
        env.build_dir + '/gzread.c',
        env.build_dir + '/gzwrite.c',
        env.build_dir + '/inflate.c',
        env.build_dir + '/infback.c',
        env.build_dir + '/inftrees.c',
        env.build_dir + '/inffast.c',
        env.build_dir + '/trees.c',
        env.build_dir + '/uncompr.c',
        env.build_dir + '/zutil.c',
    ]

    base_header_files = [

    ]

    distrib_header_files = [

    ]

    env = ConfigureEnv(env)
    env.Append(CPPPATH=['.'])
    zlib = ZlibBuilder(env)
    
    prog_name = 'z'
    prog_static_name = 'z'
    if("Windows" in platform.system()):
        prog_static_name += "_static"
        
    shared_env, shared_lib   = zlib.SetupBuildEnv('shared', prog_name, base_source_files)
    static_env, static_lib   = zlib.SetupBuildEnv('static', prog_static_name, base_source_files, shared_lib)

    if("Windows" in platform.system()):
        shared_env.Append(CPPDEFINES=['ZLIB_DLL'])
    

    if(not env['COVER']):
        example_env, example_bin = zlib.SetupBuildEnv('exec', 'example', ['build/test/example.c'], static_lib)
        minizip_env, minizip_bin = zlib.SetupBuildEnv('exec', 'minigzip', ['build/test/minigzip.c'], example_bin)

        example_env.Append(LIBS=[File('./build/libz.a')])
        minizip_env.Append(LIBS=[File('./build/libz.a')])

    else:
        infcover_env, infcover_bin = zlib.SetupBuildEnv('exec', 'infcover', ['build/test/infcover.c'], static_lib)
        infcover_env.Append(LIBS=[File('./build/libz.a')])

   
    #env = SetupInstalls(env)
    #env = ConfigPlatformIDE(env, source_files, headerFiles, resourceFiles, prog)

    
def ConfigureEnv(env):

    p = ColorPrinter()

    def CheckLargeFile64(context):
        context.Message(p.ConfigString('Checking for off64_t... ') )

        prev_defines = ""
        if('CPPDEFINES' in context.env):
            prev_defines = context.env['CPPDEFINES']

        context.env.Append(CPPDEFINES=['_LARGEFILE64_SOURCE=1'])
        result = context.TryCompile("""
            #include <sys/types.h>
            off64_t dummy = 0;
        """, 
        '.c')
        if not result:
            context.env.Replace(CPPDEFINES = prev_defines)
        context.Result(result)
        return result

    def CheckFseeko(context):
        context.Message(p.ConfigString('Checking for fseeko... ') )
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
        context.Message(p.ConfigString('Checking for size_t... ') )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdlib.h>
            size_t dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result
    
    def CheckSizeTLongLong(context):
        context.Message(p.ConfigString('Checking for long long... ') )
        result = context.TryCompile("""
            long long dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result

    def CheckSizeTPointerSize(context, longlong_result):
        result = []
        context.Message(p.ConfigString('Checking for a pointer-size integer type... ') )
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
        context.Message(p.ConfigString( 'Checking for shared library support... ') )
        result = context.TryBuild(context.env.SharedLibrary, """
            extern int getchar();
            int hello() {return getchar();}
        """,
        '.c')

        context.Result(result)
        return result

    def CheckUnistdH(context):
        context.Message(p.ConfigString('Checking for unistd.h... ') )
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
        context.Message(p.ConfigString('Checking for strerror... ') )
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
        context.Message(p.ConfigString('Checking for stdarg.h... ') )
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
        context.Message(p.ConfigString('Using z_ prefix on all symbols... ') )
        result = context.env['ZPREFIX'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sZ_PREFIX(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddSolo(context):
        context.Message(p.ConfigString('Using Z_SOLO to build... ') )
        result = context.env['SOLO'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"\#define\sZCONF_H", r"#define ZCONF_H\n#define Z_SOLO", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddCover(context):
        context.Message(p.ConfigString('Using code coverage flags... ') )
        result = context.env['COVER'] 
        if result:
            context.env.Append(CCFLAGS=[
                '-fprofile-arcs', 
                '-ftest-coverage',
            ])
            context.env.Append(LINKFLAGS=[
                '-fprofile-arcs', 
                '-ftest-coverage',
            ])
            context.env.Append(LIBS=[
                'gcov', 
            ])
        if not context.env['GCC_CLASSIC'] == "":
            context.env.Replace(CC = context.env['GCC_CLASSIC'])
        context.Result(result)
        return result
    
    def CheckVsnprintf(context):
        context.Message(p.ConfigString("Checking whether to use vs[n]printf() or s[n]printf()... ") )
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
        context.Message(p.ConfigString("Checking for vsnprintf() in stdio.h... ") )
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
            print(p.ConfigString("  WARNING: vsnprintf() not found, falling back to vsprintf(). zlib") )
            print(p.ConfigString("  can build but will be open to possible buffer-overflow security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckVsnprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of vsnprintf()... ") )
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
            print(p.ConfigString("  WARNING: apparently vsnprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckVsprintfReturn(context):
        context.Message(p.ConfigString( "Checking for return value of vsnprintf()... ") )
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
            print(p.ConfigString("  WARNING: apparently vsprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSnStdio(context):
        context.Message(p.ConfigString("Checking for snprintf() in stdio.h... ") )
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
            print(p.ConfigString("  WARNING: snprintf() not found, falling back to sprintf(). zlib") )
            print(p.ConfigString("  can build but will be open to possible buffer-overflow security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSnprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of snprintf()... ") )
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
            print(p.ConfigString("  WARNING: apparently snprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of sprintf()... ") )
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
            print(p.ConfigString("  WARNING: apparently sprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckHidden(context):
        context.Message(p.ConfigString("Checking for attribute(visibility) support... ") )
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
        if('--solo' in sys.argv):    env['SOLO']    = GetOption('option_solo')
        if('--cover' in sys.argv):   env['COVER']   = GetOption('option_cover')

        if not os.path.exists(env.project_dir + "/" + env.build_dir):
            os.makedirs(env.project_dir + "/" + env.build_dir)

        vars.Save('build.conf', env)

        configureString = ""
        if env['ZPREFIX']: configureString += "--zprefix "
        if env['SOLO']:    configureString += "--solo "
        if env['COVER']:   configureString += "--cover "

        if configureString == "": configureString = "Configuring... "
        else:                     configureString = "Configuring with " + configureString

        p.InfoPrint(configureString)

        SCons.Script.Main.progress_display.set_mode(1)

        conf = Configure(env, conf_dir = env.build_dir + "/conf_tests", log_file = env.build_dir + "/conf.log", 
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
            conf.env["ZCONFH"] = str(content_file.read())  

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

        SCons.Script.Main.progress_display.set_mode(0)

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
            'LocalDebuggerCommand':os.path.abspath(env.project_dir).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory':os.path.abspath(env.project_dir).replace('\\', '/') + "/build/",
            
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

class ZlibBuilder():

    def __init__(self, env):
        self.build_num = 0
        self.reset_callback = None
        self.env = env
        self.prog_counter = ProgressCounter()
        self.reset_callback = SCons.Action.ActionFactory( self.prog_counter.ResetProgress,
            lambda source_files, target_bins: 'Reseting Progress Counter for ' + str(target_bins))
        
        Progress(self.prog_counter, interval=1)

    def SetupBuildEnv(self, prog_type, prog_name, source_files, previous_build = None):

        self.build_num+=1
        build_env = self.env.Clone()
        variant_dir_str = build_env.build_dir + "/" + prog_name + "_objs_" + str(self.build_num)
        build_env.VariantDir ( variant_dir_str, '.', duplicate=0 )
        
        variant_source_files = []
        for file in source_files:
            variant_source_files.append(file.replace(build_env.build_dir, variant_dir_str))
        
        win_redirect = ""
        linux_redirect = "2>&1"
        if("Windows" in platform.system()):
            win_redirect = "2>&1"
            linux_redirect = ""

        source_objs = []
        for file in variant_source_files:

            if(prog_type == 'shared'):    build_obj_command = build_env.SharedObject
            elif(   prog_type == 'static' 
                 or prog_type == 'exec'): build_obj_command = build_env.Object
            else:
                ColorPrinter().ErrorPrint("Could not determine build type.")
                raise

            build_obj = build_obj_command(file, 
                CCCOM= self.env['CCCOM']  + " " + win_redirect + " > \"" + build_env.project_dir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linux_redirect,
                CXXCOM=self.env['CXXCOM'] + " " + win_redirect + " > \"" + build_env.project_dir + "/build/build_logs/" + os.path.splitext(os.path.basename(file))[0] + "_compile.txt\" " + linux_redirect)
            source_objs.append(build_obj)
           

        if("Windows" in platform.system()):
            build_env['LINKCOM'].list[0].cmd_list = self.env['LINKCOM'].list[0].cmd_list.replace('",'," 2>&1 > \\\"" + build_env.project_dir + "/build/build_logs/" + prog_name + "_link.txt\\\"\",") 
        else:
            build_env['LINKCOM'] = self.env['LINKCOM'].replace('",'," > \\\"" + build_env.project_dir + "/build/build_logs/" + prog_name + "_link.txt\\\"\" 2>&1 ,") 
       
        if(prog_type == "shared"):
            prog = build_env.SharedLibrary(build_env.build_dir + "/" + prog_name, source_objs)
        elif(prog_type == "static"):
            prog = build_env.StaticLibrary(build_env.build_dir + "/" + prog_name, source_objs)
        elif(prog_type == 'exec'):
            prog =       build_env.Program(build_env.build_dir + "/" + prog_name, source_objs)

        if not os.path.exists(build_env.project_dir + "/build/build_logs"):
            os.makedirs(build_env.project_dir + "/build/build_logs")

        if ARGUMENTS.get('fail', 0):
            Command('target', 'source', ['/bin/false'])

        atexit.register(display_build_status)

        def print_cmd_line(s, targets, sources, env):
            with open(build_env.project_dir + "/build/build_logs/build_" + env['BUILD_LOG_TIME'] + ".log", "a") as f:
                f.write(s + "\n")

        build_env['BUILD_LOG_TIME'] = datetime.datetime.fromtimestamp(time.time()).strftime('%Y_%m_%d__%H_%M_%S')
        if(not build_env['VERBOSE_COMPILE']):
            build_env['PRINT_CMD_LINE_FUNC'] = print_cmd_line
        
        built_bins = []
        if("Windows" in platform.system()):
            if(prog_type == 'shared'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$SHLIBPREFIX') + prog_name + build_env.subst('$SHLIBSUFFIX'))
            elif(prog_type == 'static'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$LIBPREFIX') + prog_name + build_env.subst('$LIBSUFFIX'))
            elif(prog_type == 'exec'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$PROGPREFIX') + prog_name + build_env.subst('$PROGSUFFIX') )
        else:
            if(prog_type == 'shared'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$SHLIBPREFIX') + prog_name + build_env.subst('$SHLIBSUFFIX'))
            elif(prog_type == 'static'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$LIBPREFIX') + prog_name + build_env.subst('$LIBSUFFIX'))
            elif(prog_type == 'exec'):
                built_bins.append(build_env.build_dir + "/" + build_env.subst('$PROGPREFIX') + prog_name + build_env.subst('$PROGSUFFIX') )

        if(not self.reset_callback == None ):
            if(previous_build == None):
                self.prog_counter.ResetProgress(variant_source_files, built_bins)
            else:
                reset = build_env.Command( None, variant_source_files, self.reset_callback(variant_source_files, built_bins))
                Depends(reset, previous_build)
                for node in source_objs:
                    Depends(node, reset)

        return [build_env, prog]

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
            #print (str(self.progress_sources) )
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
        #print("reseting sources to " + str(source_files))
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
    
    if("Windows" in platform.system()):
        HEADER = ''
        OKBLUE = ''
        OKGREEN = ''
        WARNING = ''
        FAIL = ''
        ENDC = ''
    else:
        HEADER = '\033[95m'
        OKBLUE = '\033[94m'
        OKGREEN = '\033[92m'
        WARNING = '\033[93m'
        FAIL = '\033[91m'
        ENDC = '\033[0m'
    
    if status == 'failed':
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
            print(FAIL + "Build of " + buildLink + " failed." + ENDC)
    

class ColorPrinter():

    def __init__(self):
    
        if("Windows" in platform.system()):
            self.HEADER = ''
            self.OKBLUE = ''
            self.OKGREEN = ''
            self.WARNING = ''
            self.FAIL = ''
            self.ENDC = ''
        else:
            self.HEADER = '\033[95m'
            self.OKBLUE = '\033[94m'
            self.OKGREEN = '\033[92m'
            self.WARNING = '\033[93m'
            self.FAIL = '\033[91m'
            self.ENDC = '\033[0m'

    def InfoPrint(self, message):
        print(self.HEADER + "[   INFO] " + self.ENDC + message)

    def ErrorPrint(self, message):
        print(self.FAIL + "[  ERROR] " + self.ENDC + message)

    def CompilePrint(self, percent, message):
        percent_string = "{0:.2f}".format(percent)
        if(percent < 100): percent_string = " " + percent_string
        if(percent < 10):  percent_string = " " + percent_string
        print(self.OKGREEN + "[" + percent_string + "%] " + self.ENDC + message )

    def LinkPrint(self, message):
        print(self.OKGREEN + "[   LINK] " + self.ENDC + message)

    def ConfigString(self, message):
        return self.OKBLUE + "[ CONFIG] " + self.ENDC + message


CreateNewEnv()

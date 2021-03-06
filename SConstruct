"""zlib library built with SCons"""

import os
import sys
import re
import platform

import SCons
from SCons.Environment import Environment
from SCons.Script import Variables
from SCons.Script.SConscript import Configure
from SCons.Script.Main import SetOption, AddOption, GetOption, Progress

from BuildUtils.SconsUtils import SetBuildJobs, SetupBuildEnv, ProgressCounter
from BuildUtils.ColorPrinter import ColorPrinter


def create_new_env():
    """ Entry point for build, could be called by other builds"""

    SetupOptions()

    env = Environment(
        DEBUG_BUILD=GetOption('option_debug'),
        ZPREFIX=GetOption('option_zprefix'),
        SOLO=GetOption('option_solo'),
        COVER=GetOption('option_cover'),
        GCC_CLASSIC=GetOption('option_gcc_classic'),
        VERBOSE_COMPILE=GetOption('option_verbose'),
    )

    env['PROJECT_DIR'] = os.path.abspath(
        env.Dir('.').abspath).replace('\\', '/')
    SetBuildJobs(env)

    base_source_files = [
        'repo/adler32.c',
        'repo/compress.c',
        'repo/crc32.c',
        'repo/deflate.c',
        'repo/gzclose.c',
        'repo/gzlib.c',
        'repo/gzread.c',
        'repo/gzwrite.c',
        'repo/inflate.c',
        'repo/infback.c',
        'repo/inftrees.c',
        'repo/inffast.c',
        'repo/trees.c',
        'repo/uncompr.c',
        'repo/zutil.c',
    ]

    env = configure_env(env)
    env.Append(CPPPATH=['.'])
    prog_name = 'z'
    prog_static_name = prog_name
    if "Windows" in platform.system():
        prog_static_name = prog_name + "_static"

    progress = ProgressCounter()

    _static_env, _z_static = SetupBuildEnv(
        env, progress, 'static', prog_static_name,
        base_source_files, 'build/build_static', 'deploy')
    shared_env, _z_shared = SetupBuildEnv(
        env, progress, 'shared', prog_name,
        base_source_files, 'build/build_shared', 'deploy')

    if "Windows" in platform.system():
        shared_env.Append(CPPDEFINES=['ZLIB_DLL'])

    Progress(progress, interval=1)

    zlib_static_lib = env.subst('$LIBPREFIX') + \
        prog_static_name + env.subst('$LIBSUFFIX')

    if not env['COVER']:
        example_env, _example_bin = SetupBuildEnv(env, progress, 'exec', 'example', [
            'repo/test/example.c'], 'build/build_static', 'build')
        minizip_env, _minizip_bin = SetupBuildEnv(env, progress, 'exec', 'minigzip', [
            'repo/test/minigzip.c'], 'build/build_static', 'build')

        example_env.Append(CPPPATH=[example_env.Dir('repo').abspath], LIBS=[
            example_env.File('./deploy/' + zlib_static_lib)])
        minizip_env.Append(CPPPATH=[minizip_env.Dir('repo').abspath], LIBS=[
            minizip_env.File('./deploy/' + zlib_static_lib)])

    else:
        infcover_env, _infcover_bin = SetupBuildEnv(env, progress, 'exec', 'infcover', [
            'repo/test/infcover.c'], 'build/build_static', 'build')
        infcover_env.Append(CPPPATH=[infcover_env.Dir('repo').abspath], LIBS=[
            infcover_env.File('./deploy/' + zlib_static_lib)])

    #env = SetupInstalls(env)
    #env = ConfigPlatformIDE(env, source_files, headerFiles, resourceFiles, prog)


def configure_env(env):
    """
    This function is just for organization really, groups all
    the configure stuff together.
    """

    printer = ColorPrinter()

    def check_large_file64(context):
        """Test to check if the off64_t c type is avialable."""

        context.Message(printer.ConfigString('Checking for off64_t... '))

        prev_defines = ""
        if 'CPPDEFINES' in context.env:
            prev_defines = context.env['CPPDEFINES']

        context.env.Append(CPPDEFINES=['_LARGEFILE64_SOURCE=1'])
        result = context.TryCompile("""
            #include <sys/types.h>
            off64_t dummy = 0;
        """,
                                    '.c')
        if not result:
            context.env.Replace(CPPDEFINES=prev_defines)
        context.Result(result)
        return result

    def check_fseeko(context):
        """Test to see if fseeko function is available."""

        context.Message(printer.ConfigString('Checking for fseeko... '))
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
        context.Message(printer.ConfigString('Checking for size_t... '))
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdlib.h>
            size_t dummy = 0;
        """,
                                    '.c')
        context.Result(result)
        return result

    def CheckSizeTLongLong(context):
        context.Message(printer.ConfigString('Checking for long long... '))
        result = context.TryCompile("""
            long long dummy = 0;
        """,
                                    '.c')
        context.Result(result)
        return result

    def CheckSizeTPointerSize(context, longlong_result):
        result = []
        context.Message(printer.ConfigString(
            'Checking for a pointer-size integer type... '))
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
            context.Result("Failed.")
            return False
        else:
            context.env.Append(CPPDEFINES=['NO_SIZE_T='+result[1]])
            context.Result(result[1] + ".")
            return True

    def CheckSharedLibrary(context):
        context.Message(printer.ConfigString(
            'Checking for shared library support... '))
        result = context.TryBuild(context.env.SharedLibrary, """
            extern int getchar();
            int hello() {return getchar();}
        """,
                                  '.c')

        context.Result(result)
        return result

    def CheckUnistdH(context):
        context.Message(printer.ConfigString('Checking for unistd.h... '))
        result = context.TryCompile("""
            #include <unistd.h>
            int main() { return 0; }
        """,
                                    '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_UNISTD_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"], re.MULTILINE)
        context.Result(result)
        return result

    def CheckStrerror(context):
        context.Message(printer.ConfigString('Checking for strerror... '))
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
        context.Message(printer.ConfigString('Checking for stdarg.h... '))
        result = context.TryCompile("""
            #include <stdarg.h>
            int main() { return 0; }
        """,
                                    '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_STDARG_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"], re.MULTILINE)
        context.Result(result)
        return result

    def AddZPrefix(context):
        context.Message(printer.ConfigString(
            'Using z_ prefix on all symbols... '))
        result = context.env['ZPREFIX']
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sZ_PREFIX(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"], re.MULTILINE)
        context.Result(result)
        return result

    def AddSolo(context):
        context.Message(printer.ConfigString('Using Z_SOLO to build... '))
        result = context.env['SOLO']
        if result:
            context.env["ZCONFH"] = re.sub(
                r"\#define\sZCONF_H", r"#define ZCONF_H\n#define Z_SOLO", context.env["ZCONFH"], re.MULTILINE)
        context.Result(result)
        return result

    def AddCover(context):
        context.Message(printer.ConfigString('Using code coverage flags... '))
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
            context.env.Replace(CC=context.env['GCC_CLASSIC'])
        context.Result(result)
        return result

    def CheckVsnprintf(context):
        context.Message(printer.ConfigString(
            "Checking whether to use vs[n]printf() or s[n]printf()... "))
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
        context.Message(printer.ConfigString(
            "Checking for vsnprintf() in stdio.h... "))
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
            print(printer.ConfigString(
                "  WARNING: vsnprintf() not found, falling back to vsprintf(). zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible buffer-overflow security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckVsnprintfReturn(context):
        context.Message(printer.ConfigString(
            "Checking for return value of vsnprintf()... "))
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
            print(printer.ConfigString(
                "  WARNING: apparently vsnprintf() does not return a value. zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible string-format security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckVsprintfReturn(context):
        context.Message(printer.ConfigString(
            "Checking for return value of vsnprintf()... "))
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
            print(printer.ConfigString(
                "  WARNING: apparently vsprintf() does not return a value. zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible string-format security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckSnStdio(context):
        context.Message(printer.ConfigString(
            "Checking for snprintf() in stdio.h... "))
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
            print(printer.ConfigString(
                "  WARNING: snprintf() not found, falling back to sprintf(). zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible buffer-overflow security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckSnprintfReturn(context):
        context.Message(printer.ConfigString(
            "Checking for return value of snprintf()... "))
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
            print(printer.ConfigString(
                "  WARNING: apparently snprintf() does not return a value. zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible string-format security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckSprintfReturn(context):
        context.Message(printer.ConfigString(
            "Checking for return value of sprintf()... "))
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
            print(printer.ConfigString(
                "  WARNING: apparently sprintf() does not return a value. zlib"))
            print(printer.ConfigString(
                "  can build but will be open to possible string-format security"))
            print(printer.ConfigString("  vulnerabilities."))
        return result

    def CheckHidden(context):
        context.Message(printer.ConfigString(
            "Checking for attribute(visibility) support... "))
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

        conf_vars = Variables('build.conf')

        conf_vars.Add('ZPREFIX', '')
        conf_vars.Add('SOLO', '')
        conf_vars.Add('COVER', '')
        conf_vars.Update(env)

        if('--zprefix' in sys.argv):
            env['ZPREFIX'] = GetOption('option_zprefix')
        if('--solo' in sys.argv):
            env['SOLO'] = GetOption('option_solo')
        if('--cover' in sys.argv):
            env['COVER'] = GetOption('option_cover')

        conf_vars.Save('build.conf', env)

        configureString = ""
        if env['ZPREFIX']:
            configureString += "--zprefix "
        if env['SOLO']:
            configureString += "--solo "
        if env['COVER']:
            configureString += "--cover "

        if configureString == "":
            configureString = "Configuring... "
        else:
            configureString = "Configuring with " + configureString

        printer.InfoPrint(configureString)

        SCons.Script.Main.progress_display.set_mode(1)

        conf = Configure(env, conf_dir="build/conf_tests", log_file="conf.log",
                         custom_tests={
                             'check_large_file64': check_large_file64,
                             'check_fseeko': check_fseeko,
                             'CheckSizeT': CheckSizeT,
                             'CheckSizeTLongLong': CheckSizeTLongLong,
                             'CheckSizeTPointerSize': CheckSizeTPointerSize,
                             'CheckSharedLibrary': CheckSharedLibrary,
                             'CheckUnistdH': CheckUnistdH,
                             'CheckStrerror': CheckStrerror,
                             'CheckStdargH': CheckStdargH,
                             'CheckVsnprintf': CheckVsnprintf,
                             'CheckVsnStdio': CheckVsnStdio,
                             'CheckVsnprintfReturn': CheckVsnprintfReturn,
                             'CheckVsprintfReturn': CheckVsprintfReturn,
                             'CheckSnStdio': CheckSnStdio,
                             'CheckSnprintfReturn': CheckSnprintfReturn,
                             'CheckSprintfReturn': CheckSprintfReturn,
                             'CheckHidden': CheckHidden,
                             'AddZPrefix': AddZPrefix,
                             'AddSolo': AddSolo,
                             'AddCover': AddCover,
                         })

        with open('repo/zconf.h.in', 'r') as content_file:
            conf.env["ZCONFH"] = str(content_file.read())

        conf.CheckSharedLibrary()
        # conf.CheckExternalNames()
        if not conf.CheckSizeT():
            conf.CheckSizeTPointerSize(conf.CheckSizeTLongLong())
        if not conf.check_large_file64():
            conf.check_fseeko()

        conf.CheckStrerror()
        conf.CheckUnistdH()
        conf.CheckStdargH()

        if conf.env['ZPREFIX']:
            conf.AddZPrefix()
        if conf.env['SOLO']:
            conf.AddSolo()
        if conf.env['COVER']:
            conf.AddCover()

        with open('repo/zconf.h', 'w') as content_file:
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

    if("linux" in platform.system().lower()):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
            '-O3',
            # '-fPIC',
            # "-rdynamic",
        ])

        env.Append(CXXFLAGS=[
            # "-std=c++11",
        ])

        env.Append(LDFLAGS=[
            debugFlag,
            # "-rdynamic",
        ])

        env.Append(LIBPATH=[
            # "/usr/lib/gnome-settings-daemon-3.0/",
        ])

        env.Append(LIBS=[
            # "GL",
        ])

    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):
        pass
    return env


def ConfigPlatformIDE(env, sourceFiles, headerFiles, resources, program):
    if platform == "linux" or platform == "linux2":
        print("Eclipse C++ project not implemented yet")
    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):
        variantSourceFiles = []
        for file in sourceFiles:
            variantSourceFiles.append(re.sub("^build", "../Source", file))
        variantHeaderFiles = []
        for file in headerFiles:
            variantHeaderFiles.append(re.sub("^Source", "../Source", file))
        buildSettings = {
            'LocalDebuggerCommand': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/",

        }
        buildVariant = 'Release|x64'
        cmdargs = ''
        if(env['DEBUG_BUILD']):
            buildVariant = 'Debug|x64'
            cmdargs = '--debug_build'
        env.MSVSProject(target='VisualStudio/MyLifeApp' + env['MSVSPROJECTSUFFIX'],
                        srcs=variantSourceFiles,
                        localincs=variantHeaderFiles,
                        resources=resources,
                        buildtarget=program,
                        DebugSettings=buildSettings,
                        variant=buildVariant,
                        cmdargs=cmdargs)
    return env


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


create_new_env()

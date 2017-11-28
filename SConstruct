


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


def CheckLargeFile64(context):
    context.Message('Checking for off64_t... ')

    prevDefines = ""
    if('CPPDEFINES' in context.env):
        prevDefines = context.env['CPPDEFINES']

    context.env.Append(CPPDEFINES=['_LARGEFILE64_SOURCE=1'])
    result = context.TryCompile("""

    #include <sys/types.h>
    off64_t dummy = 0;

    """, '.c')

    if not result:
        context.env.Replace(CPPDEFINES = prevDefines)
    context.Result(result)
    return result

env = Environment()
conf = Configure(env, config_h = True, custom_tests = {'CheckLargeFile64' : CheckLargeFile64})
conf.CheckCC()
conf.CheckLargeFile64()
conf.CheckCHeader('sys/types.h')
if conf.CheckCHeader('unistd.h'):
    conf.env.Append(CPPDEFINES=['HAVE_UNISTD_H'])    
if not conf.CheckFunc('fseeko'):
    conf.env.Append(CPPDEFINES=['NO_FSEEKO'])

env = conf.Finish()


#if(MSVC)
#    set(CMAKE_DEBUG_POSTFIX "d")
#    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
#    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
#    include_directories(${CMAKE_CURRENT_SOURCE_DIR})
#endif()

source_files = [
    'adler32.c',
    'compress.c',
    'crc32.c',
    'deflate.c',
    'gzclose.c',
    'gzlib.c',
    'gzread.c',
    'gzwrite.c',
    'inflate.c',
    'infback.c',
    'inftrees.c',
    'inffast.c',
    'trees.c',
    'uncompr.c',
    'zutil.c',
]

env.SharedLibrary('zlib', source_files)
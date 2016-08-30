# -*- mode: python; -*-

# This SConscript describes build rules for the "mongo" project.

import itertools
import os
import re
import sys
from buildscripts import utils

Import("env")
Import("has_option")
Import("get_option")
Import("usemozjs")
Import("use_system_version_of_library")

# Boost we need everywhere. 's2' is spammed in all over the place by
# db/geo unfortunately. pcre is also used many places.
env.InjectThirdPartyIncludePaths(libraries=['boost', 's2', 'pcre'])
env.InjectMongoIncludePaths()

env.SConscript(
    dirs=[
        'base',
        'bson',
        'client',
        'crypto',
        'db',
        'dbtests',
        'executor',
        'installer',
        'logger',
        'platform',
        'rpc',
        's',
        'scripting',
        'shell',
        'tools',
        'unittest',
        'util',
    ],
)

# NOTE: This library does not really belong here. Its presence here is
# temporary. Do not add to this library, do not remove from it, and do
# not declare other libraries in this file.

baseSource=[
    'base/data_range.cpp',
    'base/data_range_cursor.cpp',
    'base/data_type.cpp',
    'base/data_type_string_data.cpp',
    'base/data_type_terminated.cpp',
    'base/error_codes.cpp',
    'base/global_initializer.cpp',
    'base/global_initializer_registerer.cpp',
    'base/init.cpp',
    'base/initializer.cpp',
    'base/initializer_context.cpp',
    'base/initializer_dependency_graph.cpp',
    'base/make_string_vector.cpp',
    'base/parse_number.cpp',
    'base/status.cpp',
    'base/string_data.cpp',
    'base/validate_locale.cpp',
    'bson/bson_validate.cpp',
    'bson/bsonelement.cpp',
    'bson/bsonmisc.cpp',
    'bson/bsonobj.cpp',
    'bson/bsonobjbuilder.cpp',
    'bson/bsontypes.cpp',
    'bson/json.cpp',
    'bson/oid.cpp',
    'bson/timestamp.cpp',
    'logger/component_message_log_domain.cpp',
    'logger/console.cpp',
    'logger/log_component.cpp',
    'logger/log_component_settings.cpp',
    'logger/log_manager.cpp',
    'logger/log_severity.cpp',
    'logger/logger.cpp',
    'logger/logstream_builder.cpp',
    'logger/message_event_utf8_encoder.cpp',
    'logger/message_log_domain.cpp',
    'logger/ramlog.cpp',
    'logger/rotatable_file_manager.cpp',
    'logger/rotatable_file_writer.cpp',
    'platform/random.cpp',
    'platform/strnlen.cpp',
    'util/allocator.cpp',
    'util/assert_util.cpp',
    'util/base64.cpp',
    'util/concurrency/thread_name.cpp',
    'util/exception_filter_win32.cpp',
    'util/hex.cpp',
    'util/itoa.cpp',
    'util/log.cpp',
    'util/signal_handlers_synchronous.cpp',
    'util/stacktrace.cpp',
    'util/stacktrace_${TARGET_OS_FAMILY}.cpp',
    'util/static_observer.cpp',
    'util/stringutils.cpp',
    'util/system_clock_source.cpp',
    'util/system_tick_source.cpp',
    'util/text.cpp',
    'util/time_support.cpp',
    'util/version.cpp',
]

baseLibDeps=[
    # NOTE: This library *must not* depend on any libraries than
    # the ones declared here. Do not add to this list.
    '$BUILD_DIR/third_party/murmurhash3/murmurhash3',
    '$BUILD_DIR/third_party/shim_boost',
    '$BUILD_DIR/third_party/shim_pcrecpp',
    'util/debugger',
    'util/quick_exit',
]

if GetOption('experimental-decimal-support') == 'on':
    baseSource.append('platform/decimal128.cpp')
    baseLibDeps.append('$BUILD_DIR/third_party/shim_intel_decimal128')
else:
    baseSource.append('platform/decimal128_dummy.cpp')

env.Library(
    target='base',
    source=baseSource,
    LIBDEPS=baseLibDeps,
)

js_engine_ver = get_option("js-engine") if get_option("server-js") == "on" else "none"

# On windows, we need to escape the backslashes in the command-line
# so that windows paths look okay.
cmd_line = " ".join(sys.argv).encode('string-escape')
if env.TargetOSIs('windows'):
    cmd_line = cmd_line.replace('\\', r'\\')

module_list = '{ %s }' % ', '.join([ '"{0}"'.format(x) for x in env['MONGO_MODULES'] ])

# This generates a numeric representation of the version string so that
# you can easily compare versions of MongoDB without having to parse
# the version string.
#
# The rules for this are
# {major}{minor}{release}{pre/rc/final}
# If the version is pre-release and not an rc, the final number is 0
# If the version is an RC, the final number of 1 + rc number
# If the version is pre-release between RC's, the final number is 1 + rc number
# If the version is a final release, the final number is 99
#
# Examples:
# 3.1.1-123     = 3010100
# 3.1.1-rc2     = 3010103
# 3.1.1-rc2-123 = 3010103
# 3.1.1         = 3010199
#
env['MONGO_VERSION'] = "3.3.2-77-g81efb31"
version_parts = [ x for x in re.match(r'^(\d+)\.(\d+)\.(\d+)-?((?:(rc)(\d+))?.*)?',
    env['MONGO_VERSION']).groups() ]
version_extra = version_parts[3] if version_parts[3] else ""
if version_parts[4] == 'rc':
    version_parts[3] = int(version_parts[5]) + -50
elif version_parts[3]:
    version_parts[2] = int(version_parts[2]) + 1
    version_parts[3] = -100
else:
    version_parts[3] = 0
version_parts = [ int(x) for x in version_parts[:4]]

# This turns the MONGO_BUILDINFO_ENVIRONMENT_DATA tuples into a std::vector of
# std::tuple<string, string, bool, bool>.
buildInfoInitializer = []
for tup in env['MONGO_BUILDINFO_ENVIRONMENT_DATA']:
    def pyToCXXBool(val):
        return "true" if val else "false"
    def wrapInQuotes(val):
        return '"{0}"'.format(val)
    buildInfoInitializer.append(
        'std::make_tuple({0})'.format(', '.join(
            (
                wrapInQuotes(tup[0]),
                wrapInQuotes(env.subst(tup[1])),
                pyToCXXBool(tup[2]),
                pyToCXXBool(tup[3]),
            )
        ))
    )
buildInfoInitializer = '{{ {0} }}'.format(', '.join(buildInfoInitializer))

generatedVersionFile = env.Substfile(
    'util/version.cpp.in',
    SUBST_DICT=[
        ('@mongo_version@', env['MONGO_VERSION']),
        ('@mongo_version_major@', version_parts[0]),
        ('@mongo_version_minor@', version_parts[1]),
        ('@mongo_version_patch@', version_parts[2]),
        ('@mongo_version_extra@', version_parts[3]),
        ('@mongo_version_extra_str@', version_extra),
        ('@mongo_git_hash@', env['MONGO_GIT_HASH']),
        ('@buildinfo_js_engine@', js_engine_ver),
        ('@buildinfo_allocator@', env['MONGO_ALLOCATOR']),
        ('@buildinfo_modules@', module_list),
        ('@buildinfo_environment_data@', buildInfoInitializer),
    ])
env.Alias('generated-sources', generatedVersionFile)

config_header_substs = (
    ('@mongo_config_byte_order@', 'MONGO_CONFIG_BYTE_ORDER'),
    ('@mongo_config_debug_build@', 'MONGO_CONFIG_DEBUG_BUILD'),
    ('@mongo_config_experimental_decimal_support@', 'MONGO_CONFIG_EXPERIMENTAL_DECIMAL_SUPPORT'),
    ('@mongo_config_have_thread_local@', 'MONGO_CONFIG_HAVE_THREAD_LOCAL'),
    ('@mongo_config_have___declspec_thread@', 'MONGO_CONFIG_HAVE___DECLSPEC_THREAD'),
    ('@mongo_config_have___thread@', 'MONGO_CONFIG_HAVE___THREAD'),
    ('@mongo_config_have_execinfo_backtrace@', 'MONGO_CONFIG_HAVE_EXECINFO_BACKTRACE'),
    ('@mongo_config_have_fips_mode_set@', 'MONGO_CONFIG_HAVE_FIPS_MODE_SET'),
    ('@mongo_config_have_header_unistd_h@', 'MONGO_CONFIG_HAVE_HEADER_UNISTD_H'),
    ('@mongo_config_have_memset_s@', 'MONGO_CONFIG_HAVE_MEMSET_S'),
    ('@mongo_config_have_posix_monotonic_clock@', 'MONGO_CONFIG_HAVE_POSIX_MONOTONIC_CLOCK'),
    ('@mongo_config_have_std_is_trivially_copyable@', 'MONGO_CONFIG_HAVE_STD_IS_TRIVIALLY_COPYABLE'),
    ('@mongo_config_have_std_make_unique@', 'MONGO_CONFIG_HAVE_STD_MAKE_UNIQUE'),
    ('@mongo_config_have_strnlen@', 'MONGO_CONFIG_HAVE_STRNLEN'),
    ('@mongo_config_optimized_build@', 'MONGO_CONFIG_OPTIMIZED_BUILD'),
    ('@mongo_config_ssl@', 'MONGO_CONFIG_SSL'),
    ('@mongo_config_wiredtiger_enabled@', 'MONGO_CONFIG_WIREDTIGER_ENABLED'),
)

def makeConfigHeaderDefine(self, key):
    val = "// #undef {0}".format(key)
    if key in self['CONFIG_HEADER_DEFINES']:
        val = "#define {0} {1}".format(key, self['CONFIG_HEADER_DEFINES'][key])
    return val
env.AddMethod(makeConfigHeaderDefine)

generateConfigHeaderFile = env.Substfile(
    'config.h.in',
    SUBST_DICT=[(k, env.makeConfigHeaderDefine(v)) for (k, v) in config_header_substs]
)
env.Alias('generated-sources', generateConfigHeaderFile)

mongodLibDeps = [
    "db/commands/core",
    "db/conn_pool_options",
    "db/mongod_options",
    "db/mongodandmongos",
    "db/mongodwebserver",
    "db/serveronly",
    "db/repl/storage_interface_impl",
    "executor/network_interface_factory",
    's/commands/shared_cluster_commands',
    "util/ntservice",
]

if has_option('use-cpu-profiler'):
    mongodLibDeps.append('db/cpuprofiler')

mongod = env.Program(
    target="mongod",
    source=[
        "db/db.cpp",
        "db/mongod_options_init.cpp",
    ],
    LIBDEPS=mongodLibDeps,
)

env.Default(env.Install('#/', mongod))

# tools
rewrittenTools = [ "mongodump", "mongorestore", "mongoexport", "mongoimport", "mongostat", "mongotop", "bsondump", "mongofiles", "mongooplog" ]

# mongobridge and mongoperf
env.Install(
    '#/',
    [
        env.Program("mongoperf",
                    [
                        "client/examples/mongoperf.cpp",
                    ],
                    LIBDEPS=[
                        "db/serveronly",
                        "util/ntservice_mock",
                    ]),
    ])

# mongos
env.Install(
    '#/',
    env.Program(
        "mongos",
        [
            "s/server.cpp",
            "s/mongos_options.cpp",
            "s/mongos_options_init.cpp",
        ],
        LIBDEPS=[
            'db/conn_pool_options',
            'db/commands/core',
            'db/mongodandmongos',
            's/client/sharding_connection_hook',
            's/commands/cluster_commands',
            's/commands/shared_cluster_commands',
            's/coreshard',
            's/mongoscore',
            's/sharding_initialization',
            'util/ntservice',
            'util/options_parser/options_parser_init',
        ]))

env.Library("linenoise_utf8",
    source=[
        "shell/linenoise_utf8.cpp",
    ])

# --- sniffer ---
mongosniff_built = False
if env.TargetOSIs('osx') or env["_HAVEPCAP"]:
    mongosniff_built = True
    sniffEnv = env.Clone()
    sniffEnv.Append( CPPDEFINES="MONGO_EXPOSE_MACROS" )

    if not env.TargetOSIs('windows'):
        sniffEnv.Append( LIBS=[ "pcap" ] )
    else:
        sniffEnv.Append( LIBS=[ "wpcap" ] )

    sniffEnv.Install( '#/', sniffEnv.Program( "mongosniff", "tools/sniffer.cpp",
                                              LIBDEPS = [
                                                 "db/serveronly",
                                              ] ) )

# --- shell ---

if not has_option('noshell') and usemozjs:
    shell_core_env = env.Clone()
    if has_option("safeshell"):
        shell_core_env.Append(CPPDEFINES=["MONGO_SAFE_SHELL"])
    shell_core_env.Library("shell_core",
                source=[
                    "shell/bench.cpp",
                    "shell/clientAndShell.cpp",
                    "shell/linenoise.cpp",
                    "shell/mk_wcwidth.cpp",
                    "shell/mongo-server.cpp",
                    "shell/shell_utils.cpp",
                    "shell/shell_utils_extended.cpp",
                    "shell/shell_utils_launcher.cpp",
                    "shell/shell_options_init.cpp"
                ],
                LIBDEPS=[
                    'db/catalog/index_key_validate',
                    'db/index/external_key_generator',
                    'db/query/command_request_response',
                    'db/query/lite_parsed_query',
                    'linenoise_utf8',
                    'scripting/scripting',
                    'shell/mongojs',
                    'util/processinfo',
                    'util/signal_handlers',
                ],
                LIBDEPS_TAGS=[
                    # Circular with shell_options below. It is unclear
                    # why this is split into two libraries, or even
                    # why these libraries exist at all, since it seems
                    # that the only thing that links them is the shell
                    # itself.
                    'incomplete',
                ],
    )

    # mongo shell options
    shell_core_env.Library("shell_options", ["shell/shell_options.cpp"],
                LIBDEPS=[
                    '$BUILD_DIR/mongo/db/server_options_core',
                    '$BUILD_DIR/mongo/rpc/protocol',
                    '$BUILD_DIR/mongo/util/net/network',
                    '$BUILD_DIR/mongo/util/options_parser/options_parser_init',
                    'shell_core',
                ])

    shellEnv = env.Clone()
    if env.TargetOSIs('windows'):
       shellEnv.Append(LIBS=["winmm.lib"])

    mongo_shell = shellEnv.Program(
        "mongo",
        "shell/dbshell.cpp",
        LIBDEPS=["$BUILD_DIR/third_party/shim_pcrecpp",
                 "shell_options",
                 "shell_core",
                 "db/server_options_core",
                 "client/clientdriver",
                 "$BUILD_DIR/mongo/util/password",
                 ])

    shellEnv.Install( '#/', mongo_shell )
else:
    shellEnv = None

#  ----  INSTALL -------

# binaries

distBinaries = []
distDebugSymbols = []

def add_exe( v ):
    return "${PROGPREFIX}%s${PROGSUFFIX}" % v

def failMissingObjCopy(env, target, source):
    env.FatalError("Generating debug symbols requires objcopy, please set the OBJCOPY variable.")

def installBinary( e, name ):
    debug_sym_name = name
    name = add_exe( name )

    debug_sym_cmd = None
    if e.TargetOSIs('linux', 'solaris'):
        if 'OBJCOPY' not in e:
            debug_sym_cmd = failMissingObjCopy
        else:
            debug_sym_cmd = '${OBJCOPY} --only-keep-debug ${SOURCE} ${TARGET}'
        debug_sym_name += '.debug'
    elif e.TargetOSIs('osx'):
        debug_sym_name += '.dSYM'
        debug_sym_cmd = 'dsymutil -o ${TARGET} ${SOURCE}'
    elif e.ToolchainIs('msvc'):
        debug_sym_name += '.pdb'
        distBinaries.append(debug_sym_name)
        distDebugSymbols.append(debug_sym_name)

    if debug_sym_cmd:
        debug_sym = e.Command(
            debug_sym_name,
            name,
            debug_sym_cmd
        )
        e.Install("#/", debug_sym)
        e.Alias('debugsymbols', debug_sym)
        distDebugSymbols.append(debug_sym)

    if env.TargetOSIs('linux', 'solaris') and (not has_option("nostrip")):
        strip_cmd = e.Command(
            'stripped/%s' % name,
            [name, debug_sym],
            '${OBJCOPY} --strip-debug --add-gnu-debuglink ${SOURCES[1]} ${SOURCES[0]} $TARGET'
        )
        distBinaries.append('stripped/%s' % name)
    else:
        distBinaries.append(name)

    inst = e.Install( "$INSTALL_DIR/bin", name )

    if env.TargetOSIs('posix'):
        e.AddPostAction( inst, 'chmod 755 $TARGET' )

def installExternalBinary( e, name_str ):
    name = env.File("#/%s" % add_exe(name_str))
    if not name.isfile():
        env.FatalError("ERROR: external binary not found: {0}", name)

    distBinaries.append(name)
    inst = e.Install( "$INSTALL_DIR/bin", name )

    if env.TargetOSIs('posix'):
        e.AddPostAction( inst, 'chmod 755 $TARGET' )


# "--use-new-tools" adds dependencies for rewritten (Go) tools
# It is required for "dist" but optional for "install"
if has_option("use-new-tools"):
    toolsRoot = "src/mongo-tools"
    for t in rewrittenTools:
        installExternalBinary(env, "%s/%s" % (toolsRoot, t))

# legacy tools
installBinary(env, "mongoperf")
env.Alias("tools", '#/' + add_exe("mongoperf"))

env.Alias("tools", "#/" + add_exe("mongobridge"))

if mongosniff_built:
    installBinary(env, "mongosniff")
    env.Alias("tools", '#/' + add_exe("mongosniff"))

installBinary( env, "mongod" )
installBinary( env, "mongos" )

if shellEnv is not None:
    installBinary( shellEnv, "mongo" )

env.Alias( "core", [ '#/%s' % b for b in [ add_exe( "mongo" ), add_exe( "mongod" ), add_exe( "mongos" ) ] ] )

# Stage the top-level mongodb banners
distsrc = env.Dir('#distsrc')
env.Append(MODULE_BANNERS = [distsrc.File('README'),
                             distsrc.File('THIRD-PARTY-NOTICES'),
                             distsrc.File('MPL-2')])

# If no module has introduced a file named LICENSE.txt, then inject the AGPL.
if sum(itertools.imap(lambda x: x.name == "LICENSE.txt", env['MODULE_BANNERS'])) == 0:
    env.Append(MODULE_BANNERS = [distsrc.File('GNU-AGPL-3.0')])

# All module banners get staged to the top level of the tarfile, so we
# need to fail if we are going to have a name collision.
module_banner_filenames = set([f.name for f in env['MODULE_BANNERS']])
if not len(module_banner_filenames) == len(env['MODULE_BANNERS']):
    # TODO: Be nice and identify conflicts in error.
    env.FatalError("ERROR: Filename conflicts exist in module banners.")

# Build a set of directories containing module banners, and use that
# to build a --transform option for each directory so that the files
# are tar'ed up to the proper location.
module_banner_dirs = set([Dir('#').rel_path(f.get_dir()) for f in env['MODULE_BANNERS']])
module_banner_transforms = ["--transform %s=$SERVER_DIST_BASENAME" % d for d in module_banner_dirs]

# Allow modules to map original file name directories to subdirectories 
# within the archive (e.g. { "src/mongo/db/modules/enterprise/docs": "snmp"})
archive_addition_transforms = []
for full_dir, archive_dir in env["ARCHIVE_ADDITION_DIR_MAP"].items():
  archive_addition_transforms.append("--transform \"%s=$SERVER_DIST_BASENAME/%s\"" %
                                     (full_dir, archive_dir))

for target in env["DIST_BINARIES"]:
    installBinary(env, "db/modules/" + target)

# "dist" target is valid only when --use-new-tools is specified
# Attempts to build release artifacts without tools must fail
if has_option("use-new-tools"):
    env.Command(
        target='#/${SERVER_ARCHIVE}',
        source=['#buildscripts/make_archive.py'] + env["MODULE_BANNERS"] + env["ARCHIVE_ADDITIONS"] + distBinaries,
        action=' '.join(
            ['$PYTHON ${SOURCES[0]} -o $TARGET'] +
            archive_addition_transforms +
            module_banner_transforms +
            [
                '--transform $BUILD_DIR/mongo/db/modules/enterprise=$SERVER_DIST_BASENAME/bin',
                '--transform $BUILD_DIR/mongo/stripped/db/modules/enterprise=$SERVER_DIST_BASENAME/bin',
                '--transform $BUILD_DIR/mongo/stripped=$SERVER_DIST_BASENAME/bin',
                '--transform $BUILD_DIR/mongo=$SERVER_DIST_BASENAME/bin',
                '--transform $BUILD_DIR/mongo/stripped/src/mongo-tools=$SERVER_DIST_BASENAME/bin',
                '--transform src/mongo-tools=$SERVER_DIST_BASENAME/bin',
                '${TEMPFILE(SOURCES[1:])}'
            ],
        ),
        BUILD_DIR=env.Dir('$BUILD_DIR').path
    )

    env.Alias("dist", source='#/${SERVER_ARCHIVE}')
else:
    def failDist(env, target, source):
        env.FatalError("ERROR: 'dist' target only valid with --use-new-tools.")
    env.Alias("dist", [], [ failDist ] )
    env.AlwaysBuild("dist")

debug_symbols_dist = env.Command(
    target='#/${SERVER_DIST_BASENAME}-debugsymbols${DIST_ARCHIVE_SUFFIX}',
    source=['#buildscripts/make_archive.py'] + distDebugSymbols,
    action=' '.join(
        [
            '$PYTHON ${SOURCES[0]} -o $TARGET',
            '--transform $BUILD_DIR/mongo/db/modules/enterprise=$SERVER_DIST_BASENAME',
            '--transform $BUILD_DIR/mongo=$SERVER_DIST_BASENAME',
            '${TEMPFILE(SOURCES[1:])}',
        ]
    ),
    BUILD_DIR=env.Dir('$BUILD_DIR').path
)
env.Alias('dist-debugsymbols', debug_symbols_dist)

#final alias
env.Alias( "install", "$INSTALL_DIR" )

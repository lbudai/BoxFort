project('boxfort', 'c',
		meson_version   : '>= 0.48.0',
		license         : 'MIT',
		version         : '0.0.1',
		default_options : ['c_std=c99', 'warning_level=3', 'b_lundef=false'])

# standard install directories
prefix    = get_option('prefix')

# Helper scripts
auxdir  = join_paths(meson.source_root(), 'ci')
isdir   = join_paths(auxdir, 'isdir.py')

python3 = find_program('python3')
git     = find_program('git', required: false)

# Get the right version

is_git_repo = run_command([isdir, '.git']).returncode() == 0

version = 'v' + meson.project_version()
if git.found() and is_git_repo
	version = run_command([git.path(), 'describe', '--dirty', '--tags']).stdout().strip()
	branch  = run_command([git.path(), 'rev-parse', '--abbrev-ref', 'HEAD']).stdout().strip()

	if branch != 'master'
		version = '@0@ (@1@)'.format(version, branch)
	endif
endif

cc = meson.get_compiler('c')

add_project_arguments(
	cc.get_supported_arguments([
		'-Wno-unused-parameter',
		'-Wno-unused-value',
		'-fvisibility=hidden',

		# MSVC-specific stuff
		'/SAFESEH:NO',
		'/source-charset:utf-8',
	]),
	'-DBXF_BUILDING_LIB',
	'-D_GNU_SOURCE',
	language: ['c', 'cpp'])

if host_machine.system() == 'windows'
	add_project_arguments(
			'-DVC_EXTRALEAN',
			'-DWIN32_LEAN_AND_MEAN',
			'-D_CRT_SECURE_NO_WARNINGS',
			'-D_WIN32_WINNT=0x600',
			language: ['c', 'cpp'])
endif

boxfort_includedir = include_directories(
	'include',
	'src',
)

threads = dependency('threads')

# optional platform-dependent standard libs
librt = cc.find_library('rt', required: false)
libm  = cc.find_library('m',  required: false)

config = configuration_data()
config.set('package', meson.project_name())
config.set('version', version)

os = host_machine.system()
arch = host_machine.cpu_family()

mangling = 'none'
if cc.symbols_have_underscore_prefix()
	mangling = 'leading-underscore'
endif

if arch == 'x86'
	bitness = 32
elif arch == 'x86_64'
	bitness = 64
elif arch == 'arm'
	bitness = 32
elif arch == 'aarch64'
	bitness = 64
else
	error('Architecture "@0@" is not supported.'.format(arch))
endif

binfmt = 'elf'
os_family = 'posix'
if os == 'windows'
	binfmt = 'pe'
	os_family = 'windows'
elif os == 'darwin'
	binfmt = 'mach-o'
endif

config.set('BXF_EXE_FORMAT', binfmt)
config.set('BXF_EXE_FMT_@0@'.format(binfmt.to_upper().underscorify()), 1)
config.set('BXF_OS_FAMILY', os_family)
config.set('BXF_ARCH', '"' + arch + '"')
config.set('BXF_ARCH_@0@'.format(arch), 1)
config.set('BXF_MANGLING', mangling)
config.set('BXF_BITS', bitness)

checks = [
	{'fn': 'clock_gettime'},
	{'fn': 'gettimeofday'},
	{'fn': 'mincore'},
	{'fn': 'prctl'},
	{'fn': 'shm_open'},

	# some platforms define mincore with an unsigned vector
	{
		'fn': 'mincore',
		'proto': 'int _(void *, size_t, unsigned char *)',
		'headers': ['unistd.h', 'sys/mman.h'],
		'var': 'HAVE_UNSIGNED_MINCORE_VEC',
	},

	{'sym': 'PR_SET_PDEATHSIG',    'header': 'sys/prctl.h'},
	{'sym': 'CLOCK_MONOTONIC_RAW', 'header': 'time.h'},
	{'sym': 'environ',             'header': 'unistd.h'},
]

if binfmt == 'elf'
	checks += [
		{'sym': '_r_debug', 'header': 'link.h'},
		{'sym': '_DYNAMIC', 'header': 'link.h'},
	]
endif

check_prelude = '''
#define _GNU_SOURCE
'''

prototype_check = check_prelude + '''
@0@
typedef @1@;
static _ *check = @2@;
'''

foreach check : checks
	chk_prefix = check.get('prefix', '')
	if chk_prefix != ''
		chk_prefix = chk_prefix + '_'
	endif

	result = false
	if check.has_key('proto')
		includes = ''
		foreach hdr : check.get('headers')
			includes += '#include <@0@>\n'.format(hdr)
		endforeach
		result = cc.compiles(prototype_check.format(includes, check.get('proto'), check.get('fn')))
	elif check.has_key('fn')
		name = check.get('fn')
		libs = ['c']
		if librt.found()
			libs += 'rt'
		endif
		if libm.found()
			libs += 'm'
		endif
		if check.has_key('libs')
			libs += check.get('libs')
		endif
		args = []
		foreach lib : libs
			args += '-l' + lib
		endforeach
		result = cc.has_function(name, prefix: check_prelude, args: args)
	elif check.has_key('sym')
		name = check.get('sym')
		result = cc.has_header_symbol(check.get('header'), name, prefix: check_prelude)
	elif check.has_key('hdr')
		name = check.get('hdr')
		result = cc.has_header(name)
	endif

	if check.has_key('var')
		name = check.get('var')
	else
		name = 'HAVE_@1@@0@'.format(name.to_upper(), chk_prefix.to_upper())
	endif
	config.set(name, result)
endforeach

api = files('include/boxfort.h')

install_headers(api)

includedir = include_directories(
	'include',
	'src',
)

if target_machine.system() == 'windows'
	config.set10('HAVE_WIN32_THREADS', true)
else
	config.set10('HAVE_PTHREADS', true)
endif

subdir('src')
subdir('sample')
subdir('test')

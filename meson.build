project(
	'wlroots',
	'c',
	version: '0.18.0-dev',
	license: 'MIT',
	meson_version: '>=0.59.0',
	default_options: [
		'c_std=c11',
		'warning_level=2',
		'werror=true',
	],
)

little_endian = target_machine.endian() == 'little'
big_endian = target_machine.endian() == 'big'

add_project_arguments([
	'-D_POSIX_C_SOURCE=200809L',
	'-DWLR_USE_UNSTABLE',
	'-DWLR_LITTLE_ENDIAN=@0@'.format(little_endian.to_int()),
	'-DWLR_BIG_ENDIAN=@0@'.format(big_endian.to_int()),
], language: 'c')

cc = meson.get_compiler('c')

add_project_arguments(cc.get_supported_arguments([
	'-Wundef',
	'-Wlogical-op',
	'-Wmissing-include-dirs',
	'-Wold-style-definition',
	'-Wpointer-arith',
	'-Winit-self',
	'-Wstrict-prototypes',
	'-Wimplicit-fallthrough=2',
	'-Wendif-labels',
	'-Wstrict-aliasing=2',
	'-Woverflow',
	'-Wmissing-prototypes',
	'-Walloca',

	'-Wno-missing-braces',
	'-Wno-missing-field-initializers',
	'-Wno-unused-parameter',
]), language: 'c')

# Compute the relative path used by compiler invocations.
source_root = meson.current_source_dir().split('/')
build_root = meson.global_build_root().split('/')
relative_dir_parts = []
i = 0
in_prefix = true
foreach p : build_root
	if i >= source_root.length() or not in_prefix or p != source_root[i]
		in_prefix = false
		relative_dir_parts += '..'
	endif
	i += 1
endforeach
i = 0
in_prefix = true
foreach p : source_root
	if i >= build_root.length() or not in_prefix or build_root[i] != p
		in_prefix = false
		relative_dir_parts += p
	endif
	i += 1
endforeach
relative_dir = join_paths(relative_dir_parts) + '/'

# Strip relative path prefixes from the code if possible, otherwise hide them.
if cc.has_argument('-fmacro-prefix-map=/prefix/to/hide=')
	add_project_arguments(
		'-fmacro-prefix-map=@0@='.format(relative_dir),
		language: 'c',
	)
else
	add_project_arguments(
		'-D_WLR_REL_SRC_DIR="@0@"'.format(relative_dir),
		language: 'c',
	)
endif

patching = run_command('./patches/apply.sh', capture: true, check: true)

features = {
	'drm-backend': false,
	'x11-backend': false,
	'libinput-backend': false,
	'xwayland': false,
	'gles2-renderer': false,
	'vulkan-renderer': false,
	'gbm-allocator': false,
	'session': false,
}
internal_features = {
	'xcb-errors': false,
	'egl': false,
}
internal_config = configuration_data()

wayland_project_options = ['tests=false', 'documentation=false']
wayland_server = dependency('wayland-server',
	version: '>=1.22',
	fallback: 'wayland',
	default_options: wayland_project_options,
)

drm = dependency('libdrm',
	version: '>=2.4.120',
	fallback: 'libdrm',
	default_options: [
		'intel=disabled',
		'radeon=disabled',
		'amdgpu=disabled',
		'nouveau=disabled',
		'vmwgfx=disabled',
		'omap=disabled',
		'exynos=disabled',
		'freedreno=disabled',
		'tegra=disabled',
		'vc4=disabled',
		'etnaviv=disabled',
		'cairo-tests=disabled',
		'man-pages=disabled',
		'valgrind=disabled',
		'tests=false',
	],
)
xkbcommon = dependency(
	'xkbcommon',
	fallback: 'libxkbcommon',
	default_options: [
		'enable-tools=false',
		'enable-x11=false',
		'enable-docs=false',
		'enable-xkbregistry=false',
	],
)
pixman = dependency('pixman-1',
	version: '>=0.42.0',
	fallback: 'pixman',
	default_options: ['werror=false'],
)
math = cc.find_library('m')
rt = cc.find_library('rt')

wlr_files = []
wlr_deps = [
	wayland_server,
	drm,
	xkbcommon,
	pixman,
	math,
	rt,
]

subdir('protocol')
subdir('render')

subdir('backend')
subdir('types')
subdir('util')
subdir('xcursor')
subdir('xwayland')

subdir('include')

wlr_inc = include_directories('include')

symbols_file = 'wlroots.syms'
symbols_flag = '-Wl,--version-script,@0@/@1@'.format(meson.current_source_dir(), symbols_file)
lib_wlr = static_library(
	meson.project_name(), wlr_files,
	dependencies: wlr_deps,
	include_directories: [wlr_inc],
	install: true,
	link_args: symbols_flag,
	link_depends: symbols_file,
)

wlr_vars = {}
foreach name, have : features
	wlr_vars += { 'have_' + name.underscorify(): have.to_string() }
endforeach

wlroots = declare_dependency(
	link_with: lib_wlr,
	dependencies: wlr_deps,
	include_directories: wlr_inc,
	variables: wlr_vars,
)

meson.override_dependency('wlroots', wlroots)

summary(features + internal_features, bool_yn: true)

if get_option('examples')
	subdir('examples')
	subdir('tinywl')
endif

pkg_install_dir = join_paths(get_option('datadir'), 'pkgconfig')
pkgconfig = import('pkgconfig')
pkgconfig.generate(
	lib_wlr,
	description: 'Wayland compositor library',
	install_dir: pkg_install_dir,
	url: 'https://gitlab.freedesktop.org/wlroots/wlroots',
	variables: wlr_vars,
)

message(patching.stdout().strip())

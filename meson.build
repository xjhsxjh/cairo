project('cairo', 'c', 'cpp',
  meson_version: '>= 0.48.0',
  version: '1.15.12',
)

cc = meson.get_compiler('c')

if cc.get_id() == 'msvc'
  # Basic usage in the cairo type system that causes spammy and useless warnings
  add_project_arguments('/wd4244', '/wd4146',
                        # Don't warn about double -> float truncation
                        '/wd4305',
                        language : 'c')
endif

pkgmod = import('pkgconfig')
python3 = import('python').find_installation()

check_sizeofs = [
  ['void *', {'conf-name': 'SIZEOF_VOID_P'}],
  ['int'],
  ['long'],
  ['long long'],
  ['size_t'],
]

check_headers = [
  ['stdint.h'],
  ['inttypes.h'],
  ['sys/int_types.h'],
  ['fcntl.h'],
  ['unistd.h'],
  ['signal.h'],
  ['sys/stat.h'],
  ['sys/socket.h'],
  ['sys/poll.h'],
  ['sys/un.h'],
  ['sched.h', {'check-funcs': ['sched_getaffinity']}],
  ['sys/mman.h', {'check-funcs': ['mmap']}],
  ['time.h', {'check-funcs': ['clock_gettime']}],
  ['libgen.h'],
  ['byteswap.h'],
  ['signal.h'],
  ['setjmp.h'],
  ['fenv.h'],
  ['sys/wait.h'],
  ['sys/stat.h'],
  ['io.h'],
  ['fenv.h', {'check-funcs': ['feenableexcept', 'fedisableexcept', 'feclearexcept']}],
  ['xlocale.h'],
  ['sys/ioctl.h'],
]

check_types = [
  ['uint64_t', {'headers': ['stdint.h']}],
  ['uint128_t', {'headers': ['stdint.h']}],
  ['__uint128_t']
]

check_funcs = [
  'alarm',
  'ctime_r',
  'localtime_r',
  'gmtime_r',
  'drand48',
  'flockfile',
  'funlockfile',
  'getline',
  'link',
  'strndup',
  'fork',
  'waitpid',
  'raise',
  'newlocale',
  'strtod_l',
]

check_thread_flags = [
  [['-D_REENTRANT'], ['-lpthread']],
  [['-pthread'], []],
  [['-D_REENTRANT'], [], {'real': false}],
]

m_dep = cc.find_library('libm', required: false)
# Used in util
gtk_dep = dependency('gtk+-2.0', required: get_option('gtk2-utils'))

deps = [m_dep]
test_deps = []
internal_deps = []
extra_link_args = []

conf = configuration_data()

lzo_dep = dependency('lzo2', required: false)
if lzo_dep.found()
  deps += [lzo_dep]
  conf.set('HAVE_LZO', 1)
endif

dl_dep = cc.find_library('dl', required: false)
if dl_dep.found() and cc.has_function('dlsym', dependencies: [dl_dep])
  deps += [dl_dep]
  conf.set('CAIRO_HAS_DLSYM', 1)
elif cc.has_function('dlsym')
  conf.set('CAIRO_HAS_DLSYM', 1)
elif cc.has_function('dlsym', prefix: '#include <dlfcn.h>')
  conf.set('CAIRO_HAS_DLSYM', 1)
endif

feature_conf = configuration_data()

built_features = []

zlib_dep = dependency('zlib',
  required: get_option('zlib'),
  fallback : ['zlib', 'zlib_dep'],
)
if zlib_dep.found()
  if zlib_dep.type_name() == 'internal'
    internal_deps += [zlib_dep]
  else
    deps += [zlib_dep]
  endif
  conf.set('HAVE_ZLIB', 1)
endif

png_dep = dependency('libpng',
  required: get_option('png'),
  fallback: ['libpng', 'png_dep']
)
if png_dep.found()
  feature_conf.set('CAIRO_HAS_SVG_SURFACE', 1)
  feature_conf.set('CAIRO_HAS_PNG_FUNCTIONS', 1)
  built_features += [
    {
      'name': 'cairo-png',
      'description': 'PNG functions',
      'requires': 'libpng',
    },
    {
      'name': 'cairo-svg',
      'description': 'SVG surface backend',
      'requires': 'libpng',
    }
  ]

  if png_dep.type_name() == 'internal'
    internal_deps += [png_dep]
  else
    deps += [png_dep]
  endif
endif

freetype_required_version = '>= 9.7.3'

freetype_dep = dependency('freetype2',
  required: get_option('freetype'),
  version: freetype_required_version,
  fallback: ['freetype2', 'freetype_dep'],
)
if freetype_dep.found()
  feature_conf.set('CAIRO_HAS_FT_FONT', 1)
  built_features += [{
    'name': 'cairo-ft',
    'description': 'Freetype font backend',
    'requires': 'freetype2 @0@'.format(freetype_required_version)
  }]

  ft_check_funcs = [
    'FT_Get_X11_Font_Format',
    'FT_GlyphSlot_Embolden',
    'FT_GlyphSlot_Oblique',
    'FT_Load_Sfnt_Table',
    'FT_Library_SetLcdFilter',
    'FT_Get_Var_Design_Coordinates',
    'FT_Done_MM_Var',
  ]

  if freetype_dep.type_name() == 'internal'
    foreach func : ft_check_funcs
      conf.set('HAVE_@0@'.format(func.to_upper()), 1)
    endforeach
    internal_deps += [freetype_dep]
  else
    if not cc.links(files('meson-cc-tests/ft_has_color.c'), dependencies: freetype_dep, name: 'FT has color')
      conf.set('FT_HAS_COLOR', '(0)')
    endif
    check_funcs += ft_check_funcs
    deps += [freetype_dep]
  endif
endif

fontconfig_required_version = '>= 2.2.95'
fontconfig_dep = dependency('fontconfig',
  required: get_option('fontconfig'),
  version: fontconfig_required_version,
  fallback: ['fontconfig', 'fontconfig_dep'],
)
if fontconfig_dep.found()
  fc_check_funcs = [
    'FcInit',
    'FcFini'
  ]

  if fontconfig_dep.type_name() == 'internal'
    foreach func : fc_check_funcs
      conf.set('HAVE_@0@'.format(func.to_upper()), 1)
    endforeach
    internal_deps += [fontconfig_dep]
  else
    check_funcs += fc_check_funcs
    deps += [fontconfig_dep]
  endif

  feature_conf.set('CAIRO_HAS_FC_FONT', 1)
  built_features += [{
    'name': 'cairo-fc',
    'description': 'Fontconfig font backend',
    'requires': 'fontconfig @0@'.format(fontconfig_required_version)
  }]
endif

x11_dep = dependency('x11', required: get_option('xlib'))
xext_dep = dependency('xext', required: get_option('xlib'))
if x11_dep.found() and xext_dep.found()
  feature_conf.set('CAIRO_HAS_XLIB_SURFACE', 1)
  built_features += [{
    'name': 'cairo-xlib',
    'description': 'Xlib surface backend',
    'requires': 'x11 xext',
  }]

  extra_headers = ['X11/Xlibint.h', 'X11/Xproto.h']
  check_headers += [
    ['X11/extensions/XShm.h', {'extra-headers': extra_headers}],
    ['X11/extensions/shmproto.h', {'extra-headers': extra_headers}],
    ['X11/extensions/shmstr.h', {'extra-headers': extra_headers}],
  ]
  deps += [x11_dep, xext_dep]
  res = cc.run(files('meson-cc-tests/ipc_rmid_deferred_release.c'),
    dependencies: [x11_dep, xext_dep],
    name: 'shmctl IPC_RMID allowes subsequent attaches')

  if res.returncode() == 0
    conf.set('IPC_RMID_DEFERRED_RELEASE', 1)
  endif
endif

if feature_conf.get('CAIRO_HAS_XLIB_SURFACE', 0) == 1
  xrender_required_version = '>= 0.6'
  xrender_dep = dependency('xrender', required: get_option('xlib'),
                           version: xrender_required_version)

  if xrender_dep.found()
    check_funcs += [
      'XRenderCreateSolidFill',
      'XRenderCreateLinearGradient',
      'XRenderCreateRadialGradient',
      'XRenderCreateConicalGradient',
    ]

    deps += [xrender_dep]

    built_features += [{
      'name': 'cairo-xlib-xrender',
      'description': 'Xlib Xrender surface backend',
      'requires': 'xrender @0@'.format(xrender_required_version),
    }]
    feature_conf.set('CAIRO_HAS_XLIB_XRENDER_SURFACE', 1)
  endif
endif

xcb_required_version = '>= 1.6'
xcb_render_required_version = '>= 1.6'

xcb_dep = dependency('xcb', required: get_option('xcb'),
                     version: xcb_required_version)
xcb_render_dep = dependency('xcb-render', required: get_option('xcb'),
                            version: xcb_render_required_version)
if xcb_dep.found() and xcb_render_dep.found()
  feature_conf.set('CAIRO_HAS_XCB_SURFACE', 1)
  built_features += [{
    'name': 'cairo-xcb',
    'description': 'XCB surface backend',
    'requires': 'xcb @0@ xcb-render @1@'.format(xcb_required_version, xcb_render_required_version),
  }]

  deps += [xcb_dep, xcb_render_dep]
endif

if feature_conf.get('CAIRO_HAS_XCB_SURFACE', 0) == 1 and feature_conf.get('CAIRO_HAS_XLIB_SURFACE', 0) == 1
  # FIXME: automagic
  x11xcb_dep = dependency('x11-xcb', required: false)
  if x11xcb_dep.found()
    deps += [x11xcb_dep]
    feature_conf.set('CAIRO_HAS_XLIB_XCB_FUNCTIONS', 1)
    built_features += [{
      'name': 'cairo-xlib-xcb',
      'description': 'Xlib/XCB functions',
      'requires': 'x11-xcb',
    }]
  endif
endif

if feature_conf.get('CAIRO_HAS_XCB_SURFACE', 0) == 1
  xcbshm_dep = dependency('xcb-shm', required: get_option('xcb'))
  if xcbshm_dep.found()
    feature_conf.set('CAIRO_HAS_XCB_SHM_FUNCTIONS', 1)
    deps += [xcbshm_dep]
    built_features += [{
      'name': 'cairo-xcb-shm',
      'description': 'XCB/SHM functions',
      'requires': 'xcb-shm',
    }]
  endif
endif

# Tests crash and burn
qtgui_required_version = '>= 4.4.0'
qtgui_dep = dependency('QtGui', required: get_option('qt'),
                       version: qtgui_required_version)
if qtgui_dep.found() and false
  feature_conf.set('CAIRO_HAS_QT_SURFACE', 1)
  built_features += [{
    'name': 'cairo-qt',
    'description': 'Qt surface backend',
    'requires': 'QtGui @0@'.format(qtgui_required_version),
    'libs': qtgui_extra_libs,
  }]

  deps += [qtgui_dep]
  qtgui_extra_libs = ['-lstdc++']
  extra_link_args += qtgui_extra_libs
endif

# Untested
if cc.has_header('ApplicationServices/ApplicationServices.h')
  if cc.has_header('CoreGraphics/CoreGraphics.h')
    quartz_extra_libs = ['-Xlinker', '-framework', '-Xlinker', 'CoreGraphics']
  else
    quartz_extra_libs = ['-Xlinker', '-framework', '-Xlinker', 'ApplicationServices']
  endif

  extra_link_args += quartz_extra_libs

  feature_conf.set('CAIRO_HAS_QUARTZ_SURFACE', 1)
  feature_conf.set('CAIRO_HAS_QUARTZ_FONT', 1)
  feature_conf.set('CAIRO_HAS_QUARTZ_IMAGE_SURFACE', 1)

  built_features += [
    {
      'name': 'cairo-quartz',
      'description': 'Quartz surface backend',
      'libs': quartz_extra_libs,
    },
    {
      'name': 'cairo-quartz-image',
      'description': 'Quartz Image surface backend',
      'libs': quartz_extra_libs,
    },
    {
      'name': 'cairo-quartz-font',
      'description': 'Quartz font backend',
      'libs': quartz_extra_libs,
    },
  ]
endif

if host_machine.system() == 'windows'
  win32_extra_libs = ['gdi32', 'msimg32']
  foreach libname: win32_extra_libs
    deps += [cc.find_library(libname)]
  endforeach

  feature_conf.set('CAIRO_HAS_WIN32_SURFACE', 1)
  built_features += [{
    'name': 'cairo-win32',
    'description': 'Microsoft Windows surface backend',
    'libs': win32_extra_libs,
  }]
endif

if feature_conf.get('CAIRO_HAS_WIN32_SURFACE', 0) == 1
  feature_conf.set('CAIRO_HAS_WIN32_FONT', 1)
  built_features += [{
    'name': 'cairo-win32-font',
    'description': 'Microsoft Windows font backend',
    'libs': win32_extra_libs,
  }]
endif

# Doesn't build since at least 2011
libudev_required_version = '>= 136'
libdrm_required_version = '>= 2.4'
libudev_dep = dependency('libudev', required: get_option('drm'),
                         version: libudev_required_version)
libdrm_dep = dependency('libdrm', required: get_option('drm'),
                        version: libdrm_required_version)
if libudev_dep.found() and libdrm_dep.found() and false
  deps += [libudev_dep, libdrm_dep]

  feature_conf.set('CAIRO_HAS_DRM_SURFACE', 1)
  built_features += [{
    'name': 'cairo-drm',
    'description': 'DRM surface backend',
    'requires': 'libudev @0@ libdrm @1@'.format(libudev_required_version, libdrm_required_version),
  }]
endif

# Not even trying to port gallium as it depends on drm

# Not porting BEOS or OS/2 backends either, meson doesn't support those

# GL / GLESV2 / GLESV3 are mutually exclusive
gl_backend = get_option('gl-backend')
use_gl = false
need_egl_functions = false
need_wgl_functions = false
need_glx_functions = false

if ['auto', 'gl'].contains(gl_backend)
  gl_dep = dependency('gl', required: gl_backend == 'gl')
  if not gl_dep.found()
    if cc.has_header('GL/gl.h')
      gl_extra_libs = ['-lGL']
      gl_requires = ''
      extra_link_args += gl_extra_libs
      use_gl = true
    endif
  elif cc.has_header('GL/gl.h', dependencies: gl_dep) and cc.has_header('GL/glext.h', dependencies: gl_dep)
    use_gl = true
    gl_extra_libs = []
    gl_requires = 'gl'
    deps += [gl_dep]
  endif
endif


if use_gl
  need_egl_functions = true
  need_wgl_functions = true
  need_glx_functions = true

  feature_conf.set('CAIRO_HAS_GL_SURFACE', 1)
  built_features += [{
    'name': 'cairo-gl',
    'description': 'OpenGL surface backend',
    'requires': gl_requires,
    'libs': gl_extra_libs,
  }]
endif

if feature_conf.get('CAIRO_HAS_GL_SURFACE', 0) == 0 and ['auto', 'glesv2'].contains(gl_backend)
  use_glesv2 = false
  glesv2_dep = dependency('glesv2', required: gl_backend == 'glesv2')
  if not glesv2_dep.found()
    if cc.has_header('GLES2/gl2.h')
      use_glesv2 = true
      glesv2_extra_libs = ['-lGLESv2']
      glesv2_requires = ''
      extra_link_args += glesv2_extra_libs
    endif
  else
    glesv2_extra_libs = []
    glesv2_requires = 'glesv2'
    use_glesv2 = true
    deps += [glesv2_dep]
  endif

  if use_glesv2
    need_egl_functions = true

    feature_conf.set('CAIRO_HAS_GLESV2_SURFACE', 1)
    built_features += [{
      'name': 'cairo-glesv2',
      'source-key': 'cairo-gl',
      'description': 'OpenGLESv2 surface backend',
      'requires': glesv2_requires,
      'libs': glesv2_extra_libs,
    }]
  endif
endif

if feature_conf.get('CAIRO_HAS_GL_SURFACE', 0) == 0 and feature_conf.get('CAIRO_HAS_GLESV2_SURFACE', 0) == 0 and ['auto', 'glesv3'].contains(gl_backend)
  use_glesv3 = false
  # glesv3 is provided via libGLESv2.so (there is no libGLESv3, nor glesv3.pc)
  glesv3_dep = dependency('glesv2', required: gl_backend == 'glesv3')
  if cc.has_header('GLES3/gl3.h', dependencies: glesv3_dep)
    use_glesv3 = true
    if not glesv3_dep.found()
      glesv3_extra_libs = ['-lGLESv2']
      extra_link_args += glesv3_extra_libs
      glesv3_requires = ''
    else
      glesv3_extra_libs = []
      glesv3_requires = 'glesv2'
      deps += [glesv3_dep]
    endif
  endif

  if use_glesv3
    need_egl_functions = true

    feature_conf.set('CAIRO_HAS_GLESV3_SURFACE', 1)
    built_features += [{
      'name': 'cairo-glesv3',
      'source-key': 'cairo-gl',
      'description': 'OpenGLESv3 surface backend',
      'requires': glesv3_requires,
      'libs': glesv3_extra_libs,
    }]
  endif
endif

# Broken for me, depends on experimental API
cogl_dep = dependency('cogl-2.0-experimental', required: get_option('cogl'))
if cogl_dep.found() and false
  deps += [cogl_dep]

  feature_conf.set('CAIRO_HAS_COGL_SURFACE', 1)
  built_features += [{
    'name': 'cairo-cogl',
    'description': 'Cogl surface backend',
    'requires': 'cogl-2.0-experimental',
  }]
endif

# Untested
directfb_dep = dependency('directfb', required: get_option('directfb'))
if directfb_dep.found()
  deps += [directfb_dep]

  feature_conf.set('CAIRO_HAS_DIRECTFB_SURFACE', 1)
  built_features += [{
    'name': 'cairo-directfb',
    'description': 'directfb surface backend',
    'requires': 'directfb',
  }]
endif

# Untested
openvg_option = get_option('openvg')
if not openvg_option.disabled()
  if cc.has_header('VG/openvg.h')
    openvg_extra_libs = ['-lOpenVG']
    extra_link_args += openvg_extra_libs
    need_egl_functions = true
    need_glx_functions = true

    feature_conf.set('CAIRO_HAS_VG_SURFACE', 1)
    built_features += [{
      'name': 'cairo-vg',
      'description': 'OpenVG surface backend',
      'libs': openvg_extra_libs,
    }]
  elif openvg_option.enabled()
    error('openvg surface backend enabled, but VG/openvg.h not found')
  endif
endif

if need_egl_functions
  # FIXME: automagic
  egl_dep = dependency('egl', required: false)
  use_egl = false
  egl_extra_libs = []
  egl_requires = ''

  if not egl_dep.found()
    if cc.has_header('EGL/egl.h')
      egl_extra_deps = []
      csi_dep = cc.find_library('csi', required: false)
      if csi_dep.found() and cc.has_function('csi_stream_attachresource', dependencies: [csi_dep])
        egl_extra_deps = [csi_dep]
        egl_extra_libs = ['-lcsi']
      endif

      foreach libname : ['EGL', 'egl13', 'egl12', 'egl11']
        if not use_egl
          dep = cc.find_library(libname, required: false)
          if dep.found() and cc.has_function('eglGetError', dependencies: [dep])
            deps += [dep] + egl_extra_deps
            egl_extra_libs += ['-l@0@'.format(libname)]
            use_egl = true
          endif
        endif
      endforeach
    endif
  else
    use_egl = true
    egl_requires = 'egl'
    deps += [egl_dep]
  endif

  if use_egl
    feature_conf.set('CAIRO_HAS_EGL_FUNCTIONS', 1)
    built_features += [{
      'name': 'cairo-egl',
      'description': 'EGL functions',
      'libs': egl_extra_libs,
      'requires': egl_requires,
    }]
  endif
endif

if need_glx_functions
  # FIXME: automagic
  if cc.has_header('GL/glx.h')
    feature_conf.set('CAIRO_HAS_GLX_FUNCTIONS', 1)
    built_features += [{
      'name': 'cairo-glx',
      'description': 'GLX functions',
      'libs': ['-lGL'],
    }]
  endif
endif

# Untested
if need_wgl_functions
  # FIXME: automagic
  if cc.has_header('windows.h')
    feature_conf.set('CAIRO_HAS_WGL_FUNCTIONS', 1)
    built_features += [{
      'name': 'cairo-wgl',
      'description': 'WGL functions',
    }]
  endif
endif

gobject_dep = dependency('gobject-2.0',
  required: get_option('glib'),
  fallback: ['glib', 'libgobject_dep']
)
glib_required_version = '>= 2.14'
glib_dep = dependency('glib-2.0',
  required: get_option('glib'),
  version: glib_required_version,
  fallback: ['glib', 'libglib_dep'],
)
if gobject_dep.found() and glib_dep.found()
  feature_conf.set('CAIRO_HAS_GOBJECT_FUNCTIONS', 1)
  built_features += [{
    'name': 'cairo-gobject',
    'description': 'gobject functions',
    'requires': 'gobject-2.0 glib-2.0 @0@'.format(glib_required_version),
    'libs': ['-lcairo-gobject'],
  }]
endif

if zlib_dep.found()
  feature_conf.set('CAIRO_HAS_SCRIPT_SURFACE', 1)
  built_features += [{
    'name': 'cairo-script',
    'description': 'script surface backend',
    'requires': 'zlib',
  }]
endif

if zlib_dep.found()
  feature_conf.set('CAIRO_HAS_PS_SURFACE', 1)
  built_features += [{
    'name': 'cairo-ps',
    'description': 'PostScript surface backend',
    'requires': 'zlib',
  }]
endif

if zlib_dep.found()
  feature_conf.set('CAIRO_HAS_PDF_SURFACE', 1)
  built_features += [{
    'name': 'cairo-pdf',
    'description': 'PDF surface backend',
    'requires': 'zlib',
  }]
endif

if zlib_dep.found()
  feature_conf.set('CAIRO_HAS_INTERPRETER', 1)
endif

if zlib_dep.found()
  feature_conf.set('CAIRO_HAS_XML_SURFACE', 1)
  built_features += [{
    'name': 'cairo-xml',
    'description': 'XML surface backend',
    'requires': 'zlib',
  }]
endif

# Untested, libiberty.h is in a libiberty subfolder for me
# FIXME: automagic
bfd_dep = cc.find_library('bfd', required: false)
if bfd_dep.found() and cc.has_function('bfd_openr', dependencies: [bfd_dep])
  if cc.has_header('libiberty.h')
    conf.set('HAVE_BFD', 1)
    deps += [bfd_dep]
  endif
endif

# Untested, see above
if conf.get('HAVE_BFD', 0) == 1
  conf.set('CAIRO_HAS_SYMBOL_LOOKUP', 1)
endif

if feature_conf.get('CAIRO_HAS_PS_SURFACE', 0) == 1
  gs = find_program('gs', required: get_option('tests'))
  libspectre_dep = dependency('libspectre', version: '>= 0.2.0',
                              required: get_option('spectre'))
  if gs.found() and libspectre_dep.found()
    conf.set('CAIRO_CAN_TEST_PS_SURFACE', 1)
  endif

  if libspectre_dep.found()
    conf.set('CAIRO_HAS_SPECTRE', 1)
    deps += [libspectre_dep]
  endif
endif

if feature_conf.get('CAIRO_HAS_PDF_SURFACE', 0) == 1
  poppler_dep = dependency('poppler-glib', version: '>= 0.17.4',
                           required: get_option('tests'))
  if poppler_dep.found() and cc.has_function('poppler_page_render', dependencies: [poppler_dep])
    conf.set('CAIRO_CAN_TEST_PDF_SURFACE', 1)
    test_deps += [poppler_dep]
  endif
endif

if feature_conf.get('CAIRO_HAS_SVG_SURFACE', 0) == 1
  librsvg_dep = dependency('librsvg-2.0', version: '>= 2.35.0',
                           fallback: ['librsvg', 'librsvg_dep'],
                           required: get_option('tests'))
  if librsvg_dep.found()
    conf.set('CAIRO_CAN_TEST_SVG_SURFACE', 1)
    test_deps += [librsvg_dep]
  endif
endif

pixman_dep = dependency('pixman-1',
  required: get_option('image'),
  version: '>= 0.30.0',
  fallback: ['pixman', 'pixman_dep'],
)
if pixman_dep.found()
  feature_conf.set('CAIRO_HAS_IMAGE_SURFACE', 1)
  conf.set('HAS_PIXMAN_GLYPHS', 1)
  if pixman_dep.type_name() == 'internal'
    internal_deps += [pixman_dep]
  else
    deps += [pixman_dep]
  endif
endif

feature_conf.set('CAIRO_FEATURES_H', true)
feature_conf.set('CAIRO_HAS_USER_FONT', 1)

feature_conf.set('CAIRO_HAS_MIME_SURFACE', 1)
feature_conf.set('CAIRO_HAS_RECORDING_SURFACE', 1)
feature_conf.set('CAIRO_HAS_OBSERVER_SURFACE', 1)

feature_conf.set('CAIRO_HAS_TEE_SURFACE', 1)
built_features += [{
  'name': 'cairo-tee',
  'description': 'Tee surface backend',
}]

incbase = include_directories('.')

foreach check : check_sizeofs
  type = check[0]
  opts = check.length() > 1 ? check[1] : {}

  conf_name = opts.get('conf-name', 'SIZEOF_@0@'.format(type.underscorify().to_upper()))

  conf.set(conf_name, cc.sizeof(type))
endforeach

foreach check : check_headers
  name = check[0]
  opts = check.length() > 1 ? check[1] : {}
  prefix = ''

  foreach header : opts.get('extra-headers', [])
    prefix += '#include <@0@>\n'.format(header)
  endforeach

  if cc.has_header(name, prefix: prefix)
    conf.set('HAVE_@0@'.format(name.to_upper().underscorify()), 1)
    check_funcs += check.length() > 1 ? check[1].get('check-funcs', []) : []
  endif
endforeach

foreach check : check_types
  name = check[0]
  opts = check.length() > 1 ? check[1] : {}
  prefix = ''

  foreach header : opts.get('headers', [])
    prefix += '#include <@0@>\n'.format(header)
  endforeach

  if cc.has_type(name, prefix: prefix)
    conf.set('HAVE_@0@'.format(name.to_upper()), 1)
  endif
endforeach

foreach name : check_funcs
  if cc.has_function(name, dependencies: deps)
    conf.set('HAVE_@0@'.format(name.to_upper()), 1)
  endif
endforeach

pthread_c_args = []
pthread_link_args = []

foreach thread_flags : check_thread_flags
  if not conf.has('CAIRO_HAS_PTHREAD')
    cflags = thread_flags[0]
    lflags = thread_flags[1]
    real_pthread = thread_flags.length() > 2 ? thread_flags[2].get('real', true) : true

    if cc.links(files('meson-cc-tests/pthread.c'), args: cflags + lflags, name: 'pthreads')
      conf.set('CAIRO_HAS_PTHREAD', 1)
      if real_pthread
        conf.set('CAIRO_HAS_REAL_PTHREAD', 1)
      endif
      pthread_c_args = cflags
      pthread_link_args = lflags
    endif
  endif
endforeach

extra_link_args += pthread_link_args

if cc.links(files('meson-cc-tests/atomic-ops-cxx11.c'), name: 'Atomic ops: cxx11')
  conf.set('HAVE_CXX11_ATOMIC_PRIMITIVES', 1)
elif cc.links(files('meson-cc-tests/atomic-ops-gcc-legacy.c'), name: 'Atomic ops: gcc legacy')
  conf.set('HAVE_GCC_LEGACY_ATOMICS', 1)
elif cc.has_header('atomic_ops.h')
  conf.set('HAVE_LIB_ATOMIC_OPS', 1)
elif cc.has_header('libkern/OSAtomic.h')
  conf.set('HAVE_OS_ATOMIC_OPS', 1)
endif

test_mkdir_c_args = []
if conf.get('HAVE_SYS_STAT_H', 0) == 1
  test_mkdir_c_args += ['-DHAVE_SYS_STAT_H']
endif

if conf.get('HAVE_IO_H', 0) == 1
  test_mkdir_c_args += ['-DHAVE_IO_H']
endif

if cc.links(files('meson-cc-tests/mkdir-variant-1.c'), args: test_mkdir_c_args)
  conf.set('HAVE_MKDIR', 1)
elif cc.links(files('meson-cc-tests/mkdir-variant-2.c'), args: test_mkdir_c_args)
  conf.set('HAVE_MKDIR', 2)
else
  conf.set('HAVE_MKDIR', 0)
endif

if not ['x86', 'x86_64'].contains(host_machine.cpu_family())
  conf.set('ATOMIC_OP_NEEDS_MEMORY_BARRIER', 1)
endif

have_ld_preload = ['linux', 'freebsd', 'darwin', 'dragonfly'].contains(host_machine.system())

if have_ld_preload and zlib_dep.found() and conf.get('CAIRO_HAS_REAL_PTHREAD', 0) == 1 and conf.get('CAIRO_HAS_DLSYM', 0) == 1
  conf.set('CAIRO_HAS_TRACE', 1)
endif

rt_dep = cc.find_library('rt', required: false)
have_shm = false
if rt_dep.found() and cc.has_function('shm_open', dependencies: [rt_dep])
  have_shm = true
endif

# This to make sure we don't run checks against internal deps
deps += internal_deps

subdir('src')

incboilerplate = include_directories('boilerplate')
if feature_conf.get('CAIRO_HAS_PNG_FUNCTIONS', 0) == 1
  subdir('boilerplate')
else
  libcairoboilerplate = []
endif

subdir('util')

if feature_conf.get('CAIRO_HAS_PNG_FUNCTIONS', 0) == 1
  subdir('test')
endif

configure_file(output: 'config.h', configuration: conf)

foreach feature: built_features
  fconf = configuration_data()
  fconf.set('prefix', get_option('prefix'))
  fconf.set('exec_prefix', '${prefix}')
  fconf.set('libdir', '${exec_prefix}/@0@'.format(get_option('libdir')))
  fconf.set('includedir', '${prefix}/@0@'.format(get_option('includedir')))
  fconf.set('FEATURE_PC', feature['name'])
  fconf.set('FEATURE_NAME', feature['description'])
  fconf.set('VERSION', meson.project_version())
  fconf.set('FEATURE_BASE', meson.project_name())
  fconf.set('FEATURE_REQUIRES', feature.get('requires', ''))
  fconf.set('FEATURE_NONPKGCONFIG_LIBS', ' '.join(feature.get('libs', [])))
  fconf.set('FEATURE_NONPKGCONFIG_EXTRA_LIBS', '')
  fconf.set('FEATURE_NONPKGCONFIG_CFLAGS', '')

  configure_file(input: 'src/cairo-features.pc.in',
    output: '@0@.pc'.format(feature['name']),
    configuration: fconf,
    install: true,
    install_dir: join_paths(get_option('prefix'), get_option('libdir'), 'pkgconfig')
  )
endforeach

install_headers('cairo-version.h', subdir: 'cairo')

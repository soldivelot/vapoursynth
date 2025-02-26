AC_INIT([vapoursynth], [61], [https://github.com/vapoursynth/vapoursynth/issues], [vapoursynth], [http://www.vapoursynth.com/])

: ${CFLAGS=""}
: ${CXXFLAGS=""}

AM_INIT_AUTOMAKE([foreign no-dist-gzip dist-xz subdir-objects no-define])
AM_SILENT_RULES([yes])

LT_INIT([win32-dll])

AC_PROG_CC
AC_PROG_CXX

AC_CONFIG_MACRO_DIRS([m4])

AC_SYS_LARGEFILE
AC_FUNC_FSEEKO
AX_PTHREAD


AC_ARG_ENABLE([debug], AS_HELP_STRING([--enable-debug], [Enable compilation options required for debugging. (default=no)]))
AS_IF(
      [test "x$enable_debug" = "xyes"],
      [
       AC_DEFINE([VS_CORE_DEBUG])
       AC_SUBST([DEBUGCFLAGS], ["-O0 -g3 -ggdb -ftrapv"])
      ],
      [
       AC_DEFINE([NDEBUG])
      ]
)



AC_ARG_ENABLE([guard-pattern], AS_HELP_STRING([--enable-guard-pattern], [Adds 32 bytes on the left and the right sides of each frame, fills them with a certain value, and checks their integrity after each filter. It can be used to detect buggy filters that write a little outside the frame.]))
AS_IF(
      [test "x$enable_guard_pattern" = "xyes"],
      [
       AC_DEFINE([VS_FRAME_GUARD])
      ]
)



AC_ARG_WITH(
            [plugindir],
            AS_HELP_STRING([--with-plugindir], [The default value for the configuration option 'SystemPluginDir' in vapoursynth.conf. (default=LIBDIR/vapoursynth)]),
            [PLUGINDIR=$with_plugindir],
            [PLUGINDIR=$libdir/$PACKAGE_NAME]
)
dnl Can't do it with AC_DEFINE because $libdir wouldn't be expanded fully.
AC_SUBST([PLUGINDIR])



AC_CANONICAL_HOST


X86="false"
PPC="false"
ARM="false"

AS_CASE(
        [$host_cpu],
        [i?86],     [BITS="32" X86="true"],
        [x86_64],   [BITS="64" X86="true"],
        [powerpc*], [PPC="true"],
        [arm*],     [ARM="true"], # Maybe doesn't work for all arm systems?
        [aarch64*], [ARM="true"]
)

AS_CASE(
        [$host_os],
        [darwin*],
        [
         AC_DEFINE([VS_TARGET_OS_DARWIN])
         AC_SUBST([UNDEFINEDLDFLAGS], ["-Wl,-undefined,error"])
        ],
        [*linux*|gnu*|dragonfly*|*bsd*], # The BSDs are close enough, right?
        [
         AC_DEFINE([VS_TARGET_OS_LINUX])
         AC_SUBST([UNDEFINEDLDFLAGS], ["-Wl,--no-undefined"])
        ],
        [cygwin*|mingw*],
        [
         AS_IF(
               [test "x$BITS" = "x32"],
               [
                AC_SUBST([PLUGINLDFLAGS], ["-Wl,--kill-at"])
                AC_SUBST([STACKREALIGN], ["-mstackrealign"])
               ]
         )
         AC_DEFINE([VS_TARGET_OS_WINDOWS])
         AC_SUBST([UNICODECFLAGS], ["-DUNICODE -D_UNICODE"])
         AC_SUBST([UNICODELDFLAGS], ["-municode"])
        ],
        [AC_MSG_ERROR([Unknown host OS])]
)

AS_IF(
      [test "x$X86" = "xtrue"],
      [
       AC_ARG_ENABLE([x86-asm], AS_HELP_STRING([--enable-x86-asm], [Enable assembler code for x86 CPUs. (default=yes)]))

       AS_IF(
             [test "x$enable_x86_asm" != "xno"],
             [
              AC_DEFINE([VS_TARGET_CPU_X86])
             ]
       )

       AC_SUBST([MFLAGS], ["-mfpmath=sse -msse2"])
       AC_SUBST([AVX2FLAGS], ["-mavx2 -mfma -mtune=haswell"])
      ]
)

AS_IF(
      [test "x$PPC" = "xtrue"],
      [AC_DEFINE([VS_TARGET_CPU_POWERPC])]
)

AS_IF(
      [test "x$ARM" = "xtrue"],
      [AC_DEFINE([VS_TARGET_CPU_ARM])]
)



AC_ARG_ENABLE([core], AS_HELP_STRING([--enable-core], [Build the VapourSynth core library. (default=yes)]))
AS_IF(
      [test "x$enable_core" != "xno"],
      [
       AC_DEFINE([VS_CORE_EXPORTS])
       AC_DEFINE([VS_GRAPH_API])

       PKG_CHECK_MODULES([ZIMG], [zimg])

       AC_LANG_PUSH([C++])
       saved_cppflags="$CPPFLAGS"
       saved_libs="$LIBS"
       CPPFLAGS="$ZIMG_CFLAGS"
       LIBS="$ZIMG_LIBS"
       AC_LINK_IFELSE(
                         [AC_LANG_PROGRAM(
                                         [[
#include <zimg.h>

#if ZIMG_API_VERSION < ZIMG_MAKE_API_VERSION(2, 3)
#error zimg API v2.3 or greater required
#endif
                                         ]],
                                         [[
unsigned major, minor, micro;
zimg_get_version_info(&major, &minor, &micro);
                                         ]]
                         )],
                         [],
                         [AC_MSG_ERROR([failed to link zimg. See config.log for details.])]
       )


       CPPFLAGS="$saved_cppflags"
       LIBS="$saved_libs"
       AC_LANG_POP([C++])


       AC_CONFIG_FILES([pc/vapoursynth.pc])

       dnl Annoying shit...
       AS_CASE(
               [$host_os],
               [cygwin*|mingw*],
               [],
               [
                saved_libs="$LIBS"
                AC_SEARCH_LIBS([dlopen], [dl dld], [], [AC_MSG_ERROR([Unable to find the dlopen() function.])])
                AS_IF(
                      [test "x$ac_cv_search_dlopen" != "xnone required"],
                      [AC_SUBST([DLOPENLIB], ["$ac_cv_search_dlopen"])]
                )
                LIBS="$saved_libs"
               ]
       )

       AC_LANG_PUSH([C++])
       AC_MSG_CHECKING([for sched_getaffinity])
       AC_LINK_IFELSE(
                         [AC_LANG_PROGRAM(
                                         [[
#define _GNU_SOURCE
#include <sched.h>
                                         ]],
                                         [[
cpu_set_t affinity;
sched_getaffinity(0, sizeof(cpu_set_t), &affinity);
int count = CPU_COUNT(&affinity);
                                         ]]
                         )],
                         [
                          AC_DEFINE([HAVE_SCHED_GETAFFINITY])
                          AC_MSG_RESULT([yes])
                         ],
                         [AC_MSG_RESULT([no])]
       )
       AC_LANG_POP([C++])

       AC_LANG_PUSH([C++])
       AC_MSG_CHECKING([for cpuset_getaffinity])
       AC_LINK_IFELSE(
                         [AC_LANG_PROGRAM(
                                         [[
#include <sys/param.h>
#include <sys/_cpuset.h>
#include <sys/cpuset.h>
                                         ]],
                                         [[
cpuset_t affinity;
cpuset_getaffinity(CPU_LEVEL_WHICH, CPU_WHICH_PID, -1, sizeof(cpuset_t), &affinity);
int count = CPU_COUNT(&affinity);
                                         ]]
                         )],
                         [
                          AC_DEFINE([HAVE_CPUSET_GETAFFINITY])
                          AC_MSG_RESULT([yes])
                         ],
                         [AC_MSG_RESULT([no])]
       )
       AC_LANG_POP([C++])
      ]
)
AM_CONDITIONAL([VSCORE], [test "x$enable_core" != "xno"])
AM_CONDITIONAL([X86ASM], [test "x$X86" = "xtrue" -a "x$enable_x86_asm" != "xno"])



AC_ARG_ENABLE([vsscript], AS_HELP_STRING([--enable-vsscript], [Enable VSScript. Requires Python 3. (default=yes)]))
AS_IF(
      [test "x$enable_vsscript" != "xno"],
      [
       AM_PATH_PYTHON([3])

       PKG_CHECK_MODULES([PYTHON3],
                         [python-$PYTHON_VERSION-embed],
                         [],
                         [
                          PKG_CHECK_MODULES([PYTHON3],
                                            [python-$PYTHON_VERSION])
                         ])

       AC_CONFIG_FILES([pc/vapoursynth-script.pc])
      ]
)
AM_CONDITIONAL([VSSCRIPT], [test "x$enable_vsscript" != "xno"])



AC_ARG_ENABLE([vspipe], AS_HELP_STRING([--enable-vspipe], [Build vspipe. Requires VSScript. (default=yes)]))
AS_IF(
      [test "x$enable_vsscript" = "xno"],
      [
       AC_DEFINE([VS_GRAPH_API])
       AS_IF(
             [test "x$enable_vspipe" = "xyes"],
             [AC_MSG_ERROR([Cannot build vspipe when VSScript is disabled.])],
             [enable_vspipe="no"]
       )
      ]
)
AM_CONDITIONAL([VSPIPE], [test "x$enable_vspipe" != "xno"])



AC_ARG_ENABLE([python-module], AS_HELP_STRING([--enable-python-module], [Build the Python module. Requires Cython, Python, and the core. (default=yes)]))
AS_IF(
      [test "x$enable_core" = "xno"],
      [
       AS_IF(
             [test "x$enable_python_module" = "xyes"],
             [AC_MSG_ERROR([Cannot build the Python module when the core is disabled.])],
             [enable_python_module="no"]
       )
      ],
      [
       AS_IF(
             [test "x$enable_python_module" != "xno"],
             [
              AC_ARG_WITH([cython], AS_HELP_STRING([--with-cython], [Override the automatic detection of the Cython executable. (default=check)]), [], [with_cython=check])
              AS_IF(
                    [test "x$with_cython" = "xcheck"],
                    [AC_CHECK_PROGS([CYTHON], [cython3 cython])],
                    [CYTHON="$with_cython"]
              )

              AS_IF(
                    [test "x$CYTHON" = "x"],
                    [AC_MSG_ERROR([Cython required but not found.])]
              )
              AC_SUBST([CYTHON])

              AS_IF(
                    [test -z "$PYTHON_VERSION"],
                    [AM_PATH_PYTHON([3])]
              )

              AS_IF(
                    [test -z "$PYTHON3_LIBS"],
                    [
                     PKG_CHECK_MODULES([PYTHON3],
                                       [python-$PYTHON_VERSION-embed],
                                       [],
                                       [
                                        PKG_CHECK_MODULES([PYTHON3],
                                                          [python-$PYTHON_VERSION])
                                       ])
                    ]
              )

              AS_CASE(
                      [$host_os],
                      [darwin*],
                      [
                       AC_SUBST([PYTHON_MODULE_UNDEFINED], ["-undefined dynamic_lookup"])
                      ],
                      [cygwin*|mingw*],
                      [
                       AC_SUBST([PYTHON_MODULE_UNDEFINED], ["-no-undefined"])
                       AC_SUBST([MAYBE_PYTHON3_LIBS], ["$PYTHON3_LIBS"])
                      ]
                     )
             ]
       )
      ]
)
AM_CONDITIONAL([PYTHONMODULE], [test "x$enable_python_module" != "xno"])



dnl Workaround for a bug in libtool
dnl The windows libtool uses a file magic checking method that only accepts
dnl dynamic libraries. Change it for libtool's alternative checking method.

dnl Workaround found in configure.ac from ffms2.

dnl Reproducing the error message below, for search engines and people
dnl looking for a solution...

dnl *** Warning: linker path does not have real file for library -lfftw3f.
dnl *** I have the capability to make that library automatically link in when
dnl *** you link to this library.  But I can only do this if you have a
dnl *** shared version of the library, which you do not appear to have
dnl *** because I did check the linker path looking for a file starting
dnl *** with libfftw3f and none of the candidates passed a file format test
dnl *** using a file magic. Last file checked: /home/asdf/mingw-w64/i686/mingw/lib/libfftw3f.a
dnl *** The inter-library dependencies that have been dropped here will be
dnl *** automatically added whenever a program is linked with this library
dnl *** or is declared to -dlopen it.
dnl *
dnl *** Since this library must not contain undefined symbols,
dnl *** because either the platform does not support them or
dnl *** it was explicitly requested with -no-undefined,
dnl *** libtool will only create a static version of it.

if test "$lt_cv_file_magic_cmd" = "func_win32_libid" ; then
    deplibs_check_method='file_magic file format pei*-(i386|x86-64)|(.*architecture: i386)?'
    file_magic_cmd='$OBJDUMP -f'
fi



AC_CONFIG_FILES([Makefile])
AC_OUTPUT

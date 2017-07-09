# MIT License

# Copyright (c) 2017 Cristian Adam

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

file(READ version.c version_c_file)
string(REGEX REPLACE 
    ".*CCACHE_VERSION\\[\\] = \"([0-9\.]+)\".*" "\\1" 
    version ${version_c_file})

project(ccache VERSION ${version} LANGUAGES C)

add_executable(ccache ccache.c)

target_sources(ccache PRIVATE 
    args.c
    cleanup.c
    compopt.c
    conf.c
    #confitems_lookup.c
    counters.c
    #envtoconfitems_lookup.c
    execute.c
    exitfn.c
    getopt_long.c
    hash.c
    hashtable.c
    hashtable_itr.c
    hashutil.c
    language.c
    lockfile.c
    main.c
    manifest.c
    mdfour.c
    murmurhashneutral2.c
    stats.c
    unify.c
    util.c
    version.c
    zlib/adler32.c
    zlib/crc32.c
    zlib/deflate.c
    zlib/gzclose.c
    zlib/gzlib.c
    zlib/gzread.c
    zlib/gzwrite.c
    zlib/inffast.c
    zlib/inflate.c
    zlib/inftrees.c
    zlib/trees.c
    zlib/zutil.c
)

target_include_directories(ccache PRIVATE ${CMAKE_BINARY_DIR})

# I'm going to override the original CMake capabilities check functions
# in order not to edit the config.h.in file by hand. This file will be auto generated!

file(REMOVE ${CMAKE_BINARY_DIR}/config.h.in)

include(CheckIncludeFile)
macro(check_include_file header variable)
    _check_include_file(${header} ${variable})
    
    set(define "#cmakedefine01")
    set (extra_macro_args ${ARGN})
    list(LENGTH extra_macro_args num_extra_args)
    if(${num_extra_args} GREATER 0)
        list(GET extra_macro_args 0 optional_arg)
        set(define ${optional_arg})
    endif()
    
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "${define} ${variable}\n")
endmacro()

check_include_file(ctype.h HAVE_CTYPE_H)
check_include_file(dirent.h HAVE_DIRENT_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)
check_include_file(locale.h HAVE_LOCALE_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(ndir.h HAVE_NDIR_H)
check_include_file(pwd.h HAVE_PWD_H "#cmakedefine")
check_include_file(stdarg.h HAVE_STDARG_H)
check_include_file(stddef.h HAVE_STDDEF_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(sys/dir.h HAVE_SYS_DIR_H)
check_include_file(sys/mman.h HAVE_SYS_MMAN_H "#cmakedefine")
check_include_file(sys/ndir.h HAVE_SYS_NDIR_H "#cmakedefine")
check_include_file(sys/stat.h HAVE_SYS_STAT_H "#cmakedefine")
check_include_file(sys/time.h HAVE_SYS_TIME_H "#cmakedefine")
check_include_file(sys/types.h HAVE_SYS_TYPES_H "#cmakedefine")
check_include_file(sys/wait.h HAVE_SYS_WAIT_H "#cmakedefine")
check_include_file(termios.h HAVE_TERMIOS_H)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(varargs.h HAVE_VARARGS_H)

include(CheckFunctionExists)
macro(check_function_exists function variable)
    _check_function_exists(${function} ${variable})

    set(define "#cmakedefine01")
    set (extra_macro_args ${ARGN})
    list(LENGTH extra_macro_args num_extra_args)
    if (${num_extra_args} GREATER 0)
        list(GET extra_macro_args 0 optional_arg)
        set(define ${optional_arg})
    endif ()

    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "${define} ${variable}\n")
endmacro()

check_function_exists(asprintf HAVE_ASPRINTF)
check_function_exists(__compar_fn_t HAVE_COMPAR_FN_T)
check_function_exists(GetFinalPathNameByHandleW HAVE_GETFINALPATHNAMEBYHANDLEW)
check_function_exists(getopt_long HAVE_GETOPT_LONG)
check_function_exists(getpwuid HAVE_GETPWUID "#cmakedefine")
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
check_function_exists(localeconv HAVE_LOCALECONV)
check_function_exists(mkstemp HAVE_MKSTEMP)
check_function_exists(realpath HAVE_REALPATH)
check_function_exists(snprintf HAVE_SNPRINTF)
check_function_exists(strndup HAVE_STRNDUP "#cmakedefine")
check_function_exists(strtok_r HAVE_STRTOK_R)
check_function_exists(unsetenv HAVE_UNSETENV "#cmakedefine")
check_function_exists(utimes HAVE_UTIMES "#cmakedefine")
check_function_exists(vasprintf HAVE_VASPRINTF)
check_function_exists(va_copy HAVE_VA_COPY)
check_function_exists(vsnprintf HAVE_VSNPRINTF)
check_function_exists(__va_copy HAVE___VA_COPY)

# Found a bug in cmake, they already have a _check_symbol_exists implementation macro o_O
include(CheckSymbolExists)
macro(my_check_symbol_exists symbol header variable)
    check_symbol_exists(${symbol} ${header} ${variable})
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine01 ${variable}\n")
endmacro()

list(APPEND CMAKE_REQUIRED_LIBRARIES ws2_32)
my_check_symbol_exists(gethostname winsock2.h HAVE_GETHOSTNAME)
list(REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES ws2_32)

include(CheckTypeSize)
macro(check_type_size type variable)
    _check_type_size(${type} ${variable})
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine ${variable}\n")
endmacro()

check_type_size(intmax_t HAVE_INTMAX_T)
check_type_size(uintmax_t HAVE_UINTMAX_T)
check_type_size(uintptr_t HAVE_UINTPTR_T)
check_type_size(ptrdiff_t HAVE_PTRDIFF_T)
check_type_size("long long" HAVE_LONG_LONG)
check_type_size("long long int" HAVE_LONG_LONG_INT)
check_type_size("unsigned long long int" HAVE_UNSIGNED_LONG_LONG_INT)

include(CheckStructHasMember)
macro(check_struct_has_member struct member header variable)
    _check_struct_has_member(${struct} ${member} ${header} ${variable} ${ARGN})
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine ${variable}\n")
endmacro()
check_struct_has_member("struct lconv" "decimal_point" locale.h HAVE_STRUCT_LCONV_DECIMAL_POINT)
check_struct_has_member("struct lconv" "thousands_sep" locale.h HAVE_STRUCT_LCONV_THOUSANDS_SEP)

if(WIN32)
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#define _WIN32_WINNT 0x0600\n")
    target_link_libraries(ccache PRIVATE ws2_32)
endif()

configure_file(${CMAKE_BINARY_DIR}/config.h.in ${CMAKE_BINARY_DIR}/config.h)

if(NOT HAVE_SNPRINTF)
    target_sources(ccache PRIVATE snprintf.c)
endif()

#
# Defines which I have omitted. Some of the checks are not even 
# refereced in the source code!
#
#undef AC_APPLE_UNIVERSAL_BUILD
#undef HAVE_EXTERN_INLINE
#undef PACKAGE_BUGREPORT
#undef PACKAGE_NAME
#undef PACKAGE_STRING
#undef PACKAGE_TARNAME
#undef PACKAGE_URL
#undef PACKAGE_VERSION
#undef STDC_HEADERS
#undef TIME_WITH_SYS_TIME
#if defined AC_APPLE_UNIVERSAL_BUILD
# if defined __BIG_ENDIAN__
#  define WORDS_BIGENDIAN 1
# endif
#else
# ifndef WORDS_BIGENDIAN
#  undef WORDS_BIGENDIAN
# endif
#endif
#undef _BSD_SOURCE
#undef _BSD_TYPES
#undef _DARWIN_C_SOURCE
#undef _GNU_SOURCE
#undef _NETBSD_SOURCE
#undef _POSIX_C_SOURCE
#undef _XOPEN_SOURCE
#undef _XOPEN_SOURCE_EXTENDED
#undef __BSD_VISIBLE
#undef __EXTENSIONS__

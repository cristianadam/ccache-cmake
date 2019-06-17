# MIT License

# Copyright (c) 2017-2019 Cristian Adam

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

cmake_minimum_required(VERSION 3.12 FATAL_ERROR)

find_package(Git)
execute_process(COMMAND ${GIT_EXECUTABLE} describe
    OUTPUT_VARIABLE git_output
    ERROR_VARIABLE git_error
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
if (NOT git_error)
    string(REGEX MATCH "[0-9.]+" CCACHE_VERSION ${git_output})
else()
    set(CCACHE_VERSION 3.7.1)
endif()

file(WRITE ${CMAKE_BINARY_DIR}/version.c.in [=[
extern const char CCACHE_VERSION[];
const char CCACHE_VERSION[] = "@CCACHE_VERSION@";
]=])
configure_file(${CMAKE_BINARY_DIR}/version.c.in ${CMAKE_BINARY_DIR}/version.c @ONLY)

project(ccache VERSION ${CCACHE_VERSION} LANGUAGES C CXX)

#
# gperf
#
find_program(gperf_program gperf)
if (NOT gperf_program)
    file(WRITE ${CMAKE_BINARY_DIR}/gperf/config.h "")
    set(gperf_sources
        gperf/lib/getline.cc
        gperf/lib/hash.cc
        gperf/src/bool-array.cc
        gperf/src/hash-table.cc
        gperf/src/input.cc
        gperf/src/keyword-list.cc
        gperf/src/keyword.cc
        gperf/src/main.cc
        gperf/src/options.cc
        gperf/src/output.cc
        gperf/src/positions.cc
        gperf/src/search.cc
        gperf/src/version.cc
    )
    add_executable(gperf ${gperf_sources})
    target_include_directories(gperf PRIVATE gperf/lib ${CMAKE_BINARY_DIR}/gperf)
endif()

#
# ccache
#

# CMake script that extracts the value of TOTAL_KEYWORDS and writes a C file
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/total_keywords.cmake [=[
    file(STRINGS ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c ${SOURCE}_numbers_line
        REGEX "TOTAL_KEYWORDS = [0-9]+,")
    string(REGEX MATCH "([0-9]+)" ${SOURCE}_numbers ${${SOURCE}_numbers_line})
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c
        "size_t ${SOURCE}_count(void) { return ${${SOURCE}_numbers}; }\n")
]=])

# CMake script that will assign the sorted index of ITEM
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/add_sorted_indexes.cmake [=[
    file(STRINGS ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c raw_items
        REGEX ",.*ITEM|ITEM_V(\\1)")
    foreach(line IN LISTS raw_items)
        string(REGEX REPLACE "^.*\"([a-z_]+)\".*$" "\\1" item ${line})
        list(APPEND items ${item})
    endforeach()
    list(SORT items)

    file(READ ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c raw_file)
    list(LENGTH items items_length)
    math(EXPR items_length "${items_length} - 1")
    foreach(index RANGE ${items_length})
        list(GET items ${index} item)
        string(REGEX REPLACE "((ITEM|ITEM_V)\\(${item},)" " ${index}, \\1" raw_file "${raw_file}")
    endforeach()

    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c "${raw_file}")
]=])

# Function which adds custom commands to generate c files from gperf files
function(process_gperf out_var)
    set(result)
    foreach(in_f ${ARGN})
        get_filename_component(source ${in_f} NAME_WE)
        set(in_f ${CMAKE_CURRENT_SOURCE_DIR}/${in_f})
        set(out_f "${CMAKE_CURRENT_BINARY_DIR}/${source}_lookup.c")
        set(out_count_f "${CMAKE_CURRENT_BINARY_DIR}/${source}_count.c")
        add_custom_command(OUTPUT ${out_f}
            COMMAND gperf --output-file ${out_f} < ${in_f}
            COMMAND ${CMAKE_COMMAND} -DSOURCE=${source} -P ${CMAKE_CURRENT_BINARY_DIR}/add_sorted_indexes.cmake
            COMMAND ${CMAKE_COMMAND} -DSOURCE=${source} -P ${CMAKE_CURRENT_BINARY_DIR}/total_keywords.cmake
            DEPENDS ${in_f}
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            COMMENT "Creating preprocessed file ${out_f}"
            VERBATIM
        )
        list(APPEND result ${out_f})
    endforeach()
    set(${out_var} "${result}" PARENT_SCOPE)
endfunction()

process_gperf(gperf_processed_files
    ccache/src/confitems.gperf
    ccache/src/envtoconfitems.gperf
)

set(ccache_sources
    ccache/src/args.c
    ccache/src/ccache.c
    ccache/src/cleanup.c
    ccache/src/compopt.c
    ccache/src/conf.c
    ccache/src/confitems.c
    ccache/src/counters.c
    ccache/src/execute.c
    ccache/src/exitfn.c
    ccache/src/getopt_long.c
    ccache/src/hash.c
    ccache/src/hashtable.c
    ccache/src/hashtable_itr.c
    ccache/src/hashutil.c
    ccache/src/language.c
    ccache/src/lockfile.c
    ccache/src/manifest.c
    ccache/src/mdfour.c
    ccache/src/minitrace.c
    ccache/src/murmurhashneutral2.c
    ccache/src/stats.c
    ccache/src/unify.c
    ccache/src/util.c
    ${CMAKE_BINARY_DIR}/version.c
    ccache/src/zlib/adler32.c
    ccache/src/zlib/crc32.c
    ccache/src/zlib/deflate.c
    ccache/src/zlib/gzclose.c
    ccache/src/zlib/gzlib.c
    ccache/src/zlib/gzread.c
    ccache/src/zlib/gzwrite.c
    ccache/src/zlib/inffast.c
    ccache/src/zlib/inflate.c
    ccache/src/zlib/inftrees.c
    ccache/src/zlib/trees.c
    ccache/src/zlib/zutil.c
)
add_library(ccache_object_lib OBJECT ${ccache_sources} ${gperf_processed_files})
target_include_directories(ccache_object_lib PUBLIC ${CMAKE_BINARY_DIR} ccache/src)

add_executable(ccache ccache/src/main.c)
target_link_libraries(ccache PRIVATE ccache_object_lib)

#
# unittest
#

enable_testing()
set(unittest_files
    ccache/unittest/framework.c
    ccache/unittest/main.c
    ccache/unittest/test_args.c
    ccache/unittest/test_argument_processing.c
    ccache/unittest/test_compopt.c
    ccache/unittest/test_conf.c
    ccache/unittest/test_counters.c
    ccache/unittest/test_hash.c
    ccache/unittest/test_hashutil.c
    ccache/unittest/test_lockfile.c
    ccache/unittest/test_stats.c
    ccache/unittest/test_util.c
    ccache/unittest/util.c
)

file(WRITE ${CMAKE_BINARY_DIR}/suites.h.in)
foreach(source IN LISTS unittest_files)
    file(STRINGS ${source} source_test_suite REGEX "TEST_SUITE\\(")
    string(REGEX MATCH "([a-z_]+)" suite_name "${source_test_suite}")
    if (suite_name)
        file(APPEND ${CMAKE_BINARY_DIR}/suites.h.in "SUITE(${suite_name})\n")
    endif()
endforeach()
configure_file(${CMAKE_BINARY_DIR}/suites.h.in ${CMAKE_BINARY_DIR}/suites.h)

add_executable(unittest ${unittest_files})
target_link_libraries(unittest PRIVATE ccache_object_lib)

add_test(NAME unittest COMMAND unittest)

# I'm going to override the original CMake capabilities check functions
# in order not to edit the config.h.in file by hand. This file will be auto generated!

file(WRITE ${CMAKE_BINARY_DIR}/config.h.in "#pragma once\n")

include(CheckIncludeFile)
macro(check_include_file header variable)
    _check_include_file(${header} ${variable})

    cmake_parse_arguments(arg ONLY_DEFINE "" "" ${ARGV})
    set(define "#cmakedefine01")
    if (arg_ONLY_DEFINE)
      set(define "#cmakedefine")
    endif()

    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "${define} ${variable}\n")
    unset(define)
endmacro()

include(CheckFunctionExists)
macro(check_function_exists function variable)
    _check_function_exists(${function} ${variable})

    cmake_parse_arguments(arg ONLY_DEFINE "" "" ${ARGV})
    set(define "#cmakedefine01")
    if (arg_ONLY_DEFINE)
      set(define "#cmakedefine")
    endif()

    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "${define} ${variable}\n")
    unset(define)
endmacro()

include(CheckSymbolExists)
macro(check_symbol_exists symbol header variable)
    _check_symbol_exists(${symbol} ${header} ${variable})
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine01 ${variable}\n")
endmacro()

include(CheckTypeSize)
macro(check_type_size type variable)
    # Don't check for check_include_file(sys/types.h) in check_type_size
    _check_type_size(${type} ${variable} BUILTIN_TYPES_ONLY)
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine ${variable}\n")
endmacro()

include(CheckStructHasMember)
macro(check_struct_has_member struct member header variable)
    _check_struct_has_member(${struct} ${member} ${header} ${variable} ${ARGN})
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#cmakedefine ${variable}\n")
endmacro()

check_include_file(ctype.h HAVE_CTYPE_H)
check_include_file(dirent.h HAVE_DIRENT_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)
check_include_file(locale.h HAVE_LOCALE_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(ndir.h HAVE_NDIR_H)
check_include_file(pwd.h HAVE_PWD_H ONLY_DEFINE)
check_include_file(stdarg.h HAVE_STDARG_H)
check_include_file(stddef.h HAVE_STDDEF_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(sys/dir.h HAVE_SYS_DIR_H)
check_include_file(sys/mman.h HAVE_SYS_MMAN_H ONLY_DEFINE)
check_include_file(sys/ndir.h HAVE_SYS_NDIR_H ONLY_DEFINE)
check_include_file(sys/stat.h HAVE_SYS_STAT_H ONLY_DEFINE)
check_include_file(sys/time.h HAVE_SYS_TIME_H ONLY_DEFINE)
check_include_file(sys/types.h HAVE_SYS_TYPES_H ONLY_DEFINE)
check_include_file(sys/wait.h HAVE_SYS_WAIT_H ONLY_DEFINE)
check_include_file(termios.h HAVE_TERMIOS_H ONLY_DEFINE)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(varargs.h HAVE_VARARGS_H)

check_function_exists(asprintf HAVE_ASPRINTF)
check_function_exists(__compar_fn_t HAVE_COMPAR_FN_T ONLY_DEFINE)
check_function_exists(GetFinalPathNameByHandleW HAVE_GETFINALPATHNAMEBYHANDLEW)
check_function_exists(getopt_long HAVE_GETOPT_LONG)
check_function_exists(getpwuid HAVE_GETPWUID ONLY_DEFINE)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
check_function_exists(localeconv HAVE_LOCALECONV)
check_function_exists(mkstemp HAVE_MKSTEMP)
check_function_exists(realpath HAVE_REALPATH)
check_function_exists(snprintf HAVE_SNPRINTF)
check_function_exists(strndup HAVE_STRNDUP ONLY_DEFINE)
check_function_exists(strtok_r HAVE_STRTOK_R)
check_function_exists(unsetenv HAVE_UNSETENV ONLY_DEFINE)
check_function_exists(utimes HAVE_UTIMES ONLY_DEFINE)
check_function_exists(vasprintf HAVE_VASPRINTF)
check_function_exists(va_copy HAVE_VA_COPY)
check_function_exists(vsnprintf HAVE_VSNPRINTF)
check_function_exists(__va_copy HAVE___VA_COPY)

list(APPEND CMAKE_REQUIRED_LIBRARIES ws2_32)
check_symbol_exists(gethostname winsock2.h HAVE_GETHOSTNAME)
list(REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES ws2_32)

check_type_size(intmax_t HAVE_INTMAX_T)
check_type_size(uintmax_t HAVE_UINTMAX_T)
check_type_size(uintptr_t HAVE_UINTPTR_T)
check_type_size(ptrdiff_t HAVE_PTRDIFF_T)
check_type_size("long long" HAVE_LONG_LONG)
check_type_size("long long int" HAVE_LONG_LONG_INT)
check_type_size("unsigned long long int" HAVE_UNSIGNED_LONG_LONG_INT)

check_struct_has_member("struct lconv" "decimal_point" locale.h HAVE_STRUCT_LCONV_DECIMAL_POINT)
check_struct_has_member("struct lconv" "thousands_sep" locale.h HAVE_STRUCT_LCONV_THOUSANDS_SEP)

if(WIN32)
    file(APPEND ${CMAKE_BINARY_DIR}/config.h.in "#define _WIN32_WINNT 0x0600\n")
    target_link_libraries(ccache PRIVATE ws2_32)
    target_link_libraries(unittest PRIVATE ws2_32)
endif()

configure_file(${CMAKE_BINARY_DIR}/config.h.in ${CMAKE_BINARY_DIR}/config.h)

if(NOT HAVE_SNPRINTF)
    target_sources(ccache PRIVATE ccache/src/snprintf.c)
endif()

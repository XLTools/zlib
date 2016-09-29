#  :copyright: (c) 2015-2016 The Regents of the University of California.
#  :license: MIT, see LICENSE for more details.

cmake_minimum_required(VERSION 3.4)
project(ZLib C)

# VERSION
# -------

set(${PROJECT}_VERSION_MAJOR 1)
set(${PROJECT}_VERSION_MINOR 2)
set(${PROJECT}_VERSION_PATCH 8)

# ARGUMENTS
# ---------

option(BUILD_STATIC "Build static libz" ON)
option(BUILD_ZLIB "Build ZLib compressor/decompressor" OFF)
option(BUILD_ZLIB_TESTS "Build ZLib tests" OFF)

# CHECKS
# ------

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h    HAVE_STDINT_H)
check_include_file(stddef.h    HAVE_STDDEF_H)

# OFFSET
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
   add_definitions(-D_LARGEFILE64_SOURCE=1)
endif()

# FSEEKO
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
    add_definitions(-DNO_FSEEKO)
endif()

# ASM
# ---

if(CMAKE_COMPILER_IS_GNUCC)
    if(ASM686)
        set(ZLIB_ASMS contrib/asm686/match.S)
    elseif(AMD64)
        set(ZLIB_ASMS contrib/amd64/amd64-match.S)
    endif()
elseif(MSVC)
    if(ASM686)
        set(ZLIB_ASMS
            contrib/masmx86/inffas32.asm
            contrib/masmx86/match686.asm
        )
    elseif(AMD64)
        set(ZLIB_ASMS
            contrib/masmx64/gvmat64.asm
            contrib/masmx64/inffasx64.asm
        )
    endif()
endif()

if(ZLIB_ASMS)
    add_definitions(-DASMV)
    if(MSVC)
        add_definitions(-DASMINF)
    else()
        set_source_files_properties(${ZLIB_ASMS}
            PROPERTIES LANGUAGE C
            COMPILE_FLAGS -DNO_UNDERLINE
        )
    endif()
endif()

# DLLS
# ----

if(MSVC)
    set(ZLIB_DLL_SRCS win32/zlib1.rc)
elseif(MINGW)
    set(ZLIB_DLL_SRCS ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj)
endif()

# LIBRARIES
# ---------

set(ZLIB_PUBLIC_HDRS
    zconf.h
    zlib.h
)
set(ZLIB_PRIVATE_HDRS
    crc32.h
    deflate.h
    gzguts.h
    inffast.h
    inffixed.h
    inflate.h
    inftrees.h
    trees.h
    zutil.h
)
set(ZLIB_SRCS
    adler32.c
    compress.c
    crc32.c
    deflate.c
    gzclose.c
    gzlib.c
    gzread.c
    gzwrite.c
    inflate.c
    infback.c
    inftrees.c
    inffast.c
    trees.c
    uncompr.c
    zutil.c
)

if(BUILD_STATIC)
    add_library(zlib
        STATIC
        ${ZLIB_SRCS}
        ${ZLIB_ASMS}
        ${ZLIB_PUBLIC_HDRS}
        ${ZLIB_PRIVATE_HDRS}
    )
else()
    add_library(zlib
        SHARED
        ${ZLIB_SRCS}
        ${ZLIB_ASMS}
        ${ZLIB_DLL_SRCS}
        ${ZLIB_PUBLIC_HDRS}
        ${ZLIB_PRIVATE_HDRS}
    )
    set_target_properties(zlib
        PROPERTIES DEFINE_SYMBOL ZLIB_DLL
    )
endif()

# NAMING
# ------

if(UNIX)
    set_target_properties(zlib
        PROPERTIES OUTPUT_NAME z
    )
elseif(NOT BUILD_STATIC AND WIN32)
    set_target_properties(zlib
        PROPERTIES SUFFIX "1.dll"
    )
endif()

# EXECUTABLES
# -----------

if(BUILD_ZLIB_TESTS)
    # EXAMPLE
    add_executable(example
        test/example.c
    )
    target_link_libraries(example zlib)

    # MINI GZIP
    add_executable(minigzip
        test/minigzip.c
    )
    target_link_libraries(minigzip zlib)

    if(HAVE_OFF64_T)
        # EXAMPLE 64
        add_executable(example64
            test/example.c
        )
        target_link_libraries(example64 zlib)
        set_target_properties(example64
            PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64"
        )

        # MINI GZIP 64
        add_executable(minigzip64
            test/minigzip.c
        )
        target_link_libraries(minigzip64 zlib)
        set_target_properties(minigzip64
            PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64"
        )
    endif()
endif()

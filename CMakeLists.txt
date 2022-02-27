CMAKE_MINIMUM_REQUIRED(VERSION 3.14)
SET(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)

PROJECT(ZLib LANGUAGES C CXX VERSION 2022.2.27)

SET(VERSION "1.2.11")
SET(ZLIB_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
SET(ZLIB_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})

OPTION(ASM686 "Enable building i686 assembly implementation")
OPTION(AMD64 "Enable building amd64 assembly implementation")
OPTION(ZLIB_BUILD_SHARED_LIBS "Build Shared Lib for ZLib" OFF)

INCLUDE(CheckTypeSize)
INCLUDE(CheckFunctionExists)
INCLUDE(CheckIncludeFile)
INCLUDE(CheckCSourceCompiles)
ENABLE_TESTING()

CHECK_INCLUDE_FILE(sys/types.h HAVE_SYS_TYPES_H)
CHECK_INCLUDE_FILE(stdint.h HAVE_STDINT_H)
CHECK_INCLUDE_FILE(stddef.h HAVE_STDDEF_H)

#
# Check to see if we have large file support
#
SET(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE=1)
# We add these other definitions here because CheckTypeSize.cmake
# in CMake 2.4.x does not automatically do so and we want
# compatibility with CMake 2.4.x.
IF (HAVE_SYS_TYPES_H)
    LIST(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_SYS_TYPES_H)
ENDIF ()
IF (HAVE_STDINT_H)
    LIST(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDINT_H)
ENDIF ()
IF (HAVE_STDDEF_H)
    LIST(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDDEF_H)
ENDIF ()
CHECK_TYPE_SIZE(off64_t OFF64_T)
IF (HAVE_OFF64_T)
    ADD_DEFINITIONS(-D_LARGEFILE64_SOURCE=1)
ENDIF ()
SET(CMAKE_REQUIRED_DEFINITIONS) # clear variable

#
# Check for fseeko
#
CHECK_FUNCTION_EXISTS(fseeko HAVE_FSEEKO)
IF (NOT HAVE_FSEEKO)
    ADD_DEFINITIONS(-DNO_FSEEKO)
ENDIF ()

#
# Check for unistd.h
#
CHECK_INCLUDE_FILE(unistd.h Z_HAVE_UNISTD_H)

IF (MSVC)
    SET(CMAKE_DEBUG_POSTFIX "d")
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE)
    ADD_DEFINITIONS(-D_CRT_NONSTDC_NO_DEPRECATE)
    INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
ENDIF ()

IF (NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    # If we're doing an out of source build and the user has a zconf.h
    # in their source tree...
    IF (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h)
        MESSAGE(STATUS "Renaming")
        MESSAGE(STATUS "    ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h")
        MESSAGE(STATUS "to 'zconf.h.included' because this file is included with zlib")
        MESSAGE(STATUS "but CMake generates it automatically in the build directory.")
        FILE(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.included)
    ENDIF ()
ENDIF ()

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein ${ZLIB_SOURCE_DIR}/Include/ZLib/zconf.h @ONLY)


#============================================================================
# zlib
#============================================================================

IF (CMAKE_COMPILER_IS_GNUCC)
    IF (ASM686)
        SET(ZLIB_ASMS Contrib/asm686/match.S)
    ELSEIF (AMD64)
        SET(ZLIB_ASMS Contrib/amd64/amd64-match.S)
    ENDIF ()

    IF (ZLIB_ASMS)
        ADD_DEFINITIONS(-DASMV)
        SET_SOURCE_FILES_PROPERTIES(${ZLIB_ASMS} PROPERTIES LANGUAGE C COMPILE_FLAGS -DNO_UNDERLINE)
    ENDIF ()
ENDIF ()

IF (MSVC)
    IF (ASM686)
        ENABLE_LANGUAGE(ASM_MASM)
        SET(ZLIB_ASMS
                Contrib/masmx86/inffas32.asm
                Contrib/masmx86/match686.asm
                )
    ELSEIF (AMD64)
        ENABLE_LANGUAGE(ASM_MASM)
        SET(ZLIB_ASMS
                Contrib/masmx64/gvmat64.asm
                Contrib/masmx64/inffasx64.asm
                )
    ENDIF ()

    IF (ZLIB_ASMS)
        ADD_DEFINITIONS(-DASMV -DASMINF)
    ENDIF ()
ENDIF ()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
FILE(READ Include/ZLib/zlib.h _zlib_h_contents)
STRING(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
        "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents})



# The OBJECT's can be used to compile the sources in the list given to add_library
# to object files, but then neither archiving them into a static library nor
# linking them into a shared object. The use of object libraries is
# particularly useful if one needs to create both static and shared libraries
# in one go.

# Property: OBJECT
ADD_LIBRARY(ZLib.Framework.Object OBJECT
        Source/adler32.c
        Source/compress.c
        Source/crc32.c
        Source/deflate.c
        Source/gzclose.c
        Source/gzlib.c
        Source/gzread.c
        Source/gzwrite.c
        Source/inflate.c
        Source/infback.c
        Source/inftrees.c
        Source/inffast.c
        Source/trees.c
        Source/uncompr.c
        Source/zutil.c

        ${ZLIB_ASMS}
        )

SET_TARGET_PROPERTIES(ZLib.Framework.Object PROPERTIES C_STANDARD 11)
TARGET_INCLUDE_DIRECTORIES(ZLib.Framework.Object PUBLIC $<BUILD_INTERFACE:${ZLIB_SOURCE_DIR}/Include/>)

IF (ZLIB_BUILD_SHARED_LIBS)

    # The POSITION_INDEPENDENT_CODE property determines whether position independent executables or
    # shared libraries will be created. This property is True by default for SHARED and MODULE library
    # targets and False otherwise.

    # Property: SHARED, POSITION_INDEPENDENT_CODE
    ADD_LIBRARY(ZLib.Framework SHARED $<TARGET_OBJECTS:ZLib.Framework.Object>)

ELSE ()

    ADD_LIBRARY(ZLib.Framework STATIC $<TARGET_OBJECTS:ZLib.Framework.Object>)

ENDIF ()

ADD_LIBRARY(ZLib::Framework ALIAS ZLib.Framework)
SET_TARGET_PROPERTIES(ZLib.Framework PROPERTIES C_STANDARD 11)
SET_TARGET_PROPERTIES(ZLib.Framework PROPERTIES DEFINE_SYMBOL ZLIB_DLL)
# It is necessary to repeat these instructions again, in case of not doing it, the executables
# will be unable to locate the Headers needed to perform the linking
TARGET_INCLUDE_DIRECTORIES(ZLib.Framework PUBLIC $<BUILD_INTERFACE:${ZLIB_SOURCE_DIR}/Include/>)



#============================================================================
# Example binaries
#============================================================================

ADD_EXECUTABLE(example Test/Example.cpp)
TARGET_LINK_LIBRARIES(example PRIVATE ZLib::Framework)
ADD_TEST(example example)

ADD_EXECUTABLE(minigzip Test/minigzip.c)
TARGET_LINK_LIBRARIES(minigzip PRIVATE ZLib::Framework)

IF (HAVE_OFF64_T)
    ADD_EXECUTABLE(example64 Test/Example.cpp)
    TARGET_LINK_LIBRARIES(example64 PRIVATE ZLib::Framework)
    SET_TARGET_PROPERTIES(example64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
    ADD_TEST(example64 example64)

    ADD_EXECUTABLE(minigzip64 Test/minigzip.c)
    TARGET_LINK_LIBRARIES(minigzip64 PRIVATE ZLib::Framework)
    SET_TARGET_PROPERTIES(minigzip64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
ENDIF ()

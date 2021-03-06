#**************************************************************************
#    Lightspark, a free flash player implementation
#
#    Copyright (C) 2010-2013  Alessandro Pignotti <a.pignotti@sssup.it>
#    Copyright (C) 2010  Giacomo Spigler <g.spigler@sssup.it>
#
#    This program is free software: you can redistribute it and/or modify
#    it under the terms of the GNU Lesser General Public License as published by
#    the Free Software Foundation, either version 3 of the License, or
#    (at your option) any later version.
#
#   This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU Lesser General Public License for more details.
#
#   You should have received a copy of the GNU Lesser General Public License
#   along with this program.  If not, see <http://www.gnu.org/licenses/>.
#**************************************************************************


CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

# Name & Version
PROJECT(LIGHTSPARK)
SET(MAJOR_VERSION "0")
SET(MINOR_VERSION "7")
SET(PATCH_VERSION "2")
SET(SUBPATCH_VERSION "0")

IF(EXISTS "${CMAKE_SOURCE_DIR}/.git/")
	SET(GIT_VERSION "-git")

	#Find out if we are exactly at a tag. Then the version
	#is enough. Only consider the official release tags like
	#'lightspark-0.5.3'
	EXECUTE_PROCESS(COMMAND git describe --match "lightspark-*" --tags
			WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
			OUTPUT_VARIABLE GIT_DESC
			ERROR_QUIET
			OUTPUT_STRIP_TRAILING_WHITESPACE)
	#Check if that command failed (maybe git is not installed at all)
	IF(GIT_DESC)
		EXECUTE_PROCESS(COMMAND git describe --match "lightspark-*" --tags --abbrev=0
				WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
				OUTPUT_VARIABLE GIT_DESC_ABBREV
				ERROR_QUIET
				OUTPUT_STRIP_TRAILING_WHITESPACE)

		IF(${GIT_DESC} STREQUAL ${GIT_DESC_ABBREV})
			#We are exactly at a tag
			SET(GIT_VERSION "")
		ELSE()
			#Append to GIT_VERSION
			#GIT_DESC_ABBREV is of the form lightspark-0.5.3 (the name of the previous tag)
			#GIT_DESC is of the form lightspark-0.5.3-69-gf49ec56 (tag-numberOfCommits-currentHash)
			#We want GIT_REVISION to be 69-gf49ec56
			#which denotes the number of commits since the last tag (=release)
			#and the current commit's hash
			STRING(REPLACE "${GIT_DESC_ABBREV}-" "" GIT_REVISION "${GIT_DESC}")
			SET(GIT_VERSION "-${GIT_REVISION}")
		ENDIF()
	ENDIF()
ELSE(EXISTS "${CMAKE_SOURCE_DIR}/.git/")
	SET(GIT_VERSION "")
ENDIF(EXISTS "${CMAKE_SOURCE_DIR}/.git/")

IF(SUBPATCH_VERSION EQUAL "0")
	SET(FULLVERSION "${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}${GIT_VERSION}")
ELSE(SUBPATCH_VERSION EQUAL "0")
	SET(FULLVERSION "${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}.${SUBPATCH_VERSION}${GIT_VERSION}")
ENDIF(SUBPATCH_VERSION EQUAL "0")

SET(SHORTVERSION "${MINOR_VERSION}${PATCH_VERSION}${SUBPATCH_VERSION}")

CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/src/version.h.in" "${CMAKE_CURRENT_BINARY_DIR}/version.h" @ONLY)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

# GCC 4.6+ is required, GCC_VERSION macro taken from GCC manual
INCLUDE(CheckCSourceCompiles)
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
   CHECK_C_SOURCE_COMPILES("
   #define GCC_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100 + __GNUC_PATCHLEVEL__)
   #if GCC_VERSION < 40600
   #error
   #endif
   int main() { return 0; }" GCC_IS_4_6)
   IF(NOT GCC_IS_4_6)
	  MESSAGE(FATAL_ERROR "GCC 4.6+ is required.")
   ENDIF(NOT GCC_IS_4_6)
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

# Find put the path of the gnash executable, if available
FIND_PROGRAM(GNASH_EXE_PATH NAMES gnash)

IF(GNASH_EXE_PATH)
	MESSAGE(STATUS "Found gnash path: ${GNASH_EXE_PATH}")
ELSE(GNASH_EXE_PATH)
# Set a default path
	MESSAGE(STATUS "Gnash not found")
	SET(GNASH_EXE_PATH "/usr/bin/gnash")
ENDIF(GNASH_EXE_PATH)

# Find out CPU architecture ...
# we could also use IF(CMAKE_SIZEOF_VOID_P EQUAL 4) to determine if it's a 32 or a 64bit arch
#
# Setting the value of LIB_SUFFIX if not specified (We could build 32bit version on 64bit)
# However, some distro don't implement FHS the same way
# Suse, Redhat put 64bit libs in /lib64; Debian use /lib for 64bit, but specifies /lib32 for 32bit libs
# See FHS 2.3 for Lib directories under each architecture 

# Some directory shortcuts
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/conf)
INCLUDE(Pack)
# If we're gcc, then use nasm to get fastpath.  If MSVC, just use inline asm to get around
# CMake issues 
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
   INCLUDE(CMakeASM-NASMCompiler)
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
IF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^i[3-6]86$|^x86$")
	SET(i386 1)
	SET(LIB_SUFFIX "" CACHE STRING "Choose the suffix of the lib folder (if any) : None 32")
	# nasm for assembly optimizations
	IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	   ENABLE_LANGUAGE(ASM-NASM)        
	ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
ELSEIF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "unknown" AND ${CMAKE_SYSTEM} MATCHES "GNU-0.3")
	# GNU Hurd is i386
	SET(i386 1)
	SET(LIB_SUFFIX "" CACHE STRING "Choose the suffix of the lib folder (if any) : None 32")
	# nasm for assembly optimizations
	IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	   ENABLE_LANGUAGE(ASM-NASM)
	ENDIF ()
ELSEIF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^x86_64$|^amd64$")
	SET(x86_64 1)
	SET(LIB_SUFFIX "" CACHE STRING "Choose the suffix of the lib folder (if any) : None 64")
	# nasm for assembly optimizations
	IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	  ENABLE_LANGUAGE(ASM-NASM)
	ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
ELSEIF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "ppc")
	SET(ppc 1)
	SET(LIB_SUFFIX "" CACHE STRING "Choose the suffix of the lib folder (if any) : None ppc")
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__STDC_CONSTANT_MACROS -fPIC")
	IF(NOT(DISABLE_ALTIVEC))
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maltivec")
	ENDIF()
ELSE(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^i[3-6]86$|^x86$")
	# All non-x86 non-ppc architectures.
	SET(LIB_SUFFIX "" CACHE STRING "Choose the suffix of the lib folder (if any) : None")
ENDIF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^i[3-6]86$|^x86$")

IF(CMAKE_SYSTEM_NAME STREQUAL CMAKE_HOST_SYSTEM_NAME)
  SET(CROSSCOMPILING FALSE)
ELSE()
  SET(CROSSCOMPILING TRUE)
ENDIF()

#On mingw-cross-env, all depedencies are static libs
IF(CROSSCOMPILING AND MINGW)
  SET(STATICDEPS TRUE)
ELSE()
  SET(STATICDEPS FALSE)
ENDIF()

# Using relative folders, cmake happens by itself CMAKE_INSTALL_PREFIX
IF(UNIX)
   SET(ETCDIR "/etc") #Using absolute folder
   SET(BINDIR "bin")
   SET(DATADIR "share")
   # Absolute path, because this path is embedded in the binary
   SET(LSDATADIR "${CMAKE_INSTALL_PREFIX}/${DATADIR}/lightspark")
   SET(LIBDIR "lib${LIB_SUFFIX}")
   #We need a absolute path here because we use it for rpath
   #later on
   SET(PRIVATELIBDIR "${CMAKE_INSTALL_PREFIX}/${LIBDIR}/lightspark")
   SET(CMAKE_INSTALL_RPATH "${PRIVATELIBDIR}")
   SET(PLUGINSDIR "${PRIVATELIBDIR}/plugins")
ELSE()
  SET(ETCDIR ".")
  SET(BINDIR ".")
  SET(DATADIR ".")
  SET(LSDATADIR ".")
  SET(LIBDIR ".")
  SET(PRIVATELIBDIR ".")
  SET(PLUGINSDIR ".")
ENDIF(UNIX)

# Setting variables
IF(WIN32)
  SET(AUDIO_BACKEND "winmm" CACHE STRING "Which audio backends should be built?")
ELSE()
  SET(AUDIO_BACKEND "pulseaudio" CACHE STRING "Which audio backends should be built?")
ENDIF()

SET(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Install prefix, default is /usr/local (UNIX) and C:\\Program Files (Windows)")
SET(COMPILE_LIGHTSPARK TRUE CACHE BOOL "Compile Lightspark?")
SET(COMPILE_TIGHTSPARK TRUE CACHE BOOL "Compile Tightspark?")
SET(COMPILE_PLUGIN TRUE CACHE BOOL "Compile the browser plugin?")
SET(ENABLE_CURL TRUE CACHE BOOL "Enable CURL? (Required for Downloader functionality)")
SET(ENABLE_GLES2 FALSE CACHE BOOL "Build with OpenGLES 2.0 support instead of OpenGL")
SET(ENABLE_LIBAVCODEC TRUE CACHE BOOL "Enable libavcodec and dependent functionality?")
SET(ENABLE_RTMP TRUE CACHE BOOL "Enable librtmp and dependent functionality?")
SET(ENABLE_PROFILING FALSE CACHE BOOL "Enable profiling support? (Causes performance issues)")
SET(ENABLE_MEMORY_USAGE_PROFILING FALSE CACHE BOOL "Enable profiling of memory usage? (Causes performance issues)")
SET(PLUGIN_DIRECTORY "${LIBDIR}/mozilla/plugins" CACHE STRING "Directory to install Firefox plugin to")

IF(ENABLE_DEBIAN_ALTERNATIVES OR WIN32)
  SET(PLUGIN_DIRECTORY ${PRIVATELIBDIR})
ENDIF()

IF(NOT CMAKE_BUILD_TYPE)
	SET(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING
		"Choose the type of build, options are: Debug Release RelWithDebInfo Profile" FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

# The library is searched in its private library directory
SET(CMAKE_SKIP_BUILD_RPATH FALSE)
SET(CMAKE_INSTALL_RPATH "${PRIVATELIBDIR}")

# Libraries we need
INCLUDE(FindPkgConfig REQUIRED)
INCLUDE(FindGettext REQUIRED)
INCLUDE(FindLLVM REQUIRED)
IF(${LLVM_STRING_VERSION} VERSION_LESS 2.8)
	MESSAGE(FATAL_ERROR "LLVM >=2.8 is required!")
ENDIF(${LLVM_STRING_VERSION} VERSION_LESS 2.8)
IF(${LLVM_STRING_VERSION} VERSION_EQUAL 2.9)
	MESSAGE(FATAL_ERROR "LLVM !=2.9 is required!")
ENDIF(${LLVM_STRING_VERSION} VERSION_EQUAL 2.9)
IF(${LLVM_STRING_VERSION} VERSION_EQUAL 2.8)
	ADD_DEFINITIONS(-DLLVM_28)
ENDIF(${LLVM_STRING_VERSION} VERSION_EQUAL 2.8)
IF(${LLVM_STRING_VERSION} VERSION_EQUAL 3.0)
	ADD_DEFINITIONS(-DLLVM_30)
ENDIF(${LLVM_STRING_VERSION} VERSION_EQUAL 3.0)
IF(${LLVM_STRING_VERSION} VERSION_GREATER 3.0)
	ADD_DEFINITIONS(-DLLVM_31)
ENDIF(${LLVM_STRING_VERSION} VERSION_GREATER 3.0)
IF(${LLVM_STRING_VERSION} VERSION_GREATER 3.3)
	ADD_DEFINITIONS(-DLLVM_34)
ENDIF(${LLVM_STRING_VERSION} VERSION_GREATER 3.3)
IF(${LLVM_STRING_VERSION} VERSION_GREATER 3.4)
	ADD_DEFINITIONS(-DLLVM_35)
ENDIF(${LLVM_STRING_VERSION} VERSION_GREATER 3.4)
INCLUDE(FindZLIB REQUIRED)
INCLUDE(FindFreetype REQUIRED)
IF(NOT(ENABLE_GLES2))
INCLUDE(FindOpenGL REQUIRED)
INCLUDE(FindGLEW REQUIRED)
ENDIF(NOT(ENABLE_GLES2))
INCLUDE(FindPCRE REQUIRED)
INCLUDE(FindJPEG REQUIRED)
INCLUDE(FindPNG REQUIRED)
pkg_check_modules(XMLPP REQUIRED libxml++-2.6>=2.33.1)

pkg_check_modules(LZMA REQUIRED liblzma)
INCLUDE_DIRECTORIES(${LZMA_INCLUDE_DIRS})
SET(OPTIONAL_LIBRARIES ${OPTIONAL_LIBRARIES} ${LZMA_LIBRARIES})

IF(XMLPP_FOUND AND NOT(XMLPP_VERSION VERSION_LESS 2.35.1))
  ADD_DEFINITIONS(-DXMLPP_2_35_1)
ENDIF(XMLPP_FOUND AND NOT(XMLPP_VERSION VERSION_LESS 2.35.1))

find_package(Boost COMPONENTS filesystem system REQUIRED)
if(Boost_FOUND)
  INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})
  #Without BOOST_MULTI_INDEX_DISABLE_SERIALIZATION, it will indirectly include
  #windows.h and pollute our namespace
  ADD_DEFINITIONS(-DBOOST_MULTI_INDEX_DISABLE_SERIALIZATION)
ENDIF(Boost_FOUND) 

pkg_check_modules(CAIRO REQUIRED cairo pangocairo)

IF(UNIX)
  pkg_check_modules(X11 REQUIRED x11)
  INCLUDE_DIRECTORIES(${X11_INCLUDE_DIRS})
ENDIF(UNIX)

IF(ENABLE_GLES2)
  pkg_check_modules(GLES2 REQUIRED egl glesv2)
  ADD_DEFINITIONS(-DENABLE_GLES2)
ENDIF(ENABLE_GLES2)

IF(WIN32)
  SET(EXTRA_LIBS_LIBRARIES ${EXTRA_LIBS_LIBRARIES} ws2_32.lib imagehlp.lib)
ENDIF(WIN32)

IF(AUDIO_BACKEND)
  ADD_DEFINITIONS(-DAUDIO_BACKEND="${AUDIO_BACKEND}")
ENDIF(AUDIO_BACKEND)

IF(ENABLE_LIBAVCODEC)
	pkg_check_modules(FFMPEG libavcodec libavutil libavformat)
	pkg_check_modules(LIBAVRESAMPLE libavresample)
	IF(NOT(FFMPEG_FOUND))
		INCLUDE(FindFFMpeg REQUIRED)
	ENDIF(NOT(FFMPEG_FOUND))
	# Compatibility checks for ffmpeg deprecated functions
  INCLUDE(CheckFunctionExists REQUIRED)
  INCLUDE(CheckCSourceCompiles)
  SET(FFMPEG_FLAGS "${LIBAVCODEC_CFLAGS} ${LIBAVRESAMPLE_FLAGS}")
  SET(FFMPEG_INCLUDE_DIRS "${FFMPEG_INCLUDE_DIRS} ${LIBAVRESAMPLE_INCLUDE_DIRS}")
  SET(FFMPEG_LIBRARIES "${FFMPEG_LIBRARIES};${LIBAVRESAMPLE_LIBRARIES}")
  SET(CMAKE_REQUIRED_FLAGS ${FFMPEG_FLAGS})
  SET(CMAKE_REQUIRED_INCLUDES ${FFMPEG_INCLUDE_DIRS})
  SET(CMAKE_REQUIRED_LIBRARIES ${FFMPEG_LIBRARIES})
  CHECK_FUNCTION_EXISTS(avcodec_decode_video2 HAVE_AVCODEC_DECODE_VIDEO2)
  CHECK_FUNCTION_EXISTS(avcodec_decode_audio3 HAVE_AVCODEC_DECODE_AUDIO3)
  CHECK_FUNCTION_EXISTS(avcodec_decode_audio4 HAVE_AVCODEC_DECODE_AUDIO4)
  CHECK_FUNCTION_EXISTS(avio_alloc_context HAVE_AVIO_ALLOC_CONTEXT)
  CHECK_FUNCTION_EXISTS(avcodec_alloc_context3 HAVE_AVCODEC_ALLOC_CONTEXT3)
  CHECK_FUNCTION_EXISTS(avcodec_open2 HAVE_AVCODEC_OPEN2)
  CHECK_FUNCTION_EXISTS(avformat_close_input HAVE_AVFORMAT_CLOSE_INPUT)
  CHECK_FUNCTION_EXISTS(avformat_find_stream_info HAVE_AVFORMAT_FIND_STREAM_INFO)
  CHECK_C_SOURCE_COMPILES("#include <libavcodec/avcodec.h>\nint main() { enum AVCodecID c; return 0; }" HAVE_AVCODECID)

  SET(CMAKE_REQUIRED_FLAGS)
  SET(CMAKE_REQUIRED_INCLUDES)
  SET(CMAKE_REQUIRED_LIBRARIES)
  IF(HAVE_AVCODEC_DECODE_VIDEO2)
    ADD_DEFINITIONS(-DHAVE_AVCODEC_DECODE_VIDEO2)
  ENDIF(HAVE_AVCODEC_DECODE_VIDEO2)
  IF(HAVE_AVCODEC_DECODE_AUDIO3)
    ADD_DEFINITIONS(-DHAVE_AVCODEC_DECODE_AUDIO3)
  ENDIF(HAVE_AVCODEC_DECODE_AUDIO3)
  IF(HAVE_AVCODEC_DECODE_AUDIO4)
    ADD_DEFINITIONS(-DHAVE_AVCODEC_DECODE_AUDIO4)
  ENDIF(HAVE_AVCODEC_DECODE_AUDIO4)
  IF(HAVE_AVIO_ALLOC_CONTEXT)
    ADD_DEFINITIONS(-DHAVE_AVIO_ALLOC_CONTEXT)
  ENDIF(HAVE_AVIO_ALLOC_CONTEXT)
  IF(HAVE_AVCODEC_ALLOC_CONTEXT3)
    ADD_DEFINITIONS(-DHAVE_AVCODEC_ALLOC_CONTEXT3)
  ENDIF(HAVE_AVCODEC_ALLOC_CONTEXT3)
  IF(HAVE_AVCODEC_OPEN2)
    ADD_DEFINITIONS(-DHAVE_AVCODEC_OPEN2)
  ENDIF(HAVE_AVCODEC_OPEN2)
  IF(HAVE_AVFORMAT_CLOSE_INPUT)
    ADD_DEFINITIONS(-DHAVE_AVFORMAT_CLOSE_INPUT)
  ENDIF(HAVE_AVFORMAT_CLOSE_INPUT)
  IF(HAVE_AVFORMAT_FIND_STREAM_INFO)
    ADD_DEFINITIONS(-DHAVE_AVFORMAT_FIND_STREAM_INFO)
  ENDIF(HAVE_AVFORMAT_FIND_STREAM_INFO)
  IF(HAVE_AVCODECID)
    ADD_DEFINITIONS(-DHAVE_AVCODECID)
  ENDIF(HAVE_AVCODECID)
  IF(LIBAVRESAMPLE_FOUND)
    ADD_DEFINITIONS(-DHAVE_LIBAVRESAMPLE)
  ENDIF(LIBAVRESAMPLE_FOUND)
  ADD_DEFINITIONS(-DENABLE_LIBAVCODEC)
ENDIF(ENABLE_LIBAVCODEC)

pkg_check_modules(GLIB REQUIRED glib-2.0)
pkg_check_modules(GTHREAD REQUIRED gthread-2.0)
pkg_check_modules(GMODULE REQUIRED gmodule-2.0)
pkg_check_modules(GLIBMM REQUIRED glibmm-2.4)
INCLUDE_DIRECTORIES(${GLIBMM_INCLUDE_DIRS})
pkg_check_modules(GTK REQUIRED gtk+-2.0)

IF(GLIBMM_FOUND AND NOT(GLIBMM_VERSION VERSION_LESS 2.31.0))
  ADD_DEFINITIONS(-DHAVE_NEW_GLIBMM_THREAD_API)
ENDIF(GLIBMM_FOUND AND NOT(GLIBMM_VERSION VERSION_LESS 2.31.0))
IF(GTHREAD_FOUND AND (GTHREAD_VERSION VERSION_LESS 2.32.0))
  #The definition was not removed in 2.32.0, but its
  #deprecated since then and makes problems in the win32 build
  ADD_DEFINITIONS(-DHAVE_G_THREAD_INIT)
ENDIF(GTHREAD_FOUND AND (GTHREAD_VERSION VERSION_LESS 2.32.0))

INCLUDE_DIRECTORIES(${LLVM_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${FREETYPE_INCLUDE_DIRS})
IF(ENABLE_GLES2)
	INCLUDE_DIRECTORIES(${GLES2_INCLUDE_DIRS})
ELSE()
	INCLUDE_DIRECTORIES(${OPENGL_INCLUDE_DIR})
	INCLUDE_DIRECTORIES(${GLEW_INCLUDE_DIR})
ENDIF(ENABLE_GLES2)
INCLUDE_DIRECTORIES(${PCRE_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${JPEG_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${XMLPP_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${CAIRO_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${GLIB_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${GTK_INCLUDE_DIRS})

IF(ENABLE_LIBAVCODEC)
  INCLUDE_DIRECTORIES(${FFMPEG_INCLUDE_DIRS})
  SET(OPTIONAL_LIBRARIES ${OPTIONAL_LIBRARIES} ${FFMPEG_LIBRARIES})
ENDIF(ENABLE_LIBAVCODEC)	

IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  IF(MINGW)
    SET(CMAKE_EXE_LINKER_FLAGS "-Wl,--as-needed,--no-undefined -mthreads ${CMAKE_EXE_LINKER_FLAGS}")
    SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--as-needed,--no-undefined,--enable-stdcall-fixup -mthreads ${CMAKE_SHARED_LINKER_FLAGS}")
  ELSE()
    SET(CMAKE_EXE_LINKER_FLAGS "-Wl,--as-needed -Wl,--no-undefined ${CMAKE_EXE_LINKER_FLAGS}")
    SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--as-needed,-z,noexecstack,--no-undefined ${CMAKE_SHARED_LINKER_FLAGS}")
  ENDIF()
  SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "-s")
  SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "-s")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
IF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  SET(CMAKE_EXE_LINKER_FLAGS "-Wl,--as-needed -Wl,--no-undefined ${CMAKE_EXE_LINKER_FLAGS}")
  SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--as-needed,-z,noexecstack,--no-undefined ${CMAKE_SHARED_LINKER_FLAGS}")
  SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "-s")
  SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "-s")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")

LINK_DIRECTORIES(${LLVM_LIB_DIR})

IF(ENABLE_CURL)
  pkg_check_modules(CURL REQUIRED libcurl)
  INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIR})
  IF(STATICDEPS)
    SET(OPTIONAL_LIBRARIES ${OPTIONAL_LIBRARIES} ${CURL_STATIC_LIBRARIES} crypt32)
	ADD_DEFINITIONS(-DCURL_STATICLIB)
  ELSE()
    SET(OPTIONAL_LIBRARIES ${OPTIONAL_LIBRARIES} ${CURL_LIBRARIES})
  ENDIF()
  ADD_DEFINITIONS(-DENABLE_CURL)
ENDIF(ENABLE_CURL)

IF(ENABLE_RTMP)
  pkg_check_modules(RTMP REQUIRED librtmp)
  SET(OPTIONAL_LIBRARIES ${OPTIONAL_LIBRARIES} ${RTMP_LIBRARIES})
  ADD_DEFINITIONS(-DENABLE_RTMP)
ENDIF(ENABLE_RTMP)

IF(ENABLE_PROFILING)
  ADD_DEFINITIONS(-DPROFILING_SUPPORT)
ENDIF(ENABLE_PROFILING)

IF(ENABLE_MEMORY_USAGE_PROFILING)
	ADD_DEFINITIONS(-DMEMORY_USAGE_PROFILING)
ENDIF(ENABLE_MEMORY_USAGE_PROFILING)

# Compiler defaults flags for different profiles
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  IF(MINGW)
    SET(CMAKE_CXX_FLAGS
	   "${CMAKE_CXX_FLAGS} -mms-bitfields -mthreads -fexceptions -Wall -Wnon-virtual-dtor -Woverloaded-virtual -pipe -std=c++0x -Wdisabled-optimization -Wextra -Wno-unused-parameter -Wno-invalid-offsetof")
  ELSE()
   SET(CMAKE_CXX_FLAGS
	  "${CMAKE_CXX_FLAGS} -Wall -Wnon-virtual-dtor -Woverloaded-virtual -pipe -fvisibility=hidden -fvisibility-inlines-hidden -std=c++11 -Wdisabled-optimization -Wextra -Wno-unused-parameter -Wno-invalid-offsetof")
  ENDIF()
  SET(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DEXPENSIVE_DEBUG")
  SET(CMAKE_CXX_FLAGS_PROFILE "-g -pg -O2")
  SET(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
  SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-g -O2 -DNDEBUG")
  SET(CMAKE_CXX_FLAGS_LEANDEBUG "-g -O2")
  ADD_DEFINITIONS(-DLS_DATADIR="${LSDATADIR}" -DGNASH_PATH="${GNASH_EXE_PATH}" -DPRIVATELIBDIR="${PRIVATELIBDIR}")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
IF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  SET(CMAKE_CXX_FLAGS
	  "${CMAKE_CXX_FLAGS} -Wall -Wnon-virtual-dtor -Woverloaded-virtual -pipe -fvisibility=hidden -fvisibility-inlines-hidden -std=c++11 -Wdisabled-optimization -Wextra -Wno-unused-parameter -Wno-invalid-offsetof")
  SET(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DEXPENSIVE_DEBUG")
  SET(CMAKE_CXX_FLAGS_PROFILE "-g -pg -O2")
  SET(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
  SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-g -O2 -DNDEBUG")
  SET(CMAKE_CXX_FLAGS_LEANDEBUG "-g -O2")
  ADD_DEFINITIONS(-DLS_DATADIR="${LSDATADIR}" -DGNASH_PATH="${GNASH_EXE_PATH}" -DPRIVATELIBDIR="${PRIVATELIBDIR}")
  ADD_DEFINITIONS(-DUSE_CLANG)
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")

IF(MSVC)
  ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS -DVPCOMPAT -DPTW32_STATIC_LIB -DPCRE_STATIC)
ENDIF(MSVC)

IF(STATICDEPS)
  ADD_DEFINITIONS(-DLIBXML_STATIC -DGLEW_STATIC)
ENDIF()

# Check if the system have atomic header ( gcc >= 4.6 )
INCLUDE (CheckIncludeFileCXX)
CHECK_INCLUDE_FILE_CXX (atomic HAVE_ATOMIC)
IF(HAVE_ATOMIC)
  ADD_DEFINITIONS(-DHAVE_ATOMIC)
ENDIF(HAVE_ATOMIC)

# Setting the output directories, so we can build all profiles without mixmatching
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/bin")
  SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/lib${LIB_SUFFIX}")
  #SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/objs/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/lib${LIB_SUFFIX}" CACHE PATH "Static libraries output directory")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
IF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/bin")
  SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/lib${LIB_SUFFIX}")
  #SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/objs/${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}/lib${LIB_SUFFIX}" CACHE PATH "Static libraries output directory")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")

#Install icons and desktop file
if(UNIX)
  FOREACH(i 16 22 24 32 36 48 64 72 96 128 192 256 )
    INSTALL(FILES media/lightspark-ico-${i}x${i}.png
            DESTINATION ${DATADIR}/icons/hicolor/${i}x${i}/apps
            RENAME lightspark.png
           )
  ENDFOREACH(i)
  INSTALL(FILES media/lightspark-ico.svg
          DESTINATION ${DATADIR}/icons/hicolor/scalable/apps
          RENAME lightspark.svg
         )
  INSTALL(FILES media/lightspark.desktop DESTINATION ${DATADIR}/applications )
endif(UNIX)

# generate locales for gettext
GETTEXT_CREATE_TRANSLATIONS(i18n/lightspark.pot ALL i18n/pl.po i18n/fr.po
        i18n/zh_CN.po)

if(UNIX)
	INSTALL(FILES etc/xdg/lightspark.conf
		      DESTINATION ${ETCDIR}/xdg/)
ELSE()
  INSTALL(FILES COPYING DESTINATION "." RENAME COPYING.txt)
  INSTALL(FILES COPYING.LESSER DESTINATION "." RENAME COPYING.LESSER.txt)
endif(UNIX)

SUBDIRS(src)

#-- CPack setup - use 'make package' to build
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Lightspark is an LGPLv3 licensed cross-platform Flash player and browser plugin")
SET(CPACK_PACKAGE_VENDOR "Lightspark Team")
SET(CPACK_PACKAGE_NAME "Lightspark")
SET(CPACK_PACKAGE_VERSION_MAJOR "${MAJOR_VERSION}")
SET(CPACK_PACKAGE_VERSION_MINOR "${MINOR_VERSION}")
IF(SUBPATCH_VERSION EQUAL "0")
  SET(CPACK_PACKAGE_VERSION_PATCH "${PATCH_VERSION}${GIT_VERSION}")
ELSE()
  SET(CPACK_PACKAGE_VERSION_PATCH "${PATCH_VERSION}.${SUBPATCH_VERSION}${GIT_VERSION}")
ENDIF()
SET(CPACK_NSIS_MENU_LINKS "")
#In release builds, we already strip symbols (see LDFLAGS), and compress by upx
#Therefore we cannot strip afterwards
SET(CPACK_STRIP_FILES FALSE)
SET(CPACK_RESOURCE_FILE_LICENSE ${PROJECT_SOURCE_DIR}/COPYING)
SET(CPACK_NSIS_HELP_LINK "http:////www.lightspark.org")
SET(CPACK_NSIS_URL_INFO_ABOUT "http:////www.lightspark.org")
SET(CPACK_PACKAGE_ICON ${PROJECT_SOURCE_DIR}/media/lightspark-logo.svg)
IF(WIN32)
  SET(CPACK_PACKAGE_INSTALL_DIRECTORY "Lightspark")
  SET(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1' 'Description' 'Lightspark'
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1' 'ProductName' 'Lightspark'
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1' 'Vendor' 'Lightspark Team'
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1' 'Version' '1'
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1' 'Path' '$INSTDIR\\\\nplightsparkplugin.dll'
  WriteRegStr HKCU 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1\\\\MimeTypes\\\\application/x-shockwave-flash' '' ''
  ")
  SET(CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS "
  DeleteRegKey HKLM 'Software\\\\MozillaPlugins\\\\@lightspark.github.com/Lightspark;version=1'
  ")
ENDIF(WIN32)
#Must come after the definitions
INCLUDE(CPack)

# Copyright (C) 2017 Franklin "Snaipe" Mathieu.
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the LICENSE file distributed with fmem.

cmake_minimum_required (VERSION 3.16)

file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" FMEM_VERSION_LINE LIMIT_COUNT 1)
string(STRIP "${FMEM_VERSION_LINE}" FMEM_VERSION)

project (fmem VERSION ${FMEM_VERSION} LANGUAGES C)

list (APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/.cmake")

include (CheckSymbolExists)
include (CheckCSourceCompiles)
include (GNUInstallDirs)
include (CMakePackageConfigHelpers)

set (FMEM_SOVERSION 0)

if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wall -Wextra -pedantic")
elseif (CMAKE_C_COMPILER_ID MATCHES "Clang")
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wall -Wextra -pedantic")
endif ()

list (APPEND CMAKE_REQUIRED_DEFINITIONS
  -D_GNU_SOURCE
  -D_CRT_RAND_S
  -DVC_EXTRALEAN
  -DWIN32_LEAN_AND_MEAN)

check_symbol_exists (open_memstream stdio.h HAVE_OPEN_MEMSTREAM)
check_symbol_exists (fopencookie stdio.h HAVE_FOPENCOOKIE)
check_symbol_exists (funopen stdio.h HAVE_FUNOPEN)
check_symbol_exists (tmpfile stdio.h HAVE_TMPFILE)
check_symbol_exists (rand_s stdlib.h HAVE_WINAPI_RAND_S)
check_symbol_exists (CreateFile windows.h HAVE_WINAPI_CREATEFILE)
check_symbol_exists (CloseHandle windows.h HAVE_WINAPI_CLOSEHANDLE)
check_symbol_exists (GetFileSize windows.h HAVE_WINAPI_GETFILESIZE)
check_symbol_exists (CreateFileMapping windows.h HAVE_WINAPI_CREATEFILEMAPPING)
check_symbol_exists (MapViewOfFile windows.h HAVE_WINAPI_MAPVIEWOFFILE)
check_symbol_exists (UnmapViewOfFile windows.h HAVE_WINAPI_UNMAPVIEWOFFILE)
check_symbol_exists (GetTempPath windows.h HAVE_WINAPI_GETTEMPPATH)
check_symbol_exists (_open_osfhandle io.h HAVE_WINAPI_OPEN_OSFHANDLE)
check_symbol_exists (_get_osfhandle io.h HAVE_WINAPI_GET_OSFHANDLE)
check_symbol_exists (_fdopen stdio.h HAVE_WINAPI_FDOPEN)
check_symbol_exists (_fileno stdio.h HAVE_WINAPI_FILENO)
check_symbol_exists (_close io.h HAVE_WINAPI_CLOSE)

set (SOURCES)

if (HAVE_OPEN_MEMSTREAM)
  list (APPEND SOURCES src/fmem-open_memstream.c)
elseif (HAVE_FOPENCOOKIE)
  list (APPEND SOURCES
    src/alloc.c
    src/alloc.h
    src/fmem-fopencookie.c)
elseif (HAVE_FUNOPEN)
  list (APPEND SOURCES
    src/alloc.c
    src/alloc.h
    src/fmem-funopen.c)
elseif (HAVE_WINAPI_CREATEFILE
    AND HAVE_WINAPI_CLOSEHANDLE
    AND HAVE_WINAPI_GETFILESIZE
    AND HAVE_WINAPI_CREATEFILEMAPPING
    AND HAVE_WINAPI_MAPVIEWOFFILE
    AND HAVE_WINAPI_UNMAPVIEWOFFILE
    AND HAVE_WINAPI_GETTEMPPATH
    AND HAVE_WINAPI_FDOPEN
    AND HAVE_WINAPI_FILENO
    AND HAVE_WINAPI_CLOSE
    AND HAVE_WINAPI_OPEN_OSFHANDLE
    AND HAVE_WINAPI_GET_OSFHANDLE
    AND HAVE_WINAPI_RAND_S)
  list (APPEND SOURCES src/fmem-winapi-tmpfile.c)
elseif (HAVE_TMPFILE)
  list (APPEND SOURCES src/fmem-tmpfile.c)
else ()
  message (FATAL_ERROR "No memory stream implementation found")
endif ()

add_library (fmem ${SOURCES})
add_library (fmem::fmem ALIAS fmem)

target_include_directories (fmem
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/gen>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src)

set_target_properties (fmem PROPERTIES
  VERSION ${FMEM_VERSION}
  SOVERSION ${FMEM_SOVERSION}
  POSITION_INDEPENDENT_CODE ON)

set (CMAKE_REQUIRED_DEFINITIONS -fvisibility=hidden)
check_c_source_compiles (
  "__attribute__((visibility(\"default\"))) int main(void) { return 0; }"
  CC_HAVE_VISIBILITY)
set (CMAKE_REQUIRED_DEFINITIONS)

get_property (FMEM_LIBTYPE
  TARGET fmem
  PROPERTY TYPE)

if ("${FMEM_LIBTYPE}" STREQUAL "SHARED_LIBRARY")
  if (WIN32)
    set (EXPORT_MACROS
"#ifdef FMEM_BUILD_LIBRARY
# define FMEM_API __declspec(dllexport)
#else /* !FMEM_BUILD_LIBRARY */
# define FMEM_API __declspec(dllimport)
#endif /* !FMEM_BUILD_LIBRARY */")
    target_compile_definitions (fmem PRIVATE FMEM_BUILD_LIBRARY)
  elseif (CC_HAVE_VISIBILITY)
    set (EXPORT_MACROS "#define FMEM_API __attribute__((visibility(\"default\")))")
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
  endif ()
else ()
  set (EXPORT_MACROS "#define FMEM_API")
endif ()

configure_file (
  ${PROJECT_SOURCE_DIR}/include/fmem.h.in
  ${PROJECT_BINARY_DIR}/gen/fmem.h
  @ONLY)

# pkg-config file
configure_file (
  ${PROJECT_SOURCE_DIR}/fmem.pc.in
  ${PROJECT_BINARY_DIR}/fmem.pc
  @ONLY)

# CMake package config
install(TARGETS fmem
  EXPORT fmemTargets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(FILES
   ${PROJECT_BINARY_DIR}/gen/fmem.h
   DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT fmemTargets
  FILE fmemTargets.cmake
  NAMESPACE fmem::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fmem)

configure_package_config_file(
  ${PROJECT_SOURCE_DIR}/cmake/fmemConfig.cmake.in
  ${PROJECT_BINARY_DIR}/fmemConfig.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fmem)

write_basic_package_version_file(
  ${PROJECT_BINARY_DIR}/fmemConfigVersion.cmake
  VERSION ${FMEM_VERSION}
  COMPATIBILITY SameMajorVersion)

install(FILES
  ${PROJECT_BINARY_DIR}/fmemConfig.cmake
  ${PROJECT_BINARY_DIR}/fmemConfigVersion.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fmem)

install(FILES
  ${PROJECT_BINARY_DIR}/fmem.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

include (CTest)

if (BUILD_TESTING)
  find_package (Criterion REQUIRED)
  add_subdirectory (test)
endif ()

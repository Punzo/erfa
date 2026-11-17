# CMake build of ERFA (Essential Routines for Fundamental Astronomy,
# https://github.com/liberfa/erfa, BSD-3), used by SlicerAstro for the
# celestial frame conversions (FK4, FK5, ICRS, Galactic) and the velocity of
# the Earth (spectral reference frames). Upstream ERFA ships autotools and
# meson builds only; this file mirrors the meson build (sources, version,
# installed headers).
cmake_minimum_required(VERSION 3.16)

# Version of the meson build
file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/meson.build _erfa_version_line
  REGEX "^[ \t]*version:[ \t]*'[0-9.]+'" LIMIT_COUNT 1)
string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+" _erfa_version "${_erfa_version_line}")
project(erfa VERSION ${_erfa_version} LANGUAGES C)

option(BUILD_TESTING "Build the ERFA tests" OFF)
set(ERFA_SOFA_VERSION "20231011")

file(GLOB ERFA_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
list(FILTER ERFA_SOURCES EXCLUDE REGEX "/t_erfa_c[^/]*\\.c$")

# config.h of the meson build (version information of erfaversion.c)
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/config.h
  "#define PACKAGE_VERSION \"${PROJECT_VERSION}\"\n"
  "#define PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR}\n"
  "#define PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR}\n"
  "#define PACKAGE_VERSION_MICRO ${PROJECT_VERSION_PATCH}\n"
  "#define SOFA_VERSION \"${ERFA_SOFA_VERSION}\"\n")

add_library(erfa ${ERFA_SOURCES})
target_compile_definitions(erfa PRIVATE HAVE_CONFIG_H)
if(MSVC)
  target_compile_definitions(erfa PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()
target_include_directories(erfa
  PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
  PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src> $<INSTALL_INTERFACE:include>)
find_library(ERFA_MATH_LIBRARY m)
if(ERFA_MATH_LIBRARY)
  target_link_libraries(erfa PRIVATE ${ERFA_MATH_LIBRARY})
endif()
# libtool versioning of the meson build (current 9, revision 1, age 8)
set_target_properties(erfa PROPERTIES
  VERSION 1.8.1
  SOVERSION 1
  C_STANDARD 99
  POSITION_INDEPENDENT_CODE ON
  WINDOWS_EXPORT_ALL_SYMBOLS ON)

install(TARGETS erfa
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)
install(FILES src/erfa.h src/erfam.h src/erfaextra.h src/erfadatextra.h DESTINATION include)

if(BUILD_TESTING)
  enable_testing()
  foreach(test t_erfa_c t_erfa_c_extra)
    add_executable(${test} src/${test}.c)
    target_link_libraries(${test} PRIVATE erfa)
    target_compile_definitions(${test} PRIVATE HAVE_CONFIG_H)
    target_include_directories(${test} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME ${test} COMMAND ${test})
  endforeach()
endif()

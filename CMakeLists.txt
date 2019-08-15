
# CMake Configuration and build added by Lianhua Zhu -- May, 2014
# shamelessly adapted from https://github.com/jacobwilliams/json-fortran 

## This is the original file header from the json-fortran project
# Copy right (c) 2014, Izaak Beekman
# All rights reserved.

# This file is contributed to the Json-fortran project, and
# is licensed under the terms of Json-fortran license. The Json-fortran
# license is located in the LICENSE file which must be distributed with
# this software. The contributing author, Izaak Beekman, retains all
# rights permitted by the terms of the Json-fortran license.
## End of the original header from json-fortran project

cmake_minimum_required ( VERSION 2.8.8 FATAL_ERROR )

# Use MSVS folders to organize projects on windows
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(PROJECT_DESCRIPTION "A Fortran GNUPLOT API")
set(PROJECT_URL "https://github.com/kookma/ogpf")

# Set the type/configuration of build to perform
set ( CMAKE_CONFIGURATION_TYPES "Debug" "Release" "MinSizeRel" "RelWithDebInfo" )
set ( CMAKE_BUILD_TYPE "Release"
  CACHE STRING "Select which configuration to build." )

enable_language ( Fortran )
include ( "cmake/pickFortranCompilerFlags.cmake" )

set_property ( CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ${CMAKE_CONFIGURATION_TYPES} )

# Check for in-source builds and error out if found
# Provides an advanced option to allow in source builds
include ( "cmake/checkOutOfSource.cmake" )

#---------------------
# Declare project name
#---------------------
project ( ogpf NONE )

if(CMAKE_Fortran_COMPILER_ID STREQUAL GNU)
  option(OGPF_FORTRAN_USE_OpenCoarrays
    "Build OGPF-Fortran with support for linking against OpenCoarray programs" OFF)
endif()
if(OGPF_FORTRAN_USE_OpenCoarrays)
  find_package(OpenCoarrays)
endif()

#---------------------
# Real and Integer kinds
#---------------------

SET(OGPF_REAL_KIND "REAL64" CACHE STRING "Real kind parameter")
SET_PROPERTY(CACHE OGPF_REAL_KIND PROPERTY STRINGS REAL32 REAL64 REAL128)
if(${OGPF_REAL_KIND} MATCHES "REAL32")
  add_definitions(-DREAL32)
elseif(${OGPF_REAL_KIND} MATCHES "REAL64")
  add_definitions(-DREAL64)
elseif(${OGPF_REAL_KIND} MATCHES "REAL128")
  add_definitions(-DREAL128)
endif()

#----------------------------------
# Get version (semantic versioning)
# C.F. semver.org
#----------------------------------
file ( STRINGS "${CMAKE_SOURCE_DIR}/.VERSION" VERSION )
string( REPLACE "." ";" VERSION_LIST ${VERSION} )
list(GET VERSION_LIST 0 VERSION_MAJOR)
list(GET VERSION_LIST 1 VERSION_MINOR)
list(GET VERSION_LIST 2 VERSION_PATCH)
set(PROJECT_VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
message ( STATUS "CMake build configuration for ogpf ${PROJECT_VERSION}" )

#-------------------------------------
# Collect source files for the library
#-------------------------------------
set ( OGPF_LIB_SRCS  ogpf.f90)
file ( GLOB OGPF_TEST_SRCS "demo.f90" )

#-----------------------------------------
# Collect all the mod files into their own
# directory to ease installation issues
#-----------------------------------------
set ( MODULE_DIR "${CMAKE_BINARY_DIR}/include" )

#-------------------------------------
# Define where our files get installed
#-------------------------------------
set ( USE_GNU_INSTALL_CONVENTION FALSE
  CACHE BOOL
  "Install library, module file and documentation to standard GNU locations. Do not use this if supporting multiple Fortran compilers" )

# Set the package name to be specific to the compiler used, so that
# versions compiled with different compilers can be installed in parallel
string ( TOLOWER ${CMAKE_PROJECT_NAME}-${CMAKE_Fortran_COMPILER_ID} PACKAGE_NAME )
set ( PACKAGE_VERSION "${PACKAGE_NAME}-${VERSION}" )

if (USE_GNU_INSTALL_CONVENTION)
  include(GNUInstallDirs)
  set ( INSTALL_MOD_DIR "${CMAKE_INSTALL_INCLUDEDIR}" )
  set ( INSTALL_LIB_DIR "${CMAKE_INSTALL_LIBDIR}")
  set( ABS_LIB_INSTALL_DIR "\${CMAKE_INSTALL_FULL_LIBDIR}" )
else ()
  # Most of this could be 'wrong' for Windows/Cygwin
  set ( INSTALL_MOD_DIR    "${PACKAGE_VERSION}/lib" )
  set ( INSTALL_LIB_DIR    "${INSTALL_MOD_DIR}" )
  set( ABS_LIB_INSTALL_DIR "\${CMAKE_INSTALL_PREFIX}/${INSTALL_LIB_DIR}" )
endif ()


# Put package export CMake files where they can be found
# use `find_package ( jsonfortran-${CMAKE_Fortran_COMPILER_ID} <version> REQUIRED )`
if (USE_GNU_INSTALL_CONVENTION)
  set ( EXPORT_INSTALL_DIR "${INSTALL_LIB_DIR}/cmake/${PACKAGE_VERSION}" )
else ()
  set ( EXPORT_INSTALL_DIR "${PACKAGE_VERSION}/cmake" )
endif ()

if ( "${CMAKE_SYSTEM_NAME}" MATCHES "Darwin" )
  set ( ENABLE_DYLIBS_USE_RPATH TRUE CACHE BOOL
    "Enable @rpath install name for dylibs" )
  mark_as_advanced ( ENABLE_DYLIBS_USE_RPATH )
endif ()

if ( ENABLE_DYLIBS_USE_RPATH )
  set ( CMAKE_MACOSX_RPATH TRUE )
else ()
  set ( CMAKE_INSTALL_NAME_DIR
    "${ABS_LIB_INSTALL_DIR}" )
endif ()


#---------------------------------------------
# Build a shared and static library by default
#---------------------------------------------

set ( LIB_NAME ${CMAKE_PROJECT_NAME} )
add_library ( ${LIB_NAME}        SHARED ${OGPF_LIB_SRCS} )
add_library ( ${LIB_NAME}-static STATIC ${OGPF_LIB_SRCS} )

if(OGPF_USE_OpenCoarrays)
  target_link_libraries(${LIB_NAME}
    PRIVATE OpenCoarrays::caf_mpi_static)
  target_link_libraries(${LIB_NAME}-static
    PRIVATE OpenCoarrays::caf_mpi_static)
endif()

set_target_properties ( ${LIB_NAME}-static
  PROPERTIES
  OUTPUT_NAME ${LIB_NAME}
  if(NOT MSVC_IDE)
  PREFIX lib
  endif()
  VERSION ${VERSION}
  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
  Fortran_MODULE_DIRECTORY ${MODULE_DIR} )
set_target_properties ( ${LIB_NAME}
  PROPERTIES
  OUTPUT_NAME ${LIB_NAME}
  if(NOT MSVC_IDE)
  PREFIX lib
  endif()
  SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR}
  VERSION ${VERSION}
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
  Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR} )

#-------------------------
# Perform the installation
#-------------------------

install ( TARGETS ${LIB_NAME} ${LIB_NAME}-static
  EXPORT ${PACKAGE_NAME}-targets
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}" )

# Code to  fix the dylib install name on Mac.
include ( cmake/fixupInstallNameDir.cmake )

set(MOD_DIR_TO_INSTALL "${MODULE_DIR}")
set(MOD_DESTINATION_DIR "${INSTALL_MOD_DIR}")
install(
  CODE "file(GLOB_RECURSE MODULE_FILES \"${MOD_DIR_TO_INSTALL}/*.mod\")"
  CODE "file(GLOB_RECURSE SUBMOD_FILES \"${MOD_DIR_TO_INSTALL}/*.smod\")"
  CODE "file(INSTALL \${MODULE_FILES} DESTINATION \"\${CMAKE_INSTALL_PREFIX}/${MOD_DESTINATION_DIR}\")"
  CODE "file(INSTALL \${SUBMOD_FILES} DESTINATION \"\${CMAKE_INSTALL_PREFIX}/${MOD_DESTINATION_DIR}\")"
  )

#------------------------------------------
# Add portable unistall command to makefile
#------------------------------------------
# Adapted from the CMake Wiki FAQ
configure_file ( "${CMAKE_SOURCE_DIR}/cmake/uninstall.cmake.in" "${CMAKE_BINARY_DIR}/uninstall.cmake"
    @ONLY)

add_custom_target ( uninstall
    COMMAND ${CMAKE_COMMAND} -P "${CMAKE_BINARY_DIR}/uninstall.cmake" )

#-----------------------------------------------------
# Publicize installed location to other CMake projects
#-----------------------------------------------------
install ( EXPORT ${PACKAGE_NAME}-targets DESTINATION "${EXPORT_INSTALL_DIR}" )

include ( CMakePackageConfigHelpers ) # Standard CMake module
write_basic_package_version_file( "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake"
  VERSION ${VERSION}
  COMPATIBILITY SameMajorVersion )

# provides COMPILER_CONSISTENCY_CHECK
include ( cmake/FCompilerConsistencyCheck.cmake )

# install package config file
configure_package_config_file (
  "${CMAKE_SOURCE_DIR}/cmake/pkg/${CMAKE_PROJECT_NAME}-config.cmake.in"
  "${CMAKE_BINARY_DIR}/pkg/${PACKAGE_NAME}-config.cmake"
  INSTALL_DESTINATION "${EXPORT_INSTALL_DIR}"
  PATH_VARS EXPORT_INSTALL_DIR INSTALL_MOD_DIR )

# Install the config and version files so that we can find this project with others
install ( FILES
  "${CMAKE_BINARY_DIR}/pkg/${PACKAGE_NAME}-config.cmake"
  "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake"
  DESTINATION "${EXPORT_INSTALL_DIR}" )

#----------------------------------------------
# Make build tree targets accessible for import
#----------------------------------------------
export ( TARGETS ${LIB_NAME} ${LIB_NAME}-static FILE ${PACKAGE_NAME}-targets.cmake )

# build tree package config file, NOT installed
configure_file (
  "${CMAKE_SOURCE_DIR}/cmake/${CMAKE_PROJECT_NAME}-config.cmake.in"
  "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config.cmake"
  @ONLY )

set ( ENABLE_BUILD_TREE_EXPORT FALSE CACHE BOOL
  "Add the ${PACKAGE_NAME} build tree to the CMake package registry?" )
if ( ENABLE_BUILD_TREE_EXPORT )
  export ( PACKAGE ${PACKAGE_NAME} )
endif ()

# pkg-config stuff
configure_file(
   "${CMAKE_CURRENT_SOURCE_DIR}/ogpf.pc.cmake.in"
   "${CMAKE_CURRENT_BINARY_DIR}/ogpf.pc"
   @ONLY
)
install(FILES
   "${CMAKE_CURRENT_BINARY_DIR}/ogpf.pc"
   DESTINATION "${INSTALL_LIB_DIR}/pkgconfig"
)

if(MSVC_IDE)
   INCLUDE_DIRECTORIES("src")
   SET(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fpp")
endif()
cmake_minimum_required(VERSION 3.7)

project(asio_vendor)

find_package(ament_cmake REQUIRED)

# Standalone Asio is a header-only C++ networking library. Fast-DDS consumes it
# (eprosima_find_thirdparty(Asio asio VERSION 1.13.0)) and REQUIRES >= 1.13.0;
# its FindAsio.cmake locates it by finding asio.hpp on the include path and
# reading asio/version.hpp. VxWorks has no system libasio-dev, so this vendor
# fetches the pristine upstream source and installs the (patched) headers into
# the shared install prefix.
#
# Version 1.30.2 matches Ubuntu 26.04 (resolute) libasio-dev AND satisfies the
# Fast-DDS >= 1.13 floor. Being header-only, "cross-compiling" is a no-op: we do
# not configure/build, we just patch and stage the include/ tree.
#
# The VxWorks source patches travel with this vendor in ./patches and are
# applied on top of the fetched upstream source by the ExternalProject
# PATCH_COMMAND below (the fetched asio tree only exists at build time, so the
# patches cannot be applied to the checked-out package source directly).
# Override the location with -DASIO_VENDOR_PATCH_DIR if needed.

option(FORCE_BUILD_VENDOR_PKG "Build Asio from source" OFF)

set(ASIO_VERSION_TAG "asio-1-30-2")  # upstream git tag for 1.30.2

# Where this vendor keeps the VxWorks source patches.
if(NOT DEFINED ASIO_VENDOR_PATCH_DIR)
  set(ASIO_VENDOR_PATCH_DIR "${CMAKE_CURRENT_SOURCE_DIR}/patches")
endif()

if(NOT FORCE_BUILD_VENDOR_PKG)
  # A system/overlay Asio >= 1.13 would satisfy Fast-DDS; on VxWorks there is
  # none, so this normally falls through to the source build.
  find_path(Asio_INCLUDE_DIR NAMES asio.hpp)
  if(Asio_INCLUDE_DIR AND EXISTS "${Asio_INCLUDE_DIR}/asio/version.hpp")
    file(READ "${Asio_INCLUDE_DIR}/asio/version.hpp" _asio_ver_hpp)
    string(REGEX MATCH "#define ASIO_VERSION ([0-9]+)" _m "${_asio_ver_hpp}")
    if(CMAKE_MATCH_1 AND NOT (CMAKE_MATCH_1 LESS 101300))  # 1.13.0 == 101300
      set(Asio_FOUND TRUE)
    endif()
  endif()
endif()

macro(build_asio)
  include(ExternalProject)

  find_program(PATCH_EXECUTABLE patch REQUIRED)

  # Collect the vendor's VxWorks patches (0001-*, 0002-*, ...) and build a chained
  # PATCH_COMMAND that applies each with patch -p1 onto the fetched source.
  file(GLOB _asio_patches "${ASIO_VENDOR_PATCH_DIR}/*.patch")
  list(SORT _asio_patches)
  if(NOT _asio_patches)
    message(FATAL_ERROR
      "asio_vendor: no patches found in ${ASIO_VENDOR_PATCH_DIR}. "
      "The vendor must ship the VxWorks source patches in ./patches "
      "(or pass -DASIO_VENDOR_PATCH_DIR).")
  endif()

  set(_patch_cmd "")
  foreach(_p IN LISTS _asio_patches)
    if(_patch_cmd STREQUAL "")
      set(_patch_cmd ${PATCH_EXECUTABLE} -p1 -i ${_p})
    else()
      list(APPEND _patch_cmd COMMAND ${PATCH_EXECUTABLE} -p1 -i ${_p})
    endif()
  endforeach()

  # Header-only: no configure/compile step. Fetch the tagged source, run the
  # VxWorks patches, then install the include/ tree in the install step.
  externalproject_add(asio_external
    URL https://github.com/chriskohlhoff/asio/archive/refs/tags/${ASIO_VERSION_TAG}.tar.gz
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    TIMEOUT 600
    PATCH_COMMAND ${_patch_cmd}
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
  )

  # The standalone headers live under asio/include/ in the extracted tree.
  externalproject_get_property(asio_external SOURCE_DIR)
  install(
    DIRECTORY ${SOURCE_DIR}/asio/include/
    DESTINATION include
    USE_SOURCE_PERMISSIONS
    PATTERN "Makefile.am" EXCLUDE
  )
endmacro()

if(NOT Asio_FOUND)
  message(STATUS "Did not find Asio >= 1.13, building standalone Asio ${ASIO_VERSION_TAG} from source")
  message(STATUS "asio_vendor: applying VxWorks patches from ${ASIO_VENDOR_PATCH_DIR}")
  build_asio()
endif()

ament_package()

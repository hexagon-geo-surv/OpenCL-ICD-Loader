cmake_minimum_required (VERSION 3.1)

# Include guard for including this project multiple times
if(TARGET OpenCL)
  return()
endif()

project (OpenCL-ICD-Loader
    VERSION 1.2
    LANGUAGES C)

find_package (Threads REQUIRED)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
# The option below allows building the ICD Loader library as a shared library
# (ON, default) or a static library (OFF).
#
# Khronos OpenCL Working Group strongly recommends building and using the ICD
# loader as a shared library due to the following benefits:
#
# 1. The shared library can be updated independent of the application. This
#    allows releasing new fixes and features in the ICD loader without updating
#    the application.
#
#    In rare cases when there are backward-incompatible changes to the ICD
#    loader (due to platform requirements, for instance), using a shared
#    library allows updating the library to make the transition seamless to
#    installed applications.
#
# 2. On platforms that require the ICD mechanism there are multiple vendors
#    shipping their OpenCL implementations. The vendor installers collaborate
#    to make sure that the installed ICD shared library version is suitable for
#    working with all vendor implementations installed on the system.
#
#    If applications statically link to ICD Loader then that version of the ICD
#    loader may not work with one or more installed vendor implementations.
#
# Using the OpenCL ICD loader as a static library is NOT recommended for
# end-user installations in general. However in some controlled environments it
# may be useful to simplify the build and distribution of the application. E.g.
# in test farms, or in cases where the end-user system configs are known in
# advance. Use it with discretion.
if(DEFINED BUILD_SHARED_LIBS)
  set(OPENCL_ICD_LOADER_BUILD_SHARED_LIBS_DEFAULT ${BUILD_SHARED_LIBS})
else()
  set(OPENCL_ICD_LOADER_BUILD_SHARED_LIBS_DEFAULT ON)
endif()
  option(OPENCL_ICD_LOADER_BUILD_SHARED_LIBS "Build OpenCL ICD Loader as shared library" ${OPENCL_ICD_LOADER_BUILD_SHARED_LIBS_DEFAULT})

# This option enables/disables support for OpenCL layers in the ICD loader.
# It is currently needed default while the specification is being formalized,
# and to study the performance impact.
option (ENABLE_OPENCL_LAYERS "Enable OpenCL Layers" ON)

include(CheckFunctionExists)
check_function_exists(secure_getenv HAVE_SECURE_GETENV)
check_function_exists(__secure_getenv HAVE___SECURE_GETENV)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/loader/icd_cmake_config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/icd_cmake_config.h)

set (OPENCL_ICD_LOADER_SOURCES
    loader/icd.c
    loader/icd.h
    loader/icd_dispatch.c
    loader/icd_dispatch.h
    loader/icd_dispatch_generated.c
    loader/icd_envvars.h
    loader/icd_platform.h)

if (WIN32)
    enable_language (CXX)
    list (APPEND OPENCL_ICD_LOADER_SOURCES
        loader/windows/adapter.h
        loader/windows/icd_windows.c
        loader/windows/icd_windows.h
        loader/windows/icd_windows_dxgk.c
        loader/windows/icd_windows_dxgk.h
        loader/windows/icd_windows_envvars.c
        loader/windows/icd_windows_hkr.c
        loader/windows/icd_windows_hkr.h
        loader/windows/icd_windows_apppackage.cpp
        loader/windows/icd_windows_apppackage.h
        loader/windows/OpenCL.def
        loader/windows/OpenCL.rc)
    # Only add the DXSDK include directory if the environment variable is
    # defined.  Since the DXSDK has merged into the Windows SDK, this is
    # only required in rare cases.
    if (DEFINED ENV{DXSDK_DIR} AND NOT (MINGW OR MSYS OR CYGWIN))
        include_directories ($ENV{DXSDK_DIR}/Include)
    endif ()
else ()
    list (APPEND OPENCL_ICD_LOADER_SOURCES
        loader/linux/icd_linux.c
        loader/linux/icd_linux_envvars.c
        loader/linux/icd_exports.map)
endif ()

set (OPENCL_ICD_LOADER_HEADERS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/inc" CACHE PATH "Path to OpenCL Headers")

if (${OPENCL_ICD_LOADER_BUILD_SHARED_LIBS})
  add_library (OpenCL SHARED ${OPENCL_ICD_LOADER_SOURCES})
else()
  add_library (OpenCL STATIC ${OPENCL_ICD_LOADER_SOURCES})
endif()

add_library (OpenCL::OpenCL ALIAS OpenCL)

set_target_properties (OpenCL PROPERTIES VERSION "1.2" SOVERSION "1")

if (WIN32)
    target_link_libraries (OpenCL PRIVATE cfgmgr32.lib runtimeobject.lib)

    option (OPENCL_ICD_LOADER_DISABLE_OPENCLON12 "Disable support for OpenCLOn12. Support for OpenCLOn12 should only be disabled when building an import lib to link with, and must be enabled when building an ICD loader for distribution!" OFF)
else()
    target_link_libraries (OpenCL PRIVATE ${CMAKE_THREAD_LIBS_INIT})
    if (NOT APPLE)
        set_target_properties (OpenCL PROPERTIES LINK_FLAGS "-Wl,--version-script -Wl,${CMAKE_CURRENT_SOURCE_DIR}/loader/linux/icd_exports.map")
        if (OPENCL_ICD_LOADER_PIC)
            set_target_properties(OpenCL PROPERTIES POSITION_INDEPENDENT_CODE ON)
        endif ()
    endif ()
endif ()

if (EXISTS ${OPENCL_ICD_LOADER_HEADERS_DIR}/CL/cl.h)
    message (STATUS "Defining OpenCL::Headers through OPENCL_ICD_LOADER_HEADERS_DIR")
    add_library (OpenCLHeaders INTERFACE)
    add_library (OpenCL::Headers ALIAS OpenCLHeaders)
    target_include_directories (OpenCLHeaders INTERFACE ${OPENCL_ICD_LOADER_HEADERS_DIR})
    target_include_directories (OpenCL PUBLIC $<BUILD_INTERFACE:${OPENCL_ICD_LOADER_HEADERS_DIR}>)
else ()
    if (NOT TARGET OpenCL::Headers)
        find_package (OpenCLHeaders REQUIRED)
    endif ()
    target_link_libraries (OpenCL PUBLIC OpenCL::Headers)
endif ()

target_compile_definitions (OpenCL
  PRIVATE
    CL_TARGET_OPENCL_VERSION=300
    $<$<BOOL:${ENABLE_OPENCL_LAYERS}>:CL_ENABLE_LAYERS>
    $<$<BOOL:${OPENCL_ICD_LOADER_DISABLE_OPENCLON12}>:OPENCL_ICD_LOADER_DISABLE_OPENCLON12>
)

target_include_directories (OpenCL
  PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}
    loader
)
target_link_libraries (OpenCL PUBLIC ${CMAKE_DL_LIBS})

option (OPENCL_ICD_LOADER_BUILD_TESTING "Enable support for OpenCL ICD Loader testing." OFF)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME OR OPENCL_ICD_LOADER_BUILD_TESTING)
    include(CTest)
endif()
if((CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME OR OPENCL_ICD_LOADER_BUILD_TESTING) AND BUILD_TESTING)
    add_subdirectory (test)
endif()

include (GNUInstallDirs)

install(
  TARGETS OpenCL
  EXPORT OpenCLICDLoaderTargets
  LIBRARY
    DESTINATION ${CMAKE_INSTALL_LIBDIR} # obtained from GNUInstallDirs
)

export(
  EXPORT OpenCLICDLoaderTargets
  FILE ${PROJECT_BINARY_DIR}/OpenCLICDLoader/OpenCLICDLoaderTargets.cmake
  NAMESPACE OpenCL::
)
file(
  WRITE ${PROJECT_BINARY_DIR}/OpenCLICDLoader/OpenCLICDLoaderConfig.cmake
  "include(\"\${CMAKE_CURRENT_LIST_DIR}/OpenCLICDLoaderTargets.cmake\")"
)

set(config_package_location ${CMAKE_INSTALL_DATADIR}/cmake/OpenCLICDLoader)
install(
  EXPORT OpenCLICDLoaderTargets
  FILE OpenCLICDLoaderTargets.cmake
  NAMESPACE OpenCL::
  DESTINATION ${config_package_location}
)
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/OpenCLICDLoader/OpenCLICDLoaderConfig.cmake
  DESTINATION ${config_package_location}
)

unset(CMAKE_SIZEOF_VOID_P)
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/OpenCLICDLoader/OpenCLICDLoaderConfigVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/OpenCLICDLoader/OpenCLICDLoaderConfigVersion.cmake
  DESTINATION ${config_package_location}
)

install (TARGETS OpenCL
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

##===------------------------------------------------------------------------------*- CMake -*-===##
##                         _       _
##                        | |     | |
##                    __ _| |_ ___| | __ _ _ __   __ _ 
##                   / _` | __/ __| |/ _` | '_ \ / _` |
##                  | (_| | || (__| | (_| | | | | (_| |
##                   \__, |\__\___|_|\__,_|_| |_|\__, | - GridTools Clang DSL
##                    __/ |                       __/ |
##                   |___/                       |___/
##
##
##  This file is distributed under the MIT License (MIT). 
##  See LICENSE.txt for details.
##
##===------------------------------------------------------------------------------------------===##

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING 
      "Choose the type of build, options are: Debug Release RelWithDebInfo." FORCE)
endif()

if(NOT BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build shared libraries." FORCE)
endif()

if(NOT CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}/install" CACHE STRING
      "Install path prefix, prepended onto install directories." FORCE)
endif()

project(gtclang CXX)
cmake_minimum_required(VERSION 3.3)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

if(NOT(DEFINED MCHBUILD_ROOT))
  message(FATAL_ERROR "MCHBUILD_ROOT not found! Try specifying it in the environment via -DMCHBUILD_ROOT=<>")
endif()

list(APPEND CMAKE_MODULE_PATH "${MCHBUILD_ROOT}/cmake")

include(mchbuildAddExecutable)
include(mchbuildAddLibrary)
include(mchbuildAddTargetCleanAll)
include(mchbuildAddTargetClangFormat)
include(mchbuildCombineLibraries)
include(mchbuildConfigureFile)
include(mchbuildCreatePackageString)
include(mchbuildEnableFullRPATH)
include(mchbuildExportPackage)
include(mchbuildGetGitHeadRevision)
include(mchbuildSetCXXStandard)
include(mchbuildAddCustomDummyTarget)
include(mchbuildInit)

mchbuild_init()

# Include the GTClang specific options, definitions and macros
include(GTClangOptions)
include(GTClangDefinitions)
include(GTClangMacros)

if(GTCLANG_BUILD_EXAMPLES_WITH_GPU)
  include(GTClangSetupCUDA)
  gtclang_setup_CUDA()
  find_package(SLURM REQUIRED)
endif(GTCLANG_BUILD_EXAMPLES_WITH_GPU)

# Set C++ standard
mchbuild_set_cxx_standard(c++11)

# Set C++ flags (Note that the LLVM/Clang package might add some other flags)
gtclang_set_cxx_flags()

# Add custom targets
mchbuild_add_target_clean_all()

# Output summary of the configuration
macro(make_config_string FIRST SECOND)
  mchbuild_make_string_pair(${FIRST} ": ${SECOND}" 20 out)
  list(APPEND config_info ${out})
endmacro()
make_config_string("gtclang version" ${GTCLANG_FULL_VERSION})
make_config_string("Platform" ${MCHBUILD_PLATFORM_STRING})
make_config_string("Architecture" ${MCHBUILD_ARCHITECTURE_STRING})
make_config_string("Compiler" ${MCHBUILD_COMPILER_STRING})
make_config_string("Build type" ${CMAKE_BUILD_TYPE})
make_config_string("Asserts" ${GTCLANG_ASSERTS})
mchbuild_report_result("Configuration summary" ${config_info})

# Include the packages (and set the correct, libraries, includes etc.)
foreach(package bash ccache Python3 Boost clang-format Clang Dawn GridTools OpenMP Threads)
  include("Add${package}")
  mchbuild_create_package_string(${package} info)
  list(APPEND package_info ${info})

  string(TOUPPER ${package} PACKAGE)
  if(${PACKAGE}_FOUND)
    list(APPEND GTCLANG_EXTERNAL_LIBRARIES ${MCHBUILD_${PACKAGE}_LIBRARIES})
    list(APPEND GTCLANG_EXTERNAL_INCLUDE_DIRS ${MCHBUILD_${PACKAGE}_INCLUDE_DIRS})
    list(APPEND GTCLANG_EXTERNAL_DEFINITIONS ${MCHBUILD_${PACKAGE}_DEFINITIONS})
  endif()
endforeach()

include_directories(SYSTEM ${GTCLANG_EXTERNAL_INCLUDE_DIRS})
add_definitions(${GTCLANG_EXTERNAL_DEFINITIONS})

# Output summary of the packages
mchbuild_report_result("Package summary" ${package_info})

# Support of RPATH-exports of dawn
if(NOT(DEFINED DAWN_RPATH_DIR))
   message(FATAL_ERROR "DAWN_RPATH_DIR is not exported by DAWN")
endif()
mchbuild_enable_full_rpath("${DAWN_RPATH_DIR}")

# Add clang-format target
set(format_directories 
  "${CMAKE_SOURCE_DIR}/src"
  "${CMAKE_SOURCE_DIR}/test/unit-test"
)

mchbuild_add_target_clang_format(DIRECTORIES ${format_directories} EXTENSION ".h;.cpp")

# Build gtclang
include_directories(${CMAKE_SOURCE_DIR}/src)
include_directories(${CMAKE_BINARY_DIR}/src)
include_directories(SYSTEM ${CMAKE_SOURCE_DIR}/test/utils/googletest/include)

add_subdirectory(src)

if(GTCLANG_BUILD_EXAMPLES)
  add_subdirectory(examples)
endif()

if(GTCLANG_TESTING)
  enable_testing()
  add_subdirectory(test)
endif()

if(GTCLANG_DOCUMENTATION)
  add_subdirectory(docs)
endif()

gtclang_gen_install_config()

# Install headers
install(
  DIRECTORY src/ 
  DESTINATION ${GTCLANG_INSTALL_INCLUDE_DIR} 
  FILES_MATCHING PATTERN "*.h" PATTERN "*.inc" PATTERN "*.hpp"
)


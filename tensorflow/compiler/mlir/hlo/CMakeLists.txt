#
# Copyright 2020 The TensorFlow Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
cmake_minimum_required(VERSION 3.13.4)

if(POLICY CMP0068)
  cmake_policy(SET CMP0068 NEW)
  set(CMAKE_BUILD_WITH_INSTALL_NAME_DIR ON)
endif()

if(POLICY CMP0075)
  cmake_policy(SET CMP0075 NEW)
endif()

if(POLICY CMP0077)
  cmake_policy(SET CMP0077 NEW)
endif()

#-------------------------------------------------------------------------------
# Options and settings
#-------------------------------------------------------------------------------
option(MHLO_BUILD_EMBEDDED "Build MHLO as part of another project" OFF)
option(MHLO_ENABLE_BINDINGS_PYTHON "Enables MHLO python bindings" OFF)

#-------------------------------------------------------------------------------
# Project setup and globals
#-------------------------------------------------------------------------------
set(MHLO_EXTERNAL_PROJECT_BUILD OFF)

if(PROJECT_NAME STREQUAL "LLVM")
  # Building as part of LLVM via the external project mechanism.
  set(MHLO_EXTERNAL_PROJECT_BUILD ON)
else()
  # Building standalone.
  project(mlir-hlo LANGUAGES CXX C)
  set(CMAKE_C_STANDARD 11)
  set(CMAKE_CXX_STANDARD 14)
  list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")
endif()

if(MHLO_ENABLE_BINDINGS_PYTHON AND NOT MHLO_EXTERNAL_PROJECT_BUILD)
  message(WARNING "MHLO python bindings are only supported in unified, external project builds")
endif()

#-------------------------------------------------------------------------------
# MLIR/LLVM Configuration
#-------------------------------------------------------------------------------

# Find MLIR to install if we are building standalone. If building as part of
# another project, let it handle the MLIR dependency. The dependent project
# might use a bundled version of MLIR instead of installing, for instance.
if(MHLO_EXTERNAL_PROJECT_BUILD)
  message(STATUS "Building MHLO as an external LLVM project")
  set(MLIR_MAIN_SRC_DIR ${LLVM_MAIN_SRC_DIR}/../mlir ) # --src-root
  set(MLIR_INCLUDE_DIR ${MLIR_MAIN_SRC_DIR}/include ) # --includedir
  set(MLIR_GENERATED_INCLUDE_DIR ${LLVM_BINARY_DIR}/tools/mlir/include)
  set(MLIR_TABLEGEN_EXE $<TARGET_FILE:mlir-tblgen>)
  include_directories(SYSTEM ${MLIR_INCLUDE_DIR})
  include_directories(SYSTEM ${MLIR_GENERATED_INCLUDE_DIR})
  include_directories(SYSTEM ${MLIR_TABLEGEN_OUTPUT_DIR})

  set(BACKEND_PACKAGE_STRING "${PACKAGE_STRING}")
  list(APPEND CMAKE_MODULE_PATH "${MLIR_MAIN_SRC_DIR}/cmake/modules")
elseif(NOT MHLO_BUILD_EMBEDDED)
  message(STATUS "Building MHLO with an installed MLIR")
  find_package(MLIR REQUIRED CONFIG)
  message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")
  message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")
  set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/bin)
  set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/lib)
  list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
  list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
else()
  message(STATUS "Building MHLO embedded in another project")
endif()

if(LLVM_ENABLE_ZLIB)
  find_package(ZLIB)
endif()

include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)
include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${MLIR_INCLUDE_DIRS})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/)
link_directories(${LLVM_BUILD_LIBRARY_DIR})
add_definitions(${LLVM_DEFINITIONS})

#-------------------------------------------------------------------------------
# Python configuration
#-------------------------------------------------------------------------------

if(MHLO_ENABLE_BINDINGS_PYTHON)
  include(MLIRDetectPythonEnv)
  mlir_detect_pybind11_install()
  find_package(Python3 ${LLVM_MINIMUM_PYTHON_VERSION}
  COMPONENTS Interpreter Development NumPy REQUIRED)
  find_package(pybind11 2.6 CONFIG REQUIRED)
endif()

#-------------------------------------------------------------------------------
# Directory setup
#-------------------------------------------------------------------------------

set(MLIR_HLO_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(MLIR_HLO_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})

add_custom_target(check-mlir-hlo)

add_subdirectory(include/mlir-hlo)
add_subdirectory(lib)
add_subdirectory(tools)
add_subdirectory(tests)

if(MHLO_ENABLE_BINDINGS_PYTHON)
  add_subdirectory(python)
endif()

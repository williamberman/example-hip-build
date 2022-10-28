# Copyright (c) 2020 - 2021 Advanced Micro Devices, Inc. All Rights Reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.

project(bit_extract)

cmake_minimum_required(VERSION 3.10)

if(NOT WIN32 AND NOT DEFINED __HIP_ENABLE_PCH)
  set(__HIP_ENABLE_PCH ON CACHE BOOL "enable/disable pre-compiled hip headers")
endif()

if (NOT DEFINED ROCM_PATH )
     set ( ROCM_PATH "/opt/rocm"  CACHE STRING "Default ROCM installation directory." )
endif ()

if(${__HIP_ENABLE_PCH})
  add_definitions(-D__HIP_ENABLE_PCH)
endif()

if (NOT DEFINED HIP_PLATFORM OR HIP_PLATFORM STREQUAL "amd")
  list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})

  find_package(hip)

  set(CMAKE_CXX_COMPILER ${HIP_HIPCC_EXECUTABLE})
  set(CMAKE_CXX_LINKER   ${HIP_HIPCC_EXECUTABLE})

  add_executable(bit_extract bit_extract.cpp)

  target_link_libraries(bit_extract hip::host)
elseif (HIP_PLATFORM STREQUAL "nvidia" OR HIP_PLATFORM STREQUAL "nvcc")
  message(FATAL_ERROR "TODO")
endif()

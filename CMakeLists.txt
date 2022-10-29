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

set( ROCM_PATH "/opt/rocm" CACHE STRING "Default ROCM installation directory.")

set(HIP_PLATFORM "amd" CACHE STRING "Platform for HIP to target. Available values: amd, nvidia, or nvcc (deprecated -- equivalent to nvidia).")

set(CUDA_ARCH "" CACHE STRING "Optional: If targeting CUDA, specific architecture to target. Standard `compute_XX` form.")

add_executable(bit_extract bit_extract.cpp)

list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})

find_package(hip)

if (HIP_PLATFORM STREQUAL "amd")
  if(NOT WIN32 AND NOT DEFINED __HIP_ENABLE_PCH)
    set(__HIP_ENABLE_PCH ON CACHE BOOL "enable/disable pre-compiled hip headers")
  endif()

  if(${__HIP_ENABLE_PCH})
    add_definitions(-D__HIP_ENABLE_PCH)
  endif()

  set(CMAKE_CXX_COMPILER ${HIP_HIPCC_EXECUTABLE})
  set(CMAKE_CXX_LINKER   ${HIP_HIPCC_EXECUTABLE})

  # The hip cmake config module defines three targets, 
  # hip::amdhip64, hip::host, and hip::device.
  #
  # hip::host and hip::device are interface targets. hip::amdhip64 is an 
  # imported target for libamdhip.
  #
  # You do not directly link to hip::amdhip64. hip::host links to hip::amdhip64
  # and hip::device links to hip::host. Link to hip::host to just use hip without 
  # compiling any GPU code. Link to hip::device to compile the GPU device code.
  #
  # Docs (outdated):
  # https://rocmdocs.amd.com/en/latest/Installation_Guide/Using-CMake-with-AMD-ROCm.html
  target_link_libraries(bit_extract hip::device)
elseif (HIP_PLATFORM STREQUAL "nvidia" OR HIP_PLATFORM STREQUAL "nvcc")
  # The targets defined by the hip cmake config only target amd devices.
  # For targeting nvidia devices, we'll make our own interface target,
  # hip_device_nvidia, that includes the rocm and hip headers. 
  add_library(hip_device_nvidia INTERFACE)

  # Configuring hipcc for nvidia:
  #
  # The platform hipcc targets is configured by the HIP_PLATFORM env var.
  # Ideally, as we could in the shell, we would call `HIP_PLATFORM=nvidia hipcc <...>`.
  # However, CMAKE_CXX_COMPILER doesn't allow configuration as such. Additionally,
  # cmake doesn't allow setting environment variables for target builds like make does
  # with exported variables.
  #
  # We write out a file, `nvidia_hipcc`, into the build directory to handle configuring hipcc. 
  # nvidia_hipcc uses `cmake -E env` to agnostically set HIP_PLATFORM before calling hipcc.
  cmake_path(APPEND _nvidia_hipcc "${CMAKE_CURRENT_BINARY_DIR}" nvidia_hipcc)
  file(WRITE ${_nvidia_hipcc} "${CMAKE_COMMAND} -E env HIP_PLATFORM=${HIP_PLATFORM} ${HIP_HIPCC_EXECUTABLE} $@\n")
  file(CHMOD ${_nvidia_hipcc} 
    PERMISSIONS 
      OWNER_READ OWNER_WRITE OWNER_EXECUTE 
      GROUP_READ GROUP_EXECUTE 
      WORLD_READ WORLD_EXECUTE)

  set(CMAKE_CXX_COMPILER ${_nvidia_hipcc})

  # Linking cuda: 
  # We do not explicitly link cuda. hipcc when targeting nvidia will 
  # use nvcc under the hood. nvcc when used for linking will handle 
  # linking cuda dependencies
  set(CMAKE_CXX_LINKER   ${_nvidia_hipcc})

  if (NOT CUDA_ARCH STREQUAL "")
    target_compile_options(hip_device_nvidia INTERFACE -arch=${CUDA_ARCH})
  endif()

  target_include_directories(hip_device_nvidia SYSTEM INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)
  target_include_directories(hip_device_nvidia INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)

  target_link_libraries(bit_extract hip_device_nvidia)
endif()

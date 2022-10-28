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

add_executable(bit_extract bit_extract.cpp)

list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})

find_package(hip)


if (NOT DEFINED HIP_PLATFORM OR HIP_PLATFORM STREQUAL "amd")
  set(CMAKE_CXX_COMPILER ${HIP_HIPCC_EXECUTABLE})
  set(CMAKE_CXX_LINKER   ${HIP_HIPCC_EXECUTABLE})

  target_link_libraries(bit_extract hip::device)
elseif (HIP_PLATFORM STREQUAL "nvidia" OR HIP_PLATFORM STREQUAL "nvcc")
  cmake_path(APPEND _nvidia_hipcc "${CMAKE_CURRENT_BINARY_DIR}" nvidia_hipcc)
  message("writing to ${_nvidia_hipcc}")
  file(WRITE ${_nvidia_hipcc} "${CMAKE_COMMAND} -E env HIP_PLATFORM=${HIP_PLATFORM} ${HIP_HIPCC_EXECUTABLE}\n")
  file(CHMOD ${_nvidia_hipcc} 
    PERMISSIONS 
      OWNER_READ OWNER_WRITE OWNER_EXECUTE 
      GROUP_READ GROUP_EXECUTE 
      WORLD_READ WORLD_EXECUTE)
  set(CMAKE_CXX_COMPILER ${_nvidia_hipcc})
  set(CMAKE_CXX_LINKER   ${_nvidia_hipcc})

  add_library(hip_device_nvidia INTERFACE)

  target_include_directories(hip_device_nvidia SYSTEM INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)
  target_include_directories(hip_device_nvidia INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)

  # target_compile_definitions(hip_device_nvidia INTERFACE __HIP_PLATFORM_NVIDIA__=1)

  # include /opt/rocm-5.3.0/hip/include;/opt/rocm/include
  # how to set HIP_PLATFORM="nvidia" env var when calling compiler?
  # link cuda?
  # device to build for?

  target_link_libraries(bit_extract hip_device_nvidia)
endif()

execute_process(COMMAND cmake --help-property-list OUTPUT_VARIABLE CMAKE_PROPERTY_LIST)

STRING(REGEX REPLACE ";" "\\\\;" CMAKE_PROPERTY_LIST "${CMAKE_PROPERTY_LIST}")
STRING(REGEX REPLACE "\n" ";" CMAKE_PROPERTY_LIST "${CMAKE_PROPERTY_LIST}")

list(REMOVE_DUPLICATES CMAKE_PROPERTY_LIST)

function(print_target_properties tgt)
    if(NOT TARGET ${tgt})
            message(NOTICE "There is no target named '${tgt}'")
      return()
    endif()

    foreach (prop ${CMAKE_PROPERTY_LIST})
        string(REPLACE "<CONFIG>" "${CMAKE_BUILD_TYPE}" prop ${prop})
        get_target_property(propval ${tgt} ${prop})
        if (propval)
                message (NOTICE "${tgt} ${prop} = ${propval}")
        endif()
    endforeach(prop)
endfunction(print_target_properties)

# print_target_properties(hip::device)
# print_target_properties(hip::host)
# print_target_properties(hip::amdhip64)
print_target_properties(hip_device_nvidia)

# message(NOTICE ${ROCM_PATH})
# message(NOTICE ${HIP_INCLUDE_DIRS})
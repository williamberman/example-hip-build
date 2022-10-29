## Example HIP build

Example of building HIP targeting nvidia and rocm. Modified from [HIP examples](https://github.com/ROCm-Developer-Tools/HIP/tree/d62eaeeda560c0c33d1e58fe9e44d7fd28220ac1/samples/0_Intro/bit_extract) to build for both amd and nvidia.

#### Cmake variables

| variable name | default value | description |
|-----------|-----------|-----------------------------------------|
| ROCM_PATH | /opt/rocm | Default ROCM installation directory. |
| HIP_PLATFORM | amd | Platform for HIP to target. Available values: amd, nvidia, or nvcc (deprecated -- equivalent to nvidia). |
| CUDA_ARCH |  | Optional: If targeting CUDA, specific architecture to target. Standard `compute_XX` form. |

#### Notes

The rocm install comes with a cmake config file for hip, but it only works when targeting amd. The HIP source has cmake
find modules which have code for handling targetting both nvidia and amd. The find modules could be vendored with our
source but there are no docs and they give opaque errors.

##### AMD Targets

The hip cmake config module defines three targets, 
hip::amdhip64, hip::host, and hip::device.

hip::host and hip::device are interface targets. hip::amdhip64 is an 
imported target for libamdhip.

You do not directly link to hip::amdhip64. hip::host links to hip::amdhip64
and hip::device links to hip::host. Link to hip::host to just use hip without 
compiling any GPU code. Link to hip::device to compile the GPU device code.

Docs (outdated):
https://rocmdocs.amd.com/en/latest/Installation_Guide/Using-CMake-with-AMD-ROCm.html

##### Nvidia Targets

The targets defined by the hip cmake config only target amd devices.
For targeting nvidia devices, we'll make our own interface target,
hip_device_nvidia, that includes the rocm and hip headers. 

##### Configuring hipcc for nvidia:

The platform hipcc targets is configured by the HIP_PLATFORM env var.
Ideally, as we could in the shell, we would call `HIP_PLATFORM=nvidia hipcc <...>`.
However, CMAKE_CXX_COMPILER doesn't allow configuration as such. Additionally,
cmake doesn't allow setting environment variables for target builds like make does
with exported variables.

We write out a file, `nvidia_hipcc`, into the build directory to handle configuring hipcc. 
nvidia_hipcc uses `cmake -E env` to agnostically set HIP_PLATFORM before calling hipcc.

cmake_minimum_required(VERSION 3.18)
project(${SKBUILD_PROJECT_NAME} VERSION ${SKBUILD_PROJECT_VERSION} LANGUAGES CXX)

# General settings
set(BUILD_TARGET torchmcubes_module)

# CUDA settings
include(CheckLanguage)
check_language(CUDA)

if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)

  add_definitions(-DWITH_CUDA)
  find_package(CUDAToolkit REQUIRED)
  set(CMAKE_CUDA_STANDARD 17)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)

  if(CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL 12)
    message(STATUS "PyTorch NVTX headers workaround: Yes")
    # only do this if nvToolsExt is not defined and CUDA::nvtx3 exists
    if(NOT TARGET CUDA::nvToolsExt AND TARGET CUDA::nvtx3)
      add_library(CUDA::nvToolsExt INTERFACE IMPORTED)
      # The TORCH_CUDA_USE_NVTX3 compile definition tells PyTorch to use the NVTX3 headers
      # instead of the legacy NVTX headers. This is necessary when building with CUDA 12+
      # and the CUDA::nvtx3 target is available, but CUDA::nvToolsExt is not defined.
      # Without this definition, PyTorch may fail to find the correct NVTX headers.
      target_compile_definitions(CUDA::nvToolsExt INTERFACE TORCH_CUDA_USE_NVTX3)
      target_link_libraries(CUDA::nvToolsExt INTERFACE CUDA::nvtx3)
    endif()
  else()
    message(STATUS "PyTorch NVTX headers workaround: No")
  endif()

  message(STATUS "INSTALLING EXTENSIONS WITH CUDA!")
  string(REGEX REPLACE ".[0-9][0-9]|\\." "" CUDA_V ${CMAKE_CUDA_COMPILER_VERSION})
  message(STATUS "CMAKE_CUDA_COMPILER = ${CMAKE_CUDA_COMPILER}")
  message(STATUS "CMAKE_CUDA_COMPILER_ID = ${CMAKE_CUDA_COMPILER_ID}")
  message(STATUS "CMAKE_CUDA_COMPILER_VERSION = ${CUDA_V}")
else()
  message(WARNING "NO CUDA INSTALLATION FOUND, TRYING TO INSTALL CPU VERSION ONLY!")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(MSVC)
  add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/utf-8>")
  add_compile_options("$<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=/utf-8>")
  add_definitions(-DNOMINMAX -D_CRT_SECURE_NO_WARNINGS)
endif()

# Find packages
find_package(Python COMPONENTS Interpreter Development.Module REQUIRED)
find_package(pybind11 CONFIG REQUIRED)
find_package(OpenMP)

find_package(Torch CONFIG REQUIRED)
find_library(TORCH_PYTHON_LIBRARY torch_python PATH "${TORCH_INSTALL_PREFIX}/lib")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")

add_subdirectory(cxx)
install(TARGETS ${BUILD_TARGET} LIBRARY DESTINATION .)

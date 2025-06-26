set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

if(NOT APPLE AND NOT WIN32)
  # symbolic functions causes references in our libraries to be resolved
  # to definitions within the libraries themselves, if possible.
  # If both runtime library and SYCL headers define some of the same
  # symbols, this can prevent code using hijacked USM malloc for standard
  # parallelism to bleed into code that is executed within the runtime, where
  # USM is a big no-no.
  set(HIPSYCL_STDPAR_RT_LINKER_FLAGS "-Wl,-Bsymbolic-functions")
  set(base $ORIGIN)
else()
  set(HIPSYCL_STDPAR_RT_LINKER_FLAGS "")
  set(base @loader_path)
endif()

if(NOT COMPILE_FOR_MICROCONTROLLERS)
  add_subdirectory(common)
endif()

set(WITH_LLVM_TO_AMDGPU_AMDHSA false)
set(WITH_LLVM_TO_PTX false)
set(WITH_LLVM_TO_SPIRV false)
set(WITH_LLVM_TO_HOST false)

if(WITH_SSCP_COMPILER)
  if(WITH_LEVEL_ZERO_BACKEND OR WITH_OPENCL_BACKEND)
    set(WITH_LLVM_TO_SPIRV true)
  endif()

  if(WITH_CUDA_BACKEND)
    set(WITH_LLVM_TO_PTX true)
  endif()

  if(WITH_ROCM_BACKEND)
    set(WITH_LLVM_TO_AMDGPU_AMDHSA true)
  endif()

  set(WITH_LLVM_TO_HOST true)
endif()

if(BUILD_CLANG_PLUGIN)
  set(ACPP_BINARY_DIR ${CMAKE_BINARY_DIR})
  # If cross-compiling, acpp clang plugin must be added as external project
  # to compile it for the host and not for the target
  if (COMPILE_FOR_MICROCONTROLLERS)
    include(ExternalProject)
    ExternalProject_Add(acpp-clang
        SOURCE_DIR ${CMAKE_SOURCE_DIR}/src/compiler
        BINARY_DIR ${CMAKE_BINARY_DIR}/src/compiler
        CMAKE_ARGS
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DACPP_BINARY_DIR=${ACPP_BINARY_DIR}
        -DLLVM_DIR=${LLVM_DIR}
        -DCLANG_EXECUTABLE_PATH=${CLANG_EXECUTABLE_PATH}
        -DHIPSYCL_DEBUG_LEVEL=${HIPSYCL_DEBUG_LEVEL}
        -DWITH_ACCELERATED_CPU=${WITH_ACCELERATED_CPU}
        -DWITH_SSCP_COMPILER=${WITH_SSCP_COMPILER}
        # disable the install step during build
        INSTALL_COMMAND ""
    )
    install(CODE [[
      message(STATUS "Installing acpp-clang")
      execute_process(
        COMMAND "${CMAKE_COMMAND}"
                --install "${CMAKE_BINARY_DIR}/src/compiler"
                --prefix  "${CMAKE_INSTALL_PREFIX}"
        RESULT_VARIABLE ACPP_CLANG_INSTALL_RESULT
      )
      if(NOT ACPP_CLANG_INSTALL_RESULT EQUAL 0)
        message(FATAL_ERROR "acpp-clang installation failed")
      endif()
    ]])
  else()
    add_subdirectory(compiler)
  endif()
  add_subdirectory(libkernel)
endif()

set(ACPP_RT_SANITIZE "" CACHE STRING
    "Enable building AdaptiveCpp runtime with specified sanitizers")

if(NOT ${ACPP_RT_SANITIZE} STREQUAL "")
  set(ACPP_RT_SANITIZE_FLAGS "-fsanitize=${ACPP_RT_SANITIZE}")
else()
  set(ACPP_RT_SANITIZE_FLAGS "")
endif()


add_subdirectory(runtime)
if(COMPILE_TOOLS)
  add_subdirectory(tools)
endif()

set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME ${HIPSYCL_RT_LIBRARY_OUTPUT_NAME} PARENT_SCOPE)
set(hipSYCL_OpenMP_CXX_LIBRARIES ${hipSYCL_OpenMP_CXX_LIBRARIES} PARENT_SCOPE)

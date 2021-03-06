# ########################################################################
# Copyright 2016 Advanced Micro Devices, Inc.
# ########################################################################

set( hipblas_samples_common ../common/utility.cpp )

add_executable( example-sscal example_sscal.cpp ${hipblas_samples_common} )
add_executable( example_sgemm example_sgemm.cpp ${hipblas_samples_common} )
add_executable( example_sgemm_strided_batched example_sgemm_strided_batched.cpp ${hipblas_samples_common} )


if( NOT TARGET hipblas )
  find_package( hipblas CONFIG PATHS /opt/rocm/hipblas )

  if( NOT hipblas_FOUND )
    message( FATAL_ERROR "hipBLAS is a required dependency and is not found; try adding rocblas path to CMAKE_PREFIX_PATH")
  endif( )
endif( )

foreach( exe example-sscal;example_sgemm;example_sgemm_strided_batched )
  if( TARGET hipblas )
    target_link_libraries( ${exe} PRIVATE hipblas )
  else( )
    target_link_libraries( ${exe} PRIVATE roc::hipblas )
  endif( )

  # Try to test for specific compiler features if cmake version is recent enough
  if( CMAKE_VERSION VERSION_GREATER "3.0" )
    target_compile_features( ${exe} PRIVATE cxx_static_assert cxx_nullptr cxx_auto_type )
  else( )
    # Otherwise, just try to compile the library with a standards flag
    if( CMAKE_COMPILER_IS_GNUCXX OR ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" ) )
      target_compile_options( ${exe} PRIVATE -std=c++11 -pthread )
    endif( )
  endif( )

  set_target_properties( ${exe} PROPERTIES DEBUG_POSTFIX "-d" CXX_EXTENSIONS NO )
  set_target_properties( ${exe} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging" )
  target_include_directories( ${exe} PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include> )

  if( CMAKE_CXX_COMPILER MATCHES ".*/hcc$" OR CMAKE_CXX_COMPILER MATCHES ".*/hipcc$" )
    # Remove following when hcc is fixed; hcc emits following spurious warning ROCm v1.6.1
    # "clang-5.0: warning: argument unused during compilation: '-isystem /opt/rocm/include'"
    target_compile_options( ${exe} PRIVATE -Wno-unused-command-line-argument )

    foreach( target ${AMDGPU_TARGETS} )
      target_link_libraries( ${exe} PRIVATE --amdgpu-target=${target} )
    endforeach( )
  else( )
    find_package( CUDA REQUIRED )

    target_include_directories( ${exe}
    PRIVATE
      $<BUILD_INTERFACE:${CUDA_INCLUDE_DIRS}>
    )

    target_link_libraries( ${exe} PRIVATE ${CUDA_LIBRARIES} )
  endif( )

endforeach( )


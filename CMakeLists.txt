# Does your system only have an older version of CMake? Not to worry!
# CMake offers download-and-use binary packages, with no installation 
# necessary...  Visit https://cmake.org/download/ and grab one for 
# your platform. They are not finicky with library dependencies, so
# compatability is very likely. Also, the package's CMake binary will
# not mistake any other local CMake-related files for its own.
cmake_minimum_required(VERSION 3.18 FATAL_ERROR)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules")

# -----------------------------------
# Project name, version & build type
# -----------------------------------

if(WIN32 AND "${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
	# Avoid a build failure when the BUILD_SHARED_LIBS variable is set to true.
	set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

PROJECT(cuda-api-wrappers
	VERSION 0.5.2
	DESCRIPTION "Thin C++-flavored wrappers for the CUDA Runtime API"
	HOMEPAGE_URL https://github.com/eyalroz/cuda-api-wrappers
	LANGUAGES CUDA CXX)

include(GNUInstallDirs)

find_package(CUDAToolkit 9.0 REQUIRED)
find_package(Threads REQUIRED)
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "lib/")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "lib/")

# -------------------
# Our library targets
# -------------------

# They're all header-only
add_library(runtime-and-driver INTERFACE)
add_library(nvrtc INTERFACE)
add_library(nvtx INTERFACE)

set(wrapper-libraries runtime-and-driver nvtx nvrtc)

foreach(wrapper_lib ${wrapper-libraries})
	target_compile_features(${wrapper_lib} INTERFACE cxx_std_11) # This means _at least_ C++11
	target_include_directories(
		${wrapper_lib}
		INTERFACE
		"$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
		"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
	)
	target_link_libraries(${wrapper_lib} INTERFACE CUDA::cudart CUDA::cuda_driver)
	# Targets using these libraries should be compiled with C++11 _at least_,
	# but it doesn't seem there's a way to express that at the CMake level
endforeach()
target_link_libraries(nvrtc INTERFACE CUDA::nvrtc)
target_link_libraries(nvtx INTERFACE CUDA::nvToolsExt)

target_link_libraries(nvtx INTERFACE runtime-and-driver)
target_link_libraries(nvtx INTERFACE Threads::Threads CUDA::nvToolsExt)


# Note: This is a bit like a poor man's configure.h file;
# but for two settings I won't bother creating one of those
if(DEFINED CMAKE_USE_PTHREADS_INIT)
	target_compile_definitions(nvtx INTERFACE "" "CUDA_API_WRAPPERS_USE_PTHREADS")
elseif(DEFINED CMAKE_USE_WIN32_THREADS_INIT)
	target_compile_definitions(nvtx INTERFACE "" "CUDA_API_WRAPPERS_USE_WIN32_THREADS")
endif()

# --------
# Examples
# --------

option(CAW_BUILD_EXAMPLES "Build example programs" OFF)

if (CAW_BUILD_EXAMPLES)
	add_subdirectory(examples)
endif()

# ------------------------
# Installing the libraries
# ------------------------

configure_file("${PROJECT_SOURCE_DIR}/cmake/cuda-api-wrappers-config.cmake.in"
	"${PROJECT_BINARY_DIR}/cuda-api-wrappers-config.cmake" @ONLY)

install(FILES "${PROJECT_BINARY_DIR}/cuda-api-wrappers-config.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/cuda-api-wrappers")

install(
	TARGETS ${wrapper-libraries}
	EXPORT cuda-api-wrappers_export
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# If you want to use the library without installing it, you'll need to
# copy the file we generate here, cuda-api-wrappers-targets.cmake, into the
# build directory, and set `cuda-api-wrappers_DIR` to that build directory
# in your own CMake project.
export(
	EXPORT cuda-api-wrappers_export
	NAMESPACE "cuda-api-wrappers::"
	FILE "${PROJECT_BINARY_DIR}/cuda-api-wrappers-targets.cmake"
)

install(
	DIRECTORY src/cuda
	DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
	FILES_MATCHING REGEX "\\.(h|hpp|cuh)$"
)

install(
	EXPORT cuda-api-wrappers_export
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/cuda-api-wrappers"
	NAMESPACE "cuda-api-wrappers::"
	FILE "cuda-api-wrappers-targets.cmake"
)

include(CMakePackageConfigHelpers)

# The SameMinorVersion parameter requires CMake 3.11.
# If not supported, fall back to SameMajorVersion.
if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.11)
	set(COMPAT_SETTING SameMinorVersion)
else()
	set(COMPAT_SETTING SameMajorVersion)
endif()
write_basic_package_version_file(
	"cuda-api-wrappers-config-version.cmake"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY ${COMPAT_SETTING}
)

install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/cuda-api-wrappers-config-version.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/cuda-api-wrappers"
)

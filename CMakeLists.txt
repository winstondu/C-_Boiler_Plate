cmake_minimum_required(VERSION 3.2)
#-------------------
# Set initial variables
#-------------------
# Use Clang compiler here
#SET (CMAKE_C_COMPILER "clang")
#SET (CMAKE_C_FLAGS "-target x86_64-pc-w64-mingw32")
#SET (CMAKE_CXX_COMPILER "clang++")
#SET (CMAKE_CXX_FLAGS "-target x86_64-pc-w64-mingw32")

SET (CMAKE_C_COMPILER "gcc")
SET (CMAKE_CXX_COMPILER "g++")

# Name of project
set(PROJECT_NAME_STR SampleProject)
project(${PROJECT_NAME_STR})
# Set compiler flags (gnu++ over c++)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11 -Wall -Werror -Wextra -pedantic -pedantic-errors")
set (CMAKE_SHARED_LINKER_FLAGS ${CMAKE_SHARED_LINKER_FLAGS_INIT} -lshlwapi)

#-------------------
# Set common include folder for module (project headers)
#-------------------
set(COMMON_INCLUDES ${PROJECT_SOURCE_DIR}/include)
include_directories(${COMMON_INCLUDES})
include_directories(./include)

#-------------------
# Set external include folder for imported libraries
#-------------------
# Set directory for external projects/libraries
set(EXT_PROJECTS_DIR ${PROJECT_SOURCE_DIR}/lib)

include(CTest REQUIRED) ## for ctests with benchmarks ???

# Enable ExternalProject CMake module
include(ExternalProject)

    #-------------------
    # Googletest
    #-------------------

# We need thread support
find_package(Threads REQUIRED)

# Download and install GoogleTest
ExternalProject_Add(googletest
        GIT_REPOSITORY https://github.com/google/googletest
        GIT_TAG master
        CMAKE_ARGS
        -Dgtest_force_shared_crt=${GTEST_FORCE_SHARED_CRT}
        -DCMAKE_INSTALL_PREFIX=${GTEST_DIRECTORY}
        )

# Get GTest source and binary directories from CMake project
ExternalProject_Get_Property(googletest source_dir binary_dir)

# Create a libgtest target to be used as a dependency by test programs
add_library(libgtest IMPORTED STATIC GLOBAL)
add_dependencies(libgtest googletest)

# Set libgtest properties
set_target_properties(libgtest PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/googlemock/gtest/libgtest.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# Create a libgmock target to be used as a dependency by test programs
add_library(libgmock IMPORTED STATIC GLOBAL)
add_dependencies(libgmock googletest)

# Set libgmock properties
set_target_properties(libgmock PROPERTIES
    "IMPORTED_LOCATION"  "${GTEST_DIRECTORY}/lib/libgmock.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# Create a libgtestmain target to be used as a dependency by test programs
add_library(libgtestmain IMPORTED STATIC GLOBAL)
add_dependencies(libgtestmain googletest)

# Set libgtestmain properties
set_target_properties(libgtestmain PROPERTIES
    "IMPORTED_LOCATION"  "${binary_dir}/googlemock/gtest/libgtest_main.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# I couldn't make it work with INTERFACE_INCLUDE_DIRECTORIES
include_directories("${source_dir}/googletest/include"
                    "${source_dir}/googlemock/include"
            "${GTEST_DIRECTORY}/include")

    #-------------------
    # Google MicroBenchmarks
    #-------------------

set(BENCHMARK_ENABLE_TESTING OFF) # If you want to self-test benchmark lib too, turn me ON

### Set GoogleBM directory
set(GBENCHMARK_DIRECTORY ${EXT_PROJECTS_DIR}/googlebenchmark)
ExternalProject_Add(googlebenchmark
        GIT_REPOSITORY https://github.com/google/benchmark
        GIT_TAG master
        UPDATE_COMMAND ""
        CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${GBENCHMARK_DIRECTORY}
	-DBENCHMARK_ENABLE_GTEST_TESTS=OFF
        )
### Include the GoogleBM headers and libraries
include_directories(${GBENCHMARK_DIRECTORY}/include)
link_directories(${GBENCHMARK_DIRECTORY}/lib)

#-------------------
# Module source
#-------------------
set(PROJECT_LIB_NAME ${PROJECT_NAME_STR}_impl)

file(GLOB SRC_FILES ${PROJECT_SOURCE_DIR}/src/*.cpp)
add_library(${PROJECT_LIB_NAME} ${SRC_FILES})
# Make the project root directory the working directory when we run
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bin)

## Executables
# add_executable(executable ${SRC_FILES})

#-------------------
# Testing
#-------------------
add_definitions(${MSVC_COMPILER_DEFS}) #Uncomment if you use MSVC
# Define the source files and dependencies for the testing executable
set(PROJECT_TEST_NAME ${PROJECT_NAME_STR}_test)
# The test files.
file(GLOB TEST_SRC_FILES ${PROJECT_SOURCE_DIR}/tests/*.cpp)

# Create test executable.
add_executable(${PROJECT_TEST_NAME} ${TEST_SRC_FILES})
add_dependencies(${PROJECT_TEST_NAME} googletest)
# Link gtest modules
# We need gtest_main to drive the gtests (otherwise we'll get a WinMain error)
target_link_libraries(${PROJECT_TEST_NAME} libgtest libgtestmain ${CMAKE_THREAD_LIBS_INIT})

## Make Ctest invoke that executable.
enable_testing()
add_test(main_test ${PROJECT_TEST_NAME})

#-------------------
# Benchmarking
#-------------------
set(PROJECT_BM_NAME ${PROJECT_NAME_STR}_bm)
# The BM files.
file(GLOB BM_SRC_FILES ${PROJECT_SOURCE_DIR}/tests/benchmarks/*.cpp)
# Create test executable.
add_executable(${PROJECT_BM_NAME} ${BM_SRC_FILES})
add_dependencies(${PROJECT_BM_NAME} googlebenchmark)
target_link_libraries(${PROJECT_BM_NAME} libgtest libgtestmain benchmark shlwapi.lib ${CMAKE_THREAD_LIBS_INIT})
add_test(main_bm ${PROJECT_BM_NAME})
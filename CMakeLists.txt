cmake_minimum_required(VERSION 3.2)
# Name of project
set(PROJECT_NAME_STR Dijkstras)
project(${PROJECT_NAME_STR})
# Set compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Werror -Wextra -pedantic -pedantic-errors")
set (CMAKE_SHARED_LINKER_FLAGS ${CMAKE_SHARED_LINKER_FLAGS_INIT} -lshlwapi)

message(STATUS ${PROJECT_SOURCE_DIR})

#-------------------
# Set common include folder for module (project headers)
#-------------------
set(COMMON_INCLUDES ${PROJECT_SOURCE_DIR}/include)
include_directories(${COMMON_INCLUDES})
include_directories(./include)
#-------------------
# Set external include folder for module
#-------------------
include(ExternalProject)
set(EXT_PROJECTS_DIR ${PROJECT_SOURCE_DIR}/lib)

#
#-GTEST
#
# We need thread support
find_package(Threads REQUIRED)

# Enable ExternalProject CMake module
include(ExternalProject)

# Download and install GoogleTest
ExternalProject_Add(
    gtest
    URL https://github.com/google/googletest/archive/master.zip
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/gtest
    # Disable install step
    INSTALL_COMMAND ""
)

# Get GTest source and binary directories from CMake project
ExternalProject_Get_Property(gtest source_dir binary_dir)

# Create a libgtest target to be used as a dependency by test programs
add_library(libgtest IMPORTED STATIC GLOBAL)
add_dependencies(libgtest gtest)

# Set libgtest properties
set_target_properties(libgtest PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/googlemock/gtest/libgtest.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# Create a libgtest target to be used as a dependency by test programs
add_library(libgtestmain IMPORTED STATIC GLOBAL)
add_dependencies(libgtestmain gtest)

# Set libgtestmain properties
set_target_properties(libgtestmain PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/googlemock/gtest/libgtest_main.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# Create a libgmock target to be used as a dependency by test programs
add_library(libgmock IMPORTED STATIC GLOBAL)
add_dependencies(libgmock gtest)

# Set libgmock properties
set_target_properties(libgmock PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/googlemock/libgmock.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# Create a libgmockmain target to be used as a dependency by test programs
add_library(libgmockmain IMPORTED STATIC GLOBAL)
add_dependencies(libgmockmain gtest)

# Set libgmock properties
set_target_properties(libgmockmain PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/googlemock/libgmock_main.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)

# I couldn't make it work with INTERFACE_INCLUDE_DIRECTORIES
include_directories("${source_dir}/googletest/include"
                    "${source_dir}/googlemock/include")

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
add_definitions(${MSVC_COMPILER_DEFS})
# Define the source files and dependencies for the testing executable
set(PROJECT_TEST_NAME ${PROJECT_NAME_STR}_test)
# The test files.
file(GLOB TEST_SRC_FILES ${PROJECT_SOURCE_DIR}/tests/*.cpp)

# Create test executable.
add_executable(${PROJECT_TEST_NAME} ${TEST_SRC_FILES})
add_dependencies(${PROJECT_TEST_NAME} gtest)
# Link gtest modules
# We need gtest_main to drive the gtests (otherwise we'll get a WinMain error)
target_link_libraries(${PROJECT_TEST_NAME} libgtest libgtestmain libgmock)

## Make Ctest invoke that executable.
enable_testing()
add_test(main_test ${PROJECT_TEST_NAME})

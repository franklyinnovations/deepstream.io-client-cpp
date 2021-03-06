cmake_minimum_required(VERSION 2.8.12)

project(deepstream.io-client-cpp)

enable_language(C)
enable_language(CXX)
enable_testing()
include(CTest)
include(ExternalProject)

set(PROJECT_VERSION_MAJOR 0)
set(PROJECT_VERSION_MINOR 1)
set(PROJECT_VERSION_PATCH 0)
set(PROJECT_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake/modules)

configure_file(version.hpp.in ${CMAKE_BINARY_DIR}/include/deepstream/core/version.hpp @ONLY)

# Put the libaries and binaries that get built into directories at the
# top of the build tree rather than in hard-to-find leaf directories.

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

option(CCACHE "Use ccache if available" ON)
find_program(CCACHE_PROGRAM ccache)
if(CCACHE AND CCACHE_PROGRAM)
  message(STATUS "using ccache")
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

# We absolutely depend on C++11 support from the toolchain; exit if
# the toolchain doesn't support it.

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
if(COMPILER_SUPPORTS_CXX11)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
else()
  message(FATAL_ERROR "Compiler ${CMAKE_CXX_COMPILER} has no C++11 support.")
endif()

CHECK_C_COMPILER_FLAG("-std=c11" COMPILER_SUPPORTS_C11)
if(COMPILER_SUPPORTS_C11)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11")
else()
  message(FATAL_ERROR "Compiler ${CMAKE_C_COMPILER} has no C11 support.")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wextra -Wall -pedantic -g")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wextra -Wall -pedantic -g")

if(NOT DEFINED(CMAKE_BUILD_TYPE) OR "Debug" STREQUAL "${CMAKE_BUILD_TYPE}")
  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_ASSERTIONS")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_DEBUG")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_DEBUG_PEDANTIC")
  endif()
endif()

if(BUILD_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --coverage")
endif()

if(BUILD_POCO)
  externalproject_add(poco
    PREFIX thirdparty
    URL https://github.com/pocoproject/poco/archive/poco-1.7.8-release.tar.gz
    URL_HASH MD5=4dbf02e14b9f20940ca0e8c70d8f6036
    CMAKE_GENERATOR ${CMAKE_GENERATOR}
    CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE}
    CMAKE_ARGS
    -DENABLE_MONGODB=OFF
    -DENABLE_PDF=OFF
    -DENABLE_DATA=OFF
    -DENABLE_DATA_SQLITE=OFF
    -DENABLE_DATA_MYSQL=OFF
    -DENABLE_DATA_ODBC=OFF
    -DENABLE_SEVENZIP=OFF
    -DENABLE_ZIP=OFF
    -DENABLE_APACHECONNECTOR=OFF
    -DENABLE_CPPPARSER=OFF
    -DENABLE_POCODOC=OFF
    -DENABLE_PAGECOMPILER=OFF
    -DENABLE_PAGECOMPILER_FILE2PAGE=OFF
    -DENABLE_JSON=OFF
    -DENABLE_XML=OFF
    -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/thirdparty
    LOG_CONFIGURE ON
    UPDATE_COMMAND "")

  set(Poco_LIBRARIES PocoNetSSL PocoCrypto PocoNet PocoUtil PocoFoundation)
else()
  find_package(Poco 1.7.8 REQUIRED NetSSL Crypto Net Util Foundation)
endif()

find_package(SWIG 3)
find_package(FLEX 2.5 REQUIRED)

link_directories(${CMAKE_BINARY_DIR}/thirdparty/lib)
include_directories(${CMAKE_BINARY_DIR}/thirdparty/include)

include_directories(${CMAKE_SOURCE_DIR}/include ${CMAKE_BINARY_DIR}/include)

option(BUILD_SWIG "Build the SWIG python wrapper" OFF)
if(BUILD_SWIG AND SWIG_FOUND)
  # We need to fix the generated Swig wrapper as the current version
  # of Swig does not handle std::unique_ptr. Our fix directly calls
  # std::move() for the few cases where this smart pointer type is
  # exposed in the deepstream public API.

  include(${SWIG_USE_FILE})
  find_package(PythonLibs REQUIRED)
  include_directories(${PYTHON_INCLUDE_PATH})
  set(CMAKE_SWIG_FLAGS "")
  set(CMAKE_SWIG_OUTDIR python)
  set_source_files_properties(swig/deepstream.i PROPERTIES CPLUSPLUS ON)
  set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/python/deepstreamPYTHON_wrap.cxx PROPERTIES COMPILE_FLAGS -Wno-self-assign)
  swig_add_module(deepstream python swig/deepstream.i)
  swig_link_libraries(deepstream libdeepstream_core ${POCO_LIBS} ${PYTHON_LIBRARIES})

  add_test(
    NAME    python-swig-test
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/test/swig/python-test.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()

add_subdirectory(doc)
add_subdirectory(src/core)
add_subdirectory(src/lib)
add_subdirectory(examples)

if(BUILD_TESTING)
  find_package(Boost 1.46 REQUIRED COMPONENTS unit_test_framework)
  if(Boost_FOUND)
      add_subdirectory(test/core)
      add_subdirectory(test/lib)
  endif()
endif()

install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/include/
    DESTINATION include
    FILES_MATCHING PATTERN "*.hpp"
)

install(
    FILES ${CMAKE_BINARY_DIR}/include/deepstream/core/version.hpp
    DESTINATION include/deepstream/core
)

message(STATUS "CMAKE_SYSTEM=${CMAKE_SYSTEM}")
message(STATUS "CMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}")
message(STATUS "CMAKE_SYSTEM_PROCESSOR=${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "CMAKE_GENERATOR=${CMAKE_GENERATOR}")
message(STATUS "CMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
message(STATUS "CMAKE_C_COMPILER=${CMAKE_C_COMPILER}")
message(STATUS "CMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}")

message(STATUS "BUILD_TESTING=${BUILD_TESTING}")
message(STATUS "Boost_FOUND=${Boost_FOUND}")
message(STATUS "BUILD_COVERAGE=${BUILD_COVERAGE}")
message(STATUS "BUILD_POCO=${BUILD_POCO}")
message(STATUS "Poco_LIBRARIES=${Poco_LIBRARIES}")

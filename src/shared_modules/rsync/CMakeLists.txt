cmake_minimum_required(VERSION 3.12.4)

project(rsync)

enable_testing()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

get_filename_component(SHARED_MODULES ${CMAKE_SOURCE_DIR}/../ ABSOLUTE)
get_filename_component(SRC_FOLDER     ${CMAKE_SOURCE_DIR}/../../ ABSOLUTE)


if(COVERITY)
  add_definitions(-D__GNUC__=8)
endif(COVERITY)

set(CMAKE_CXX_FLAGS "-Wall -Wextra -Wshadow -Wnon-virtual-dtor -Woverloaded-virtual -Wunused -Wcast-align -Wformat=2 -std=c++14 -pthread")

set(CMAKE_CXX_FLAGS_DEBUG "-g")
set(CMAKE_CXX_FLAGS_RELEASE "-O3")

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if(APPLE)
  set(CMAKE_MACOSX_RPATH 1)
endif(APPLE)

include_directories(${SRC_FOLDER}/external/cJSON/)
include_directories(${SRC_FOLDER}/external/nlohmann/)
include_directories(${SRC_FOLDER}/external/openssl/include/)
include_directories(${CMAKE_SOURCE_DIR}/include/)
include_directories(${CMAKE_SOURCE_DIR}/src/)
include_directories(${SHARED_MODULES}/dbsync/include/)
include_directories(${SHARED_MODULES}/utils/)
include_directories(${SHARED_MODULES}/common/)

link_directories(${SRC_FOLDER}/external/cJSON/)
link_directories(${SHARED_MODULES}/dbsync/build/lib/)
link_directories(${SRC_FOLDER}/external/openssl/)

file(GLOB RSYNC_SRC
    "${CMAKE_SOURCE_DIR}/src/*.cpp")

add_library(rsync SHARED
    ${RSYNC_SRC} )

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  add_definitions(-DWIN_EXPORT)
  set_target_properties(rsync PROPERTIES
        PREFIX ""
        SUFFIX ".dll"
        LINK_FLAGS "-Wl,--add-stdcall-alias -static-libgcc -static-libstdc++"
        POSITION_INDEPENDENT_CODE 0 # this is to avoid MinGW warning;
        # MinGW generates position-independent-code for DLL by default
  )
  target_link_libraries(rsync crypto ssl ws2_32 crypt32)
elseif(UNIX AND NOT APPLE)
  set_target_properties(rsync PROPERTIES
        LINK_FLAGS "-static-libgcc -static-libstdc++")
  if(NOT CMAKE_SYSTEM_NAME STREQUAL "AIX")
    string(APPEND CMAKE_SHARED_LINKER_FLAGS "-Wl,-rpath=$ORIGIN")
  endif(NOT CMAKE_SYSTEM_NAME STREQUAL "AIX")
endif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_link_libraries(rsync -fprofile-arcs)
    target_link_libraries(rsync cjson dbsync crypto pthread)
  else()
    target_link_libraries(rsync cjson dbsync crypto gcov pthread)
  endif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")

  if(UNIT_TEST)
    include_directories(${SRC_FOLDER}/external/googletest/googletest/include/)
    include_directories(${SRC_FOLDER}/external/googletest/googlemock/include/)

    link_directories(${SRC_FOLDER}/external/googletest/lib/)

    add_subdirectory(tests)
    add_subdirectory(integrationTests)
  endif(UNIT_TEST)
else()
  if(NOT CMAKE_SYSTEM_NAME STREQUAL "AIX")
    target_link_libraries(rsync dbsync cjson crypto pthread)
  else()
    string(REPLACE ";" ":" CXX_IMPLICIT_LINK_DIRECTORIES_STR "${CMAKE_CXX_IMPLICIT_LINK_DIRECTORIES}")
    string(REPLACE ";" ":" PLATFORM_REQUIRED_RUNTIME_PATH_STR "${CMAKE_PLATFORM_REQUIRED_RUNTIME_PATH}")
    target_link_libraries(rsync dbsync cjson crypto pthread -Wl,-blibpath:${INSTALL_PREFIX}/lib:${CXX_IMPLICIT_LINK_DIRECTORIES_STR}:${PLATFORM_REQUIRED_RUNTIME_PATH_STR})
  endif(NOT CMAKE_SYSTEM_NAME STREQUAL "AIX")
endif(CMAKE_BUILD_TYPE STREQUAL "Debug")

if(NOT DEFINED COVERITY)
add_subdirectory(testtool)
endif(NOT DEFINED COVERITY)

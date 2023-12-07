cmake_minimum_required(VERSION 3.14)
project(EDIE VERSION 3.1.0)

option(COVERAGE "Coverage" OFF)

set(CMAKE_VERBOSE_MAKEFILE OFF)
set(CMAKE_CXX_STANDARD 20)

include_directories(".")
include_directories("lib") # spdlog's include structure is relative to its root folder

include(FetchContent)

if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE Release)
endif()

# Check platforms
if(CMAKE_HOST_WIN32)
   set(WINDOWS 1)
   cmake_host_system_information(RESULT OS_NAME QUERY OS_NAME)
   set(DISTRIB_NAME ${OS_NAME})
   message("CMAKE Host: Windows")
   message("CMAKE DISTRIB_NAME: ${DISTRIB_NAME}")
elseif(CMAKE_HOST_UNIX)
   set(LINUX 1)
   cmake_host_system_information(RESULT DISTRIB_NAME QUERY DISTRIB_NAME)
   cmake_host_system_information(RESULT DISTRIB_VERSION_ID QUERY DISTRIB_VERSION_ID)
   set(DISTRIB_NAME "${DISTRIB_NAME}-${DISTRIB_VERSION_ID}")
   message("CMAKE Host: Linux")
   message("CMAKE DISTRIB_NAME: ${DISTRIB_NAME}")
else()
   set(DISTRIB_NAME "UNKNOWN")
   message(FATAL_ERROR "Unable to identify OS. Update script to support distribution or OS")
endif()

message("CMAKE_SYSTEM_PROCESSOR: " ${CMAKE_SYSTEM_PROCESSOR})
message("CMAKE_LIB_SHARED: "       ${CMAKE_LIB_SHARED})

if(NOT DEFINED ARCH)
   message("ARCH is not Set, default to x64")
   set(ARCH "x64")
endif()

if(NOT DEFINED GIT_BRANCH)
   set(GIT_BRANCH "")
endif()

if(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
   set_property(GLOBAL PROPERTY USE_FOLDERS ON)

   string(REGEX REPLACE "/W3"  "/W4"  CMAKE_CXX_FLAGS         "${CMAKE_CXX_FLAGS}")
   string(REGEX REPLACE "/MDd" "/MTd" CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG}")
   string(REGEX REPLACE "/MD"  "/MT"  CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
   string(REGEX REPLACE "/O2"  "/Ox"  CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")

   # Disable narrowing conversion, deprecated warnings, and constant conditional expression warnings
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /utf-8 /wd4244 /wd4996 /wd4127 /WX")

   set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /Zi")
   
   message("ARCH is: "                  ${ARCH})
   message("cxx Flags: "                ${CMAKE_CXX_FLAGS})
   message("cxx Flags Debug: "          ${CMAKE_CXX_FLAGS_DEBUG})
   message("cxx Flags Release: "        ${CMAKE_CXX_FLAGS_RELEASE})
   message("CMAKE_GENERATOR_PLATFORM: " ${CMAKE_GENERATOR_PLATFORM})
   message("CMAKE_GENERATOR_TOOLSET: "  ${CMAKE_GENERATOR_TOOLSET})

elseif(CMAKE_C_COMPILER_ID STREQUAL "GNU")
    set(COMMON_GCC_CXX_FLAGS "-Wall -fPIC -Wfatal-errors -Werror -Wold-style-cast -Wextra -pedantic -Wno-format-truncation")
    set(COMMON_GCC_FLAGS_DEBUG "-g")
    set(COMMON_GCC_FLAGS_RELEASE "-O3 -DNDEBUG -s")

    if(LINUX)
        set(COMMON_GCC_CXX_FLAGS "${COMMON_GCC_CXX_FLAGS} -D_LINUX_")
    endif()
    if(MINGW)
        set(COMMON_GCC_FLAGS_DEBUG "${COMMON_GCC_FLAGS_DEBUG} -O1")
    endif()

    set(CMAKE_C_FLAGS_DEBUG     "${COMMON_GCC_FLAGS_DEBUG}")
    set(CMAKE_C_FLAGS_RELEASE   "${COMMON_GCC_FLAGS_RELEASE}")
    set(CMAKE_CXX_FLAGS_DEBUG   "${COMMON_GCC_CXX_FLAGS} ${COMMON_GCC_FLAGS_DEBUG}")
    set(CMAKE_CXX_FLAGS_RELEASE "${COMMON_GCC_CXX_FLAGS} ${COMMON_GCC_FLAGS_RELEASE}")

    if(ARCH STREQUAL "x86" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "i686")
        message("ARCH is x86 & Building for 32 bit")
        set(CMAKE_C_FLAGS_DEBUG     "-m32 ${CMAKE_C_FLAGS_DEBUG}")
        set(CMAKE_C_FLAGS_RELEASE   "-m32 ${CMAKE_C_FLAGS_RELEASE}")
        set(CMAKE_CXX_FLAGS_DEBUG   "-m32 ${CMAKE_CXX_FLAGS_DEBUG}")
        set(CMAKE_CXX_FLAGS_RELEASE "-m32 ${CMAKE_CXX_FLAGS_RELEASE}")
    else()
        message("ARCH is x64 & Building for 64 bit")
    endif()

    if(COVERAGE)
        message("Coverage is On")
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} --coverage")
    endif()

    message("ARCH is: "                ${ARCH})
    message("CMAKE_C_FLAGS_DEBUG: "    ${CMAKE_C_FLAGS_DEBUG})
    message("CMAKE_CXX_FLAGS_DEBUG: "  ${CMAKE_CXX_FLAGS_DEBUG})
    message("CMAKE_C_FLAGS_RELEASE: "  ${CMAKE_C_FLAGS_RELEASE})
    message("CMAKE_CXX_FLAGS_RELEASE:" ${CMAKE_CXX_FLAGS_RELEASE})
endif()

set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_SOURCE_DIR}")

# Fetch the GoogleTest library
FetchContent_Declare(
  googletest
   URL https://github.com/google/googletest/archive/refs/tags/v1.13.0.zip
)
FetchContent_MakeAvailable(googletest)

# build EDIE components
add_subdirectory(src/decoders)
add_subdirectory(src/hw_interface)

# Find Git package, if not need to install manually or through .yml file
find_package(Git)
if(Git_FOUND)
  message("Git found: " ${GIT_EXECUTABLE})
endif()

# Build version of EDIE through cmake
if(GIT_EXECUTABLE)
    execute_process(COMMAND ${CMAKE_COMMAND}
        -D SRC=${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
        -D DST=${CMAKE_CURRENT_SOURCE_DIR}/src/version.h
        -D GIT_EXECUTABLE=${GIT_EXECUTABLE}
        -D GIT_BRANCH=${GIT_BRANCH}
        -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/GenerateVersionHeader.cmake)
endif()

add_subdirectory(src/decoders/common/test)
add_subdirectory(src/decoders/novatel/test)
add_subdirectory(src/hw_interface/stream_interface/test)

add_subdirectory(examples/novatel/command_encoding)
add_subdirectory(examples/novatel/converter_fileparser)
add_subdirectory(examples/novatel/converter_components)
add_subdirectory(examples/novatel/converter_parser)
add_subdirectory(examples/novatel/json_parser)
add_subdirectory(examples/novatel/rangecmp)
add_subdirectory(examples/novatel/rxconfig)
add_subdirectory(examples/novatel/dynamic_components)
add_subdirectory(examples/novatel/dynamic_fileparser)
add_subdirectory(examples/novatel/dynamic_parser)

# This is an archive Static build(after combining stream, common, novatel component into libEDIE.a) in Linux
# if we do not require it , We can comment this section.
if(LINUX)
    add_custom_target(${PROJECT_NAME} ALL COMMAND ${CMAKE_AR} rc libEDIE.a $<TARGET_FILE:stream_interface> $<TARGET_FILE:common> $<TARGET_FILE:novatel>)
    install(FILES ${CMAKE_BINARY_DIR}/libEDIE.a DESTINATION lib)
endif()

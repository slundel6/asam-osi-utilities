# SPDX-FileCopyrightText: Copyright (c) 2026, Bayerische Motoren Werke Aktiengesellschaft (BMW AG)
# SPDX-License-Identifier: MPL-2.0
cmake_minimum_required(VERSION 3.16)

# Add cmake modules directory
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cpp/cmake")

project(asam-osi-utilities)

# Generate compile_commands.json for clang based tools
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# User-configurable paths for OSI library and includes
set(OSI_LIBRARY_DIR "" CACHE PATH "Path to directory containing OSI static library")
set(OSI_INCLUDE_DIR "" CACHE PATH "Path to directory containing OSI headers")

# Optional: produce a single bundled static archive containing OSIUtilities + compression libs
option(OSIUTILITIES_BUNDLE_COMPRESSION_LIBS "Bundle lz4/zstd into a merged static archive (Linux/Unix)" OFF)
set(LZ4_STATIC_LIBRARY "" CACHE FILEPATH "Absolute path to liblz4.a for bundling")
set(ZSTD_STATIC_LIBRARY "" CACHE FILEPATH "Absolute path to libzstd.a for bundling")

if (NOT OSI_LIBRARY_DIR)
    message(FATAL_ERROR "OSI_LIBRARY_DIR must be set to the directory containing OSI libraries.\n"
                        "Example: cmake -DOSI_LIBRARY_DIR=/path/to/osi/lib -DOSI_INCLUDE_DIR=/path/to/osi/include ..")
endif ()

if (NOT OSI_INCLUDE_DIR)
    message(FATAL_ERROR "OSI_INCLUDE_DIR must be set to the directory containing OSI headers.\n"
                        "Example: cmake -DOSI_LIBRARY_DIR=/path/to/osi/lib -DOSI_INCLUDE_DIR=/path/to/osi/include ..")
endif ()

if (NOT EXISTS "${OSI_LIBRARY_DIR}")
    message(FATAL_ERROR "OSI_LIBRARY_DIR path does not exist: ${OSI_LIBRARY_DIR}")
endif ()

if (NOT EXISTS "${OSI_INCLUDE_DIR}")
    message(FATAL_ERROR "OSI_INCLUDE_DIR path does not exist: ${OSI_INCLUDE_DIR}")
endif ()

# Create imported target for OSI library
add_library(open_simulation_interface_static STATIC IMPORTED)

# Construct library path directly (no find_library, just direct path)
# Try _pic variant first (Position Independent Code), then standard
set(OSI_STATIC_LIB "${OSI_LIBRARY_DIR}/libopen_simulation_interface_pic.a")
if (NOT EXISTS "${OSI_STATIC_LIB}")
    set(OSI_STATIC_LIB "${OSI_LIBRARY_DIR}/libopen_simulation_interface.a")
endif ()

# For Windows, try .lib extension as well
if (WIN32 AND NOT EXISTS "${OSI_STATIC_LIB}")
    set(OSI_STATIC_LIB "${OSI_LIBRARY_DIR}/open_simulation_interface.lib")
endif ()

if (NOT EXISTS "${OSI_STATIC_LIB}")
    message(FATAL_ERROR "Could not find open_simulation_interface static library at:\n"
                        "  ${OSI_LIBRARY_DIR}/libopen_simulation_interface_pic.a\n"
                        "  ${OSI_LIBRARY_DIR}/libopen_simulation_interface.a\n"
                        "  ${OSI_LIBRARY_DIR}/open_simulation_interface.lib\n"
                        "Please check OSI_LIBRARY_DIR: ${OSI_LIBRARY_DIR}")
endif ()

# Check for protobuf library (needed by OSI)
set(PROTOBUF_STATIC_LIB "${OSI_LIBRARY_DIR}/libprotobuf.a")
if (NOT EXISTS "${PROTOBUF_STATIC_LIB}")
    set(PROTOBUF_STATIC_LIB "")
endif ()

set_target_properties(open_simulation_interface_static PROPERTIES
    IMPORTED_LOCATION "${OSI_STATIC_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES "${OSI_INCLUDE_DIR}"
)

# Add protobuf as a dependency if found in the same directory
if (PROTOBUF_STATIC_LIB)
    set_target_properties(open_simulation_interface_static PROPERTIES
        INTERFACE_LINK_LIBRARIES "${PROTOBUF_STATIC_LIB}"
    )
endif ()

# Create an alias for consistency with other approaches
add_library(open_simulation_interface::open_simulation_interface_static ALIAS open_simulation_interface_static)

message(STATUS "Found OSI library: ${OSI_STATIC_LIB}")
message(STATUS "OSI includes: ${OSI_INCLUDE_DIR}")
if (PROTOBUF_STATIC_LIB)
    message(STATUS "Found protobuf library: ${PROTOBUF_STATIC_LIB}")
endif ()

# Add the include folder for public headers
include_directories(${PROJECT_SOURCE_DIR}/cpp/include)

# Build the library
add_subdirectory(cpp/src)

message(STATUS "Build configuration:")
message(STATUS "  OSI library: ${OSI_LIBRARY_DIR}")
message(STATUS "  Output: Static library OSIUtilities")

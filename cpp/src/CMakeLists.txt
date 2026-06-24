# SPDX-FileCopyrightText: Copyright (c) 2026, Bayerische Motoren Werke Aktiengesellschaft (BMW AG)
# SPDX-License-Identifier: MPL-2.0

include_directories(tracefile)

# specify library source files
set(OSIUtilities_SRCS
        tracefile/FilenameUtils.cpp
        tracefile/reader/Reader.cpp
        tracefile/writer/Writer.cpp
        tracefile/MCAPImplementation.cpp
        tracefile/reader/SingleChannelBinaryTraceFileReader.cpp
        tracefile/writer/SingleChannelBinaryTraceFileWriter.cpp
        tracefile/reader/TXTHTraceFileReader.cpp
        tracefile/writer/TXTHTraceFileWriter.cpp
        tracefile/reader/MCAPTraceFileReader.cpp
        tracefile/writer/MCAPTraceFileWriter.cpp
        tracefile/writer/MCAPTraceFileChannel.cpp
)

# Create static library target
add_library(OSIUtilities STATIC ${OSIUtilities_SRCS})
add_library(OSIUtilities::OSIUtilities ALIAS OSIUtilities)

# Compiler configuration
target_compile_features(OSIUtilities PUBLIC cxx_std_17)
set_target_properties(OSIUtilities PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
    POSITION_INDEPENDENT_CODE ON
    EXPORT_NAME OSIUtilities
)

if (NOT MSVC)
    target_compile_options(OSIUtilities PRIVATE -fPIC)
endif ()

# Compile definition for trace file spec version
target_compile_definitions(OSIUtilities PRIVATE OSI_TRACE_FILE_SPEC_VERSION="${OSI_TRACE_FILE_SPEC_VERSION}")

# Include paths
target_include_directories(OSIUtilities
    PUBLIC
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/cpp/include>
)

# MCAP header-only library support
if (WIN32)
    target_compile_definitions(OSIUtilities PUBLIC MCAP_PUBLIC=)
endif ()
set(MCAP_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../submodules/mcap/cpp/mcap/include)
target_include_directories(OSIUtilities
    PUBLIC
        $<BUILD_INTERFACE:${MCAP_INCLUDE_DIR}>
)

# Link against static OSI library (target created in main CMakeLists.txt)
target_link_libraries(OSIUtilities PUBLIC open_simulation_interface::open_simulation_interface_static)

# Find and link compression libraries (LZ4, ZSTD)
include(FindCompression)
find_compression_libraries()

if (COMPRESSION_FOUND)
    target_link_libraries(OSIUtilities PRIVATE ${LZ4_TARGET} ${ZSTD_TARGET})
else ()
    message(FATAL_ERROR "Could not find LZ4 and ZSTD compression libraries")
endif ()

message(STATUS "OSIUtilities static library configured")

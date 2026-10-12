cmake_minimum_required(VERSION 3.16)
project(NetSpector VERSION 1.0)
set(CMAKE_PROJECT_AUTHOR "Priyanshu Raj")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Add -O3 optimization flag for release builds
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")

# Source files
set(SOURCES
    src/main.cpp
    src/pcap_reader.cpp
    src/packet_parser.cpp
)

# Header files location
include_directories(${CMAKE_SOURCE_DIR}/include)

# Create the executable (updated name to match the new brand)
add_executable(netspector ${SOURCES})

# For macOS, we might need to link against system libraries later
if(APPLE)
    # Add any macOS-specific settings here
endif()
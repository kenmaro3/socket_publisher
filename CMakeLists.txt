cmake_minimum_required(VERSION 3.1)
project(socket_publisher LANGUAGES CXX C)

# ----- Find dependencies -----
if(APPLE)
    # On macOS, use Homebrew's OpenMP
    execute_process(COMMAND brew --prefix libomp
                    OUTPUT_VARIABLE OpenMP_HOME
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(EXISTS "${OpenMP_HOME}")
        set(OpenMP_C_FLAGS "-Xpreprocessor -fopenmp -I${OpenMP_HOME}/include")
        set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp -I${OpenMP_HOME}/include")
        set(OpenMP_C_LIB_NAMES "omp")
        set(OpenMP_CXX_LIB_NAMES "omp")
        set(OpenMP_omp_LIBRARY "${OpenMP_HOME}/lib/libomp.dylib")
        link_directories("${OpenMP_HOME}/lib")
    else()
        message(FATAL_ERROR "OpenMP not found!")
    endif()
else()
    find_package(OpenMP REQUIRED)
    if(NOT OpenMP_FOUND)
        message(FATAL_ERROR "OpenMP not found!")
    endif()
endif()

find_package(stella_vslam REQUIRED)

# Find sioclient package
find_package(sioclient CONFIG REQUIRED)

# Protobuf
find_package(Protobuf REQUIRED)
if(NOT PROTOBUF_PROTOC_EXECUTABLE)
    message(FATAL_ERROR "Could not find protoc executable (PROTOBUF_PROTOC_EXECUTABLE)")
endif()
message(STATUS "Found protoc executable: ${PROTOBUF_PROTOC_EXECUTABLE}")

# ----- Protobuf transpile -----

protobuf_generate_cpp(MAP_PB_SOURCE MAP_PB_HEADER protobuf/map_segment.proto)

# ----- Set build type -----

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

option(BUILD_SHARED_LIBS "Build stella_vslam as a shared library" ON)

# ----- Configure SocketPublisher library -----

add_library(socket_publisher
    ${CMAKE_CURRENT_SOURCE_DIR}/src/data_serializer.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/src/publisher.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/src/socket_client.cc
    ${MAP_PB_SOURCE})

if(NOT MSVC)
    set_source_files_properties(${MAP_PB_HEADER} ${MAP_PB_SOURCE}
                                COMPILE_FLAGS -Wno-unused-variable)
endif()

set_target_properties(socket_publisher PROPERTIES
    OUTPUT_NAME socket_publisher
    ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
    LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

target_include_directories(socket_publisher PUBLIC "/usr/local/include")

target_include_directories(socket_publisher
    PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/>"
    "$<BUILD_INTERFACE:${SIOCLIENT_INCLUDE_DIR}>"
    "$<BUILD_INTERFACE:${PROTOBUF_INCLUDE_DIRS}>"
    "$<INSTALL_INTERFACE:include/>"
    PRIVATE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>")

target_link_libraries(socket_publisher
    PUBLIC
    stella_vslam::stella_vslam
    opencv_imgcodecs
    sioclient::sioclient
    ${PROTOBUF_LIBRARIES})

# ----- Install configuration -----
# (No changes needed)

include(CMakePackageConfigHelpers)

set(SOCKET_PUBLISHER_GENERATED_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
set(SOCKET_PUBLISHER_VERSION_CONFIG ${SOCKET_PUBLISHER_GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake)
set(SOCKET_PUBLISHER_PROJECT_CONFIG ${SOCKET_PUBLISHER_GENERATED_DIR}/${PROJECT_NAME}Config.cmake)
set(SOCKET_PUBLISHER_TARGETS_EXPORT_NAME ${PROJECT_NAME}Targets)
set(SOCKET_PUBLISHER_CONFIG_INSTALL_DIR lib/cmake/${PROJECT_NAME})
set(SOCKET_PUBLISHER_NAMESPACE "${PROJECT_NAME}::")
set(SOCKET_PUBLISHER_VERSION 0.0.1)

# Create a version config file
write_basic_package_version_file(${SOCKET_PUBLISHER_VERSION_CONFIG}
    VERSION ${SOCKET_PUBLISHER_VERSION}
    COMPATIBILITY SameMajorVersion)

configure_file(${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in ${SOCKET_PUBLISHER_PROJECT_CONFIG} @ONLY)

# Install to the specified directory
install(FILES ${SOCKET_PUBLISHER_VERSION_CONFIG} ${SOCKET_PUBLISHER_PROJECT_CONFIG}
    DESTINATION ${SOCKET_PUBLISHER_CONFIG_INSTALL_DIR})
install(EXPORT ${SOCKET_PUBLISHER_TARGETS_EXPORT_NAME}
    NAMESPACE ${SOCKET_PUBLISHER_NAMESPACE}
    DESTINATION ${SOCKET_PUBLISHER_CONFIG_INSTALL_DIR})

# Set standard installation directories
# (No changes needed)

set(RUNTIME_DESTINATION bin)
set(LIBRARY_DESTINATION lib)
set(ARCHIVE_DESTINATION lib)
set(INCLUDES_DESTINATION include)

set(SOCKER_PUBLISHER_INCLUDE_INSTALL_DIR ${INCLUDES_DESTINATION}/socket_publisher)

install(TARGETS socket_publisher
    EXPORT ${SOCKET_PUBLISHER_TARGETS_EXPORT_NAME}
    RUNTIME DESTINATION ${RUNTIME_DESTINATION}
    LIBRARY DESTINATION ${LIBRARY_DESTINATION}
    ARCHIVE DESTINATION ${ARCHIVE_DESTINATION}
    INCLUDES DESTINATION ${SOCKER_PUBLISHER_INCLUDE_INSTALL_DIR})

file(GLOB HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/socket_publisher/*.h")
install(FILES ${HEADERS}
    DESTINATION ${SOCKER_PUBLISHER_INCLUDE_INSTALL_DIR})


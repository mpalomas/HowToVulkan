# Copyright (c) 2025-2026, Sascha Willems
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")
include(FetchContent)
set(NAME HowToVulkan)
project(${NAME})

if (NOT IS_DIRECTORY $ENV{VULKAN_SDK})
    message(WARNING "Vulkan SDK not detected, if installed please set VULKAN_SDK env var or manually provide libraries")
else()
    message(STATUS "Found Vulkan SDK in $ENV{VULKAN_SDK}")
endif()

find_package(Vulkan REQUIRED)

# These are not part of the SDK
include_directories(external/tinyobj)
include_directories(external/ktx/include)
include_directories(external/ktx/other_include)

FetchContent_Declare(SFML
    GIT_REPOSITORY https://github.com/SFML/SFML.git
    GIT_TAG 3.0.1
    GIT_SHALLOW ON)
FetchContent_MakeAvailable(SFML)

# VMA, Volk, and Slang may be installed with the Vulkan SDK, or not

# Try to find Volk from the Vulkan SDK first
find_library(Volk_LIBRARY NAMES volk HINTS "$ENV{VULKAN_SDK}/lib")
find_path(Volk_INCLUDE_DIR NAMES volk.h HINTS "$ENV{VULKAN_SDK}/include")

if(Volk_LIBRARY AND Volk_INCLUDE_DIR)
    message(STATUS "Found Volk in Vulkan SDK: ${Volk_LIBRARY}")
    # Note: Since we use VOLK_HEADERS_ONLY, we only need includes
    add_library(volk::volk_headers INTERFACE IMPORTED)
    target_include_directories(volk::volk_headers INTERFACE ${Volk_INCLUDE_DIR})
else()
    message(STATUS "Volk not found in Vulkan SDK, fetching from GitHub")
    FetchContent_Declare(volk
        GIT_REPOSITORY https://github.com/zeux/volk
        GIT_TAG 1.4.304
        GIT_SHALLOW ON)
    set(VOLK_HEADERS_ONLY ON)
    FetchContent_MakeAvailable(volk)
endif()

# Try to find GLM from the Vulkan SDK first
find_path(GLM_INCLUDE_DIR NAMES glm/glm.hpp HINTS "$ENV{VULKAN_SDK}/include")

if(GLM_INCLUDE_DIR)
    message(STATUS "Found GLM in Vulkan SDK: ${GLM_INCLUDE_DIR}")
    add_library(glm::glm INTERFACE IMPORTED)
    target_include_directories(glm::glm INTERFACE ${GLM_INCLUDE_DIR})
else()
    message(STATUS "GLM not found in Vulkan SDK, fetching from GitHub")
    FetchContent_Declare(glm
        GIT_REPOSITORY https://github.com/g-truc/glm.git
        GIT_TAG 1.0.3
        GIT_SHALLOW ON)
    FetchContent_MakeAvailable(glm)
endif()

# Try to find VMA (AMD Vulkan Memory Allocator) from the Vulkan SDK first
find_path(VMA_INCLUDE_DIR NAMES vma/vk_mem_alloc.h HINTS "$ENV{VULKAN_SDK}/include")

if(VMA_INCLUDE_DIR)
    message(STATUS "Found VMA in Vulkan SDK: ${VMA_INCLUDE_DIR}")
    add_library(vma INTERFACE IMPORTED)
    target_include_directories(vma INTERFACE ${VMA_INCLUDE_DIR})
else()
    message(STATUS "VMA not found in Vulkan SDK, fetching from GitHub")
    FetchContent_Declare(vma
        GIT_REPOSITORY https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator.git
        GIT_TAG e722e57c891a8fbe3cc73ca56c19dd76be242759
        GIT_SHALLOW OFF)
    FetchContent_MakeAvailable(vma)
endif()

OPTION(USE_D2D_WSI "Build the project using Direct to Display swapchain" OFF)
OPTION(USE_WAYLAND_WSI "Build the project using Wayland swapchain" OFF)

set(KTX_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/ktx)
set(KTX_SOURCES
    ${KTX_DIR}/lib/texture.c
    ${KTX_DIR}/lib/hashlist.c
    ${KTX_DIR}/lib/checkheader.c
    ${KTX_DIR}/lib/swap.c
    ${KTX_DIR}/lib/memstream.c
    ${KTX_DIR}/lib/filestream.c
    ${KTX_DIR}/lib/vkloader.c)

add_library(ktx STATIC ${KTX_SOURCES})
target_link_libraries(ktx PRIVATE Vulkan::Headers)

find_library(Slang_LIBRARY NAMES slang HINTS "$ENV{VULKAN_SDK}/lib" REQUIRED)

if(WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_WIN32_KHR")
elseif(LINUX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DVK_USE_PLATFORM_WAYLAND_KHR")
endif()
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX -D_USE_MATH_DEFINES")
if(MSVC)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")
endif()
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/")

add_executable(${NAME} main.cpp assets/shader.slang)
target_compile_definitions(${NAME} PRIVATE VK_NO_PROTOTYPES)
set_target_properties(${NAME} PROPERTIES DEBUGGER_WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_features(${NAME} PRIVATE cxx_std_20)
target_link_libraries(${NAME} PRIVATE SFML::Graphics volk::volk_headers ktx glm::glm vma Vulkan::Headers ${Slang_LIBRARY})

# Copy assets to build directory
add_custom_command(TARGET ${NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_CURRENT_SOURCE_DIR}/assets
    ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/assets
    COMMENT "Copying assets to build directory"
)
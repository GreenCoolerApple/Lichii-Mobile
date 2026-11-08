# SPDX-FileCopyrightText: Copyright 2026 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later

include_guard(GLOBAL)

if(NOT ANDROID AND NOT CMAKE_SYSTEM_NAME STREQUAL "iOS")
    message(STATUS "Lichii mobile configuration skipped on ${CMAKE_SYSTEM_NAME}")
    return()
endif()

option(LICHII_MOBILE_ENABLE_VULKAN "Enable Vulkan support on Android" ON)
option(LICHII_MOBILE_ENABLE_METAL "Enable Metal framework linking on iOS" ON)
option(LICHII_MOBILE_STRIP_RELEASE "Strip native release binaries when supported" OFF)

add_library(lichii_mobile_config INTERFACE)
add_library(lichii::mobile_config ALIAS lichii_mobile_config)
target_compile_features(lichii_mobile_config INTERFACE cxx_std_20)
target_compile_definitions(lichii_mobile_config INTERFACE LICHII_MOBILE=1)

if(ANDROID)
    if(NOT CMAKE_ANDROID_ARCH_ABI STREQUAL "arm64-v8a" AND NOT ANDROID_ABI STREQUAL "arm64-v8a")
        message(FATAL_ERROR "Lichii Android currently targets arm64-v8a; set ANDROID_ABI=arm64-v8a")
    endif()

    target_compile_definitions(lichii_mobile_config INTERFACE LICHII_ANDROID=1)
    find_library(LICHII_ANDROID_LOG_LIBRARY log REQUIRED)
    find_library(LICHII_ANDROID_LIBRARY android REQUIRED)
    target_link_libraries(lichii_mobile_config INTERFACE
        ${LICHII_ANDROID_LOG_LIBRARY}
        ${LICHII_ANDROID_LIBRARY}
    )

    if(LICHII_MOBILE_ENABLE_VULKAN)
        find_library(LICHII_ANDROID_VULKAN_LIBRARY vulkan REQUIRED)
        target_link_libraries(lichii_mobile_config INTERFACE ${LICHII_ANDROID_VULKAN_LIBRARY})
        target_compile_definitions(lichii_mobile_config INTERFACE LICHII_MOBILE_VULKAN=1)
    endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "iOS")
    if(NOT CMAKE_OSX_ARCHITECTURES MATCHES "(^|;)arm64($|;)")
        message(FATAL_ERROR "Lichii iOS requires CMAKE_OSX_ARCHITECTURES=arm64")
    endif()

    target_compile_definitions(lichii_mobile_config INTERFACE LICHII_IOS=1)
    foreach(framework Foundation UIKit QuartzCore CoreGraphics AudioToolbox AVFoundation)
        find_library(LICHII_${framework}_FRAMEWORK ${framework} REQUIRED)
        target_link_libraries(lichii_mobile_config INTERFACE ${LICHII_${framework}_FRAMEWORK})
    endforeach()
    if(LICHII_MOBILE_ENABLE_METAL)
        find_library(LICHII_Metal_FRAMEWORK Metal REQUIRED)
        target_link_libraries(lichii_mobile_config INTERFACE ${LICHII_Metal_FRAMEWORK})
        target_compile_definitions(lichii_mobile_config INTERFACE LICHII_MOBILE_METAL=1)
    endif()
endif()

if(LICHII_MOBILE_STRIP_RELEASE AND CMAKE_STRIP)
    message(STATUS "Release stripping requested: apply ${CMAKE_STRIP} to final mobile artifacts in packaging")
endif()

message(STATUS "Lichii mobile interface target configured for ${CMAKE_SYSTEM_NAME}")

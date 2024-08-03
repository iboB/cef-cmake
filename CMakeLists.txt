# CEF-CMake 1.0.0
# Copyright (c) 2019 Borislav Stanimirov
#
# Distributed under the MIT Software License
# See accompanying file LICENSE.txt or copy at
# http://opensource.org/licenses/MIT
#
if(NOT CEF_CMAKE_INCLUDED)
    message(FATAL_ERROR "CEF-CMake: Configuration not included. You need to include `cef_cmake` in your root CMakeLists.txt file")
endif()

if(NOT CEF_VERSION)
    set(CEF_VERSION 126.2.18+g3647d39+chromium-126.0.6478.183)
    message(STATUS "CEF-CMake: CEF_VERSION not specified. Defaulting to ${CEF_VERSION}")
endif()

if(NOT CEF_CMAKE_OUTPUT_DIR)
    if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
        message(SEND_ERROR "CEF-CMake: Neither CEF_CMAKE_OUTPUT_DIR nor CMAKE_RUNTIME_OUTPUT_DIRECTORY was specified. You need to specify one for the binary assets of CEF to be copied.")
    endif()
    # CEF_CMAKE_OUTPUT_DIR is used to copy the required shared libraries next to the executable
    set(CEF_CMAKE_OUTPUT_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
endif()

if(CEF_CMAKE_OS_LINUX)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(cefName cef_binary_${CEF_VERSION}_linux64)
    else()
        set(cefName cef_binary_${CEF_VERSION}_linux32)
    endif()
elseif(CEF_CMAKE_OS_WINDOWS)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(cefName cef_binary_${CEF_VERSION}_windows64)
    else()
        set(cefName cef_binary_${CEF_VERSION}_windows32)
    endif()
else()
    message(FATAL_ERROR "CEF-CMake: Download platform not supported yet")
endif()

set(cefArchiveURL https://cef-builds.spotifycdn.com/${cefName}.tar.bz2)
# fix the url as the version may contain pluses
string(REGEX REPLACE "\\+" "%2B" cefArchiveURL ${cefArchiveURL})
set(cefArchive ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}.tar.bz2)

if(NOT EXISTS ${cefArchive})
    # download cef version
    message(STATUS "CEF-CMake: Downloading CEF ${cefArchiveURL}")
    file(DOWNLOAD ${cefArchiveURL} ${cefArchive}
        SHOW_PROGRESS
    )

    # ... and extract
    message(STATUS "CEF-CMake: Extracting ${cefArchive}")
    execute_process(
        COMMAND ${CMAKE_COMMAND} -E tar xzf ${cefArchive}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
endif()

set(srcs)
set(wdir ${cefName}/libcef_dll)
file(GLOB_RECURSE srcs "${wdir}/*.cc" "${wdir}/*.mm" "${wdir}/*.h" "${cefName}/include/*.h")

add_library(cefdll_wrapper STATIC ${srcs})

target_compile_definitions(cefdll_wrapper
    PRIVATE
        # Creating the CEF wrapper library. Do not define this for dependent targets.
        -DWRAPPING_CEF_SHARED
    PUBLIC
        # Allow C++ programs to use stdint.h macros specified in the C99 standard that aren't
        # in the C++ standard (e.g. UINT8_MAX, INT64_MIN, etc)
        -D__STDC_CONSTANT_MACROS
        -D__STDC_FORMAT_MACROS
)

target_include_directories(cefdll_wrapper
    PUBLIC ${cefName}
    INTERFACE include
)

add_custom_command(TARGET cefdll_wrapper POST_BUILD
    COMMENT "cefdll_wrapper: Copying CEF resources"
    COMMAND ${CMAKE_COMMAND} -E
        make_directory ${CEF_CMAKE_OUTPUT_DIR}
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/locales
        ${CEF_CMAKE_OUTPUT_DIR}/locales
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/chrome_100_percent.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/chrome_200_percent.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/resources.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/icudtl.dat
        ${CEF_CMAKE_OUTPUT_DIR}
)

if(CEF_CMAKE_OS_LINUX)
    target_link_libraries(cefdll_wrapper INTERFACE
        X11
        pthread
        debug ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Debug/libcef.so
        optimized ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Release/libcef.so
    )
    
    if (NOT $<CONFIG> STREQUAL "Debug")
        set(CEF_RESOURCE_CONFIG "Release")
    else()
        set(CEF_RESOURCE_CONFIG "Debug")
    endif()

    if (NOT CEF_USE_SANDBOX)
        set(CEF_SANDBOX_BINARY_PATH ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/chrome-sandbox)
    endif()

    add_custom_command(TARGET cefdll_wrapper POST_BUILD
        COMMENT "cefdll_wrapper: Copying CEF binaries"
        COMMAND ${CMAKE_COMMAND} -E
            make_directory ${CEF_CMAKE_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CEF_SANDBOX_BINARY_PATH}
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libcef.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libEGL.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libGLESv2.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/snapshot_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/v8_context_snapshot.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/vk_swiftshader_icd.json
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libvk_swiftshader.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libvulkan.so.1
            ${CEF_CMAKE_OUTPUT_DIR}
    )


    if(CEF_USE_SANDBOX)
        target_compile_definitions(cefdll_wrapper
            PUBLIC -DCEF_USE_SANDBOX
        )
    endif()

elseif(CEF_CMAKE_OS_WINDOWS)

    target_compile_definitions(cefdll_wrapper PUBLIC
        -DNOMINMAX
        -DWIN32_LEAN_AND_MEAN
        -DUNICODE
        -D_UNICODE
    )

    target_link_libraries(cefdll_wrapper PUBLIC
        comctl32.lib
        rpcrt4.lib
        shlwapi.lib
        ws2_32.lib
        debug ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Debug/libcef.lib
        optimized ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Release/libcef.lib
    )

    if (NOT $<CONFIG> STREQUAL "Debug")
        set(CEF_RESOURCE_CONFIG "Release")
    else()
        set(CEF_RESOURCE_CONFIG "Debug")
    endif()

    add_custom_command(TARGET cefdll_wrapper POST_BUILD
        COMMENT "cefdll_wrapper: Copying CEF binaries"
        COMMAND ${CMAKE_COMMAND} -E
            make_directory ${CEF_CMAKE_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/chrome_elf.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/d3dcompiler_47.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/dxcompiler.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/dxil.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libcef.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libEGL.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/libGLESv2.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/snapshot_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/v8_context_snapshot.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/vk_swiftshader_icd.json
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/vk_swiftshader.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/${CEF_RESOURCE_CONFIG}/vulkan-1.dll
            ${CEF_CMAKE_OUTPUT_DIR}
    )

    if(CEF_USE_SANDBOX)
        target_compile_definitions(cefdll_wrapper
            PUBLIC
                -DCEF_USE_SANDBOX
                -DPSAPI_VERSION=1
        )

        target_link_libraries(cefdll_wrapper PUBLIC
            dbghelp.lib
            psapi.lib
            version.lib
            wbemuuid.lib
            winmm.lib
            debug ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Debug/cef_sandbox.lib
            optimized ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Release/cef_sandbox.lib
        )
    endif()

else()
    # TODO: Copy macos shared libraries
    message(FATAL_ERROR "CEF-CMake: Build platform not supported yet")
endif()

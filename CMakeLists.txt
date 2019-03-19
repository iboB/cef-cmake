# CEF-CMake
# Copyright (c) 2019 Borislav Stanimirov
#
# Distributed under the MIT Software License
# See accompanying file LICENSE.txt or copy at
# http://opensource.org/licenses/MIT
#
if(NOT CEF_VERSION)
    set(CEF_VERSION 3.3626.1894.g90eb8cc)
    message(STATUS "CEF-CMake: CEF_VERSION not specified. Defaulting to ${CEF_VERSION}")
endif()

if(NOT CEF_CMAKE_OUTPUT_DIR)
    if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
        set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
        message(STATUS "CEF-CMake: CMAKE_RUNTIME_OUTPUT_DIRECTORY was not specified. Defaulting to: ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
    endif()
    # CEF_CMAKE_OUTPUT_DIR is used to copy the required shared libraries next to the executable
    set(CEF_CMAKE_OUTPUT_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
    message(STATUS "CEF-CMake: CEF_CMAKE_OUTPUT_DIR was not specified. Defaulting to CMAKE_RUNTIME_OUTPUT_DIRECTORY: ${CEF_CMAKE_OUTPUT_DIR}")
endif()

if(MSVC)
    set(CEF_CMAKE_OUTPUT_DIR ${CEF_CMAKE_OUTPUT_DIR}/$<CONFIG>)
endif()

if(OS_LINUX)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(cefName cef_binary_${CEF_VERSION}_linux64)
    else()
        set(cefName cef_binary_${CEF_VERSION}_linux32)
    endif()
elseif(OS_WINDOWS)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(cefName cef_binary_${CEF_VERSION}_windows64)
    else()
        set(cefName cef_binary_${CEF_VERSION}_windows32)
    endif()
else()
    message(FATAL_ERROR "CEF-CMake: Download platform not supported yet")
endif()

set(cefArchiveURL http://opensource.spotify.com/cefbuilds/${cefName}.tar.bz2)
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

option(CEF_USE_SANDBOX "CEF-CMake: Enable or disable use of the CEF sandbox." ON)

if(MSVC AND CEF_USE_SANDBOX)
    message(STATUS "CEF-CMake: Replacing /MD->/MT in C and CXX_FLAGS. Required by CEF sandbox.")
    set(CompilerFlags
        CMAKE_CXX_FLAGS
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_RELEASE
        CMAKE_C_FLAGS
        CMAKE_C_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELEASE
    )
    foreach(CompilerFlag ${CompilerFlags})
        string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
    endforeach()
endif()


include(cefdll_wrapper_sources.cmake)

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
)

add_custom_command(TARGET cefdll_wrapper POST_BUILD
    COMMENT "cefdll_wrapper: Copying CEF resources"
    COMMAND ${CMAKE_COMMAND} -E
        make_directory ${CEF_CMAKE_OUTPUT_DIR}
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/locales
        ${CEF_CMAKE_OUTPUT_DIR}/locales
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/cef.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/cef_100_percent.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/cef_200_percent.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/cef_extensions.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/devtools_resources.pak
        ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Resources/icudtl.dat
        ${CEF_CMAKE_OUTPUT_DIR}
)

if(OS_LINUX)
    target_link_libraries(cefdll_wrapper INTERFACE
        X11
        pthread
        debug ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Debug/libcef.so
        optimized ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/Release/libcef.so
    )

    add_custom_command(TARGET cefdll_wrapper POST_BUILD
        COMMENT "cefdll_wrapper: Copying CEF binaries"
        COMMAND ${CMAKE_COMMAND} -E
            make_directory ${CEF_CMAKE_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/swiftshader
            ${CEF_CMAKE_OUTPUT_DIR}/swiftshader
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/chrome-sandbox
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libcef.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libEGL.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libGLESv2.so
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/natives_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/snapshot_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/v8_context_snapshot.bin
            ${CEF_CMAKE_OUTPUT_DIR}
    )


    if(CEF_USE_SANDBOX)
        target_compile_definitions(cefdll_wrapper
            PUBLIC -DCEF_USE_SANDBOX
        )
    endif()

elseif(OS_WINDOWS)

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

    add_custom_command(TARGET cefdll_wrapper POST_BUILD
        COMMENT "cefdll_wrapper: Copying CEF binaries"
        COMMAND ${CMAKE_COMMAND} -E
            make_directory ${CEF_CMAKE_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/swiftshader
            ${CEF_CMAKE_OUTPUT_DIR}/swiftshader
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/chrome_elf.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/d3dcompiler_43.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/d3dcompiler_47.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libcef.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libEGL.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/libGLESv2.dll
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/natives_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/snapshot_blob.bin
            ${CMAKE_CURRENT_SOURCE_DIR}/${cefName}/$<CONFIG>/v8_context_snapshot.bin
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
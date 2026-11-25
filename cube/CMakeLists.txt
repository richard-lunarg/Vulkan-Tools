# ~~~
# Copyright (c) 2018 Valve Corporation
# Copyright (c) 2018 LunarG, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# ~~~

if(APPLE)
    option(APPLE_USE_SYSTEM_ICD "Use system Vulkan ICD discovery instead of bundling drivers" OFF)

    # Remove drivers left behind when reconfiguring an existing app bundle.
    set(CUBE_UNUSED_ICD_FILES
        Frameworks/libMoltenVK.dylib
        Resources/vulkan/icd.d/MoltenVK_icd.json
        Frameworks/libvulkan_kosmickrisp.dylib
        Resources/vulkan/icd.d/libkosmickrisp_icd.json)

    if(APPLE_USE_SYSTEM_ICD)
        message(STATUS "Using system Vulkan loader for ICD discovery")
    else()
        # Select drivers for the target architectures, including universal builds.
        set(cube_macos_architectures ${CMAKE_OSX_ARCHITECTURES})
        if(NOT cube_macos_architectures)
            set(cube_macos_architectures ${CMAKE_SYSTEM_PROCESSOR})
        endif()

        set(CUBE_ICD_LIBRARIES)
        set(CUBE_ICD_MANIFESTS)
        set(cube_icd_sources)
        if("arm64" IN_LIST cube_macos_architectures)
            set(KOSMICKRISP_PATH "" CACHE PATH "SDK library directory containing libvulkan_kosmickrisp.dylib")
            if(DEFINED ENV{VULKAN_SDK} AND NOT "$ENV{VULKAN_SDK}" STREQUAL "")
                set(cube_kosmickrisp_path "$ENV{VULKAN_SDK}/lib")
            else()
                set(cube_kosmickrisp_path "${KOSMICKRISP_PATH}")
            endif()
            if(NOT cube_kosmickrisp_path)
                message(FATAL_ERROR "Set VULKAN_SDK or KOSMICKRISP_PATH to locate KosmicKrisp -- see BUILD.md")
            endif()
            get_filename_component(cube_kosmickrisp_path "${cube_kosmickrisp_path}" ABSOLUTE)
            message(STATUS "Using KosmicKrisp from ${cube_kosmickrisp_path}")
            list(APPEND CUBE_ICD_LIBRARIES "${cube_kosmickrisp_path}/libvulkan_kosmickrisp.dylib")
            list(APPEND cube_icd_sources "${cube_kosmickrisp_path}/../share/vulkan/icd.d/libkosmickrisp_icd.json")
        endif()

        if("x86_64" IN_LIST cube_macos_architectures)
            set(MOLTENVK_REPO_ROOT "MOLTENVK-NOTFOUND" CACHE PATH "Absolute path to a MoltenVK repo directory")
            if(NOT MOLTENVK_REPO_ROOT)
                message(FATAL_ERROR "Must define location of MoltenVK repo -- see BUILD.md")
            endif()
            message(STATUS "Using MoltenVK repo location at ${MOLTENVK_REPO_ROOT}")
            list(APPEND CUBE_ICD_LIBRARIES "${MOLTENVK_REPO_ROOT}/Package/Release/MoltenVK/dynamic/dylib/macOS/libMoltenVK.dylib")
            list(APPEND cube_icd_sources "${MOLTENVK_REPO_ROOT}/MoltenVK/icd/MoltenVK_icd.json")
        endif()

        # Preserve each SDK/driver manifest's metadata and point it into the bundle.
        file(MAKE_DIRECTORY "${PROJECT_BINARY_DIR}/staging-json")
        foreach(library manifest IN ZIP_LISTS CUBE_ICD_LIBRARIES cube_icd_sources)
            if(NOT EXISTS "${library}" OR NOT EXISTS "${manifest}")
                message(FATAL_ERROR "Bundled Vulkan driver requires both ${library} and ${manifest}")
            endif()
            get_filename_component(library_name "${library}" NAME)
            get_filename_component(manifest_name "${manifest}" NAME)
            set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${manifest}")
            file(READ "${manifest}" manifest_json)
            string(JSON manifest_json SET "${manifest_json}" ICD library_path "\"../../../Frameworks/${library_name}\"")
            set(staged_manifest "${PROJECT_BINARY_DIR}/staging-json/${manifest_name}")
            file(WRITE "${staged_manifest}" "${manifest_json}\n")
            list(APPEND CUBE_ICD_MANIFESTS "${staged_manifest}")
            list(REMOVE_ITEM CUBE_UNUSED_ICD_FILES "Frameworks/${library_name}" "Resources/vulkan/icd.d/${manifest_name}")
        endforeach()
    endif()

    find_library(COCOA NAMES Cocoa)

    # Locate Interface Builder Tool, needed to build things like Storyboards outside of Xcode.
    if(NOT XCODE)
        # Make sure we can find the 'ibtool' program. If we can NOT find it we skip generation of this project.
        find_program(IBTOOL ibtool HINTS "/usr/bin" "${OSX_DEVELOPER_ROOT}/usr/bin")
        if(${IBTOOL} STREQUAL "IBTOOL-NOTFOUND")
            message(SEND_ERROR "ibtool can not be found and is needed to compile the .xib files. "
                            "It should have been installed with the Apple developer tools. "
                            "The default system paths were searched in addition to ${OSX_DEVELOPER_ROOT}/usr/bin.")
        endif()
    endif()
endif()

if(ANDROID OR APPLE)
    set(WSI_DISPLAY_DEFAULT_SETTING "OFF")
else()
    set(WSI_DISPLAY_DEFAULT_SETTING "ON")
endif()

option(BUILD_WSI_DISPLAY_SUPPORT "Build DISPLAY WSI support" ${WSI_DISPLAY_DEFAULT_SETTING})

if (CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    option(BUILD_WSI_XCB_SUPPORT "Build XCB WSI support" ON)
    option(BUILD_WSI_XLIB_SUPPORT "Build Xlib WSI support" ON)
    option(BUILD_WSI_WAYLAND_SUPPORT "Build Wayland WSI support" ON)
    option(BUILD_WSI_DIRECTFB_SUPPORT "Build DirectFB WSI support" OFF)

    find_package(PkgConfig REQUIRED QUIET) # Use PkgConfig to find Linux system libraries

    if(BUILD_WSI_XCB_SUPPORT)
        pkg_check_modules(XCB REQUIRED QUIET IMPORTED_TARGET xcb)
        pkg_get_variable(XCB_INCLUDE_DIRS xcb includedir)
        message(DEBUG "XCB_INCLUDE_DIRS = ${XCB_INCLUDE_DIRS}")
    endif()

    if(BUILD_WSI_XLIB_SUPPORT)
        pkg_check_modules(X11 REQUIRED QUIET IMPORTED_TARGET x11)
        pkg_get_variable(XLIB_INCLUDE_DIRS x11 includedir)
        message(DEBUG "XLIB_INCLUDE_DIRS = ${XLIB_INCLUDE_DIRS}")
    endif()

    if(BUILD_WSI_WAYLAND_SUPPORT)
        pkg_check_modules(WAYLAND_CLIENT REQUIRED IMPORTED_TARGET wayland-client)
        pkg_get_variable(WAYLAND_INCLUDE_DIRS wayland-client includedir)

        pkg_check_modules(WAYLAND_SCANNER REQUIRED wayland-scanner)
        pkg_get_variable(WAYLAND_SCANNER_EXECUTABLE wayland-scanner wayland_scanner)
        message(DEBUG "WAYLAND_SCANNER_EXECUTABLE = ${WAYLAND_SCANNER_EXECUTABLE}")

        pkg_get_variable(WAYLAND_CLIENT_PATH wayland-client pkgdatadir)
        message(DEBUG "WAYLAND_CLIENT_PATH = ${WAYLAND_CLIENT_PATH}")
        set(WAYLAND_CODE_PROTOCOL ${WAYLAND_CLIENT_PATH}/wayland.xml)

        pkg_check_modules(WAYLAND_PROTOCOLS REQUIRED wayland-protocols)
        pkg_get_variable(WAYLAND_PROTOCOLS_PATH wayland-protocols pkgdatadir)
        message(DEBUG "WAYLAND_PROTOCOLS_PATH = ${WAYLAND_PROTOCOLS_PATH}")
        set(XDG_SHELL_PROTOCOL ${WAYLAND_PROTOCOLS_PATH}/stable/xdg-shell/xdg-shell.xml)

        add_custom_command(COMMENT "Generating wayland client protocol dispatch data"
                          OUTPUT wayland-client.c
                          COMMAND ${WAYLAND_SCANNER_EXECUTABLE}
                                  private-code
                                  ${WAYLAND_CODE_PROTOCOL}
                                  ${CMAKE_CURRENT_BINARY_DIR}/wayland-client.c
                          MAIN_DEPENDENCY ${WAYLAND_CODE_PROTOCOL}
                          DEPENDS ${WAYLAND_CODE_PROTOCOL} ${WAYLAND_SCANNER_EXECUTABLE})

        add_custom_command(COMMENT "Generating xdg-shell protocol dispatch data"
                           OUTPUT xdg-shell-code.c
                           COMMAND ${WAYLAND_SCANNER_EXECUTABLE}
                                   private-code
                                   ${XDG_SHELL_PROTOCOL}
                                   ${CMAKE_CURRENT_BINARY_DIR}/xdg-shell-code.c
                           MAIN_DEPENDENCY ${XDG_SHELL_PROTOCOL}
                           DEPENDS ${XDG_SHELL_PROTOCOL} ${WAYLAND_SCANNER_EXECUTABLE})
        add_custom_command(COMMENT "Generating xdg-shell protocol header"
                           OUTPUT xdg-shell-client-header.h
                           COMMAND ${WAYLAND_SCANNER_EXECUTABLE}
                                   client-header
                                   ${XDG_SHELL_PROTOCOL}
                                   ${CMAKE_CURRENT_BINARY_DIR}/xdg-shell-client-header.h
                           MAIN_DEPENDENCY ${XDG_SHELL_PROTOCOL}
                           DEPENDS ${XDG_SHELL_PROTOCOL} ${WAYLAND_SCANNER_EXECUTABLE})

        set(XDG_DECORATION_PROTOCOL ${WAYLAND_PROTOCOLS_PATH}/unstable/xdg-decoration/xdg-decoration-unstable-v1.xml)
        add_custom_command(COMMENT "Generating xdg-decoration protocol dispatch data"
                           OUTPUT xdg-decoration-code.c
                           COMMAND ${WAYLAND_SCANNER_EXECUTABLE}
                                   private-code
                                   ${XDG_DECORATION_PROTOCOL}
                                   ${CMAKE_CURRENT_BINARY_DIR}/xdg-decoration-code.c
                           MAIN_DEPENDENCY ${XDG_DECORATION_PROTOCOL}
                           DEPENDS ${XDG_DECORATION_PROTOCOL} ${WAYLAND_SCANNER_EXECUTABLE})
        add_custom_command(COMMENT "Generating xdg-decoration protocol header"
                           OUTPUT xdg-decoration-client-header.h
                           COMMAND ${WAYLAND_SCANNER_EXECUTABLE}
                                   client-header
                                   ${XDG_DECORATION_PROTOCOL}
                                   ${CMAKE_CURRENT_BINARY_DIR}/xdg-decoration-client-header.h
                           MAIN_DEPENDENCY ${XDG_DECORATION_PROTOCOL}
                           DEPENDS ${XDG_DECORATION_PROTOCOL} ${WAYLAND_SCANNER_EXECUTABLE})

        set(WAYLAND_ADDITIONAL_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/wayland-client.c
                           ${CMAKE_CURRENT_BINARY_DIR}/xdg-shell-code.c
                           ${CMAKE_CURRENT_BINARY_DIR}/xdg-shell-client-header.h
                           ${CMAKE_CURRENT_BINARY_DIR}/xdg-decoration-code.c
                           ${CMAKE_CURRENT_BINARY_DIR}/xdg-decoration-client-header.h)
    endif()

    if(BUILD_WSI_DIRECTFB_SUPPORT)
        pkg_check_modules(DirectFB REQUIRED QUIET IMPORTED_TARGET directfb)
    endif()
endif()

if(BUILD_WSI_DISPLAY_SUPPORT)
    list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_DISPLAY_KHR)
endif()

if(WIN32)
    add_definitions(-DWIN32_LEAN_AND_MEAN -DNOMINMAX)
    list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_WIN32_KHR)
elseif(ANDROID)
    list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_ANDROID_KHR)
elseif(APPLE)
    list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_METAL_EXT)
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    if(BUILD_WSI_XCB_SUPPORT)
        list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_XCB_KHR)
    endif()
    if(BUILD_WSI_XLIB_SUPPORT)
        list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_XLIB_KHR)
    endif()
    if(BUILD_WSI_WAYLAND_SUPPORT)
        list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_WAYLAND_KHR)
    endif()
    if(BUILD_WSI_DIRECTFB_SUPPORT)
        list(APPEND ENABLED_CUBE_PLATFORMS VK_USE_PLATFORM_DIRECTFB_EXT)
    endif()
    link_libraries(${API_LOWERCASE} m)
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

if(NOT DEFINED ENABLED_CUBE_PLATFORMS)
        message(FATAL_ERROR "There are no supported WSI platforms on this system, vkcube requires a WSI platform be available to be able to render its output")
endif()

if (COMPILE_CUBE_SHADERS)
    # Try to find glslang in system paths or in an SDK if the VULKAN_SDK env-var is set
    find_program(GLSLANG_VALIDATOR names glslang glslangValidator HINTS $ENV{GLSLANG_INSTALL_DIR} $ENV{VULKAN_SDK}/bin $ENV{VULKAN_SDK}/Bin)

    add_custom_command(COMMENT "Compiling cube vertex shader"
                    OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/cube.vert.inc
                    COMMAND ${GLSLANG_VALIDATOR} -V -x -o ${CMAKE_CURRENT_SOURCE_DIR}/cube.vert.inc
                            ${PROJECT_SOURCE_DIR}/cube/cube.vert
                    MAIN_DEPENDENCY ${PROJECT_SOURCE_DIR}/cube/cube.vert
                    DEPENDS ${PROJECT_SOURCE_DIR}/cube/cube.vert ${GLSLANG_VALIDATOR})
    add_custom_command(COMMENT "Compiling cube fragment shader"
                    OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/cube.frag.inc
                    COMMAND ${GLSLANG_VALIDATOR} -V -x -o ${CMAKE_CURRENT_SOURCE_DIR}/cube.frag.inc
                            ${PROJECT_SOURCE_DIR}/cube/cube.frag
                    MAIN_DEPENDENCY ${PROJECT_SOURCE_DIR}/cube/cube.frag
                    DEPENDS ${PROJECT_SOURCE_DIR}/cube/cube.frag ${GLSLANG_VALIDATOR})
endif()

if(WIN32)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
    # vkcube / vkcube make use of M_PI and various other math defines.
    add_compile_definitions(_USE_MATH_DEFINES)
endif()


# On Ubuntu 20.04, it was found that the validation layers fail to launch in vkcube & vkcubepp due to
# a missing dependency on libpthread. While newer Ubuntu versions use a glibc version where libpthread
# is integrated into libc, older ubuntu's do not so we need to link threads directly in order for
# validation layers to be loadable.
if (CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    find_package(Threads REQUIRED)
endif()

# ----------------------------------------------------------------------------
# vkcube

if(APPLE)
    add_subdirectory(macOS/cube)
elseif (ANDROID)
    add_library(vkcube MODULE)

    target_sources(vkcube PRIVATE cube.c)

    add_subdirectory(android)

elseif(CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    add_executable(vkcube)
    target_sources(vkcube PRIVATE
        cube.c
        ${PROJECT_SOURCE_DIR}/cube/cube.vert
        ${PROJECT_SOURCE_DIR}/cube/cube.frag
        cube.vert.inc
        cube.frag.inc
    )
    target_link_libraries(vkcube  Threads::Threads)
    if(BUILD_WSI_XCB_SUPPORT)
        target_sources(vkcube PRIVATE xcb_loader.h)
        target_include_directories(vkcube PRIVATE ${xcb_INCLUDE_DIRS})
    endif()
    if(BUILD_WSI_XLIB_SUPPORT)
        target_sources(vkcube PRIVATE xlib_loader.h)
        target_include_directories(vkcube PRIVATE ${XLIB_INCLUDE_DIRS})
    endif()
    if(BUILD_WSI_WAYLAND_SUPPORT)
        target_include_directories(vkcube PRIVATE ${WAYLAND_INCLUDE_DIRS} ${CMAKE_CURRENT_BINARY_DIR})
        target_sources(vkcube PRIVATE PRIVATE ${WAYLAND_ADDITIONAL_SOURCES})
    endif()
    if(BUILD_WSI_DIRECTFB_SUPPORT)
        target_link_libraries(vkcube PkgConfig::DirectFB)
    endif()
    include(CheckLibraryExists)
    CHECK_LIBRARY_EXISTS("rt" clock_gettime "" NEED_RT)
    if (NEED_RT)
        target_link_libraries(vkcube  rt)
    endif()
elseif(WIN32)
    add_executable(vkcube WIN32)
    target_sources(vkcube PRIVATE
        cube.c
        ${PROJECT_SOURCE_DIR}/cube/cube.vert
        ${PROJECT_SOURCE_DIR}/cube/cube.frag
        cube.vert.inc
        cube.frag.inc
    )
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

target_compile_definitions(vkcube PRIVATE ${ENABLED_CUBE_PLATFORMS})
target_include_directories(vkcube PRIVATE .)
target_link_libraries(vkcube ${CMAKE_DL_LIBS} Vulkan::Headers)

if (ANDROID)
    install(TARGETS vkcube DESTINATION ${CMAKE_INSTALL_LIBDIR})
elseif(APPLE)
    install(
        TARGETS vkcube
        # Install runtime dependencies like the Vulkan::Loader so the app is self-contained
        RUNTIME_DEPENDENCIES DESTINATION "cube/vkcube.app/Contents/Frameworks/"
        BUNDLE DESTINATION "cube"
    )
    set_target_properties(vkcube PROPERTIES
        INSTALL_RPATH @executable_path/../Frameworks
    )
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    install(TARGETS vkcube)
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/vkcube.desktop
            DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/vkcube.png
            DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/256x256/apps
            RENAME vkcube.png)
else()
    install(TARGETS vkcube)
endif()

if (ANDROID)
    return()
endif()

# ----------------------------------------------------------------------------
# vkcubepp

if(APPLE)
    add_subdirectory(macOS/cubepp)
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
    add_executable(vkcubepp
                   cube.cpp
                   ${PROJECT_SOURCE_DIR}/cube/cube.vert
                   ${PROJECT_SOURCE_DIR}/cube/cube.frag
                   cube.vert.inc
                   cube.frag.inc)
    target_link_libraries(vkcubepp  Threads::Threads)

    if(BUILD_WSI_XCB_SUPPORT)
        target_sources(vkcubepp PRIVATE xcb_loader.h)
        target_include_directories(vkcubepp PRIVATE ${xcb_INCLUDE_DIRS})
    endif()
    if(BUILD_WSI_XLIB_SUPPORT)
        target_sources(vkcubepp PRIVATE xlib_loader.h)
        target_include_directories(vkcubepp PRIVATE ${XLIB_INCLUDE_DIRS})
    endif()
    if(BUILD_WSI_WAYLAND_SUPPORT)
        target_include_directories(vkcubepp PRIVATE ${WAYLAND_INCLUDE_DIRS} ${CMAKE_CURRENT_BINARY_DIR})
        target_sources(vkcubepp PRIVATE ${WAYLAND_ADDITIONAL_SOURCES})
    endif()
    if(BUILD_WSI_DIRECTFB_SUPPORT)
        target_link_libraries(vkcubepp  PkgConfig::DirectFB)
    endif()
else()
    add_executable(vkcubepp
                   WIN32
                   cube.cpp
                   ${PROJECT_SOURCE_DIR}/cube/cube.vert
                   ${PROJECT_SOURCE_DIR}/cube/cube.frag
                   cube.vert.inc
                   cube.frag.inc)
endif()

target_include_directories(vkcubepp PRIVATE .)
target_compile_definitions(vkcubepp PRIVATE ${ENABLED_CUBE_PLATFORMS})
target_link_libraries(vkcubepp ${CMAKE_DL_LIBS} Vulkan::Headers)

if(APPLE)
    install(
        TARGETS vkcubepp
        # Install runtime dependencies like the Vulkan::Loader so the app is self-contained
        RUNTIME_DEPENDENCIES DESTINATION "cube/vkcubepp.app/Contents/Frameworks/"
        BUNDLE DESTINATION "cube"
    )
    set_target_properties(vkcubepp PROPERTIES
        INSTALL_RPATH @executable_path/../Frameworks
    )
else()
    install(TARGETS vkcubepp)
    if(CMAKE_SYSTEM_NAME MATCHES "Linux|BSD|GNU")
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/vkcubepp.desktop
                DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)

        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/vkcube.png
                DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/256x256/apps
                RENAME vkcubepp.png)
    endif()
endif()

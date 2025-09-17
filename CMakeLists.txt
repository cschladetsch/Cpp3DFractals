cmake_minimum_required(VERSION 3.10)
project(BoxPlorer2 LANGUAGES CXX C)

# Set C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Platform-specific configuration
if(WIN32)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDLL")
endif()

# Find packages
find_package(OpenGL REQUIRED)

# Find PkgConfig only on Linux
if(NOT WIN32 AND NOT APPLE)
    find_package(PkgConfig REQUIRED)
endif()

# Try to find SDL2 using modern CMake
find_package(SDL2 QUIET)

# If not found, try legacy methods
if(NOT SDL2_FOUND)
    if(WIN32)
        # On Windows, look for SDL2 in common locations
        find_path(SDL2_INCLUDE_DIR SDL.h
            HINTS
            ${CMAKE_CURRENT_SOURCE_DIR}/SDL2/include
            PATH_SUFFIXES SDL2
        )
        find_library(SDL2_LIBRARY
            NAMES SDL2 SDL2main
            HINTS
            ${CMAKE_CURRENT_SOURCE_DIR}/SDL2/lib/x86
            ${CMAKE_CURRENT_SOURCE_DIR}/SDL2/lib/x64
        )
        if(SDL2_INCLUDE_DIR AND SDL2_LIBRARY)
            set(SDL2_FOUND TRUE)
        endif()
    elseif(APPLE)
        # On macOS, look for SDL2 framework
        find_library(SDL2_LIBRARY SDL2 PATHS /Library/Frameworks)
        if(SDL2_LIBRARY)
            set(SDL2_FOUND TRUE)
            set(SDL2_INCLUDE_DIR ${SDL2_LIBRARY}/Headers)
        endif()
    else()
        # On Linux, use pkg-config
        pkg_check_modules(SDL2 QUIET sdl2)
    endif()
endif()

# If still not found, fetch SDL2 automatically
if(NOT SDL2_FOUND)
    include(FetchContent)
    message(STATUS "SDL2 not found, fetching from GitHub...")

    FetchContent_Declare(
        SDL2
        GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
        GIT_TAG release-2.28.5
        GIT_SHALLOW TRUE
        GIT_PROGRESS TRUE
    )

    # Configure SDL2 options
    set(SDL_SHARED OFF CACHE BOOL "Build SDL2 as shared library")
    set(SDL_STATIC ON CACHE BOOL "Build SDL2 as static library")
    set(SDL_TEST OFF CACHE BOOL "Build SDL2 test programs")

    FetchContent_MakeAvailable(SDL2)

    # Set SDL2 variables for compatibility
    set(SDL2_FOUND TRUE)
    set(SDL2_INCLUDE_DIR ${SDL2_SOURCE_DIR}/include)
    set(SDL2_LIBRARY SDL2-static)
    set(SDL2_LIBRARIES SDL2-static)
endif()

# Include directories
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/AntTweakBar/include
    ${CMAKE_CURRENT_SOURCE_DIR}/AntTweakBar/src
    ${CMAKE_CURRENT_SOURCE_DIR}/GL
    ${OPENGL_INCLUDE_DIRS}
)

# Add SDL2 include directory if found
if(SDL2_FOUND AND SDL2_INCLUDE_DIR)
    include_directories(${SDL2_INCLUDE_DIR})
endif()

# Platform-specific include directories
if(WIN32)
    include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}/OculusSDK4/LibOVR/Include
        ${CMAKE_CURRENT_SOURCE_DIR}/sixense
    )
    # Add Windows DDK ATL if available
    if(EXISTS "C:/WinDDK/7600.16385.1/inc/atl71")
        include_directories("C:/WinDDK/7600.16385.1/inc/atl71")
    endif()
elseif(APPLE)
    include_directories(
        /System/Library/Frameworks/OpenGL.framework/Headers
        /System/Library/Frameworks/AppKit.framework/Headers
    )
endif()

# Compiler-specific flags
if(WIN32 AND MSVC)
    add_compile_definitions(
        _CONSOLE
        WIN32
        TW_NO_LIB_PRAGMA
        TW_STATIC
        HYDRAX
        ANT_DEBUG
    )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W3 /EHsc /Ox")
elseif(APPLE)
    add_compile_definitions(
        TW_NO_LIB_PRAGMA
        TW_STATIC
        _MACOSX
        __PLACEMENT_NEW_INLINE
    )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -ObjC++")
else()
    add_compile_definitions(
        TW_NO_LIB_PRAGMA
        TW_STATIC
        _UNIX
    )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -Wall")
endif()

# AntTweakBar source files
set(ANTTWEAKBAR_SOURCES
    AntTweakBar/src/TwBar.cpp
    AntTweakBar/src/TwFonts.cpp
    AntTweakBar/src/TwMgr.cpp
    AntTweakBar/src/TwColors.cpp
    AntTweakBar/src/TwOpenGL.cpp
    AntTweakBar/src/TwPrecomp.cpp
    AntTweakBar/src/LoadOGL.cpp
    AntTweakBar/src/LoadOGLCore.cpp
    AntTweakBar/src/TwOpenGLCore.cpp
)

# Add SDL event handlers only if SDL2 is available
if(SDL2_FOUND)
    add_compile_definitions(HAVE_SDL2)
    list(APPEND ANTTWEAKBAR_SOURCES
        AntTweakBar/src/TwEventSDL.c
        AntTweakBar/src/TwEventSDL12.c
        AntTweakBar/src/TwEventSDL13.c
        AntTweakBar/src/TwEventSDL20.c
    )
endif()

# BoxPlorer2 core source files
set(BOXPLORER2_SOURCES
    main.cpp
    Interpolate.cpp
    Uniforms.cpp
    Camera.cpp
    Shader.cpp
)

# Check for Oculus SDK availability on Windows
if(WIN32)
    find_path(OCULUS_SDK_INCLUDE_DIR OVR.h
        HINTS
        ${CMAKE_CURRENT_SOURCE_DIR}/OculusSDK4/LibOVR/Include
        ${CMAKE_CURRENT_SOURCE_DIR}/OculusSDK/LibOVR/Include
    )

    if(OCULUS_SDK_INCLUDE_DIR)
        add_compile_definitions(HAVE_OCULUS_SDK)
        list(APPEND BOXPLORER2_SOURCES OculusSdk4.cpp)
        include_directories(${OCULUS_SDK_INCLUDE_DIR})
        message(STATUS "Oculus SDK found at: ${OCULUS_SDK_INCLUDE_DIR}")
    else()
        message(STATUS "Oculus SDK not found - building without Oculus support")
    endif()
endif()

# Create AntTweakBar static library
add_library(AntTweakBar STATIC ${ANTTWEAKBAR_SOURCES})

# Main BoxPlorer2 executable
add_executable(boxplorer2 ${BOXPLORER2_SOURCES})
target_link_libraries(boxplorer2 AntTweakBar ${OPENGL_LIBRARIES})

# SDL2 test executable (only if SDL2 is available)
if(SDL2_FOUND)
    add_executable(sdl2-test Sdl2Test.cpp)
endif()

# Shader shrink utility
add_executable(shadershrink ShaderShrink.cpp)

# Edit config utility
add_executable(edit-cfg EditCfg.cpp)

# GLSL utility
add_executable(glsl Glsl.cpp)

# Platform-specific linking
if(WIN32)
    # Windows linking
    if(SDL2_FOUND)
        if(TARGET SDL2-static)
            target_link_libraries(boxplorer2 SDL2-static SDL2main)
            if(TARGET sdl2-test)
                target_link_libraries(sdl2-test SDL2-static SDL2main)
            endif()
        else()
            target_link_libraries(boxplorer2 ${SDL2_LIBRARY})
            if(TARGET sdl2-test)
                target_link_libraries(sdl2-test ${SDL2_LIBRARY})
            endif()
        endif()
    endif()

    # Add Windows-specific libraries
    target_link_libraries(boxplorer2 opengl32 glu32 user32 shell32 comdlg32)
    if(TARGET sdl2-test)
        target_link_libraries(sdl2-test opengl32 glu32)
    endif()

    # Set console subsystem
    set_target_properties(boxplorer2 PROPERTIES LINK_FLAGS "/SUBSYSTEM:CONSOLE")
    if(TARGET sdl2-test)
        set_target_properties(sdl2-test PROPERTIES LINK_FLAGS "/SUBSYSTEM:CONSOLE")
    endif()
    set_target_properties(shadershrink PROPERTIES LINK_FLAGS "/SUBSYSTEM:CONSOLE")
    set_target_properties(edit-cfg PROPERTIES LINK_FLAGS "/SUBSYSTEM:CONSOLE")
    set_target_properties(glsl PROPERTIES LINK_FLAGS "/SUBSYSTEM:CONSOLE")

elseif(APPLE)
    # macOS linking
    if(SDL2_FOUND)
        target_link_libraries(boxplorer2 ${SDL2_LIBRARY})
        if(TARGET sdl2-test)
            target_link_libraries(sdl2-test ${SDL2_LIBRARY})
        endif()
    endif()

    # Add macOS frameworks
    target_link_libraries(boxplorer2 "-framework OpenGL" "-framework Cocoa")
    if(TARGET sdl2-test)
        target_link_libraries(sdl2-test "-framework OpenGL" "-framework Cocoa")
    endif()

else()
    # Linux linking
    if(SDL2_FOUND)
        target_link_libraries(boxplorer2 ${SDL2_LIBRARIES} ${OPENGL_LIBRARIES} m X11)
        if(TARGET sdl2-test)
            target_link_libraries(sdl2-test ${SDL2_LIBRARIES} ${OPENGL_LIBRARIES} m X11)
        endif()

        # Set SDL2 compile flags
        target_compile_options(boxplorer2 PRIVATE ${SDL2_CFLAGS_OTHER})
        if(TARGET sdl2-test)
            target_compile_options(sdl2-test PRIVATE ${SDL2_CFLAGS_OTHER})
        endif()
    else()
        target_link_libraries(boxplorer2 ${OPENGL_LIBRARIES} m X11)
    endif()
endif()

# Copy shader and config files to build directory
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/cfgs DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/fragment.glsl DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/vertex.glsl DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/textures)
    file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/textures DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()

# Install targets
set(INSTALL_TARGETS boxplorer2 shadershrink edit-cfg glsl)
if(TARGET sdl2-test)
    list(APPEND INSTALL_TARGETS sdl2-test)
endif()
install(TARGETS ${INSTALL_TARGETS}
    RUNTIME DESTINATION bin
)

# Install data files
install(DIRECTORY cfgs DESTINATION share/boxplorer2)
install(FILES fragment.glsl vertex.glsl DESTINATION share/boxplorer2)
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/textures)
    install(DIRECTORY textures DESTINATION share/boxplorer2)
endif()

cmake_minimum_required(VERSION 3.8)
project(angle CXX C)
include(config.cmake)

if(XBOX360)
    set(XBOX360_DEFAULT_XEX_CONFIG "${CMAKE_CURRENT_SOURCE_DIR}/xexconf.xml" CACHE PATH "Default XEX config file")

    function(angle_add_library target_name)
        set(ARG_LIST ${ARGN})

        # Detect target type flags
        list(FIND ARG_LIST "ALIAS" IS_ALIAS)
        list(FIND ARG_LIST "INTERFACE" IS_INTERFACE)
        list(FIND ARG_LIST "SHARED" IS_SHARED)

        if(IS_ALIAS GREATER -1)
            # The signature requires: add_library(<name> ALIAS <target>)
            # Remove ALIAS from the list so only the target name remains
            list(REMOVE_ITEM ARG_LIST "ALIAS")
            add_library(${target_name} ALIAS ${ARG_LIST})

        elseif(IS_INTERFACE GREATER -1)
            # The signature requires: add_library(<name> INTERFACE)
            list(REMOVE_ITEM ARG_LIST "INTERFACE")
            add_library(${target_name} INTERFACE)

            # If there are trailing source arguments left over, append them properly
            if(ARG_LIST)
                target_sources(${target_name} INTERFACE ${ARG_LIST})
            endif()

        elseif(IS_SHARED GREATER -1)
            list(REMOVE_ITEM ARG_LIST "SHARED")
            add_xenon_module(${target_name} "${XBOX360_DEFAULT_XEX_CONFIG}" "${ARG_LIST}")

        else()
            list(REMOVE_ITEM ARG_LIST "STATIC")
            add_xenon_library(${target_name} "${ARG_LIST}")
        endif()
    endfunction()

	include_directories("${CMAKE_CURRENT_SOURCE_DIR}/xbox360-compat-include")

	add_compile_definitions(XBOX360)
else()
	function(angle_add_library)
		add_library(${ARGV})
	endfunction()
endif()

option(USE_FRAME_CAPTURE "build with frame capture support" OFF)
option(USE_LOADER "build with EGL/GLESv2 loader" ON)
option(USE_VULKAN "build with vulkan support" OFF)
option(USE_METAL "build with metal support" OFF)

if(WINDOWS_DESKTOP AND NOT CMAKE_SYSTEM_PROCESSOR MATCHES arm)
  option(USE_D3D9 "build with d3d9 support" ON)
  option(USE_GL "build with native opengl/es support via wgl" ON)
endif()

# Force D3D9 on, GL off for Xbox 360
if(XBOX360)
  option(USE_D3D9 "build with d3d9 support" ON)
  option(USE_GL "build with native opengl/es support via wgl" OFF)
endif()

if (APPLE)
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -framework Cocoa -framework OpenGL -framework IOKit -framework CoreFoundation -framework IOSurface -framework QuartzCore -framework CoreGraphics")
endif()
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}")
set(CMAKE_STATIC_LIBRARY_PREFIX "")

add_definitions(
    -D_CRT_SECURE_NO_DEPRECATE
    -D_SCL_SECURE_NO_WARNINGS
    -D_HAS_EXCEPTIONS=0
    -DNOMINMAX
    -DANGLE_STANDALONE_BUILD
    -DANGLE_ENABLE_DEBUG_ANNOTATIONS
)

file(WRITE "${CMAKE_CURRENT_SOURCE_DIR}/src/common/commit.h"
    "#pragma once\n"
    "#define ANGLE_COMMIT_HASH \"unknown\"\n"
    "#define ANGLE_COMMIT_HASH_SIZE 7\n"
    "#define ANGLE_COMMIT_DATE \"unknown\"\n"
    "#define ANGLE_COMMIT_POSITION 0\n"
)
file(WRITE "${CMAKE_CURRENT_SOURCE_DIR}/src/common/angle_commit.h"
    "#pragma once\n"
    "#define ANGLE_COMMIT_HASH \"unknown\"\n"
    "#define ANGLE_COMMIT_HASH_SIZE 7\n"
    "#define ANGLE_COMMIT_DATE \"unknown\"\n"
    "#define ANGLE_COMMIT_POSITION 0\n"
)
file(WRITE "${CMAKE_CURRENT_SOURCE_DIR}/src/common/ANGLEShaderProgramVersion.h"
    "#pragma once\n"
    "#define ANGLE_PROGRAM_VERSION_HASH \"unknown\"\n"
    "#define ANGLE_PROGRAM_VERSION_HASH_SIZE 7\n"
    "#define ANGLE_PROGRAM_VERSION \"unknown\"\n"
)

include_directories(include src ${CMAKE_CURRENT_BINARY_DIR}/include)


##########
# angle::common
if(WINDOWS_DESKTOP)
    set(ANGLE_COMMON_PLATFORM_FILTER "_winuwp|_linux|_mac|_apple|_ios|_posix")
elseif(WINDOWS_STORE)
    set(ANGLE_COMMON_PLATFORM_FILTER "_win|_linux|_mac|_apple|_ios|_posix")
elseif(LINUX)
    set(ANGLE_COMMON_PLATFORM_FILTER "_win|_mac|_apple|_ios")
elseif(APPLE)
    set(ANGLE_COMMON_PLATFORM_FILTER "_linux|_win")
elseif(XBOX360)
    set(ANGLE_COMMON_PLATFORM_FILTER "_linux|_mac|_apple|_ios|_winuwp|_posix")
else()
    set(ANGLE_COMMON_PLATFORM_FILTER "_linux|_mac|_apple|_ios|_winuwp")
endif()

file(GLOB ANGLE_COMMON_SOURCES "src/common/*.h" "src/common/*.inl" "src/common/*.cpp" "src/common/third_party/base/anglebase/*.h" "src/common/third_party/base/anglebase/*.cc" "src/common/third_party/base/anglebase/containers/*.h" "src/common/third_party/base/anglebase/numerics/*.h" "src/common/third_party/base/anglebase/numerics/*.cc" "src/common/third_party/xxhash/*.h" "src/common/third_party/xxhash/*.c" "src/common/third_party/smhasher/src/*.h" "src/common/third_party/smhasher/src/*.cpp")
list(FILTER ANGLE_COMMON_SOURCES EXCLUDE REGEX "_unittest|event_tracer|${ANGLE_COMMON_PLATFORM_FILTER}")
if(NOT USE_FRAME_CAPTURE)
    list(FILTER ANGLE_COMMON_SOURCES EXCLUDE REGEX "frame_capture")
endif()
angle_add_library(angle_common STATIC ${ANGLE_COMMON_SOURCES})
target_include_directories(angle_common PUBLIC src/common/third_party/base)
target_include_directories(angle_common PUBLIC src/common/base)
angle_add_library(angle::common ALIAS angle_common)
if(CMAKE_CL_64 OR CMAKE_SYSTEM_PROCESSOR MATCHES 64)
  target_compile_definitions(angle_common INTERFACE -DANGLE_IS_64_BIT_CPU)
else()
  target_compile_definitions(angle_common INTERFACE -DANGLE_IS_32_BIT_CPU)
  if(MSVC AND CMAKE_SYSTEM_PROCESSOR MATCHES 86)
    target_compile_options(angle_common INTERFACE -arch:SSE2) # clang-cl, loadimage.cpp
  endif()
endif()


##########
# angle::image_util
file(GLOB ANGLE_IMAGE_UTIL_SOURCES "src/image_util/*.h" "src/image_util/*.inl" "src/image_util/*.cpp")
list(FILTER ANGLE_IMAGE_UTIL_SOURCES EXCLUDE REGEX "AstcDecompressor")
angle_add_library(angle_image_util STATIC ${ANGLE_IMAGE_UTIL_SOURCES})
target_link_libraries(angle_image_util PRIVATE angle::common)
angle_add_library(angle::image_util ALIAS angle_image_util)

set(ANGLE_GPU_INFO_UTIL_SOURCES src/gpu_info_util/SystemInfo.cpp)
if(APPLE)
  list(APPEND ANGLE_GPU_INFO_UTIL_SOURCES src/gpu_info_util/SystemInfo_mac.mm)
elseif(WIN32)
  list(APPEND ANGLE_GPU_INFO_UTIL_SOURCES src/gpu_info_util/SystemInfo_win.cpp)
elseif(ANDROID)
  list(APPEND ANGLE_GPU_INFO_UTIL_SOURCES src/gpu_info_util/SystemInfo_android.cpp)
else()
  list(APPEND ANGLE_GPU_INFO_UTIL_SOURCES src/gpu_info_util/SystemInfo_linux.cpp)
endif()
angle_add_library(angle_gpu_info_util STATIC ${ANGLE_GPU_INFO_UTIL_SOURCES})
target_link_libraries(angle_gpu_info_util PRIVATE angle::common)
angle_add_library(angle::gpu_info_util ALIAS angle_gpu_info_util)

if(WIN32)
  if(WINDOWS_DESKTOP)
    set_property(SOURCE src/gpu_info_util/SystemInfo_win.cpp APPEND PROPERTY COMPILE_FLAGS "-DGPU_INFO_USE_SETUPAPI")
    target_link_libraries(angle_gpu_info_util PRIVATE setupapi)
  elseif(NOT XBOX360) # Xbox doesn't use standard DXGI either
    set_property(SOURCE src/gpu_info_util/SystemInfo_win.cpp APPEND PROPERTY COMPILE_FLAGS "-DGPU_INFO_USE_DXGI")
  endif()
endif()

##########
# angle::translator
file(GLOB_RECURSE TRANSLATOR_SOURCES
    "src/compiler/translator/glslang.l"
    "src/compiler/translator/glslang.y"
    "src/compiler/translator/*.h"
    "src/compiler/translator/*.cpp"
    "src/third_party/compiler/ArrayBoundsClamper.cpp"
)
# if(NOT WIN32)
#     list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "HLSL|hlsl")
# endif()
if(NOT USE_VULKAN)
    list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "SPIRV|spirv")
endif()
if(NOT APPLE)
    list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "APPLE|apple")
    list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "MSL|msl")
endif()
list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "WGSL|wgsl")
list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "src/compiler/translator/ir/src")
list(FILTER TRANSLATOR_SOURCES EXCLUDE REGEX "Vulkan")
angle_add_library(angle_translator STATIC ${TRANSLATOR_SOURCES})
target_compile_definitions(angle_translator PRIVATE -DANGLE_ENABLE_HLSL)
target_compile_definitions(angle_translator PRIVATE
    -DANGLE_ENABLE_ESSL
    -DANGLE_ENABLE_GLSL
)
if(WIN32) # XBOX360 uses WIN32 so HLSL is correctly enabled here
    target_compile_definitions(angle_translator PRIVATE -DANGLE_ENABLE_HLSL)
endif()

if(NOT XBOX360)
    find_package(GTest)
endif()

if(NOT GTest_FOUND)
  include(FetchContent)
  FetchContent_Declare(
    googletest
    URL https://github.com/google/googletest/archive/release-1.10.0.tar.gz
    URL_HASH SHA1=9c89be7df9c5e8cb0bc20b3c4b39bf7e82686770
  )
  FetchContent_MakeAvailable(googletest)
endif()

target_link_libraries(angle_translator PRIVATE angle::common gtest)
angle_add_library(angle::translator ALIAS angle_translator)

##########
# angle::preprocessor
file(GLOB PREPROCESSOR_SOURCES
    "src/compiler/preprocessor/*.h"
    "src/compiler/preprocessor/*.cpp"
)
angle_add_library(angle_preprocessor STATIC ${PREPROCESSOR_SOURCES})
target_link_libraries(angle_preprocessor PRIVATE angle::common)
angle_add_library(angle::preprocessor ALIAS angle_preprocessor)

##########
# libANGLE

## OpenGL Renderer
if(USE_GL AND (WINDOWS_DESKTOP OR LINUX OR APPLE))
    if(WINDOWS_DESKTOP)
        set(ANGLE_RENDERER_OPENGL_PLATFORM
            "src/libANGLE/renderer/gl/wgl/*.cpp"
            "src/libANGLE/renderer/gl/wgl/*.inl"
            "src/libANGLE/renderer/gl/wgl/*.h"
        )
    elseif(APPLE)
        set(ANGLE_RENDERER_OPENGL_PLATFORM
            "src/libANGLE/renderer/gl/cgl/*.mm"
            "src/libANGLE/renderer/gl/cgl/*.cpp"
            "src/libANGLE/renderer/gl/cgl/*.inl"
            "src/libANGLE/renderer/gl/cgl/*.h"
        )
    elseif(LINUX)
        set(ANGLE_RENDERER_OPENGL_PLATFORM
            "src/libANGLE/renderer/gl/glx/*.cpp"
            "src/libANGLE/renderer/gl/glx/*.inl"
            "src/libANGLE/renderer/gl/glx/*.h"
        )
    endif()

    file(GLOB LIBANGLE_GL_SOURCES
        "src/libANGLE/renderer/gl/*.cpp"
        "src/libANGLE/renderer/gl/*.inl"
        "src/libANGLE/renderer/gl/*.h"

        ${ANGLE_RENDERER_OPENGL_PLATFORM}
    )
    list(FILTER LIBANGLE_GL_SOURCES EXCLUDE REGEX "_unittest")
    angle_add_library(angle_renderer_opengl INTERFACE)
    target_sources(angle_renderer_opengl INTERFACE ${LIBANGLE_GL_SOURCES})
    target_compile_definitions(angle_renderer_opengl INTERFACE -DANGLE_ENABLE_OPENGL)
    angle_add_library(angle::renderer::opengl ALIAS angle_renderer_opengl)
endif()

# D3D Renderers
if(WIN32 OR XBOX360)
    ## All D3D Sources
    file(GLOB_RECURSE LIBANGLE_D3D_SOURCES
        "src/libANGLE/renderer/d3d/*.cpp"
        "src/libANGLE/renderer/d3d/*.inl"
        "src/libANGLE/renderer/d3d/*.h"
    )
    list(FILTER LIBANGLE_D3D_SOURCES EXCLUDE REGEX "_unittest")

    ## WinRT D3D Renderer
    if(WINDOWS_STORE)
        set(LIBANGLE_D3D_WINRT_SOURCES ${LIBANGLE_D3D_SOURCES})
        list(FILTER LIBANGLE_D3D_WINRT_SOURCES INCLUDE REGEX "winrt")
        angle_add_library(angle_renderer_winrt INTERFACE)
        target_sources(angle_renderer_winrt INTERFACE ${LIBANGLE_D3D_WINRT_SOURCES})
        angle_add_library(angle::renderer::winrt ALIAS angle_renderer_winrt)
    endif()

    ## Win32/d3d9 D3D Renderer (Used by Desktop AND Xbox 360)
    if(WINDOWS_DESKTOP OR XBOX360)
        set(LIBANGLE_D3D_DESKTOP_SOURCES ${LIBANGLE_D3D_SOURCES})
        if(USE_D3D9)
            list(FILTER LIBANGLE_D3D_DESKTOP_SOURCES INCLUDE REGEX "d3d9|win32|converged")
        else()
            list(FILTER LIBANGLE_D3D_DESKTOP_SOURCES INCLUDE REGEX "win32|converged")
        endif()
        angle_add_library(angle_renderer_win32 INTERFACE)
        target_sources(angle_renderer_win32 INTERFACE ${LIBANGLE_D3D_DESKTOP_SOURCES})

        if(USE_D3D9)
            target_compile_definitions(angle_renderer_win32 INTERFACE -DANGLE_ENABLE_D3D9)
            # Xbox 360 toolchain manages its own graphics libs automatically, don't force desktop d3d9 link
            if(NOT XBOX360)
                target_link_libraries(angle_renderer_win32 INTERFACE d3d9)
            endif()
        endif()

        # Don't link standard Windows UI libraries on a console
        if(NOT XBOX360)
            target_link_libraries(angle_renderer_win32 INTERFACE user32 gdi32)
        endif()

        angle_add_library(angle::renderer::win32 ALIAS angle_renderer_win32)

        # Avoid forcing D3D compiler DLL load hacks on the Xbox 360
        if(NOT XBOX360)
            if(CMAKE_HOST_WIN32)
                set_property(SOURCE src/libANGLE/renderer/d3d/HLSLCompiler.cpp APPEND PROPERTY COMPILE_FLAGS "-DANGLE_PRELOADED_D3DCOMPILER_MODULE_NAMES={\\\"d3dcompiler_47.dll\\\",\\\"d3dcompiler_46.dll\\\",\\\"d3dcompiler_43.dll\\\"}")
            else()
                set_property(SOURCE src/libANGLE/renderer/d3d/HLSLCompiler.cpp APPEND PROPERTY COMPILE_FLAGS "-DANGLE_PRELOADED_D3DCOMPILER_MODULE_NAMES='{\"d3dcompiler_47.dll\",\"d3dcompiler_46.dll\",\"d3dcompiler_43.dll\"}'")
            endif()
        endif()
    endif()

    ## D3D11 Base renderer - SKIP ENTIRELY FOR XBOX 360
    if(NOT XBOX360)
        list(FILTER LIBANGLE_D3D_SOURCES EXCLUDE REGEX "winrt|d3d9|win32")
        find_library(DXGUID_LIB NAMES dxguid)
        find_library(D3D11_LIB NAMES d3d11)
        angle_add_library(angle_renderer_d3d INTERFACE)
        target_sources(angle_renderer_d3d INTERFACE ${LIBANGLE_D3D_SOURCES})
        target_compile_definitions(angle_renderer_d3d INTERFACE
            -DANGLE_ENABLE_D3D11
        )
        target_link_libraries(angle_renderer_d3d INTERFACE d3d11 dxgi dxguid)
        angle_add_library(angle::renderer::d3d ALIAS angle_renderer_d3d)
    endif()
endif()

## Core libANGLE library
if(WIN32)
    set(LIBANGLE_SOURCES_PLATFORM
        "src/third_party/systeminfo/SystemInfo.cpp"
    )
elseif(APPLE)
    file(GLOB LIBANGLE_SOURCES_PLATFORM
         "src/libANGLE/renderer/*.mm"
    )
else()
    set(LIBANGLE_SOURCES_PLATFORM )
endif()

file(GLOB LIBANGLE_SOURCES
    "src/common/third_party/murmurhash/MurmurHash3.cpp"
    "src/common/event_tracer.cpp"

    "src/libANGLE/*.cpp"
    "src/libANGLE/*.inl"
    "src/libANGLE/*.h"

    "src/libANGLE/renderer/*.cpp"
    "src/libANGLE/renderer/*.inl"
    "src/libANGLE/renderer/*.h"

    ${LIBANGLE_SOURCES_PLATFORM}
)
list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "_unittest")
if(USE_FRAME_CAPTURE)
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "FrameCapture_mock.cpp")
else()
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "FrameCapture.cpp")
endif()
if(NOT USE_VULKAN AND NOT USE_METAL)
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "glslang_wrapper_utils.cpp")
endif()

# Purge OpenCL
list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/.*CL.*")
list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/.*cl_.*")
list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/renderer/.*CL.*")
list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/renderer/.*cl_.*")

# Drop Windows-specific render utilities on Linux/macOS, drop DXGI entirely for Xbox 360
if(NOT WIN32)
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/renderer/d3d")
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/renderer/dxgi")
elseif(XBOX360)
    list(FILTER LIBANGLE_SOURCES EXCLUDE REGEX "src/libANGLE/renderer/dxgi")
endif()

if(LINUX OR APPLE)
    # Fallback stub for building the translator interface on Linux without standard rendering dependencies
    if(NOT TARGET angle::renderer::opengl)
        add_library(angle_renderer_opengl_stub INTERFACE)
        add_library(angle::renderer::opengl ALIAS angle_renderer_opengl_stub)
    endif()
    set(LIBANGLE_RENDERER_PLATFORM
        angle::renderer::opengl
    )
elseif(WINDOWS_STORE)
    set(LIBANGLE_RENDERER_PLATFORM
        angle::renderer::d3d
        angle::renderer::winrt
    )
elseif(WINDOWS_DESKTOP)
    set(LIBANGLE_RENDERER_PLATFORM
        angle::renderer::d3d
        angle::renderer::win32
    )
    if(USE_GL)
        list(APPEND LIBANGLE_RENDERER_PLATFORM angle::renderer::opengl)
    endif()
elseif(XBOX360)
    set(LIBANGLE_RENDERER_PLATFORM
        # Deliberately omit angle::renderer::d3d (D3D11)
        angle::renderer::win32 # Contains D3D9
    )
else()
    set(LIBANGLE_RENDERER_PLATFORM )
endif()

if(LINUX)
    set(LIBANGLE_RENDERER_COMPILEDEF
        -DANGLE_USE_X11
    )
else()
    set(LIBANGLE_RENDERER_COMPILEDEF )
endif()

angle_add_library(libANGLE STATIC ${LIBANGLE_SOURCES})
target_link_libraries(libANGLE PRIVATE
    angle::common
    angle::gpu_info_util
    angle::image_util
    angle::translator
    angle::preprocessor
    ${LIBANGLE_RENDERER_PLATFORM}
)
target_include_directories(libANGLE PRIVATE "src/third_party/khronos")
target_compile_definitions(libANGLE
    PUBLIC
        -DLIBANGLE_IMPLEMENTATION
        ${LIBANGLE_RENDERER_COMPILEDEF}
        -DANGLE_CAPTURE_ENABLED=$<BOOL:${USE_FRAME_CAPTURE}>
)
angle_add_library(angle::libANGLE ALIAS libANGLE)

##########
# libGLESv2
file(GLOB LIBGLESV2_SOURCES "src/libGLESv2/*.h" "src/libGLESv2/*.cpp" "src/libGLESv2/libGLESv2_autogen.def")
list(FILTER LIBGLESV2_SOURCES EXCLUDE REGEX "proc_table_autogen|proc_table_gles_autogen")
list(FILTER LIBGLESV2_SOURCES EXCLUDE REGEX "src/libGLESv2/.*CL.*")
list(FILTER LIBGLESV2_SOURCES EXCLUDE REGEX "src/libGLESv2/.*cl_.*")
angle_add_library(libGLESv2 SHARED ${LIBGLESV2_SOURCES})
target_link_libraries(libGLESv2 PRIVATE angle::common angle::libANGLE)
target_compile_definitions(libGLESv2
    PRIVATE -DLIBGLESV2_IMPLEMENTATION
    PUBLIC
        -DGL_GLEXT_PROTOTYPES
        -DGL_GLES_PROTOTYPES=1
)
target_include_directories(libGLESv2 PUBLIC "$<INSTALL_INTERFACE:include>")

##########
# libEGL
file(GLOB LIBEGL_SOURCES "src/libEGL/*.cpp" "src/libEGL/*.h" "src/libEGL/libEGL.def")
angle_add_library(libEGL SHARED
    ${LIBEGL_SOURCES}
)
target_link_libraries(libEGL PRIVATE angle::common angle::libANGLE libGLESv2)
target_include_directories(libEGL PUBLIC "$<INSTALL_INTERFACE:include>")
target_include_directories(libEGL PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")

if(NOT MSVC)
    target_compile_definitions(libGLESv2 PRIVATE -DGL_APICALL=__attribute__\(\(visibility\(\"default\"\)\)\))
    target_compile_definitions(libEGL PRIVATE -DEGLAPI=__attribute__\(\(visibility\(\"default\"\)\)\))
else()
    target_compile_definitions(libGLESv2 PRIVATE -DGL_API= -DGL_APICALL=)
    target_compile_definitions(libEGL PRIVATE -DEGLAPI=)
endif()

target_compile_definitions(libEGL PRIVATE
    -DLIBEGL_IMPLEMENTATION
    -DEGL_EGL_PROTOTYPES=1
    -DEGL_EGLEXT_PROTOTYPES
    -DANGLE_USE_EGL_LOADER
    -DANGLE_EGL_LIBRARY_NAME=\"libEGL\"
    -DANGLE_DISPATCH_LIBRARY=\"libGLESv2\"
)

SET_TARGET_PROPERTIES(libANGLE PROPERTIES PREFIX "")
SET_TARGET_PROPERTIES(libGLESv2 PROPERTIES PREFIX "")
SET_TARGET_PROPERTIES(libEGL PROPERTIES PREFIX "")

install(TARGETS libEGL libGLESv2 EXPORT ANGLEExport
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

if(NOT DISABLE_INSTALL_HEADERS)
    install(
        DIRECTORY include/
        DESTINATION include
        FILES_MATCHING PATTERN "*.h"
        PATTERN "GLSLANG" EXCLUDE
        PATTERN "EGL" EXCLUDE
        PATTERN "KHR" EXCLUDE
        PATTERN "export.h" EXCLUDE
    )
endif()
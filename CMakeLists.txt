# static link msvcrt
cmake_policy(SET CMP0091 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0091 NEW)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

cmake_minimum_required(VERSION 3.15)
project(vapoursynth-srmd-ncnn-vulkan VERSION 1.1.0 LANGUAGES C CXX)
set(CMAKE_BUILD_TYPE Release)

# check glslangValidator
macro(srmd_add_shader SHADER_SRC)
    get_filename_component(SHADER_SRC_NAME_WE ${SHADER_SRC} NAME_WE)
    set(SHADER_COMP_HEADER ${CMAKE_CURRENT_BINARY_DIR}/${SHADER_SRC_NAME_WE}.comp.hex.h)

    add_custom_command(
            OUTPUT ${SHADER_COMP_HEADER}
            COMMAND ${CMAKE_COMMAND} -DSHADER_SRC=${CMAKE_CURRENT_SOURCE_DIR}/${SHADER_SRC} -DSHADER_COMP_HEADER=${SHADER_COMP_HEADER} -P "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_shader_comp_header.cmake"
            DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${SHADER_SRC}
            COMMENT "Preprocessing shader source ${SHADER_SRC_NAME_WE}.comp"
            VERBATIM
    )
    set_source_files_properties(${SHADER_COMP_HEADER} PROPERTIES GENERATED TRUE)

    list(APPEND SHADER_SPV_HEX_FILES ${SHADER_COMP_HEADER})
endmacro()

# enable global link time optimization
cmake_policy(SET CMP0069 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0069 NEW)
include(CheckIPOSupported)
check_ipo_supported(RESULT ipo_supported OUTPUT ipo_supported_output)
if(ipo_supported)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
    message(WARNING "IPO is not supported: ${ipo_supported_output}")
endif()

# build ncnn library
if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/deps/ncnn/CMakeLists.txt")
    message(FATAL_ERROR "The submodules were not downloaded! Please update submodules with \"git submodule update --init --recursive\" and try again.")
endif()
option(NCNN_INSTALL_SDK "" OFF)
option(NCNN_PIXEL_ROTATE "" OFF)
option(NCNN_VULKAN "" ON)
option(NCNN_VULKAN_ONLINE_SPIRV "" ON)
# ncnn loads the Vulkan driver itself (simplevk) and builds glslang from its submodule, no Vulkan SDK needed.
option(NCNN_SIMPLEVK "" ON)
option(NCNN_SYSTEM_GLSLANG "" OFF)
option(NCNN_SHARED_LIB "" OFF)
option(NCNN_BUILD_SHARED_LIBS "" OFF)
option(NCNN_BUILD_BENCHMARK "" OFF)
option(NCNN_BUILD_TESTS "" OFF)
option(NCNN_BUILD_TOOLS "" OFF)
option(NCNN_BUILD_EXAMPLES "" OFF)
option(NCNN_DISABLE_RTTI "" ON)
option(NCNN_DISABLE_EXCEPTION "" OFF)
add_subdirectory(deps/ncnn)

# generate-spirv target
set(SHADER_SPV_HEX_FILES)
srmd_add_shader(src/srmd_preproc.comp)
srmd_add_shader(src/srmd_postproc.comp)
srmd_add_shader(src/srmd_preproc_tta.comp)
srmd_add_shader(src/srmd_postproc_tta.comp)
add_custom_target(generate-spirv DEPENDS ${SHADER_SPV_HEX_FILES})

include_directories(${CMAKE_CURRENT_BINARY_DIR})

# VapourSynth API 4 headers, vendored
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include/vapoursynth)

add_library(srmdnv MODULE src/main.cpp src/srmd.cpp)
target_link_libraries(srmdnv ncnn)
add_dependencies(srmdnv generate-spirv)
if(APPLE)
    # VapourSynth only autoloads *.dylib on macOS.
    set_target_properties(srmdnv PROPERTIES SUFFIX ".dylib")
endif()

# Wheel layout: the plugin and its models in site-packages/vapoursynth/plugins/srmd; the plugin finds the models next to itself.
set(PLUGIN_DIR "vapoursynth/plugins/srmd" CACHE STRING "Install directory of the plugin and its models")
install(TARGETS srmdnv LIBRARY DESTINATION ${PLUGIN_DIR} RUNTIME DESTINATION ${PLUGIN_DIR})
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/models")
    install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/models DESTINATION ${PLUGIN_DIR})
endif()

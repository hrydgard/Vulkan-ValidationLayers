# ~~~
# Copyright (c) 2014-2022 Valve Corporation
# Copyright (c) 2014-2022 LunarG, Inc.
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
include(GoogleTest) # gtest_discover_tests

# GoogleTest is required for testing framework
find_package(GTest REQUIRED CONFIG)

# glslang is required for testing framework
find_package(glslang CONFIG REQUIRED)

# Needed to make structure definitions match with glslang libraries
add_definitions(-DNV_EXTENSIONS -DAMD_EXTENSIONS)

if(ANNOTATED_SPEC_LINK)
    add_definitions(-DANNOTATED_SPEC_LINK=${ANNOTATED_SPEC_LINK})
endif()

if(WIN32)
    add_definitions(-DVK_USE_PLATFORM_WIN32_KHR -DWIN32_LEAN_AND_MEAN -DNOMINMAX)
    # Workaround for TR1 deprecation in Visual Studio 15.5 until Google Test is updated
    add_definitions(-D_SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
    # If MSVC, allow Windows to use multiprocessor compilation
    if(MSVC)
        add_compile_options(/MP)
    endif()
elseif(ANDROID)
    add_definitions(-DVK_USE_PLATFORM_ANDROID_KHR)
elseif(APPLE)
    add_definitions(-DVK_USE_PLATFORM_METAL_EXT)
elseif(LINUX)
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

if(WIN32)
    file(COPY vk_layer_validation_tests.vcxproj.user DESTINATION ${CMAKE_BINARY_DIR}/tests)
endif()

if(WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_USE_MATH_DEFINES")

    # If MSVC, disable some signed/unsigned mismatch warnings.
    if(MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267")
    endif()

else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

set(LIBGLM_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/libs)

set(COMMON_CPP
    vklayertests_instanceless.cpp
    vklayertests_pipeline_shader.cpp
    vklayertests_buffer_image_memory_sampler.cpp
    vklayertests_others.cpp
    vklayertests_best_practices.cpp
    vklayertests_arm_best_practices.cpp
    vklayertests_amd_best_practices.cpp
    vklayertests_gpu.cpp
    vklayertests_descriptor_renderpass_framebuffer.cpp
    vklayertests_dynamic_rendering.cpp
    vklayertests_command.cpp
    vklayertests_imageless_framebuffer.cpp
    vklayertests_nvidia_best_practices.cpp
    vklayertests_portability_subset.cpp
    vklayertests_viewport_inheritance.cpp
    vklayertests_wsi.cpp
    vklayertests_graphics_library.cpp
    positive/command.cpp
    positive/descriptors.cpp
    positive/dynamic_rendering.cpp
    positive/image_buffer.cpp
    positive/instance.cpp
    positive/other.cpp
    positive/pipeline.cpp
    positive/render_pass.cpp
    positive/shaderval.cpp
    positive/sync.cpp
    positive/tooling.cpp
    positive/graphics_library.cpp
    vksyncvaltests.cpp
    vkrenderframework.cpp
    vktestbinding.cpp
    vktestframework.cpp)

set_source_files_properties(${PROJECT_BINARY_DIR}/vk_safe_struct.cpp PROPERTIES GENERATED TRUE)
add_executable(vk_layer_validation_tests
               layer_validation_tests.cpp
               ../layers/generated/vk_format_utils.cpp
               ../layers/convert_to_renderpass2.cpp
               ../layers/generated/vk_safe_struct.cpp
               ../layers/generated/lvt_function_pointers.cpp
               ${COMMON_CPP})

# gtest_discover_tests has problem with cross-compiling, but it is faster and more robust
if (CMAKE_CROSSCOMPILING)
    gtest_add_tests(TARGET vk_layer_validation_tests)
else()
    gtest_discover_tests(vk_layer_validation_tests)
endif()

add_dependencies(vk_layer_validation_tests VkLayer_khronos_validation VkLayer_khronos_validation-json)
target_include_directories(vk_layer_validation_tests
                           PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}
                                  ${PROJECT_SOURCE_DIR}/layers
                                  ${PROJECT_SOURCE_DIR}/layers/generated
                                  ${CMAKE_CURRENT_BINARY_DIR}
                                  ${CMAKE_BINARY_DIR}
                                  ${PROJECT_BINARY_DIR}
                                  ${VulkanHeaders_INCLUDE_DIR}
                                  ${PROJECT_BINARY_DIR}/layers)

option(PORTABILITY_TESTS_USE_DEVSIM "Automatically add the devsim layer for portability tests" OFF)
if (PORTABILITY_TESTS_USE_DEVSIM)
    target_compile_definitions(vk_layer_validation_tests PRIVATE PORTABILITY_TESTS_USE_DEVSIM=1)
endif()

if (VVL_ENABLE_ASAN)
    target_compile_options(vk_layer_validation_tests  PRIVATE -fsanitize=address)
    # NOTE: Use target_link_options when cmake 3.13 is available on CI
    target_link_libraries(vk_layer_validation_tests  PRIVATE "-fsanitize=address")
endif()

if (NOT MSVC)
    target_compile_options(vk_layer_validation_tests PRIVATE "-Wno-sign-compare")
endif()

target_include_directories(vk_layer_validation_tests PRIVATE ${SPIRV_HEADERS_INCLUDE_DIR})

target_link_libraries(vk_layer_validation_tests PRIVATE
    VkLayer_utils
    glslang::glslang
    glslang::OGLCompiler
    glslang::OSDependent
    glslang::MachineIndependent
    glslang::GenericCodeGen
    glslang::HLSL
    glslang::SPIRV
    glslang::SPVRemapper
    ${SPIRV_TOOLS_TARGET}
    SPIRV-Tools-opt
    GTest::gtest
    GTest::gtest_main
)

if (NOT WIN32)
    target_link_libraries(vk_layer_validation_tests PRIVATE ${CMAKE_DL_LIBS})

    if(BUILD_WSI_XCB_SUPPORT)
        target_link_libraries(vk_layer_validation_tests PRIVATE PkgConfig::XCB)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
        target_link_libraries(vk_layer_validation_tests PRIVATE PkgConfig::X11)
    endif()
endif()

if(INSTALL_TESTS)
    install(TARGETS vk_layer_validation_tests DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

add_subdirectory(layers)

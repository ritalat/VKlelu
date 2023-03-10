cmake_minimum_required(VERSION 3.16)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "..." FORCE)
endif()

project(VKlelu)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install CACHE PATH "..." FORCE)
endif()

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

set(CMAKE_INSTALL_BINDIR ".")
set(CMAKE_INSTALL_LIBDIR ".")

add_subdirectory(external)

find_package(Vulkan REQUIRED COMPONENTS glslc)

set(ASSETS kultainenapina.jpg
           kultainenapina.mtl
           kultainenapina.obj)

set(SHADERS shader.frag
            shader.vert)

foreach(SHADER ${SHADERS})
    set(GLSLC_OUT "${SHADER}.spv")
    add_custom_command(
        OUTPUT ${GLSLC_OUT}
        COMMAND ${Vulkan_GLSLC_EXECUTABLE} ${CMAKE_SOURCE_DIR}/${SHADER} -o ${GLSLC_OUT}
        DEPENDS ${SHADER})
    list(APPEND SPIRV_BINARIES ${GLSLC_OUT})
endforeach()

add_custom_target(Shaders DEPENDS ${SPIRV_BINARIES})

set(SOURCES main.cc
            utils.cc
            vklelu.cc)

set(HEADERS utils.hh
            vklelu.hh)

add_executable(VKlelu ${SOURCES} ${HEADERS})

if(WIN32)
    # This only works with generated VS solutions
    # When using VS builtin cmake support you must edit launch.vs.json instead
    set(VSENV "VKLELU_ASSETDIR=${CMAKE_SOURCE_DIR}\nVKLELU_SHADERDIR=${CMAKE_BINARY_DIR}")
    # Only set the correct subsystem for release builds
    # This way we can easily see console output in debug builds
    set_target_properties(VKlelu PROPERTIES WIN32_EXECUTABLE $<CONFIG:Release>
                                            VS_DEBUGGER_ENVIRONMENT "${VSENV}")
endif()

if(UNIX)
    target_link_options(VKlelu PUBLIC "-Wl,--enable-new-dtags")
    set_target_properties(VKlelu PROPERTIES INSTALL_RPATH "\${ORIGIN}")
endif()

add_dependencies(VKlelu Shaders)
target_link_libraries(VKlelu Vulkan::Vulkan
                             SDL2::SDL2
                             SDL2::SDL2main
                             glm::glm
                             stb
                             tinyobjloader
                             vk-bootstrap::vk-bootstrap
                             VulkanMemoryAllocator)

install(TARGETS VKlelu SDL2 RUNTIME DESTINATION ".")

foreach(ASSET ${ASSETS})
    install(FILES ${CMAKE_SOURCE_DIR}/${ASSET} DESTINATION ".")
endforeach()

foreach(SPIRV_BINARY ${SPIRV_BINARIES})
    install(FILES ${CMAKE_BINARY_DIR}/${SPIRV_BINARY} DESTINATION ".")
endforeach()

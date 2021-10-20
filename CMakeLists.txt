#
# Copyright (C) 2005-2019 Intel Corporation
#
# SPDX-License-Identifier: GPL-2.0-only OR BSD-3-Clause
#

cmake_minimum_required(VERSION 2.8.12)

project(ittapi)

OPTION(FORCE_32 "Force a 32bit compile on 64bit" OFF)
OPTION(ITT_API_IPT_SUPPORT "ptmarks support")

if(CMAKE_SIZEOF_VOID_P MATCHES "8" AND NOT(FORCE_32))
    set(ARCH_64 1)
endif()

set(ITT_PUBLIC_HDRS
    include/ittnotify.h
    include/jitprofiling.h
    include/libittnotify.h
)

file(GLOB ITT_SRCS "src/ittnotify/*.c" "src/ittnotify/*.h")



add_library(ittnotify STATIC ${ITT_SRCS} ${ITT_PUBLIC_HDRS} ${ITT_PT})


IF(FORCE_32 AND UNIX)
    target_compile_options(ittnotify PRIVATE "-m32")
ENDIF()

if (ITT_API_IPT_SUPPORT)
    target_compile_definitions(ittnotify PRIVATE ITT_API_IPT_SUPPORT)
    if (NOT WIN32)
        enable_language(ASM)
        if (ARCH_64)
            target_sources(ittnotify PRIVATE src/ittnotify/ittptmark64.S)
        else()
            target_sources(ittnotify PRIVATE src/ittnotify/ittptmark32.S)
            # set_property(SOURCE src/ittnotify/ittptmark32.S APPEND PROPERTY COMPILE_OPTIONS "-m32")
        endif()
        # set(CMAKE_ASM_FLAGS "${CFLAGS} ${ASM_OPTIONS}" )
    else()
        enable_language(ASM_MASM)
        if (ARCH_64)
            target_sources(ittnotify PRIVATE src/ittnotify/ittptmark64.asm)
        else()
            target_sources(ittnotify PRIVATE src/ittnotify/ittptmark32.asm)
        endif()
    endif()
endif()

target_compile_definitions(ittnotify PUBLIC $<IF:$<CONFIG:Debug>,_DEBUG,NDEBUG>)

set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
foreach( OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
    string( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
    set( CMAKE_LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG}     ${LIBRARY_OUTPUT_PATH} )
endforeach( OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES )

if(WIN32)
    SET_TARGET_PROPERTIES(ittnotify PROPERTIES OUTPUT_NAME libittnotify)
else()
    SET_TARGET_PROPERTIES(ittnotify PROPERTIES OUTPUT_NAME ittnotify)
endif()

if (NOT WIN32)
    set_target_properties(ittnotify PROPERTIES POSITION_INDEPENDENT_CODE On)
    target_link_libraries(ittnotify PUBLIC dl pthread)
endif()

SET_TARGET_PROPERTIES(ittnotify PROPERTIES LINKER_LANGUAGE C)

target_include_directories(ittnotify
    PUBLIC 
      $<INSTALL_INTERFACE:include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
    PRIVATE src/ittnotify
)

# Installation:
set(config_install_dir "lib/cmake/ittapi")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(project_config "${generated_dir}/ittapiConfig.cmake")

install(TARGETS ittnotify EXPORT ittapiTargets)
install(FILES ${ITT_PUBLIC_HDRS} TYPE INCLUDE)
install(EXPORT ittapiTargets DESTINATION "lib/cmake/ittapi" NAMESPACE ittapi::)

include(CMakePackageConfigHelpers)
configure_package_config_file(
  "Config.cmake.in"
  "${project_config}"
  INSTALL_DESTINATION "${config_install_dir}"
)
install(
    FILES "${project_config}"
    DESTINATION "${config_install_dir}"
)



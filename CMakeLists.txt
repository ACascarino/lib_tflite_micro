cmake_minimum_required(VERSION 3.23)

# **********************
# Disable in-source build.
# **********************
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
    message(FATAL_ERROR "In-source build is not allowed! Please specify a build folder.\n\tex:cmake -B build")
endif()

set(LIB_NAME "xtflitemicro" CACHE STRING "Name of the library to be built")

# **********************
# Targets
# **********************
set(TOP_DIR
    "${CMAKE_CURRENT_SOURCE_DIR}")
include(${TOP_DIR}/cmakefiles/xtflm.cmake)

add_library(${LIB_NAME})

set(header_list "")
foreach(DIR ${ALL_INCLUDES})
    file(GLOB_RECURSE headers "${DIR}/*.h")
    list(APPEND header_list ${headers})
endforeach()

target_sources(${LIB_NAME}
    PRIVATE ${TFLM_KERNEL_SOURCES}
    PRIVATE ${TFLITE_SOURCES}
    PRIVATE ${NN_SOURCES}
    PRIVATE ${XTFLIB_KERNEL_SOURCES}
    PUBLIC FILE_SET HEADERS
    BASE_DIRS "${TOP_DIR}/.."
    FILES ${header_list}
)

set(DEFINITIONS
    "__xtflm_conf_h_exists__"
    "NO_INTERPRETER"
    "TF_LITE_STATIC_MEMORY"
    "TF_LITE_DISABLE_X86_NEON"
    "TF_LITE_STRIP_ERROR_STRINGS"
)

if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL XCORE_XS3A)
    # # Set the target
    set(XCORE_TARGET "XCORE-AI-EXPLORER")

    enable_language(CXX C ASM)

    target_sources(${LIB_NAME}
        PRIVATE ${XTFLIB_XCORE_SOURCES}
    )

    set(BUILD_FLAGS
        "-g"
        "-fcomment-asm"
        "-mcmodel=large"
        "-fxscope"
        "-Wno-xcore-fptrgroup"
        "-Wno-unknown-pragmas"
        "-Wno-bidirectional-buffered-port"
    )

    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        list(APPEND BUILD_FLAGS "-00")
    else()
        list(APPEND BUILD_FLAGS "-O3")
    endif()
else()
    set(BUILD_FLAGS
        "-g"
        "-O0"
    )
    SET(CMAKE_OSX_DEPLOYMENT_TARGET 12.0)

    list(APPEND DEFINITIONS "NN_USE_REF" "FLATBUFFERS_LOCALE_INDEPENDENT=0")

    target_compile_features(${LIB_NAME} PUBLIC cxx_std_17)
endif()

target_compile_options(${LIB_NAME} PRIVATE ${BUILD_FLAGS})
target_link_options(${LIB_NAME} PRIVATE ${BUILD_FLAGS})
target_compile_definitions(${LIB_NAME} PUBLIC
    ${DEFINITIONS}
)

target_include_directories(${LIB_NAME}
    PRIVATE ${ALL_INCLUDES}
)
install(TARGETS ${LIB_NAME} DESTINATION "${CMAKE_BINARY_DIR}/lib"
    FILE_SET HEADERS DESTINATION "${CMAKE_BINARY_DIR}/include")

# Move headers to clean up include directory
add_custom_target(project-install DEPENDS ${LIB_NAME}
    COMMAND rm -rf "${CMAKE_BINARY_DIR}/include" "${CMAKE_BINARY_DIR}/lib"
    COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target install
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/lib_tflite_micro/submodules/tflite-micro/tensorflow" "${CMAKE_BINARY_DIR}/include"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/lib_tflite_micro/submodules/tflite-micro/signal" "${CMAKE_BINARY_DIR}/include"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/lib_tflite_micro/submodules/flatbuffers/include/flatbuffers" "${CMAKE_BINARY_DIR}/include"
    COMMAND rm -rf "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/lib_tflite_micro/submodules"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/lib_tflite_micro" "${CMAKE_BINARY_DIR}/include/temp_ltm"
    COMMAND rm -rf "${CMAKE_BINARY_DIR}/include/lib_tflite_micro"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/temp_ltm" "${CMAKE_BINARY_DIR}/include/lib_tflite_micro"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_nn/lib_nn" "${CMAKE_BINARY_DIR}/include/temp_lnn"
    COMMAND rm -rf "${CMAKE_BINARY_DIR}/include/lib_nn"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/temp_lnn" "${CMAKE_BINARY_DIR}/include/lib_nn"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/api/flash_server.h" "${CMAKE_BINARY_DIR}/include"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/api/ioserver.h" "${CMAKE_BINARY_DIR}/include"
    COMMAND mv "${CMAKE_BINARY_DIR}/include/lib_tflite_micro/api/tile_ram_server.h" "${CMAKE_BINARY_DIR}/include"
    )

# Create zip of lib and headers
add_custom_target(create_zip DEPENDS project-install COMMAND
    ${CMAKE_COMMAND} -E tar "cfv" "release_archive.zip" --format=zip
    "${CMAKE_BINARY_DIR}/lib" "${CMAKE_BINARY_DIR}/include")

if(EXISTS "${CMAKE_BINARY_DIR}/compile_commands.json")
  file(COPY "${CMAKE_BINARY_DIR}/compile_commands.json"
       DESTINATION "${CMAKE_SOURCE_DIR}")
endif()

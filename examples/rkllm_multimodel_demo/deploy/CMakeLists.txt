cmake_minimum_required(VERSION 3.15.1)

project(rkllm-mulmod)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,--allow-shlib-undefined")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wl,--allow-shlib-undefined")

if(${CMAKE_VERSION} VERSION_GREATER "3.15.0" AND CMAKE_SYSTEM_NAME STREQUAL "Linux")
  add_link_options("-Wl,-Bsymbolic")
endif()

if (CMAKE_SYSTEM_NAME STREQUAL "Android")
    set (TARGET_LIB_ARCH ${CMAKE_ANDROID_ARCH_ABI})
else()
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set (TARGET_LIB_ARCH aarch64)
    else()
        set (TARGET_LIB_ARCH armhf)
    endif()
    if (CMAKE_C_COMPILER MATCHES "uclibc")
        set (TARGET_LIB_ARCH ${TARGET_LIB_ARCH}_uclibc)
    endif()
endif()

# opencv
if (CMAKE_SYSTEM_NAME MATCHES "Linux")
    find_library(LIBOPENCV_CORE NAMES opencv_core PATHS /usr/lib /usr/local/lib)
    if(NOT LIBOPENCV_CORE)
        set(OpenCV_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/opencv-linux-aarch64/share/OpenCV)
    endif()
elseif(CMAKE_SYSTEM_NAME MATCHES "Android")
    set(OpenCV_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/opencv-mobile-3.4.20-android/sdk/native/jni/abi-${CMAKE_ANDROID_ARCH_ABI})
endif()
find_package(OpenCV REQUIRED)

# rknn runtime
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if(NOT LIBRKNN_RT_PATH)
        find_path(RKNN_INCLUDE_DIR NAMES rknn_api.h rknn_custom_op.h rkinn_matmul_api.h PATHS /usr/include /usr/local/include)
	find_library(LIBRKNN_RT NAMES rknnrt PATHS /usr/lib /usr/local/lib)
    endif()
endif()
if(NOT RKNN_INCLUDE_DIR OR NOT LIBRKNN_RT)
    set(LIBRKNN_RT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/librknnrt)
    set(LIBRKNN_RT ${LIBRKNN_RT_PATH}/${CMAKE_SYSTEM_NAME}/librknn_api/${TARGET_LIB_ARCH}/librknnrt.so)
    set(RKNN_INCLUDE_DIR ${LIBRKNN_RT_PATH}/${CMAKE_SYSTEM_NAME}/librknn_api/include)
endif()

# imgenc
include_directories(src/image_enc.h ${RKNNRT_INCLUDE_DIR})
add_executable(rkllm-imgenc src/image_enc.cc src/img_encoder.cpp)
target_link_libraries(rkllm-imgenc PRIVATE ${OpenCV_LIBS} ${LIBRKNN_RT})

# rkllm runtime
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
if(NOT RKLLM_RT_PATH)
    find_path(RKLLM_INCLUDE_DIR NAMES rkllm.h PATHS /usr/include /usr/local/include)
    find_library(LIBRKLLM_RT NAMES rkllmrt PATHS /usr/lib /usr/local/lib)
endif()
endif()
if(NOT LIBRKLLM_RT OR NOT RKLLM_INCLUDE_DIR)
    set(RKLLM_RT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../../rkllm-runtime)
    set(LIBRKLLM_RT ${RKLLM_RT_PATH}/${CMAKE_SYSTEM_NAME}/librkllm_api/${TARGET_LIB_ARCH}/librkllmrt.so)
    set(RKLLM_INCLUDE_DIR ${RKLLM_RT_PATH}/${CMAKE_SYSTEM_NAME}/librkllm_api/include)
endif()

# llm
include_directories(${RKLLM_INCLUDE_DIR})
add_executable(rkllm-text src/llm.cpp)
target_link_libraries(rkllm-text PRIVATE ${LIBRKLLM_RT})

# llm_test
include_directories(${RKLLM_INCLUDE_DIR})
add_executable(rkllm-text_test src/llm_test.cpp)
target_link_libraries(rkllm-text_test PRIVATE ${LIBRKLLM_RT})

# demo
add_executable(${PROJECT_NAME} src/image_enc.cc src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE ${OpenCV_LIBS} ${LIBRKNN_RT} ${LIBRKLLM_RT})

# Hide symbols to avoid possible symbol conflicts with RKLLM
set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS  -Wl,--no-undefined,--version-script,${CMAKE_SOURCE_DIR}/c_export.map)

# Install the executable file to the specified directory
if(NOT CMAKE_INSTALL_PREFIX)
    set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install/demo_${CMAKE_SYSTEM_NAME}_${TARGET_LIB_ARCH})
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    install(TARGETS rkllm-imgenc rkllm-text_test ${PROJECT_NAME} DESTINATION bin)
    if(NOT RKLLM_RT_PATH)
        install(PROGRAMS ${LIBRKNN_RT} ${LIBRKLLM_RT} DESTINATION lib)
    endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    install(TARGETS rkllm-imgenc rkllm-text_test ${PROJECT_NAME} DESTINATION bin)
    install(PROGRAMS ${LIBRKNN_RT} ${LIBRKLLM_RT} DESTINATION lib)
    install(PROGRAMS ../data/demo.jpg DESTINATION ./)
endif()

cmake_minimum_required(VERSION 3.3)

# Name of the project (will be the name of the plugin)
project (pc-ble-driver-js)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# All projects depending on pc-ble-driver need to include this first
include(pc-ble-driver/cmake/nrf-ble-driver.cmake)
add_subdirectory(pc-ble-driver)

set(UECC_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src/uECC)
# Essential include files to build a node addon,
# you should add this line in every CMake.js based project.

include_directories(${CMAKE_JS_INC} ${NRF_BLE_DRIVER_INCLUDE_DIR} ${UECC_INCLUDE_DIR})
include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/common)
include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/common/config)
include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/common/sdk_compat)
include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/common/internal)
include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/common/internal/transport)

# Specify source files (include win_delay_load_hook.cpp only for Windows)
if(WIN32)
    file (GLOB SOURCE_FILES
        "src/adapter.cpp"
        "src/serialadapter.cpp"
        "src/common.cpp"
        "src/driver.cpp"
        "src/driver_gap.cpp"
        "src/driver_gatt.cpp"
        "src/driver_gattc.cpp"
        "src/driver_gatts.cpp"
        "src/driver_uecc.cpp"
        "src/win_delay_load_hook.cpp"
        "src/*.h"
    )
else()
    file (GLOB SOURCE_FILES
        "src/adapter.cpp"
        "src/serialadapter.cpp"
        "src/common.cpp"
        "src/driver.cpp"
        "src/driver_gap.cpp"
        "src/driver_gatt.cpp"
        "src/driver_gattc.cpp"
        "src/driver_gatts.cpp"
        "src/driver_uecc.cpp"
        "src/*.h"
    )
endif()

file (GLOB UECC_SOURCE_FILES
    "src/uECC/*.c"
)

# Force .c files to be compiled with the C++ compiler
set_source_files_properties(
    ${UECC_SOURCE_FILES}
    PROPERTIES
    LANGUAGE CXX
)

# Build the pc-ble-driver as a static library
add_definitions(
    -DPC_BLE_DRIVER_STATIC
)

# Compiler specific
if(MSVC)
    include(${CMAKE_CURRENT_LIST_DIR}/cmake/msvc.cmake)
elseif(APPLE)
    include(${CMAKE_CURRENT_LIST_DIR}/cmake/apple.cmake)
else()
    # Linux
    include(${CMAKE_CURRENT_LIST_DIR}/cmake/gcc.cmake)
endif()

foreach(SD_API_VER ${SD_API_VERS})
    string(TOLOWER ${SD_API_VER} SD_API_VER_L)
    set(CURRENT_TARGET pc-ble-driver-js-${SD_API_VER_L})

    add_library(${CURRENT_TARGET} SHARED ${SOURCE_FILES} ${UECC_SOURCE_FILES} ${LIB_PLATFORM_SRC_FILES})

    # This line will give our library file a .node extension without any "lib" prefix
    set_target_properties(${CURRENT_TARGET}
        PROPERTIES
        COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -DBUILDING_NODE_EXTENSION"
        PREFIX ""
        SUFFIX ".node")

	string(REGEX MATCH "[0-9]+$" _SD_API_VER_NUM "${SD_API_VER}")
	set_target_properties(${CURRENT_TARGET} PROPERTIES COMPILE_OPTIONS -DNRF_SD_BLE_API_VERSION=${_SD_API_VER_NUM})
    include_directories(${NRF_BLE_DRIVER_INCLUDE_DIR}/${SD_API_VER})
    target_include_directories(${CURRENT_TARGET} PRIVATE ${NRF_BLE_DRIVER_${SD_API_VER}_PUBLIC_INCLUDE_DIRS})
	
    if(WIN32)
        # suppress C4251 v8/msvc related warning, for more info: https://github.com/nodejs/node/pull/15570
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4251")

        target_include_directories(${CURRENT_TARGET} PRIVATE "${CMAKE_JS_INC}/win")
        set_target_properties(${CURRENT_TARGET} PROPERTIES COMPILE_DEFINITIONS "_CRT_SECURE_NO_WARNINGS")
        set_target_properties(${CURRENT_TARGET} PROPERTIES LINK_FLAGS "/DELAYLOAD:node.exe" )
    elseif(APPLE)
        target_link_libraries(${CURRENT_TARGET} "-framework CoreFoundation")
        target_link_libraries(${CURRENT_TARGET} "-framework IOKit")
        set_property(TARGET ${CURRENT_TARGET} PROPERTY MACOSX_RPATH ON)
    else()
        # Assume Linux
        target_link_libraries(${CURRENT_TARGET} "udev")
    endif()

    # actual shared and static libraries built from the same object files
    target_link_libraries(${CURRENT_TARGET} ${CMAKE_JS_LIB} ${NRF_BLE_DRIVER_${SD_API_VER}_STATIC_LIB})
endforeach(SD_API_VER)

cmake_minimum_required(VERSION 3.20)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(MQTT_CONFIG_FILE ${CMAKE_CURRENT_LIST_DIR}/mqtt_device_config)

if(DEFINED ENV{PICO_SDK_PATH})
    set(PICO_SDK_PATH "$ENV{PICO_SDK_PATH}" CACHE PATH "Path to Pico SDK")
endif()
if(DEFINED ENV{PICO_SDK_PATH})
    set(PICO_EXAMPLES_PATH "$ENV{PICO_EXAMPLES_PATH}" CACHE PATH "Path to Pico Examples")
endif()
if(DEFINED ENV{PICO_SDK_PATH})
    set(FREERTOS_KERNEL_PATH "$ENV{FREERTOS_KERNEL_PATH}" CACHE PATH "Path to FreeRTOS Kernel")
endif()
if(DEFINED ENV{WIFI_SSID})
    set(WIFI_SSID $ENV{WIFI_SSID})
endif()
if(DEFINED ENV{WIFI_PASSWORD})
    set(WIFI_PASSWORD $ENV{WIFI_PASSWORD})
endif()
if(DEFINED ENV{MQTT_SERVER})
    set(MQTT_SERVER $ENV{MQTT_SERVER})
endif()
if(DEFINED ENV{MQTT_USERNAME})
    set(MQTT_USERNAME $ENV{MQTT_USERNAME})
endif()
if(DEFINED ENV{MQTT_PASSWORD})
    set(MQTT_PASSWORD $ENV{MQTT_PASSWORD})
endif()
if(DEFINED ENV{MQTT_CERT_INC})
    set(MQTT_CERT_INC $ENV{MQTT_CERT_INC})
endif()
if(DEFINED ENV{MQTT_CERT_PATH})
    set(MQTT_CERT_PATH $ENV{MQTT_CERT_PATH} CACHE PATH "Path to MQTT cert directory")
endif()

if (EXISTS ${MQTT_CONFIG_FILE})
    file(STRINGS ${MQTT_CONFIG_FILE} CONFIG_LINES REGEX "^[^#]")
    foreach(LINE ${CONFIG_LINES})
        if(${LINE} MATCHES "^[ \t]*([A-Za-z0-9_]+)[ \t]*=[ \t]*(.*)$")
            if (${CMAKE_MATCH_1} STREQUAL "wifi_ssid")
                set(WIFI_SSID ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "wifi_password")
                set(WIFI_PASSWORD ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "mqtt_server")
                set(MQTT_SERVER ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "mqtt_username")
                set(MQTT_USERNAME ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "mqtt_password")
                set(MQTT_PASSWORD ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "mqtt_cert_include")
                set(MQTT_CERT_INC ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "mqtt_cert_path")
                set(MQTT_CERT_PATH ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "pico_sdk_path")
                set(PICO_SDK_PATH ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "pico_examples_path")
                set(PICO_EXAMPLES_PATH ${CMAKE_MATCH_2})
            elseif(${CMAKE_MATCH_1} STREQUAL "freertos_kernel_path")
                set(FREERTOS_KERNEL_PATH ${CMAKE_MATCH_2})
            endif()
        endif()
    endforeach()
endif()

if (NOT WIFI_SSID OR NOT MQTT_SERVER)
    message(FATAL_ERROR "WIFI SSID and MQTT server must be specified in environment variables or ${MQTT_CONFIG_FILE}")
    return()
endif()

# Set path to the certificate include file
if (NOT MQTT_CERT_PATH)
    set(MQTT_CERT_PATH ${CMAKE_CURRENT_LIST_DIR}/certs/${MQTT_SERVER})
endif()

# Set the name of the certificate include file
if (NOT MQTT_CERT_INC)
    set(MQTT_CERT_INC mqtt_client.inc)
endif()

set(sdkVersion 2.1.1)
set(toolchainVersion 14_2_Rel1)
set(picotoolVersion 2.1.1)
set(picoVscode $ENV{PICO_SDK_PATH}/cmake/pico-vscode.cmake)
if (EXISTS ${picoVscode})
    include(${picoVscode})
endif()
# ====================================================================================
set(PICO_BOARD pico2_w CACHE STRING "Board type")

# Pull in Raspberry Pi Pico SDK (must be before project)
include(${PICO_SDK_PATH}/external/pico_sdk_import.cmake)

project(pico-mqtt C CXX ASM)

# Initialise the Raspberry Pi Pico SDK
pico_sdk_init()

add_executable(pico-mqtt
    mqtt_client.c
    )

pico_set_program_name(pico-mqtt "pico-mqtt")
pico_set_program_version(pico-mqtt "0.1")

pico_enable_stdio_uart(pico-mqtt 0)
pico_enable_stdio_usb(pico-mqtt 1)

target_include_directories(pico-mqtt PRIVATE
    ${CMAKE_CURRENT_LIST_DIR}
    ${CMAKE_SOURCE_DIR}/include
    ${PICO_EXAMPLES_PATH}/pico_w/wifi # For common wifi headers
)

target_link_libraries(pico-mqtt
    pico_stdlib
    hardware_adc
    pico_cyw43_arch_lwip_threadsafe_background
    pico_lwip_mqtt
    pico_mbedtls
    pico_lwip_mbedtls
)

target_compile_definitions(pico-mqtt PRIVATE
    WIFI_SSID=\"${WIFI_SSID}\"
    WIFI_PASSWORD=\"${WIFI_PASSWORD}\"
    MQTT_SERVER=\"${MQTT_SERVER}\"
    )

if (MQTT_CERT_PATH AND MQTT_CERT_INC AND EXISTS "${MQTT_CERT_PATH}/${MQTT_CERT_INC}")
    message("Using MQTT_CERT_PATH \"${MQTT_CERT_PATH}\" and MQTT_CERT_INC \"${MQTT_CERT_INC}\"")
    target_compile_definitions(pico-mqtt PRIVATE
        MQTT_CERT_INC=\"${MQTT_CERT_INC}\" # contains the tls certificates for MQTT_SERVER needed by the client
        ALTCP_MBEDTLS_AUTHMODE=MBEDTLS_SSL_VERIFY_REQUIRED
        )
    target_include_directories(pico-mqtt PRIVATE
        ${MQTT_CERT_PATH}
        )
endif()
if (MQTT_USERNAME AND MQTT_PASSWORD)
    target_compile_definitions(pico-mqtt PRIVATE
        MQTT_USERNAME=\"${MQTT_USERNAME}\"
        MQTT_PASSWORD=\"${MQTT_PASSWORD}\"
    )
endif()
pico_enable_stdio_usb(pico-mqtt 1)
pico_enable_stdio_uart(pico-mqtt 1)
pico_add_extra_outputs(pico-mqtt)



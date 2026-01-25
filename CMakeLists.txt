cmake_minimum_required(VERSION 3.20)

set(MQTT_SERVER <my_mqtt_server>)
set(WIFI_SSID <my_wifi_ssid>)
set(WIFI_PASSWORD <my_wifi_password>)
# set(MQTT_USERNAME username)
# set(MQTT_PASSWORD password)
# set(MQTT_CERT_PATH ${CMAKE_CURRENT_LIST_DIR}/certs/${MQTT_SERVER})
# set(MQTT_CERT_INC mqtt_client.inc)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(MQTT_SERVER "${MQTT_SERVER}" CACHE INTERNAL "MQTT server for examples")

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
set(PICO_SDK_PATH "$ENV{PICO_SDK_PATH}" CACHE PATH "Path to Pico SDK")
include(${PICO_SDK_PATH}/external/pico_sdk_import.cmake)

project(pico-mqtt C CXX ASM)

# Initialise the Raspberry Pi Pico SDK
pico_sdk_init()

set(PICO_EXAMPLES_PATH "$ENV{PICO_EXAMPLES_PATH}" CACHE PATH "Path to Pico Examples")
set(FREERTOS_KERNEL_PATH "$ENV{FREERTOS_KERNEL_PATH}" CACHE PATH "Path to FreeRTOS Kernel")
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



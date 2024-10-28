cmake_minimum_required(VERSION 3.15)

set(PICOTOOL_FETCH_FROM_GIT_PATH ${CMAKE_CURRENT_LIST_DIR}/thirdParty/picotool)
set(PICO_SDK_FETCH_FROM_GIT ON)
set(PICO_BOARD none)
set(PICO_PLATFORM rp2350)
set(PICO_COPY_TO_RAM 1)

string(TOLOWER ${CMAKE_BUILD_TYPE} build_type)

set(LOG 0)

if (build_type STREQUAL debug)
    set(PICO_DEOPTIMIZED_DEBUG 1)
else ()
    set(PICO_DEOPTIMIZED_DEBUG 0)
endif ()

include(pico_sdk_import.cmake)
include(pico_extras_import_optional.cmake)

set(FREERTOS_KERNEL_PATH ${CMAKE_CURRENT_LIST_DIR}/thirdParty/FreeRTOS-Kernel)
include(FreeRTOS_Kernel_import.cmake)

project(YukaLink C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# Use FreeRTOS support in tinyusb
set(TINYUSB_OPT_OS OPT_OS_FREERTOS)
set(PICO_TINYUSB_PATH ${CMAKE_CURRENT_LIST_DIR}/thirdParty/tinyusb)

# Don't use Pico SDK stdio
set(PICO_STDIO_UART OFF)

# initialize the Raspberry Pi Pico SDK
pico_sdk_init()

# Add project executable
add_executable(YukaLink)

# Raspberry Pi Pico SDK parameters
target_compile_definitions(YukaLink PUBLIC PICO_FLASH_SIZE_BYTES=4194304)
target_compile_definitions(YukaLink PUBLIC PICO_BOOT_STAGE2_CHOOSE_W25Q080=1)
target_compile_definitions(YukaLink PUBLIC PICO_FLASH_SPI_CLKDIV=2)
target_compile_definitions(YukaLink PUBLIC PICO_RP2350_A2_SUPPORTED=1)
target_compile_definitions(YukaLink PUBLIC PICO_STDIO_SHORT_CIRCUIT_CLIB_FUNCS=0)

pico_set_printf_implementation(YukaLink "compiler")

# BlackMagic parameters
if (build_type STREQUAL debug)
    target_compile_definitions(YukaLink PUBLIC ENABLE_DEBUG=1)
    #target_compile_definitions(YukaLink PUBLIC ENABLE_DEBUG=0)
else ()
    target_compile_definitions(YukaLink PUBLIC ENABLE_DEBUG=0)
endif ()

target_compile_definitions(YukaLink PUBLIC GDB_PACKET_BUFFER_SIZE=8192)
target_compile_definitions(YukaLink PUBLIC PC_HOSTED=0)
target_compile_definitions(YukaLink PUBLIC NO_LIBOPENCM3)
target_compile_definitions(YukaLink PUBLIC DFU_SERIAL_LENGTH=PICO_UNIQUE_BOARD_ID_SIZE_BYTES+1)
target_compile_definitions(YukaLink PUBLIC SYSTICKHZ=100)
target_compile_definitions(YukaLink PUBLIC ENABLE_RTT=1)
target_compile_definitions(YukaLink PUBLIC PLATFORM_IDENT_DYNAMIC)

# BlackMagic headers
target_include_directories(YukaLink PUBLIC thirdParty/blackmagic/src/include)
target_include_directories(YukaLink PUBLIC thirdParty/blackmagic/src/target)

# YukaLink headers
target_include_directories(YukaLink PUBLIC include)
target_include_directories(YukaLink PUBLIC include/bmp)
target_include_directories(YukaLink PUBLIC ${CMAKE_CURRENT_BINARY_DIR})

# Generate header with BlackMagic and FreeRTOS-Kernel versions
execute_process(COMMAND ${GIT_EXECUTABLE} --git-dir ${CMAKE_CURRENT_SOURCE_DIR}/thirdParty/blackmagic/.git describe --tags --always
        OUTPUT_VARIABLE GIT_BMP_REPO_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND ${GIT_EXECUTABLE} --git-dir ${CMAKE_CURRENT_SOURCE_DIR}/thirdParty/FreeRTOS-Kernel/.git describe --tags --always
        OUTPUT_VARIABLE GIT_FREERTOS_REPO_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND ${GIT_EXECUTABLE} --git-dir ${CMAKE_CURRENT_SOURCE_DIR}/../.git describe --tags --always
        OUTPUT_VARIABLE GIT_MIOLINK_REPO_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
configure_file("git_version.h.in" "git_version.h")

# BlackMagic sources
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/command.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/exception.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/gdb_main.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/gdb_packet.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/hex_utils.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/maths_utils.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/morse.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/remote.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/rtt.c)
target_sources(YukaLink PUBLIC thirdParty/blackmagic/src/timing.c)

# BlackMagic targets sources
file(GLOB blackmagic_targets CONFIGURE_DEPENDS "thirdParty/blackmagic/src/target/*.c")
list(FILTER blackmagic_targets EXCLUDE REGEX ".*adiv5_swd\\.c$")
list(FILTER blackmagic_targets EXCLUDE REGEX ".*swdptap_generic\\.c$")
target_sources(YukaLink PUBLIC ${blackmagic_targets})

# YukaLink sources
file(GLOB miolink_bmp_sources CONFIGURE_DEPENDS "source/bmp/*.c")
target_sources(YukaLink PUBLIC ${miolink_bmp_sources}
                              source/main.c)

# SEGGER RTT sources (need only in the debug configuration)
if (build_type STREQUAL debug)
    target_sources(YukaLink PUBLIC source/SysView/SEGGER_RTT.c)
    target_sources(YukaLink PUBLIC source/SysView/SEGGER_RTT_printf.c)
    target_sources(YukaLink PUBLIC source/SysView/SEGGER_RTT_ASM_ARMv7M.S)

    target_compile_definitions(YukaLink PUBLIC ENABLE_SYSVIEW_TRACE=1)

    target_include_directories(YukaLink PUBLIC source/SysView)

    target_sources(YukaLink PUBLIC source/SysView/SEGGER_SYSVIEW.c)
    target_sources(YukaLink PUBLIC source/SysView/SEGGER_SYSVIEW_Config_FreeRTOS.c)
    target_sources(YukaLink PUBLIC source/SysView/SEGGER_SYSVIEW_FreeRTOS.c)

else ()
    target_compile_definitions(YukaLink PUBLIC ENABLE_SYSVIEW_TRACE=0)
endif ()

# Raspberry Pi Pico SDK libraries
target_link_libraries(YukaLink PUBLIC hardware_pio
                                      hardware_uart
                                      hardware_dma
                                      hardware_timer
                                      hardware_adc)

target_link_libraries(YukaLink PUBLIC pico_stdlib
                                      pico_unique_id
                                      pico_bootrom
                                      tinyusb_device
                                      tinyusb_board
                                      tinyusb_additions)

target_link_libraries(YukaLink PUBLIC FreeRTOS-Kernel
                                     FreeRTOS-Kernel-Heap4)

# Compile PIO programs
pico_generate_pio_header(YukaLink ${CMAKE_CURRENT_LIST_DIR}/source/bmp/miolink_rev_a_swd.pio)
pico_generate_pio_header(YukaLink ${CMAKE_CURRENT_LIST_DIR}/source/bmp/miolink_rev_b_swd.pio)
pico_generate_pio_header(YukaLink ${CMAKE_CURRENT_LIST_DIR}/source/bmp/miolink_pico_swd.pio)
pico_generate_pio_header(YukaLink ${CMAKE_CURRENT_LIST_DIR}/source/bmp/pico_swd.pio)
pico_generate_pio_header(YukaLink ${CMAKE_CURRENT_LIST_DIR}/source/bmp/target_jtag.pio)

# Set linker script and set parameter to print memory usage
target_link_options(YukaLink PRIVATE -Xlinker --print-memory-usage)

# Add YukaLink output
pico_add_extra_outputs(YukaLink)
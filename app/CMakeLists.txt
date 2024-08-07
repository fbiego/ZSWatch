cmake_minimum_required(VERSION 3.20.0)

# Apply patches before build
file(GLOB_RECURSE files RELATIVE ${CMAKE_SOURCE_DIR} "patches/*.patch")
foreach(file ${files})
    execute_process(COMMAND 
                    patch -p1 -d $ENV{ZEPHYR_BASE} -i ${CMAKE_CURRENT_SOURCE_DIR}/${file} -r - --no-backup-if-mismatch)
endforeach()

string(REGEX MATCH "zswatch_nrf5340_cpuapp(_ns)?@([0-9]+)$" ZSWATCH_BOARD "${BOARD}")
if(ZSWATCH_BOARD)
    set(ZSWATCH_REV ${CMAKE_MATCH_2})
    if(${ZSWATCH_REV} EQUAL 2)
        set(DFU_BUILD true)
        set(PM_STATIC_YML_FILE ${CMAKE_CURRENT_SOURCE_DIR}/partition_mcuboot.yml)
        set(OVERLAY_CONFIG ${CMAKE_CURRENT_SOURCE_DIR}/boards/dfu.conf)
        set(mcuboot_DTC_OVERLAY_FILE ${CMAKE_CURRENT_SOURCE_DIR}/overlays/mcuboot.overlay)
    elseif(${ZSWATCH_REV} GREATER_EQUAL 3)
        set(PM_STATIC_YML_FILE ${CMAKE_CURRENT_SOURCE_DIR}/partition_external.yml)
    else()
        set(PM_STATIC_YML_FILE ${CMAKE_CURRENT_SOURCE_DIR}/partition_internal.yml)
    endif()
endif()

if("nrf5340dk_nrf5340_cpuapp" STREQUAL "${BOARD}")
    set(PM_STATIC_YML_FILE ${CMAKE_CURRENT_SOURCE_DIR}/partition_external.yml)
endif()

set(BOARD_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(ZSWatchFW)

# These are XIP related anomalies and aren't applicable for us and cause
# throughput issues and QSPI issues on NCS 2.6.0
zephyr_compile_definitions(
  -DNRF53_ERRATA_43_ENABLE_WORKAROUND=0
  -DNRF52_ERRATA_215_ENABLE_WORKAROUND=0
)

add_subdirectory(drivers)
add_subdirectory(src/history)
add_subdirectory(src/applications)
add_subdirectory(src/ui/watchfaces)
add_subdirectory(src/sensors)
add_subdirectory(src/sensor_fusion)
add_subdirectory(src/events)
add_subdirectory(src/managers)
add_subdirectory(src/drivers)
add_subdirectory(src/ble)
add_subdirectory(src/images)

include_directories(src/)
include_directories(src/ui)
include_directories(src/applications)
include_directories(src/ui/watchfaces)

if (CONFIG_DT_HAS_NORDIC_NPM1300_ENABLED)
    add_subdirectory(src/fuel_gauge)
else()
    add_subdirectory(src/basic_battery)
endif()

# TODO
target_sources(app PRIVATE src/zsw_clock.c)
#if (CONFIG_RTC)
#    target_sources(app PRIVATE src/zsw_rtc.c)
#else()
#endif()

target_sources(app PRIVATE src/main.c)
target_sources(app PRIVATE src/zsw_cpu_freq.c)
target_sources(app PRIVATE src/zsw_retained_ram_storage.c)
target_sources(app PRIVATE src/zsw_coredump.c)

target_sources(app PRIVATE src/ui/notification/zsw_popup_notifcation.c)
target_sources(app PRIVATE src/ui/popup/zsw_popup_window.c)
target_sources(app PRIVATE src/ui/utils/zsw_ui_utils.c)

target_sources(app PRIVATE src/ui/watchfaces/zsw_watchface_dropdown_ui.c)

target_sources_ifdef(CONFIG_SPI_FLASH_LOADER app PRIVATE src/filesystem/zsw_rtt_flash_loader.c)
target_sources_ifdef(CONFIG_FILE_SYSTEM_LITTLEFS app PRIVATE src/filesystem/zsw_filesystem.c)
target_sources_ifdef(CONFIG_FILE_SYSTEM_LITTLEFS app PRIVATE src/filesystem/zsw_lvgl_spi_decoder.c)

if(DFU_BUILD)
    target_sources(app PRIVATE src/dfu.c)
endif()

FILE(GLOB events_sources src/events/*.c)
target_sources(app PRIVATE ${events_sources})

add_compile_definitions(LV_LVGL_H_INCLUDE_SIMPLE)

file(GLOB ext_fs_files "src/images/binaries/S/*.*")
list(LENGTH ext_fs_files num_ext_fs_files)
add_compile_definitions(NUM_RAW_FS_FILES=${num_ext_fs_files})
message("Number of external filesystem files:" ${num_ext_fs_files})
cmake_minimum_required(VERSION 3.13)

# Disable in-source build.
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
    message(FATAL_ERROR "In-source build is not allowed, please use a separate build folder.")
endif()

project(amazon-freertos)
set(PROJECT_VERSION "201906.00 Major")
set(PROJECT_VERSION_MAJOR "201906")
set(PROJECT_VERSION_MINOR "00")

# Import global configurations.
include("tools/cmake/afr.cmake")

# Add 3rdparty modules.
add_subdirectory("libraries/3rdparty")

# -------------------------------------------------------------------------------------------------
# Configure target board
# -------------------------------------------------------------------------------------------------
# Get list of supported boards.
afr_get_boards(AFR_SUPPORTED_BOARDS)

set(AFR_BOARD "vendor.board" CACHE STRING "Target board chosen by the user at configure time")
set_property(CACHE AFR_BOARD PROPERTY STRINGS ${AFR_SUPPORTED_BOARDS})

string(REGEX MATCH [[(.+)\.(.+)]] __match_result ${AFR_BOARD})
set(AFR_VENDOR_NAME ${CMAKE_MATCH_1} CACHE INTERNAL "MCU vendor name")
set(AFR_BOARD_NAME ${CMAKE_MATCH_2} CACHE INTERNAL "MCU board name")

# Abort if the target board is not supported, i.e., corresponding folder is not present.
if(NOT AFR_BOARD IN_LIST AFR_SUPPORTED_BOARDS)
    message(FATAL_ERROR "Board is not supported: ${AFR_BOARD}")
endif()

# Import board CMake build.
set(AFR_VENDOR_PATH "vendors/${AFR_VENDOR_NAME}" CACHE INTERNAL "")
include("${AFR_VENDOR_PATH}/manifest.cmake")
if(DEFINED AFR_MANIFEST_BOARD_DIR_${AFR_BOARD_NAME})
    set(AFR_BOARD_PATH "${AFR_VENDOR_PATH}/${AFR_MANIFEST_BOARD_DIR_${AFR_BOARD_NAME}}" CACHE INTERNAL "")
elseif(DEFINED AFR_MANIFEST_BOARD_DIR)
    set(AFR_BOARD_PATH "${AFR_VENDOR_PATH}/${AFR_MANIFEST_BOARD_DIR}/${AFR_BOARD_NAME}" CACHE INTERNAL "")
else()
    message(FATAL_ERROR "Could not import board CMakeLists.txt.")
endif()
add_subdirectory("${AFR_BOARD_PATH}")

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS modules
# -------------------------------------------------------------------------------------------------
# Initialize all modules.
add_subdirectory("libraries")
add_subdirectory("demos")
add_subdirectory("tests")

# Resolve dependencies.
afr_status("=========================Resolving dependencies==========================")
afr_resolve_dependencies()

# -------------------------------------------------------------------------------------------------
# Summary
# -------------------------------------------------------------------------------------------------
afr_status("")
afr_status("====================Configuration for Amazon FreeRTOS====================")
afr_status("  Version:                 " "${AFR_VERSION}")
afr_status("  Git version:             " "${AFR_VERSION_VCS}")

# ================ Target microcontroller =================
afr_status("")
afr_status("Target microcontroller:")

afr_get_board_metadata(vendor_name VENDOR_NAME)
afr_get_board_metadata(board_name  DISPLAY_NAME)
afr_get_board_metadata(description DESCRIPTION)
afr_get_board_metadata(family      FAMILY_NAME)
afr_get_board_metadata(data_ram    DATA_RAM_MEMORY)
afr_get_board_metadata(program_mem PROGRAM_MEMORY)

afr_status("  vendor:                  " "${vendor_name}")
afr_status("  board:                   " "${board_name}")
afr_status("  description:             " "${description}")
afr_status("  family:                  " "${family}")
afr_status("  data ram size:           " "${data_ram}")
afr_status("  program memory size:     " "${program_mem}")

# ===================== Host platform =====================
afr_status("")
afr_status("Host platform:")

afr_status("  OS:                      " "${CMAKE_HOST_SYSTEM}")
afr_status("  Toolchain:               " "${AFR_TOOLCHAIN}")
afr_status("  Toolchain path:          " "${CMAKE_FIND_ROOT_PATH}")
afr_status("  CMake generator:         " "${CMAKE_GENERATOR}")

# ================ Amazon FreeRTOS modules ================
afr_status("")
afr_status("Amazon FreeRTOS modules:")

afr_status("  Modules to build:        " "${AFR_MODULES_BUILD}")
afr_status("  Enabled by user:         " "${AFR_MODULES_ENABLED_USER}")
afr_status("  Enabled by dependency:   " "${AFR_MODULES_ENABLED_DEPS}")
afr_status("  3rdparty dependencies:   " "${3RDPARTY_MODULES_ENABLED}")
afr_status("  Available demos:         " "${AFR_DEMOS_ENABLED}")
afr_status("  Available tests:         " "${AFR_TESTS_ENABLED}")

afr_status("=========================================================================")
afr_status("")

# -------------------------------------------------------------------------------------------------
# Demos and tests
# -------------------------------------------------------------------------------------------------
# Enable us to use targets defined in vendor CMake files.
cmake_policy(SET CMP0079 NEW)

if(TARGET aws_demos)
    list(TRANSFORM AFR_DEMOS_ENABLED PREPEND "AFR::" OUTPUT_VARIABLE demos_list)
    target_link_libraries(aws_demos PRIVATE ${demos_list})
endif()

if(TARGET aws_tests)
    list(TRANSFORM AFR_TESTS_ENABLED PREPEND "AFR::" OUTPUT_VARIABLE tests_list)
    target_link_libraries(aws_tests PRIVATE ${tests_list})
endif()

# -------------------------------------------------------------------------------------------------
# Output metadata information
# -------------------------------------------------------------------------------------------------
if(AFR_METADATA_MODE)
    afr_write_metadata()
endif()

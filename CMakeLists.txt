cmake_minimum_required(VERSION 3.21)

# -----PROJECT NAME-----
# this hack puts dir name as project name
cmake_path(GET CMAKE_CURRENT_SOURCE_DIR FILENAME ProjectName)
string(REPLACE " " "_" ProjectId ${ProjectName})
project(${ProjectName})
set(OUTPUT_NAME ${OUTPUT_BIN_NAME})

# -----MCU-----
set(MCPU cortex-m7)
set(MCU_MODEL STM32H743xx)
set(MFPU fpv5-d16)

# ----COMPILER SETTINGS----
enable_language(C CXX ASM)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_SYSTEM_NAME Generic) # bare metal
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_AR arm-none-eabi-ar)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_OBJDUMP arm-none-eabi-objdump)
set(SIZE arm-none-eabi-size)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# ----SOURCES, HEADERS, SCRIPTS----
file(GLOB_RECURSE C_SRC CONFIGURE_DEPENDS Src/*.c)

file(GLOB_RECURSE C_LIBS CONFIGURE_DEPENDS Drivers/*.c Middlewares/*.c)

# macro(HEADER_DIRECTORIES return_list) file(GLOB_RECURSE new_list
# CONFIGURE_DEPENDS *.h) set(dir_list "") foreach(file_path ${new_list})
# get_filename_component(dir_path ${file_path} PATH) set(dir_list ${dir_list}
# ${dir_path}) endforeach() list(REMOVE_DUPLICATES dir_list) set(${return_list}
# ${dir_list}) endmacro() header_directories(AUTO_INCLUDE_DIRS)

set(INCLUDE_DIRS Inc)

set(INCLUDE_DIRS_LIBS
    Drivers/CMSIS/Include
    Drivers/STM32H7xx_HAL_Driver/Inc
    Drivers/STM32H7xx_HAL_Driver/Inc/Legacy
    Drivers/CMSIS/Device/ST/STM32H7xx/Include
    Drivers/BSP/Components/lan8742
    Middlewares/Third_Party/FreeRTOS/Source/include
    Middlewares/Third_Party/FreeRTOS/Source/CMSIS_RTOS_V2
    Middlewares/Third_Party/FreeRTOS/Source/portable/GCC/ARM_CM4F
    Middlewares/Third_Party/LwIP/src/include
    Middlewares/Third_Party/LwIP/src/include/netif/ppp
    Middlewares/Third_Party/LwIP/system
    Middlewares/Third_Party/LwIP/src/include/lwip
    Middlewares/Third_Party/LwIP/src/include/lwip/apps
    Middlewares/Third_Party/LwIP/src/include/lwip/priv
    Middlewares/Third_Party/LwIP/src/include/lwip/prot
    Middlewares/Third_Party/LwIP/src/include/netif
    Middlewares/Third_Party/LwIP/src/include/compat/posix
    Middlewares/Third_Party/LwIP/src/include/compat/posix/arpa
    Middlewares/Third_Party/LwIP/src/include/compat/posix/net
    Middlewares/Third_Party/LwIP/src/include/compat/posix/sys
    Middlewares/Third_Party/LwIP/src/include/compat/stdc
    Middlewares/Third_Party/LwIP/system/arch
    # Middlewares/Third_Party/mbedTLS/include/mbedtls
    # Middlewares/Third_Party/mbedTLS/include
)

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/STM32H743ZITx_FLASH.ld)
set(STARTUP_SCRIPT ${CMAKE_SOURCE_DIR}/startup_stm32h743xx.s)

# -----COMPILE OPTIONS-----
set(COMP_DEFS USE_HAL_DRIVER ${MCU_MODEL} ARM_MATH_CM7 ARM_MATH_MATRIX_CHECK
              ARM_MATH_ROUNDING)

set(COMP_OPTS
    -mcpu=${MCPU} # needs to be here too for correct instuction set
    -flto # link time optimization
    -mthumb
    -ffunction-sections # functions in their own section
    -fdata-sections # data in its own section
    -fno-common # global vaiables are only defined once
    -fmessage-length=0 # no wrap for warnings
    -Wall
    -Wno-unused-parameter
    -mfloat-abi=hard
    -mfpu=${MFPU})

set(LINK_OPTS
    -mcpu=${MCPU}
    -mthumb
    -Wl,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref
    -T
    ${LINKER_SCRIPT}
    -lm
    -lnosys
    --specs=nosys.specs
    -Wl,-gc-sections,--print-memory-usage
    -mfloat-abi=hard
    -mfpu=${MFPU})

# -----TARGET NAME---------
set(TARGET_NAME ${OUTPUT_NAME}.elf)
set(HEX_FILE ${PROJECT_BINARY_DIR}/${OUTPUT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${OUTPUT_NAME}.bin)

# -----SET COMPILE OPTIONS FOR MAIN TARGET -------
add_executable(${TARGET_NAME} ${C_SRC} ${C_LIBS} ${STARTUP_SCRIPT}
                              ${LINKER_SCRIPT})
target_compile_definitions(${TARGET_NAME} PRIVATE ${COMP_DEFS})
target_compile_options(${TARGET_NAME} PRIVATE ${COMP_OPTS})
target_link_options(${TARGET_NAME} PRIVATE ${LINK_OPTS})
target_link_libraries(${TARGET_NAME} m)
target_include_directories(${TARGET_NAME} PRIVATE ${INCLUDE_DIRS}
                                                  ${INCLUDE_DIRS_LIBS})

# -----BUILD TYPE-----
if("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
  message(STATUS "High optimization for speed")
  add_compile_options(-O3)
  # add_compile_options(-Ofast)
elseif("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
  message(STATUS "High optimization for speed, debug info included")
  add_compile_options(-O3 -g)
  # add_compile_options(-Ofast -g)
elseif("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
  message(STATUS "Maximum optimization for size")
  add_compile_options(-Os)
else()
  message(STATUS "Minimal optimization, debug info included")
  add_compile_options(-Og -g)
endif()

# -----BUILD COMMANDS-----
add_custom_command(
  TARGET ${TARGET_NAME}
  POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${TARGET_NAME}> ${HEX_FILE}
  COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${TARGET_NAME}> ${BIN_FILE}
  COMMENT "Building ${HEX_FILE}
    Building ${BIN_FILE}")

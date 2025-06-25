cmake_minimum_required(VERSION 3.6)

project(RS41HUP C CXX)

# Toolchain config
set(CMAKE_SYSTEM_NAME "Generic")
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(TOOLCHAIN_DIR "/usr/bin/")
set(CMAKE_C_COMPILER "${TOOLCHAIN_DIR}/arm-none-eabi-gcc")
set(CMAKE_CXX_COMPILER "${TOOLCHAIN_DIR}/arm-none-eabi-g++")

# Compiler/Linker flags
set(LINKER_SCRIPT ${PROJECT_SOURCE_DIR}/arm-gcc-link.ld)
set(COMMON_FLAGS " --specs=nosys.specs -mcpu=cortex-m3 -mthumb -Wall -ffunction-sections -g -O3 -nostartfiles ")
set(CMAKE_C_FLAGS "${COMMON_FLAGS} -std=gnu99")
set(CMAKE_CXX_FLAGS "${COMMON_FLAGS} -std=c++11")

set(CMAKE_EXE_LINKER_FLAGS
    "-Wl,-Map=${CMAKE_BINARY_DIR}/${PROJECT_NAME}.map -lstdc++ -Wl,--gc-sections --specs=nano.specs -T ${LINKER_SCRIPT}"
)

# Defines
add_definitions(
    -DSTM32F100C8
    -DSTM32F10X_MD_VL
    -DUSE_STDPERIPH_DRIVER
    -DSUPPORT_CPLUSPLUS
    -D__ASSEMBLY__
)

# Source files (ignora arquivos do CMake/CompilerId)
file(GLOB_RECURSE USER_SOURCES RELATIVE ${CMAKE_SOURCE_DIR} "${CMAKE_SOURCE_DIR}/*.c")
file(GLOB_RECURSE USER_SOURCES_CXX RELATIVE ${CMAKE_SOURCE_DIR} "${CMAKE_SOURCE_DIR}/*.cpp")
file(GLOB_RECURSE USER_HEADERS RELATIVE ${CMAKE_SOURCE_DIR} "${CMAKE_SOURCE_DIR}/*.h")

foreach(file ${USER_SOURCES} ${USER_SOURCES_CXX})
    if("${file}" MATCHES "CMakeFiles|CompilerIdC|CompilerIdCXX")
        list(REMOVE_ITEM USER_SOURCES ${file})
        list(REMOVE_ITEM USER_SOURCES_CXX ${file})
    endif()
endforeach()

# Include paths
include_directories(
    cmsis
    cmsis_boot
    stm_lib/inc
    .
)

# Executável final
add_executable(${PROJECT_NAME}.elf
    ${USER_SOURCES}
    ${USER_SOURCES_CXX}
    ${USER_HEADERS}
)

# Tools para pós-compilação
find_program(CMAKE_OBJCOPY arm-none-eabi-objcopy)
find_program(CMAKE_SIZE arm-none-eabi-size)

# Geração de .hex e .bin
set(HEX_FILE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.bin)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
    COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
    COMMAND ${CMAKE_SIZE} ${PROJECT_NAME}.elf
    COMMENT "Building ${HEX_FILE} and ${BIN_FILE}"
)

# Target de gravação (ajustável)
add_custom_target(program
    DEPENDS ${PROJECT_NAME}.elf
    COMMENT "Flashing ${BIN_FILE}"
)

if(WIN32)
    add_custom_command(TARGET program POST_BUILD
        COMMAND D:/Programy/stlink-1.3.0-win64/bin/st-flash --reset write ${BIN_FILE} 0x08000000
    )
endif()

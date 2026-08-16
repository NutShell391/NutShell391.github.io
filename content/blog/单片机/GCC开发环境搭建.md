---
title: 'GCC开发环境搭建'
date: "2026-08-16T13:00:56+08:00"
draft: false
tags:
  - 梁山派
  - GCC
  - 开发环境
description: 记录梁山派开发板 GCC 交叉编译开发环境的搭建过程。
---

## GCC 开发环境搭建

参考文献 [放弃 Keil5：基于 CLion + CMake + GCC 搭建 GD32F470 嵌入式开发环境](https://auzsadong.github.io/2026/04/14/GD32%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/CMake%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/CMake+GCC%E6%9E%B6%E6%9E%84%E6%90%AD%E5%BB%BA)

### 下载

从 [官网](https://www.gigadevice.com.cn/product/mcu/mcus-product-selector/gd32f470zgt6) 里面下载固件库、用户手册和数据手册。

### 配置

新建Demo_Template文件夹，将固件库里面的Firmware文件夹复制到项目里。其余新建文件夹并布置为如下结构：

```bash
.
├── CMakeLists.txt
├── Core
│   ├── Include
│   └── Source
├── Firmware
│   ├── CMSIS
│   ├── GD32F4xx_standard_peripheral
│   └── GD32F4xx_usb_library
└── Hardware
    ├── Include
    └── Source
```

接下来需要把示例的`main.c`, `systick.c`, `gd32f4xx_it.c`等source文件放在Core里面。Hardware则用于放置自己的驱动。


### CMake

CMake的核心是`CMakeLists.txt`。如下编写：

```cmake
cmake_minimum_required(VERSION 3.22.0)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)

project(LSPi_GCC_Template)

enable_language(C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 设置二进制执行文件的生成位置
# set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/Bin/${CMAKE_BUILD_TYPE}/)

# 开启链接时优化
cmake_policy(SET CMP0069 NEW)
option(USE_LTO "Enable LTO" OFF)

if(USE_LTO)
        include(CheckIPOSupported)
        check_ipo_supported(RESULT supported OUTPUT error)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

get_property(isMultiConfig GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)

if(isMultiConfig)
        set(CMAKE_CROSS_CONFIGS "Release;Debug")
        set(CMAKE_DEFAULT_BUILD_TYPE "Debug")
        set(CMAKE_DEFAULT_CONFIGS "Release;Debug")
endif()

# 彩色日志输出
set(CMAKE_COLOR_DIAGNOSTICS ON)

option(FORCE_COLORED_OUTPUT "Always produce ANSI-colored output (GNU/Clang only)." TRUE)

if(${FORCE_COLORED_OUTPUT})
        if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
                add_compile_options(-fdiagnostics-color=always)
        elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
                add_compile_options(-fcolor-diagnostics)
        endif()
endif()

# 添加全局define
add_definitions(
        -DGD32F470
)

# 添加编译指令
add_compile_options(
        "$<$<CONFIG:Debug>:-Og;-DDEBUG;-g;-funwind-tables>"
        "$<$<CONFIG:Release>:-O3;-DNDEBUG>"
        "$<$<CONFIG:MinSizeRel>:-Os;-DNDEBUG>"
        "$<$<CONFIG:RelWithDebInfo>:-O2;-g;-DNDEBUG>"
)

# 添加链接指令
add_link_options(
)

aux_source_directory(Core/Source CORE_SOURCES)
aux_source_directory(Hardware/Source HARDWARE_SOURCES)
aux_source_directory(Firmware/GD32F4xx_standard_peripheral/Source FIRMWARE_SOURCES)
aux_source_directory(Firmware/CMSIS/GD/GD32F4xx/Source CMSIS_SOURCES)
aux_source_directory(Firmware/CMSIS/GD/GD32F4xx/Source/GCC/newlib NEWLIB_SOURCES)

set(STARTUP_SOURCE
        Firmware/CMSIS/GD/GD32F4xx/Source/GCC/startup_gd32f450_470.S
)

add_executable(${PROJECT_NAME}
        ${CORE_SOURCES}
        ${HARDWARE_SOURCES}
        ${FIRMWARE_SOURCES}
        ${CMSIS_SOURCES}
        ${STARTUP_SOURCE}
)

target_include_directories(${PROJECT_NAME} PRIVATE
        Core/Include
        Hardware/Include
        Firmware/GD32F4xx_standard_peripheral/Include
        Firmware/CMSIS
        Firmware/CMSIS/GD/GD32F4xx/Include
)

target_link_options(${PROJECT_NAME} PRIVATE
        -T${CMAKE_CURRENT_SOURCE_DIR}/Firmware/CMSIS/GD/GD32F4xx/Source/GCC/Ld/gd32f470xG_flash.ld
        -Wl,-Map=${PROJECT_NAME}.map
        -Wl,--print-memory-usage
)

add_custom_target(flash
        DEPENDS ${PROJECT_NAME} # 确保烧录前先编译
        COMMAND pyocd load -t gd32f470zg --format elf ${CMAKE_PROJECT_NAME}.elf
        COMMENT "Flashing ${PROJECT_NAME} to GD32F470 and resetting"
)

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.elf
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}> $<TARGET_FILE_NAME:${PROJECT_NAME}>.bin
        COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}> $<TARGET_FILE_NAME:${PROJECT_NAME}>.hex
        COMMENT "Generating binary and hex files"
)

```

该`CMakeLists.txt`从STM32Cube的配置修改而来。具体CMake的修改由来

{{< details title="详细步骤" closed="true" >}}

### 第 0 步：最简骨架（版本 + 项目 + 空目标）

```cmake
cmake_minimum_required(VERSION 3.22.0)
project(LSPi_GCC_Template)
add_executable(${PROJECT_NAME})
```

这是任何 CMake 工程的起点。  
- `cmake_minimum_required` 指定最低版本。  
- `project` 定义项目名，默认启用 C 和 C++。  
- `add_executable` 创建可执行目标（先空壳，后续添加源文件）。

### 第 1 步：添加交叉编译工具链和硬件标志

```cmake
cmake_minimum_required(VERSION 3.22.0)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard" CACHE STRING "" FORCE)

project(LSPi_GCC_Template)
add_executable(${PROJECT_NAME})
```

新增内容：  
- 声明系统为裸机（Generic），处理器 ARM。  
- 指定编译器前缀（`arm-none-eabi-`），并设置 `CMAKE_TRY_COMPILE_TARGET_TYPE` 避免测试编译失败。  
- 缓存硬件编译标志（内核、thumb、浮点 ABI），并强制覆盖，确保链接也使用相同选项。

### 第 2 步：语言标准、构建类型选项和输出美化

```cmake
enable_language(C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_COLOR_DIAGNOSTICS ON)

option(FORCE_COLORED_OUTPUT "Always produce ANSI-colored output" TRUE)
if(FORCE_COLORED_OUTPUT)
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        add_compile_options(-fdiagnostics-color=always)
    elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        add_compile_options(-fcolor-diagnostics)
    endif()
endif()

add_compile_options(
    "$<$<CONFIG:Debug>:-Og;-DDEBUG;-g;-funwind-tables>"
    "$<$<CONFIG:Release>:-O3;-DNDEBUG>"
    "$<$<CONFIG:MinSizeRel>:-Os;-DNDEBUG>"
    "$<$<CONFIG:RelWithDebInfo>:-O2;-g;-DNDEBUG>"
)

add_definitions(-DGD32F470)
```

新增内容：  
- `enable_language` 显式启用 C、C++、汇编。  
- 强制使用 C11/C++11，禁止 GNU 扩展。  
- 打开编译命令导出（`compile_commands.json`）和彩色诊断。  
- 根据编译器类型添加彩色输出参数。  
- 使用生成器表达式 `$<$<CONFIG:Debug>:...>` 为不同构建类型添加差异化编译选项（优化、调试、宏）。

---

### 第 3 步：收集源文件、头文件路径和全局宏定义

```cmake
add_definitions(-DGD32F470)

aux_source_directory(Core/Source CORE_SOURCES)
aux_source_directory(Hardware/Source HARDWARE_SOURCES)
aux_source_directory(Firmware/GD32F4xx_standard_peripheral/Source FIRMWARE_SOURCES)
aux_source_directory(Firmware/CMSIS/GD/GD32F4xx/Source CMSIS_SOURCES)
aux_source_directory(Firmware/CMSIS/GD/GD32F4xx/Source/GCC/newlib NEWLIB_SOURCES)

set(STARTUP_SOURCE Firmware/CMSIS/GD/GD32F4xx/Source/GCC/startup_gd32f450_470.S)

add_executable(${PROJECT_NAME}
    ${CORE_SOURCES}
    ${HARDWARE_SOURCES}
    ${FIRMWARE_SOURCES}
    ${CMSIS_SOURCES}
    ${STARTUP_SOURCE}
)

target_include_directories(${PROJECT_NAME} PRIVATE
    Core/Include
    Hardware/Include
    Firmware/GD32F4xx_standard_peripheral/Include
    Firmware/CMSIS
    Firmware/CMSIS/GD/GD32F4xx/Include
)
```

新增内容：  
- `add_definitions(-DGD32F470)` 添加全局宏。  
- `aux_source_directory` 收集各目录下的源文件（非递归），区别于 `GLOB_RECURSE`。  
- 显式列出启动汇编文件。  
- 在 `add_executable` 中一次性指定所有源文件。  
- `target_include_directories` 添加头文件搜索路径。

---

### 第 4 步：链接脚本、链接选项和 LTO 支持

```cmake
cmake_policy(SET CMP0069 NEW)
option(USE_LTO "Enable LTO" OFF)
if(USE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT supported OUTPUT error)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

target_include_directories(${PROJECT_NAME} PRIVATE
    Core/Include
    Hardware/Include
    Firmware/GD32F4xx_standard_peripheral/Include
    Firmware/CMSIS
    Firmware/CMSIS/GD/GD32F4xx/Include
)

target_link_options(${PROJECT_NAME} PRIVATE
    -T${CMAKE_CURRENT_SOURCE_DIR}/Firmware/CMSIS/GD/GD32F4xx/Source/GCC/Ld/gd32f470xG_flash.ld
    -Wl,-Map=${PROJECT_NAME}.map
    -Wl,--print-memory-usage
)
```

新增内容：  
- 通过 `cmake_policy` 和 `CheckIPOSupported` 提供可选的 LTO 开关。  
- `target_link_options` 指定链接脚本、生成 `.map` 文件、打印内存占用。

---

### 第 5 步：后处理（生成 hex/bin）和烧录自定义目标

```cmake
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.elf
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}> $<TARGET_FILE_NAME:${PROJECT_NAME}>.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}> $<TARGET_FILE_NAME:${PROJECT_NAME}>.hex
    COMMENT "Generating binary and hex files"
)

add_custom_target(flash
    DEPENDS ${PROJECT_NAME}
    COMMAND pyocd load -t gd32f470zg --format elf ${CMAKE_PROJECT_NAME}.elf
    COMMENT "Flashing ${PROJECT_NAME} to GD32F470 and resetting"
)
```

新增内容（最终步骤）：  
- `add_custom_command` 在链接后复制 `.elf` 并生成 `.bin` 和 `.hex`。  
- `add_custom_target` 定义 `flash` 目标，依赖主目标，调用 `pyocd` 烧录。  
- 使用生成器表达式 `$<TARGET_FILE:...>` 和 `$<TARGET_FILE_NAME:...>` 获取目标文件路径和名称。

{{< /details >}}

### 编译

进入build/debug或者build/release文件夹：

```bash
cd build/debug
ninja       #编译
ninja flash #烧录
```

### 调试

配置VSCode `launch.json`

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug GD32F4",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "pyocd",
            "serverpath": "path/to/pyocd",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/debug/LSPi_GCC_Template.elf", 
            "targetId": "gd32f470zg",                                 
            "runToEntryPoint": "main",                                
            "svdFile": "${workspaceFolder}/GD32F4xx.svd",            
            "interface": "swd",                                        
            "serverArgs": ["--persist"],
            "showDevDebugOutput": "raw",    
            "overrideGDBServerStartedRegex": "(started|Listening) on port"
        }
    ]
}
```

`GD32F4xx.svd`文件可以从pyocd下载得到的pack文件里面解包出来。pyocd直接读取会报错，需要删除svd首行多出来的空格。

### 点灯测试

```c
#include "gd32f4xx.h"
#include "systick.h"
#include <stdio.h>
#include "main.h"

/*!
    \brief    main function
    \param[in]  none
    \param[out] none
    \retval     none
*/
int main(void)
{
    systick_config();
    rcu_periph_clock_enable(RCU_GPIOE);
    gpio_mode_set(GPIOE, GPIO_MODE_OUTPUT, GPIO_PUPD_NONE, GPIO_PIN_3);
    gpio_output_options_set(GPIOE, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_3);
    while(1) {
        gpio_bit_write(GPIOE,GPIO_PIN_3,1);
        delay_1ms(1000);
        gpio_bit_write(GPIOE,GPIO_PIN_3,0);
        delay_1ms(1000);
    }
}
```

修改`main.c`，输入ninja flash即可看到实验效果LED1 闪烁。
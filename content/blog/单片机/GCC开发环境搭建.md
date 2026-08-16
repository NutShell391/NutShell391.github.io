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

接下来需要把示例的`main.c`, `systick.c`等source文件放在Core里面。Hardware则用于放置自己的驱动。





CMake的核心是`CMakeLists.txt`。如下编写：

```CMake
cmake_minimum_required(VERSION 3.22)

# 1. 设置项目名称
set(CMAKE_PROJECT_NAME Demo_Template)
project(${CMAKE_PROJECT_NAME} C ASM)

# 启用 clangd 编译命令导出，方便代码补全和跳转
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)
set(CMAKE_C_STANDARD 11)

# 2. 【关键】在此处提前定义目标
# 只有先执行了这一行，后续的 target_include_directories 才能找到挂载对象
add_executable(${CMAKE_PROJECT_NAME})

# 3. 宏定义 (GD32F470 核心)
target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE
        GD32F470
        USE_STDPERIPH_DRIVER
)

# 4. 包含目录 (Include Paths)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        "Core/Include"
        "Firmware/CMSIS"
        "Firmware/CMSIS/Include"
        "Firmware/CMSIS/GD/GD32F4xx/Include"
        "Firmware/GD32F4xx_standard_peripheral/Include"
        ## "Firmware/GD32F4xx_usb_library/device/Include"
        ## "Firmware/Third_Party" ## USB和第三方库
)

# 5. 源文件搜寻
file(GLOB_RECURSE CORE_SOURCES "Core/Source/*.c")
file(GLOB_RECURSE HARDWARE_SOURCES "Hardware/Source/*.c")
file(GLOB_RECURSE PERIPH_SOURCES "Firmware/GD32F4xx_standard_peripheral/Source/*.c")

# 系统初始化与 GCC 版本的启动文件
set(STARTUP_SYSTEM_SOURCES
        "Firmware/CMSIS/GD/GD32F4xx/Source/system_gd32f4xx.c"
        "Firmware/CMSIS/GD/GD32F4xx/Source/GCC/startup_gd32f450_470.S"
)
# 将搜寻到的源文件添加到目标
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
        ${CORE_SOURCES}
        ${PERIPH_SOURCES}
        ${HARDWARE_SOURCES}
        ${STARTUP_SYSTEM_SOURCES}
)

# 6. 链接设置
# 挂载我们自定义的 GD32F470 链接脚本
set(LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/Firmware/CMSIS/GD/GD32F4xx/Source/GCC/Ld/gd32f470xG_flash.ld")


target_link_options(${CMAKE_PROJECT_NAME} PRIVATE
        -T${LINKER_SCRIPT}
        -Wl,-Map=${CMAKE_PROJECT_NAME}.map
        -Wl,--gc-sections
)

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE
        m
        c
        nosys
)

# 7. 编译后操作：生成 Hex 和 Bin 烧录文件
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.hex
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
)
```

该`CMakeLists.txt`从STM32Cube的配置修改而来。具体CMake的修改由来

{{< details title="详细步骤" closed="true" >}}

默认情况下，这部分内容会被隐藏。

#### Step 1: 骨架

任何 CMake 需要这三行。此时仅定义环境。

```cmake
cmake_minimum_required(VERSION 3.22)
project(Demo_Template C)          # 默认只开启 C 语言
add_executable(Demo_Template)     # 先创建一个空壳目标
```

`cmake_minimum_required`（版本约束）、`project`（定义项目名和语言）、`add_executable`（生成可执行目标）。

#### Step2：适配嵌入式

添加启动文件是 `.S` 汇编，加上 clangd。

```cmake
cmake_minimum_required(VERSION 3.22)
project(Demo_Template C ASM)               # 开启汇编支持
set(CMAKE_C_STANDARD 11)               # 指定 C11 标准
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)# 生成 compile_commands.json
add_executable(Demo_Template)
```

`project` 中的语言列表、`set` 修改变量、`CMAKE_EXPORT_COMPILE_COMMANDS` 用于 IDE 代码补全。

---

#### Step3：添加源文件

嵌入式需要大量源文件，用 `GLOB_RECURSE` 自动递归搜索，但启动文件和系统初始化文件必须显式列出。

```cmake
file(GLOB_RECURSE CORE_SOURCES "Core/Source/*.c")
file(GLOB_RECURSE HARDWARE_SOURCES "Hardware/Source/*.c")
file(GLOB_RECURSE PERIPH_SOURCES "Firmware/GD32F4xx_standard_peripheral/Source/*.c")

# 此处自定义 STARTUP_SOURCES 变量
set(STARTUP_SOURCES 
    "Firmware/CMSIS/GD/GD32F4xx/Source/system_gd32f4xx.c"
    "Firmware/CMSIS/GD/GD32F4xx/Source/GCC/startup_gd32f450_470.S"
)

# 把源文件挂载到目标上
target_sources(Demo_Template PRIVATE
    ${CORE_SOURCES}
    ${PERIPH_SOURCES}
    ${HARDWARE_SOURCES}
    ${STARTUP_SOURCES}
)
```

`file(GLOB_RECURSE)`（递归匹配文件）、`set()` 组合列表、`target_sources`（为目标追加源文件）。  
> 注意：`GLOB` 不会自动检测新文件，正式项目推荐显式列出，此处仅为简化。

#### Step4：头文件路径

标准库和 CMSIS 的头文件散落各处，必须用 `target_include_directories` 告诉编译器。

```cmake
# 头文件搜索路径
target_include_directories(Demo_Template PRIVATE
    "Core/Include"
    "Firmware/CMSIS"
    "Firmware/CMSIS/Include"
    "Firmware/CMSIS/GD/GD32F4xx/Include"
    "Firmware/GD32F4xx_standard_peripheral/Include"
)
```

`target_include_directories`（相当于编译器的 `-I` 参数），`PRIVATE` 表示仅本目标使用。

---

#### Step5：宏定义

`GD32F470` 宏告知标准库应该启动GD32F470寄存器映射

```cmake
# 编译宏定义
target_compile_definitions(Demo_Template PRIVATE
    GD32F470
    USE_STDPERIPH_DRIVER
)
```
`target_compile_definitions`（相当于 `-D` 参数），用于条件编译。

---

#### Step6：链接脚本与链接选项

核心的一步：指定 `.ld` 文件，生成 `.map` 映射文件，并开启 `--gc-sections` 剔除未用函数。

```cmake
# 设定链接脚本路径
set(LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/Firmware/CMSIS/GD/GD32F4xx/Source/GCC/Ld/gd32f470xG_flash.ld")

# 链接选项
target_link_options(Demo_Template PRIVATE
    -T${LINKER_SCRIPT}         # 指定 ld 脚本
    -Wl,-Map=${PROJECT_NAME}.map  # 生成 map 文件
    -Wl,--gc-sections          # 垃圾回收未用段
)

# 链接基础库（数学、C库、无主机系统）
target_link_libraries(Demo_Template PRIVATE
    m        # 数学库
    c        # C 标准库
    nosys    # 裸机 syscall 空实现
)
```
`target_link_options`（传递链接器原生参数）、`CMAKE_CURRENT_SOURCE_DIR`（当前目录路径）、`target_link_libraries`（链接外部库）。`-Wl,` 是 GCC 向链接器传递参数的语法。

---

### Step7：后处理命令

编译出的 `.elf` 调试器能用，但烧录器通常要 `.hex` 或 `.bin`，用 `objcopy` 转换。

```cmake
# 构建完成后自动生成烧录文件
add_custom_command(TARGET Demo_Template POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:Demo_Template> ${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:Demo_Template> ${PROJECT_NAME}.bin
)
```
`add_custom_command`（自定义构建步骤）、`POST_BUILD`（链接后执行）、`${CMAKE_OBJCOPY}`（工具链内置变量）、`$<TARGET_FILE:...>`（生成器表达式，表示目标文件的完整路径）。

{{< /details >}}

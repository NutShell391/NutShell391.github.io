--- 
title: 'Vim 的基本用法' 
date: "2026-08-16T14:30:00+08:00" 
draft: false 
--- 

参考内容：[【【保姆级入门】Vim编辑器】 ](https://www.bilibili.com/video/BV13t4y1t7Wg/?share_source=copy_web&vd_source=1df2d4fcb5fa4309108faa7ad011f7e8)

为了熟练vim，本博客也使用vim进行编写。

个人觉得vim不用离开键盘的操作还是比较便捷的。可以基于一个例子来逐步学习基本操作。假设我们需要写一个GD32的循环点灯程序。

## 打开和新建文件
新建文件很简单，直接使用 `vim 文件名`即可。
如果要打开或者新建多个文件，则可以输入多个文件名

这里则是输入:

```C
vim main.c
```

## Normal和Insert模式

正常模式下是输入指令执行操作，如果想要正常打字，则输入I，即可切换为Insert模式。

Normal模式下，vim 中上下左右是用hjkl进行控制，h,l分别在最左和最右边，控制左右；而j,k控制上下，k上j下。当然也可以用方向键进行控制。

可以下载vscode里面的learn vim插件，里面有走迷宫的小游戏，可以很好熟练这样的光标操作方式。

这里我们可以直接手动输入：

```C
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

## 复制和粘贴

Normal模式下，可以直接输入yy，即可复制整行。输入p可以粘贴。比如说我们需要复制 `gpio_bit_write` 操作，可以直接yy之后移动到对应行数输入p粘贴。

y这里代表yank复制，如果是y就是选定范围复制，通常和后面说到的visual模式配合，而yw则是复制一个单词。从这里可以看出vim指令的一些特点，很类似鼠标单击则是光标定位，双击则是选中单词，而三次连击可以选中一整行。通常是操作符+范围这样的组合。
p代表paste，如果是数字+p则意味着执行多次粘贴操作，非常好用。

## 行号移动

使用hjkl还是很慢，通常会使用行号或者是相对行号进行跳转。命令很简单，就是数字+方向键，此时的数字是相对位置。2j代表向下2行，5k代表向上5行。设置相对行号可以打开 .vimrc文件，增加 `set relativenumber` 即可。

另外输入shift+g可以跳跃到整个文件最后一行，gg则是反操作回到首行。w、b是按照word向左右移动，W,B则是WORD，关于word和WORD，可以参考learn vim插件的解释。 同样还有e，ge，则是word末尾左右移动，相对比较少用。

## 撤销和重做

u是撤销一次，ctrl+r则是恢复上一次操作。英文句号`.`按键则是重复上一次操作。

## 修改字符

dd 代表删除一整行， dw 是删除一个word，cw 是原位改变一个word，也就是删除z之后原位编辑，较为常用

ci表示change in，此时ci可以直接删除{}, [], ()里面的内容，只需要ci{ 或者ci[ 或者 ci( 。

## 搜索和替换

vim里面搜索很简单，直接输入正斜杠 / +需要搜索的内容，回车就可以定位位置。如果只有一个就会把光标移动到对应位置，如果有多个，按n则会向下跳转，N会向上。 另外还有一个搜索指令 ?, 代表向后搜索，此时n、N的擦左就需要反过来。

替换则类似函数的方式进行替换，:[范围]s/旧内容/新内容/[选项] ，例如单独替换一个单词 `:%s/old/new`，如果是全局替换，`:%s/old/new/g`, %是整个文件范围的意思，而g则是global全局替换选项。`:%s/old/new/gi` 则是忽略大小写全局替换，gc则是带确认的替换。
范围的理解是这样的，如果是%则是整个文件。5,10则是5-10行，.,$则是当前到最后一行。

## Visual模式

visual模式的存在类似与鼠标拖动的效果，不过更为强大。
按下v，进入普通视觉模式，此时移动光标可以按照文本流的方式选中范围，此时执行操作则是针对选中的文本执行。
ctrl+v则是visual block，此时选中则是按照block的方式理解，也就是划定文本方块，在修改重复模式的时候相当好用。
shift+v则是visual line，此时jk上下移动就选中一整行。

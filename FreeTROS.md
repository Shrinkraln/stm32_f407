## 1.1 概述

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_mainmap.jpg)

1. 调度器算法
   1. 带时间片的抢占
   2. 不带时间片的抢占
   3. 协作式
2. 任务
   1. 调度器启动时创建 vTaskStartScheduler()时
      1. idle：CPU一直在运转，但是没有可执行代码时执行
      2. 定时器任务：和软件定时器有关，需要启用软件定时器；如创建软件定时器，定期执行回调函数，和创建任务周期延迟的效果是一样的
   2. 用户创建的任务
3. 中断驱动调度器
   1. SysTick：产生时间片用于任务调度的时间单位，和系统节拍；通过时钟树配置cortex system timer即systick时钟（不是溢出产生中断的频率）；通过配置timebase source 选择hal库的hal_delay()等函数的时钟源是systick还是timx；通过HCLK配置内核时钟频率；通过configTICJ_RATE_HZ宏定义修改systick溢出产生中断的频率；SysTick 中断 -> xPortSysTickHandler() -> xTaskIncrementTick() -> xTickCount++->如果有任务可执行PendSV，如果超过位宽configTICK_TYPE_WITH_IN_WIDTH会回绕，硬件systick和软件tick的区别
   2. SVC：启动第一个任务
   3. PendSV：后续的任务切换的执行
4. 内核对象：信号量等同步量，任务，软件定时器，缓冲区
5. 堆内存管理方案
   1. 一个静态数组，指针递增，不回收
   2. 一个静态数组，最佳适配
   3. malloc和free：不可重入，执行慢，碎片化
   4. 一个静态数据，最快适配
   5. 多个静态数组，最快适配

## 1.2 文件结构

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_dir.jpg)

![FreeRTOS_kernel](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_kernel.jpg)

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_inc.jpg)

1. 总体包含：.c文件 Inc/文件 portable/文件 portable/Memang/文件 example/template_configuration/FreeRTOSConfig.h文件
2. portable文件夹负责适配各种单片机内核和工具链；先是工具链，其后是内核；
3. portable文件下包含Memang是堆内存管理文件
4. FreeRTOSConfig.h包含在FreeRTOS.h文件中包含

## 1.3 配置文件

```c
//HCL to AHB 是内核的频率
#define configCPU_CLOCK_HZ    ( ( unsigned long ) 20000000 )
//设置TickType_t数据类型的位宽，即软件计数值tick的数据类型；
//位宽通常和CPU位数一样；但是8bit是16bit_width
#define configTICK_TYPE_WIDTH_IN_BITS              TICK_TYPE_WIDTH_32_BITS
//配置PendSV和SysTick中断优先级；NVIC最高优先级是0；
#define configKERNEL_INTERRUPT_PRIORITY          15<<4
//能调用其API的最高优先级，即在比5优先级低的中断5-15里面可以调用API；在高优先级的中断里面运行FreeRTOS的内核会影响中断的实时性
#define configMAX_SYSCALL_INTERRUPT_PRIORITY     5<<4
//配置是否使用时间片
#define configUSE_TIME_SLICING                     0
//系统节拍长度，及时间片长度，而pdMS_TO_TICK就是转化为tick数，所以时间长度一定要是tick单位时间的整数倍
#define configTICK_RATE_HZ                         100
```

```c
/*
 * FreeRTOS Kernel V11.3.0
 * Copyright (C) 2021 Amazon.com, Inc. or its affiliates. All Rights Reserved.
 *
 * SPDX-License-Identifier: MIT
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
 * SOFTWARE.
 *
 * https://www.FreeRTOS.org
 * https://github.com/FreeRTOS
 *
 */

/*******************************************************************************
 * This file provides an example FreeRTOSConfig.h header file, inclusive of an
 * abbreviated explanation of each configuration item.  Online and reference
 * documentation provides more information.
 * https://www.freertos.org/a00110.html
 *
 * Constant values enclosed in square brackets ('[' and ']') must be completed
 * before this file will build.
 *
 * Use the FreeRTOSConfig.h supplied with the RTOS port in use rather than this
 * generic file, if one is available.
 ******************************************************************************/

#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

/******************************************************************************/
/* Hardware description related definitions. **********************************/
/******************************************************************************/

/* In most cases, configCPU_CLOCK_HZ must be set to the frequency of the clock
 * that drives the peripheral used to generate the kernels periodic tick
 * interrupt. The default value is set to 20MHz and matches the QEMU demo
 * settings.  Your application will certainly need a different value so set this
 * correctly. This is very often, but not always, equal to the main system clock
 * frequency. */
//HCL to AHB 是内核的频率
#define configCPU_CLOCK_HZ    ( ( unsigned long ) 20000000 )

/* configSYSTICK_CLOCK_HZ is an optional parameter for ARM Cortex-M ports only.
 *
 * By default ARM Cortex-M ports generate the RTOS tick interrupt from the
 * Cortex-M SysTick timer. Most Cortex-M MCUs run the SysTick timer at the same
 * frequency as the MCU itself - when that is the case configSYSTICK_CLOCK_HZ is
 * not needed and should be left undefined. If the SysTick timer is clocked at a
 * different frequency to the MCU core then set configCPU_CLOCK_HZ to the MCU
 * clock frequency, as normal, and configSYSTICK_CLOCK_HZ to the SysTick clock
 * frequency.  Not used if left undefined.
 * The default value is undefined (commented out).  If you need this value bring
 * it back and set it to a suitable value. */

/*
 #define configSYSTICK_CLOCK_HZ                  [Platform specific]
 */

/******************************************************************************/
/* Scheduling behaviour related definitions. **********************************/
/******************************************************************************/

/* configTICK_RATE_HZ sets frequency of the tick interrupt in Hz, normally
 * calculated from the configCPU_CLOCK_HZ value. */
#define configTICK_RATE_HZ                         100

/* Set configUSE_PREEMPTION to 1 to use pre-emptive scheduling.  Set
 * configUSE_PREEMPTION to 0 to use co-operative scheduling.
 * See https://www.freertos.org/single-core-amp-smp-rtos-scheduling.html. */
#define configUSE_PREEMPTION                       1

/* Set configUSE_TIME_SLICING to 1 to have the scheduler switch between Ready
 * state tasks of equal priority on every tick interrupt.  Set
 * configUSE_TIME_SLICING to 0 to prevent the scheduler switching between Ready
 * state tasks just because there was a tick interrupt.  See
 * https://freertos.org/single-core-amp-smp-rtos-scheduling.html. */
#define configUSE_TIME_SLICING                     0

/* Set configUSE_PORT_OPTIMISED_TASK_SELECTION to 1 to select the next task to
 * run using an algorithm optimised to the instruction set of the target
 * hardware - normally using a count leading zeros assembly instruction.  Set to
 * 0 to select the next task to run using a generic C algorithm that works for
 * all FreeRTOS ports.  Not all FreeRTOS ports have this option.  Defaults to 0
 * if left undefined. */
#define configUSE_PORT_OPTIMISED_TASK_SELECTION    0

/* Set configUSE_TICKLESS_IDLE to 1 to use the low power tickless mode.  Set to
 * 0 to keep the tick interrupt running at all times.  Not all FreeRTOS ports
 * support tickless mode. See
 * https://www.freertos.org/low-power-tickless-rtos.html Defaults to 0 if left
 * undefined. */
#define configUSE_TICKLESS_IDLE                    0

/* configMAX_PRIORITIES Sets the number of available task priorities.  Tasks can
 * be assigned priorities of 0 to (configMAX_PRIORITIES - 1).  Zero is the
 * lowest priority. */
#define configMAX_PRIORITIES                       5

/* configMINIMAL_STACK_SIZE defines the size of the stack used by the Idle task
 * (in words, not in bytes!).  The kernel does not use this constant for any
 * other purpose.  Demo applications use the constant to make the demos somewhat
 * portable across hardware architectures. */
#define configMINIMAL_STACK_SIZE                   128

/* configMAX_TASK_NAME_LEN sets the maximum length (in characters) of a task's
 * human readable name.  Includes the NULL terminator. */
#define configMAX_TASK_NAME_LEN                    16

/* Time is measured in 'ticks' - which is the number of times the tick interrupt
 * has executed since the RTOS kernel was started.
 * The tick count is held in a variable of type TickType_t.
 *
 * configTICK_TYPE_WIDTH_IN_BITS controls the type (and therefore bit-width) of
 * TickType_t:
 *
 * Defining configTICK_TYPE_WIDTH_IN_BITS as TICK_TYPE_WIDTH_16_BITS causes
 * TickType_t to be defined (typedef'ed) as an unsigned 16-bit type.
 *
 * Defining configTICK_TYPE_WIDTH_IN_BITS as TICK_TYPE_WIDTH_32_BITS causes
 * TickType_t to be defined (typedef'ed) as an unsigned 32-bit type.
 *
 * Defining configTICK_TYPE_WIDTH_IN_BITS as TICK_TYPE_WIDTH_64_BITS causes
 * TickType_t to be defined (typedef'ed) as an unsigned 64-bit type. */
#define configTICK_TYPE_WIDTH_IN_BITS              TICK_TYPE_WIDTH_64_BITS

/* Set configIDLE_SHOULD_YIELD to 1 to have the Idle task yield to an
 * application task if there is an Idle priority (priority 0) application task
 * that can run.  Set to 0 to have the Idle task use all of its timeslice.
 * Default to 1 if left undefined. */
#define configIDLE_SHOULD_YIELD                    1

/* Each task has an array of task notifications.
 * configTASK_NOTIFICATION_ARRAY_ENTRIES sets the number of indexes in the
 * array. See https://www.freertos.org/RTOS-task-notifications.html  Defaults to
 * 1 if left undefined. */
#define configTASK_NOTIFICATION_ARRAY_ENTRIES      1

/* configQUEUE_REGISTRY_SIZE sets the maximum number of queues and semaphores
 * that can be referenced from the queue registry.  Only required when using a
 * kernel aware debugger.  Defaults to 0 if left undefined. */
#define configQUEUE_REGISTRY_SIZE                  0

/* Set configENABLE_BACKWARD_COMPATIBILITY to 1 to map function names and
 * datatypes from old version of FreeRTOS to their latest equivalent.  Defaults
 * to 1 if left undefined. */
#define configENABLE_BACKWARD_COMPATIBILITY        0

/* Each task has its own array of pointers that can be used as thread local
 * storage.  configNUM_THREAD_LOCAL_STORAGE_POINTERS set the number of indexes
 * in the array.  See
 * https://www.freertos.org/thread-local-storage-pointers.html Defaults to 0 if
 * left undefined. */
#define configNUM_THREAD_LOCAL_STORAGE_POINTERS    0

/* When configUSE_MINI_LIST_ITEM is set to 0, MiniListItem_t and ListItem_t are
 * both the same. When configUSE_MINI_LIST_ITEM is set to 1, MiniListItem_t
 * contains 3 fewer fields than ListItem_t which saves some RAM at the cost of
 * violating strict aliasing rules which some compilers depend on for
 * optimization. Defaults to 1 if left undefined. */
#define configUSE_MINI_LIST_ITEM                   1

/* Sets the type used by the parameter to xTaskCreate() that specifies the stack
 * size of the task being created.  The same type is used to return information
 * about stack usage in various other API calls.  Defaults to size_t if left
 * undefined. */
#define configSTACK_DEPTH_TYPE                     size_t

/* configMESSAGE_BUFFER_LENGTH_TYPE sets the type used to store the length of
 * each message written to a FreeRTOS message buffer (the length is also written
 * to the message buffer.  Defaults to size_t if left undefined - but that may
 * waste space if messages never go above a length that could be held in a
 * uint8_t. */
#define configMESSAGE_BUFFER_LENGTH_TYPE           size_t

/* If configHEAP_CLEAR_MEMORY_ON_FREE is set to 1, then blocks of memory
 * allocated using pvPortMalloc() will be cleared (i.e. set to zero) when freed
 * using vPortFree(). Defaults to 0 if left undefined. */
#define configHEAP_CLEAR_MEMORY_ON_FREE            1

/* vTaskList and vTaskGetRunTimeStats APIs take a buffer as a parameter and
 * assume that the length of the buffer is configSTATS_BUFFER_MAX_LENGTH.
 * Defaults to 0xFFFF if left undefined. New applications are recommended to use
 * vTaskListTasks and vTaskGetRunTimeStatistics APIs instead and supply the
 * length of the buffer explicitly to avoid memory corruption. */
#define configSTATS_BUFFER_MAX_LENGTH              0xFFFF

/* Set configUSE_NEWLIB_REENTRANT to 1 to have a newlib reent structure
 * allocated for each task.  Set to 0 to not support newlib reent structures.
 * Default to 0 if left undefined.
 *
 * Note Newlib support has been included by popular demand, but is not used or
 * tested by the FreeRTOS maintainers themselves. FreeRTOS is not responsible
 * for resulting newlib operation. User must be familiar with newlib and must
 * provide system-wide implementations of the necessary stubs. Note that (at the
 * time of writing) the current newlib design implements a system-wide malloc()
 * that must be provided with locks. */
#define configUSE_NEWLIB_REENTRANT                 0

/******************************************************************************/
/* Software timer related definitions. ****************************************/
/******************************************************************************/

/* Set configUSE_TIMERS to 1 to include software timer functionality in the
 * build.  Set to 0 to exclude software timer functionality from the build.  The
 * FreeRTOS/source/timers.c source file must be included in the build if
 * configUSE_TIMERS is set to 1.  Default to 0 if left undefined.  See
 * https://www.freertos.org/RTOS-software-timer.html. */
#define configUSE_TIMERS                1

/* configTIMER_TASK_PRIORITY sets the priority used by the timer task.  Only
 * used if configUSE_TIMERS is set to 1.  The timer task is a standard FreeRTOS
 * task, so its priority is set like any other task.  See
 * https://www.freertos.org/RTOS-software-timer-service-daemon-task.html  Only
 * used if configUSE_TIMERS is set to 1. */
#define configTIMER_TASK_PRIORITY       ( configMAX_PRIORITIES - 1 )

/* configTIMER_TASK_STACK_DEPTH sets the size of the stack allocated to the
 * timer task (in words, not in bytes!).  The timer task is a standard FreeRTOS
 * task.  See
 * https://www.freertos.org/RTOS-software-timer-service-daemon-task.html Only
 * used if configUSE_TIMERS is set to 1. */
#define configTIMER_TASK_STACK_DEPTH    configMINIMAL_STACK_SIZE

/* configTIMER_QUEUE_LENGTH sets the length of the queue (the number of discrete
 * items the queue can hold) used to send commands to the timer task.  See
 * https://www.freertos.org/RTOS-software-timer-service-daemon-task.html  Only
 * used if configUSE_TIMERS is set to 1. */
#define configTIMER_QUEUE_LENGTH        10

/******************************************************************************/
/* Event Group related definitions. *******************************************/
/******************************************************************************/

/* Set configUSE_EVENT_GROUPS to 1 to include event group functionality in the
 * build. Set to 0 to exclude event group functionality from the build. The
 * FreeRTOS/source/event_groups.c source file must be included in the build if
 * configUSE_EVENT_GROUPS is set to 1. Defaults to 1 if left undefined. */

#define configUSE_EVENT_GROUPS    1

/******************************************************************************/
/* Stream Buffer related definitions. *****************************************/
/******************************************************************************/

/* Set configUSE_STREAM_BUFFERS to 1 to include stream buffer functionality in
 * the build. Set to 0 to exclude event group functionality from the build. The
 * FreeRTOS/source/stream_buffer.c source file must be included in the build if
 * configUSE_STREAM_BUFFERS is set to 1. Defaults to 1 if left undefined. */

#define configUSE_STREAM_BUFFERS    1

/******************************************************************************/
/* Memory allocation related definitions. *************************************/
/******************************************************************************/

/* Set configSUPPORT_STATIC_ALLOCATION to 1 to include FreeRTOS API functions
 * that create FreeRTOS objects (tasks, queues, etc.) using statically allocated
 * memory in the build.  Set to 0 to exclude the ability to create statically
 * allocated objects from the build.  Defaults to 0 if left undefined.  See
 * https://www.freertos.org/Static_Vs_Dynamic_Memory_Allocation.html. */
#define configSUPPORT_STATIC_ALLOCATION              1

/* Set configSUPPORT_DYNAMIC_ALLOCATION to 1 to include FreeRTOS API functions
 * that create FreeRTOS objects (tasks, queues, etc.) using dynamically
 * allocated memory in the build.  Set to 0 to exclude the ability to create
 * dynamically allocated objects from the build.  Defaults to 1 if left
 * undefined.  See
 * https://www.freertos.org/Static_Vs_Dynamic_Memory_Allocation.html. */
#define configSUPPORT_DYNAMIC_ALLOCATION             1

/* Sets the total size of the FreeRTOS heap, in bytes, when heap_1.c, heap_2.c
 * or heap_4.c are included in the build.  This value is defaulted to 4096 bytes
 * but it must be tailored to each application.  Note the heap will appear in
 * the .bss section.  See https://www.freertos.org/a00111.html. */
#define configTOTAL_HEAP_SIZE                        4096

/* Set configAPPLICATION_ALLOCATED_HEAP to 1 to have the application allocate
 * the array used as the FreeRTOS heap.  Set to 0 to have the linker allocate
 * the array used as the FreeRTOS heap.  Defaults to 0 if left undefined. */
#define configAPPLICATION_ALLOCATED_HEAP             0

/* Set configSTACK_ALLOCATION_FROM_SEPARATE_HEAP to 1 to have task stacks
 * allocated from somewhere other than the FreeRTOS heap.  This is useful if you
 * want to ensure stacks are held in fast memory.  Set to 0 to have task stacks
 * come from the standard FreeRTOS heap.  The application writer must provide
 * implementations for pvPortMallocStack() and vPortFreeStack() if set to 1.
 * Defaults to 0 if left undefined. */
#define configSTACK_ALLOCATION_FROM_SEPARATE_HEAP    0

/* Set configENABLE_HEAP_PROTECTOR to 1 to enable bounds checking and
 * obfuscation to internal heap block pointers in heap_4.c and heap_5.c to help
 * catch pointer corruptions. Defaults to 0 if left undefined. */
#define configENABLE_HEAP_PROTECTOR                  0

/******************************************************************************/
/* Interrupt nesting behaviour configuration. *********************************/
/******************************************************************************/

/* configKERNEL_INTERRUPT_PRIORITY sets the priority of the tick and context
 * switch performing interrupts.  Not supported by all FreeRTOS ports.  See
 * https://www.freertos.org/RTOS-Cortex-M3-M4.html for information specific to
 * ARM Cortex-M devices. */
#define configKERNEL_INTERRUPT_PRIORITY          0

/* configMAX_SYSCALL_INTERRUPT_PRIORITY sets the interrupt priority above which
 * FreeRTOS API calls must not be made.  Interrupts above this priority are
 * never disabled, so never delayed by RTOS activity.  The default value is set
 * to the highest interrupt priority (0).  Not supported by all FreeRTOS ports.
 * See https://www.freertos.org/RTOS-Cortex-M3-M4.html for information specific
 * to ARM Cortex-M devices. */
#define configMAX_SYSCALL_INTERRUPT_PRIORITY     0

/* Another name for configMAX_SYSCALL_INTERRUPT_PRIORITY - the name used depends
 * on the FreeRTOS port. */
#define configMAX_API_CALL_INTERRUPT_PRIORITY    0

/******************************************************************************/
/* Hook and callback function related definitions. ****************************/
/******************************************************************************/

/* Set the following configUSE_* constants to 1 to include the named hook
 * functionality in the build.  Set to 0 to exclude the hook functionality from
 * the build.  The application writer is responsible for providing the hook
 * function for any set to 1.  See https://www.freertos.org/a00016.html. */
#define configUSE_IDLE_HOOK                   0
#define configUSE_TICK_HOOK                   0
#define configUSE_MALLOC_FAILED_HOOK          0
#define configUSE_DAEMON_TASK_STARTUP_HOOK    0

/* Set configUSE_SB_COMPLETED_CALLBACK to 1 to have send and receive completed
 * callbacks for each instance of a stream buffer or message buffer. When the
 * option is set to 1, APIs xStreamBufferCreateWithCallback() and
 * xStreamBufferCreateStaticWithCallback() (and likewise APIs for message
 * buffer) can be used to create a stream buffer or message buffer instance
 * with application provided callbacks. Defaults to 0 if left undefined. */
#define configUSE_SB_COMPLETED_CALLBACK       0

/* Set configCHECK_FOR_STACK_OVERFLOW to 1 or 2 for FreeRTOS to check for a
 * stack overflow at the time of a context switch.  Set to 0 to not look for a
 * stack overflow.  If configCHECK_FOR_STACK_OVERFLOW is 1 then the check only
 * looks for the stack pointer being out of bounds when a task's context is
 * saved to its stack - this is fast but somewhat ineffective.  If
 * configCHECK_FOR_STACK_OVERFLOW is 2 then the check looks for a pattern
 * written to the end of a task's stack having been overwritten.  This is
 * slower, but will catch most (but not all) stack overflows.  The application
 * writer must provide the stack overflow callback when
 * configCHECK_FOR_STACK_OVERFLOW is set to 1. See
 * https://www.freertos.org/Stacks-and-stack-overflow-checking.html  Defaults to
 * 0 if left undefined. */
#define configCHECK_FOR_STACK_OVERFLOW        2

/******************************************************************************/
/* Run time and task stats gathering related definitions. *********************/
/******************************************************************************/

/* Set configGENERATE_RUN_TIME_STATS to 1 to have FreeRTOS collect data on the
 * processing time used by each task.  Set to 0 to not collect the data.  The
 * application writer needs to provide a clock source if set to 1.  Defaults to
 * 0 if left undefined.  See https://www.freertos.org/rtos-run-time-stats.html.
 */
#define configGENERATE_RUN_TIME_STATS           0

/* Set configUSE_TRACE_FACILITY to include additional task structure members
 * are used by trace and visualisation functions and tools.  Set to 0 to exclude
 * the additional information from the structures. Defaults to 0 if left
 * undefined. */
#define configUSE_TRACE_FACILITY                0

/* Set to 1 to include the vTaskList() and vTaskGetRunTimeStats() functions in
 * the build.  Set to 0 to exclude these functions from the build.  These two
 * functions introduce a dependency on string formatting functions that would
 * otherwise not exist - hence they are kept separate.  Defaults to 0 if left
 * undefined. */
#define configUSE_STATS_FORMATTING_FUNCTIONS    0

/******************************************************************************/
/* Co-routine related definitions. ********************************************/
/******************************************************************************/

/* Set configUSE_CO_ROUTINES to 1 to include co-routine functionality in the
 * build, or 0 to omit co-routine functionality from the build. To include
 * co-routines, croutine.c must be included in the project. Defaults to 0 if
 * left undefined. */
#define configUSE_CO_ROUTINES              0

/* configMAX_CO_ROUTINE_PRIORITIES defines the number of priorities available
 * to the application co-routines. Any number of co-routines can share the same
 * priority. Defaults to 0 if left undefined. */
#define configMAX_CO_ROUTINE_PRIORITIES    1

/******************************************************************************/
/* Debugging assistance. ******************************************************/
/******************************************************************************/

/* configASSERT() has the same semantics as the standard C assert().  It can
 * either be defined to take an action when the assertion fails, or not defined
 * at all (i.e. comment out or delete the definitions) to completely remove
 * assertions.  configASSERT() can be defined to anything you want, for example
 * you can call a function if an assert fails that passes the filename and line
 * number of the failing assert (for example, "vAssertCalled( __FILE__, __LINE__
 * )" or it can simple disable interrupts and sit in a loop to halt all
 * execution on the failing line for viewing in a debugger. */

/* *INDENT-OFF* */
#define configASSERT( x )         \
    if( ( x ) == 0 )              \
    {                             \
        taskDISABLE_INTERRUPTS(); \
        for( ; ; )                \
        ;                         \
    }
/* *INDENT-ON* */

/******************************************************************************/
/* FreeRTOS MPU specific definitions. *****************************************/
/******************************************************************************/

/* If configINCLUDE_APPLICATION_DEFINED_PRIVILEGED_FUNCTIONS is set to 1 then
 * the application writer can provide functions that execute in privileged mode.
 * See:
 * https://www.freertos.org/a00110.html#configINCLUDE_APPLICATION_DEFINED_PRIVILEGED_FUNCTIONS
 * Defaults to 0 if left undefined.  Only used by the FreeRTOS Cortex-M MPU
 * ports, not the standard ARMv7-M Cortex-M port. */
#define configINCLUDE_APPLICATION_DEFINED_PRIVILEGED_FUNCTIONS    0

/* Set configTOTAL_MPU_REGIONS to the number of MPU regions implemented on your
 * target hardware.  Normally 8 or 16.  Only used by the FreeRTOS Cortex-M MPU
 * ports, not the standard ARMv7-M Cortex-M port.  Defaults to 8 if left
 * undefined. */
#define configTOTAL_MPU_REGIONS                                   8

/* configTEX_S_C_B_FLASH allows application writers to override the default
 * values for the for TEX, Shareable (S), Cacheable (C) and Bufferable (B) bits
 * for the MPU region covering Flash.  Defaults to 0x07UL (which means TEX=000,
 * S=1, C=1, B=1) if left undefined.  Only used by the FreeRTOS Cortex-M MPU
 * ports, not the standard ARMv7-M Cortex-M port. */
#define configTEX_S_C_B_FLASH                                     0x07UL

/* configTEX_S_C_B_SRAM allows application writers to override the default
 * values for the for TEX, Shareable (S), Cacheable (C) and Bufferable (B) bits
 * for the MPU region covering RAM. Defaults to 0x07UL (which means TEX=000,
 * S=1, C=1, B=1) if left undefined.  Only used by the FreeRTOS Cortex-M MPU
 * ports, not the standard ARMv7-M Cortex-M port. */
#define configTEX_S_C_B_SRAM                                      0x07UL

/* Set configENFORCE_SYSTEM_CALLS_FROM_KERNEL_ONLY to 0 to prevent any privilege
 * escalations originating from outside of the kernel code itself.  Set to 1 to
 * allow application tasks to raise privilege.  Defaults to 1 if left undefined.
 * Only used by the FreeRTOS Cortex-M MPU ports, not the standard ARMv7-M
 * Cortex-M port. */
#define configENFORCE_SYSTEM_CALLS_FROM_KERNEL_ONLY               1

/* Set configALLOW_UNPRIVILEGED_CRITICAL_SECTIONS to 1 to allow unprivileged
 * tasks enter critical sections (effectively mask interrupts). Set to 0 to
 * prevent unprivileged tasks entering critical sections.  Defaults to 1 if left
 * undefined.  Only used by the FreeRTOS Cortex-M MPU ports, not the standard
 * ARMv7-M Cortex-M port. */
#define configALLOW_UNPRIVILEGED_CRITICAL_SECTIONS                0

/* FreeRTOS Kernel version 10.6.0 introduced a new v2 MPU wrapper, namely
 * mpu_wrappers_v2.c. Set configUSE_MPU_WRAPPERS_V1 to 0 to use the new v2 MPU
 * wrapper. Set configUSE_MPU_WRAPPERS_V1 to 1 to use the old v1 MPU wrapper
 * (mpu_wrappers.c). Defaults to 0 if left undefined. */
#define configUSE_MPU_WRAPPERS_V1                                 0

/* When using the v2 MPU wrapper, set configPROTECTED_KERNEL_OBJECT_POOL_SIZE to
 * the total number of kernel objects, which includes tasks, queues, semaphores,
 * mutexes, event groups, timers, stream buffers and message buffers, in your
 * application. The application will not be able to have more than
 * configPROTECTED_KERNEL_OBJECT_POOL_SIZE kernel objects at any point of
 * time. */
#define configPROTECTED_KERNEL_OBJECT_POOL_SIZE                   10

/* When using the v2 MPU wrapper, set configSYSTEM_CALL_STACK_SIZE to the size
 * of the system call stack in words. Each task has a statically allocated
 * memory buffer of this size which is used as the stack to execute system
 * calls. For example, if configSYSTEM_CALL_STACK_SIZE is defined as 128 and
 * there are 10 tasks in the application, the total amount of memory used for
 * system call stacks is 128 * 10 = 1280 words. */
#define configSYSTEM_CALL_STACK_SIZE                              128

/* When using the v2 MPU wrapper, set configENABLE_ACCESS_CONTROL_LIST to 1 to
 * enable Access Control List (ACL) feature. When ACL is enabled, an
 * unprivileged task by default does not have access to any kernel object other
 * than itself. The application writer needs to explicitly grant the
 * unprivileged task access to the kernel objects it needs using the APIs
 * provided for the same. Defaults to 0 if left undefined. */
#define configENABLE_ACCESS_CONTROL_LIST                          1

/******************************************************************************/
/* SMP( Symmetric MultiProcessing ) Specific Configuration definitions. *******/
/******************************************************************************/

/* Set configNUMBER_OF_CORES to the number of available processor cores.
 * Defaults to 1 if left undefined. */

/*
 #define configNUMBER_OF_CORES                     [Num of available cores]
 */

/* When using SMP (i.e. configNUMBER_OF_CORES is greater than one), set
 * configRUN_MULTIPLE_PRIORITIES to 0 to allow multiple tasks to run
 * simultaneously only if they do not have equal priority, thereby maintaining
 * the paradigm of a lower priority task never running if a higher priority task
 * is able to run. If configRUN_MULTIPLE_PRIORITIES is set to 1, multiple tasks
 * with different priorities may run simultaneously - so a higher and lower
 * priority task may run on different cores at the same time. */
#define configRUN_MULTIPLE_PRIORITIES             0

/* When using SMP (i.e. configNUMBER_OF_CORES is greater than one), set
 * configUSE_CORE_AFFINITY to 1 to enable core affinity feature. When core
 * affinity feature is enabled, the vTaskCoreAffinitySet and
 * vTaskCoreAffinityGet APIs can be used to set and retrieve which cores a task
 * can run on. If configUSE_CORE_AFFINITY is set to 0 then the FreeRTOS
 * scheduler is free to run any task on any available core. */
#define configUSE_CORE_AFFINITY                   0

/* When using SMP with core affinity feature enabled, set
 * configTASK_DEFAULT_CORE_AFFINITY to change the default core affinity mask for
 * tasks created without an affinity mask specified. Setting the define to 1
 * would make such tasks run on core 0 and setting it to (1 <<
 * portGET_CORE_ID()) would make such tasks run on the current core. This config
 * value is useful, if swapping tasks between cores is not supported (e.g.
 * Tricore) or if legacy code should be controlled. Defaults to tskNO_AFFINITY
 * if left undefined. */
#define configTASK_DEFAULT_CORE_AFFINITY          tskNO_AFFINITY

/* When using SMP (i.e. configNUMBER_OF_CORES is greater than one), if
 * configUSE_TASK_PREEMPTION_DISABLE is set to 1, individual tasks can be set to
 * either pre-emptive or co-operative mode using the vTaskPreemptionDisable and
 * vTaskPreemptionEnable APIs. */
#define configUSE_TASK_PREEMPTION_DISABLE         0

/* When using SMP (i.e. configNUMBER_OF_CORES is greater than one), set
 * configUSE_PASSIVE_IDLE_HOOK to 1 to allow the application writer to use
 * the passive idle task hook to add background functionality without the
 * overhead of a separate task. Defaults to 0 if left undefined. */
#define configUSE_PASSIVE_IDLE_HOOK               0

/* When using SMP (i.e. configNUMBER_OF_CORES is greater than one),
 * configTIMER_SERVICE_TASK_CORE_AFFINITY allows the application writer to set
 * the core affinity of the RTOS Daemon/Timer Service task. Defaults to
 * tskNO_AFFINITY if left undefined. */
#define configTIMER_SERVICE_TASK_CORE_AFFINITY    tskNO_AFFINITY

/******************************************************************************/
/* ARMv8-M secure side port related definitions. ******************************/
/******************************************************************************/

/* secureconfigMAX_SECURE_CONTEXTS define the maximum number of tasks that can
 *  call into the secure side of an ARMv8-M chip.  Not used by any other ports.
 */
#define secureconfigMAX_SECURE_CONTEXTS        5

/* Defines the kernel provided implementation of
 * vApplicationGetIdleTaskMemory() and vApplicationGetTimerTaskMemory()
 * to provide the memory that is used by the Idle task and Timer task
 * respectively. The application can provide it's own implementation of
 * vApplicationGetIdleTaskMemory() and vApplicationGetTimerTaskMemory() by
 * setting configKERNEL_PROVIDED_STATIC_MEMORY to 0 or leaving it undefined. */
#define configKERNEL_PROVIDED_STATIC_MEMORY    1

/******************************************************************************/
/* ARMv8-M port Specific Configuration definitions. ***************************/
/******************************************************************************/

/* Set configENABLE_TRUSTZONE to 1 when running FreeRTOS on the non-secure side
 * to enable the TrustZone support in FreeRTOS ARMv8-M ports which allows the
 * non-secure FreeRTOS tasks to call the (non-secure callable) functions
 * exported from secure side. */
#define configENABLE_TRUSTZONE            1

/* If the application writer does not want to use TrustZone, but the hardware
 * does not support disabling TrustZone then the entire application (including
 * the FreeRTOS scheduler) can run on the secure side without ever branching to
 * the non-secure side. To do that, in addition to setting
 * configENABLE_TRUSTZONE to 0, also set configRUN_FREERTOS_SECURE_ONLY to 1. */
#define configRUN_FREERTOS_SECURE_ONLY    1

/* Set configENABLE_MPU to 1 to enable the Memory Protection Unit (MPU), or 0
 * to leave the Memory Protection Unit disabled. */
#define configENABLE_MPU                  1

/* Set configENABLE_FPU to 1 to enable the Floating Point Unit (FPU), or 0
 * to leave the Floating Point Unit disabled. */
#define configENABLE_FPU                  1

/* Set configENABLE_MVE to 1 to enable the M-Profile Vector Extension (MVE)
 * support, or 0 to leave the MVE support disabled. This option is only
 * applicable to Cortex-M52, Cortex-M55, Cortex-M85 and STAR-MC3 ports as
 * M-Profile Vector Extension (MVE) is available only on these architectures.
 * configENABLE_MVE must be left undefined, or defined to 0 for the
 * Cortex-M23,Cortex-M33 and Cortex-M35P ports. */
#define configENABLE_MVE                  1

/******************************************************************************/
/* ARMv7-M and ARMv8-M port Specific Configuration definitions. ***************/
/******************************************************************************/

/* Set configCHECK_HANDLER_INSTALLATION to 1 to enable additional asserts to
 * verify that the application has correctly installed FreeRTOS interrupt
 * handlers.
 *
 * An application can install FreeRTOS interrupt handlers in one of the
 * following ways:
 *   1. Direct Routing  -  Install the functions vPortSVCHandler and
 * xPortPendSVHandler for SVC call and PendSV interrupts respectively.
 *   2. Indirect Routing - Install separate handlers for SVC call and PendSV
 *                         interrupts and route program control from those
 * handlers to vPortSVCHandler and xPortPendSVHandler functions. The
 * applications that use Indirect Routing must set
 * configCHECK_HANDLER_INSTALLATION to 0.
 *
 * Defaults to 1 if left undefined. */
#define configCHECK_HANDLER_INSTALLATION    1

/******************************************************************************/
/* Definitions that include or exclude functionality. *************************/
/******************************************************************************/

/* Set the following configUSE_* constants to 1 to include the named feature in
 * the build, or 0 to exclude the named feature from the build. */
#define configUSE_TASK_NOTIFICATIONS           1
#define configUSE_MUTEXES                      1
#define configUSE_RECURSIVE_MUTEXES            1
#define configUSE_COUNTING_SEMAPHORES          1
#define configUSE_QUEUE_SETS                   0
#define configUSE_APPLICATION_TASK_TAG         0

/* USE_POSIX_ERRNO enables the task global FreeRTOS_errno variable which will
 * contain the most recent error for that task. */
#define configUSE_POSIX_ERRNO                  0

/* Set the following INCLUDE_* constants to 1 to include the named API function,
 * or 0 to exclude the named API function.  Most linkers will remove unused
 * functions even when the constant is 1. */
#define INCLUDE_vTaskPrioritySet               1
#define INCLUDE_uxTaskPriorityGet              1
#define INCLUDE_vTaskDelete                    1
#define INCLUDE_vTaskSuspend                   1
#define INCLUDE_xTaskDelayUntil                1
#define INCLUDE_vTaskDelay                     1
#define INCLUDE_xTaskGetSchedulerState         1
#define INCLUDE_xTaskGetCurrentTaskHandle      1
#define INCLUDE_uxTaskGetStackHighWaterMark    0
#define INCLUDE_xTaskGetIdleTaskHandle         0
#define INCLUDE_eTaskGetState                  0
#define INCLUDE_xTimerPendFunctionCall         0
#define INCLUDE_xTaskAbortDelay                0
#define INCLUDE_xTaskGetHandle                 0
#define INCLUDE_xTaskResumeFromISR             1

#endif /* FREERTOS_CONFIG_H */
```

## 1.4 安装中断服务程序

1. 在port.c文件中

```c
void vPortSVCHandler( void )
{
    __asm volatile (
        "   ldr r3, =pxCurrentTCB           \n" /* Restore the context. */
        "   ldr r1, [r3]                    \n" /* Get the pxCurrentTCB address. */
        "   ldr r0, [r1]                    \n" /* The first item in pxCurrentTCB is the task top of stack. */
        "   ldmia r0!, {r4-r11}             \n" /* Pop the registers that are not automatically saved on exception entry and the critical nesting count. */
        "   msr psp, r0                     \n" /* Restore the task stack pointer. */
        "   isb                             \n"
        "   mov r0, #0                      \n"
        "   msr basepri, r0                 \n"
        "   orr r14, #0xd                   \n"
        "   bx r14                          \n"
        "                                   \n"
        "   .ltorg                          \n"
        );
}
void xPortPendSVHandler( void )
{
    /* This is a naked function. */

    __asm volatile
    (
        "   mrs r0, psp                         \n"
        "   isb                                 \n"
        "                                       \n"
        "   ldr r3, =pxCurrentTCB               \n" /* Get the location of the current TCB. */
        "   ldr r2, [r3]                        \n"
        "                                       \n"
        "   stmdb r0!, {r4-r11}                 \n" /* Save the remaining registers. */
        "   str r0, [r2]                        \n" /* Save the new top of stack into the first member of the TCB. */
        "                                       \n"
        "   stmdb sp!, {r3, r14}                \n"
        "   mov r0, %0                          \n"
        "   msr basepri, r0                     \n"
        "   bl vTaskSwitchContext               \n"
        "   mov r0, #0                          \n"
        "   msr basepri, r0                     \n"
        "   ldmia sp!, {r3, r14}                \n"
        "                                       \n" /* Restore the context, including the critical nesting count. */
        "   ldr r1, [r3]                        \n"
        "   ldr r0, [r1]                        \n" /* The first item in pxCurrentTCB is the task top of stack. */
        "   ldmia r0!, {r4-r11}                 \n" /* Pop the registers. */
        "   msr psp, r0                         \n"
        "   isb                                 \n"
        "   bx r14                              \n"
        "                                       \n"
        "   .ltorg                              \n"
        ::"i" ( configMAX_SYSCALL_INTERRUPT_PRIORITY )
    );
}
void xPortSysTickHandler( void )
{
    /* The SysTick runs at the lowest interrupt priority, so when this interrupt
     * executes all interrupts must be unmasked.  There is therefore no need to
     * save and then restore the interrupt mask value as its value is already
     * known. */
    portDISABLE_INTERRUPTS();
    traceISR_ENTER();
    {
        /* Increment the RTOS tick. */
        if( xTaskIncrementTick() != pdFALSE )
        {
            traceISR_EXIT_TO_SCHEDULER();

            /* A context switch is required.  Context switching is performed in
             * the PendSV interrupt.  Pend the PendSV interrupt. */
            portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
        }
        else
        {
            traceISR_EXIT();
        }
    }
    portENABLE_INTERRUPTS();
}
```

2. 替换进.s启动文件的中断向量表

```assembly
  .word SVC_Handler
  .word DebugMon_Handler
  .word 0
  .word PendSV_Handler
  .word SysTick_Handler
```

3. 更换其他定时器作为HAL库的时间基准
   1. 修改timebase source从systick改为timx
   2. HAL库里面是uwTick软件计数，在SysTick中断->SysTick_Handler()->HAL_IncTick()->uwTick++
   3. 但是产生systick之后调用的中断函数变了，从SysTick_Handler变成了xPortSysTickHandler，所以需要写定时器的中断函数内调用SysTick_Handler，直接在中断向量表改变对应定时器的回调函数为SysTick_Handler不推荐

## 2.1 变量作用域和生命周期

```c
vTaskCreate(vTask_Handler,"task_name",(uint_t)stack_depth,(uint_t)prio,Return_Handler);
```

1. 创建任务时需要指定任务的栈内存大小
2. 变量作用域
   1. 局部变量：退出当前代码块时清理；存储在栈
   2. 静态局部变量：定义的时候赋值，默认为0；后续赋值覆盖；存储在静态数据区
   3. 全局变量：其他文件要使用需要extern uint8_t num；覆盖整个项目；存储在全局数据区
   4. 静态全局变量：定义的时候赋值，默认为0；只局限于当前文件；常用于封装文件的内部；存储在全局数据区
   5. 堆变量

## 2.2 变量在SRAM的存储位置

![存储器功能分类](https://doc.embedfire.com/mcu/stm32/f407batianhu/std/zh/latest/_images/regist01.png)

![存储器Block0内部区域功能划分](https://doc.embedfire.com/mcu/stm32/f407batianhu/std/zh/latest/_images/regist02.png)

1. 32bit CPU的最大寻址地址0xffffffff，逻辑区分划分为block
2. 存储器的分类
   1. RAM
      1. SRAM结构复杂
      2. DRAM需要刷新
         1. SDRAM 异步通信
         2. DDR SDRAM带宽增加

   2. ROM


![基本存储器种类](https://doc.embedfire.com/mcu/stm32/f407batianhu/std/zh/latest/_images/storag002.jpeg)

3. flash中存储不常变化的
   1. 对于变量：变量是存储在SRAM，值存储在FLASH中
4. SRAM中存储情况
   1. 以段为单位存储，一下由低地址开始
   2. .data 有初始值的全局变量或者静态局部变量
   3. .bss 无初始值或者零值的全局变量或者静态局部变量
   4. .heap堆内存
   5. .stack栈内存：局部变量和参数
5. 堆和栈的生长方向看启动文件；EQU是告诉编译器协助编译的，AREA到下一个AREA之前分配的地址空间都是该段的空间，启动文件只是先声明这些段的大小，具体位置是靠连接文件确定
6. malloc分配的内存含有一个8byte的信息头
7. 函数调用压栈流程
   1. 调用函数时，将调用函数的入口写进PC，将下一条指令的地址写进LR，部分寄存器状态压栈
   2. 当被调用函数结束时，将LR的值返回到PC
   3. 至于是否需要将LR的值压栈进行嵌套调用，是汇编的事情
8. 栈溢出导致跑飞的原理是：SP溢出指向别的栈内，如果该栈没有修改，没有影响，如果有修改，导致应该写进PC的地址错误而跑飞

## 2.3 FreeRTOS的命名规范

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_namerule.jpg)

1. 三种重要的数据类型
   1. TickType_t 记录系统节拍xTickCount
      1. 作为xTaskGetTickCount()的返回值获取当前系统节拍计数值；使用pdTICKS_TO_MS()宏函数转换为真实的ms时间
      2. 作为vTaskDelay()的参数设置需要延迟的系统节拍数；使用pdMS_TO_TICKS()宏函数将真实转换为系统节拍数
   2. BaseType_t 记录成功/失败，选择当前架构下处理效率最高的数据类型
      1. 作为函数返回值表示任务成功与否：pdPASS成功；pdFAIL失败；将创建函数作为if判断条件，分支处理创建失败
      2. 作为函数参数设置是否使能某一功能
   3. UBaseType_t 上者的无符号，表示一个非负数
      1. 作为函数参数设置uint类型
2. 变量的命名规则
   1. 变量名=类型前缀+实际名字
3. 函数的命名规则
   1. 函数名=返回类型前缀+模块名+实际名字
   2. 当函数是static静态函数，局限在函数内部中，前缀用prv，不用模块名
4. 宏定义的命名规则
   1. 宏定义名=文件名前缀+实际名字

## 3.1 任务创建

```c
TaskHandle_t xTaskHandle;
BaseType_t xTaskCreate( TaskFunction_t pvTaskCode,
	const char * const pcName,//指向字符常量的指针，并且本身为常数，而不是指针指向字符串
	unsigned short usStackDepth,
	void *pvParameters,
	UBaseType_t uxPriority,
	TaskHandle_t *pvCreatedTask );
```

1. pvTaskCode 循环执行的任务代码
2. pcName任务名称
3. usStackDepth 栈深度
4. pvParameter 作为创建的任务的参数，实现同一任务函数的复用于多个任务，但是本身是void *类型，传入时强转，在函数内部需要强转回
5. uxPriority
6. pcCreatedTask 先创建任务句柄，随后传入作为参数，修改句柄之后，通过句柄实现对任务的操作

## 3.2 任务的状态

1. ready
2. running：当没有任务执行的时候执行idle
3. block：等待资源或者超时vTaskDelay()而阻塞
4. suspend：主动vTaskSuspend()挂起 
5. HAL_Delay()是通过在while循环内一直查询当前时间是否到达指定时间，而不释放CPU；vTaskDelay()是通过记录当前任务唤醒的时刻并且将任务放进block

## 3.3 时间片

```c
//设置systick中断
#define configTICK_RATE_HZ                         100
```

## 4. 按钮驱动

1. 相对于裸机程序是先读取当前引脚状态，如果按下开始检测，FreeRTOS是使用vTaskDelay()设置周期10ms，而抖动区间是小于1ms，而按下持续时间是大于10ms的，也就不会出现按下状态是在两次采样之间而错过，因而不需要采样
2. 创建复用函数
   1. 定义结构体包含序号掩码以区分选中的port，和变量名序号的含义不一样，前者是从底层编号区分，能够被代码识别，后者是编程定义的变量的顺序，能被程序员识别
   2. 结构体还包含引脚和回调函数
   3. 先定义结构体变量，随后Init()根据port或者引脚实现对应外设的初始化
3. 拆分文件 
   1. 控制代码长度在300行内
   2. 从定义结构体传参，到在整体头文件定义结构体变量，在各自文件引用变量，在文件内部对变量直接操作
   3. 函数使用从vKeyInit(&hkey1)到vKey1Init(void)

## 5. 堆内存管理

1. malloc()是不可重入函数，多个任务同时调用会报错；碎片化严重和执行慢
2. pvPortMalloc()和vPortFree()
3. heap1：只有pvPortMalloc，没有回收机制
   1. 内部是一个ucHeap[configTOTAL_HEAP_SIZE]数据实现
   2. 只创建内核对象而不销毁：如创建任务而不删除释放
4. heap2：过时方案，但是保留以兼容旧项目
   1. 在heap1上优化，添加回收内存
   2. 内部同样uwHeap数组
   3. 最优匹配算法：恰能满足的最小空闲块
   4. 数组头部自身是有8byte的内存头；数组结尾是有一个长度0的，但是有内存头的xEnd
   5. 内碎片化严重：不仅仅是每次分配之后的剩余外碎片，还有每次使用之后会切割内存头
   6. 不合并相邻空闲块
5. heap3：
   1. 对malloc和free改造，使其可重入
   2. 在pvPorrMalloc()和vPortFree()前后使用vTaskSuspendAll()和vTaskResumeAll()暂停调度器
6. heap4：
   1. 在heap2上优化，添加合并相邻空闲块
7. heap5：
   1. 在heap4上优化，可以控制不同地址上的堆内存数组
8. 内存使用情况分析：
   1. 根据使用的编译工具的编译输出结果判断整个项目的空间存储消耗
   2. 在FreeRTOS，任务堆是和系统的堆独立区分的
   3. 在heap 124中，ucHeap[]数组是未定义的静态变量，存储在.bss段
   4. 而FreeRTOS的内核对象是默认动态创建，即内核对象是存储在ucHeap[]里面，包括任务控制块和任务栈深度
   5. 而裸机原本的.heap段用来printf等一些c库函数调用
   6. xPortGetFreeHeapSize()获取当前剩余的内存量；xPortGetMinimalFreeHeapSize()获取历史最小的剩余量

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_memang.jpg)

## 6.1 二进制信号量

1. 用于任务与任务之间，或者任务与中断之间同步
2. 同步：通过发信号的方式，将两个不相干的任务联动起来

```c
SemaphoreHandle_t xSem;
xSem= xSemaphoreCreateBinary();
xSemaphoreGive(xSem);
xSemaphoreTask(xSem,TickType_t);
vSemaphoreDelete(xSem);
```

## 6.2 中断和二进制信号量

1. 中断的优先级是高于任务的，所以当触发中断且中断占用时间长的时候，任务是不执行的
2. 所以实际使用是在中断中释放信号量，在一个新的任务中执行
3. 在中断中使用的API必须是带FromISR后缀的函数
4. 有一个返回的参数，表示当前中断释放信号量之后是否有更高优先级任务被唤醒，定义时设置初始值pdFALSE
5. 需要修改configMAX_SYSCALL_INTERRUPT_PRIORITY设置允许调用FromISR的最高中断优先级

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_SemFromISR.jpg)

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_SemFromISR_Prio.jpg)

## 6.3 二进制信号量和DMA

1. vTaskDelay()是执行到当前语句的时候开始suspend，是前后两条语句之间的gap
2. 如果在高优先级通讯中，大部分时间用于轮询和传输，会导致低优先级任务ready之后不能正常执行，而被强迫改变gap
3. 传输完成的中断函数中释放二进制信号量通知任务传输完成以继续

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_SemDMA.jpg)

## 7. 计数信号量

1. 多个生产者，多个消费者；但是多个消费者哪一个获取到是根据任务本身优先级或者同优先级FIFO确定
2. 计数信号量用于事件计数：初始计数值0，向上计数；累计还未处理的事件的数量，保证事件全部处理
3. 计数信号量用于守护资源：初始计数值和最大计数值保持一致，向下计数；记录剩余资源的数量，保证资源不超出最大值

```c
xSemaphoreHandle_t xSem;
xSem=xSemaphoreCreateCounting(uxMaxCount,uxInitialCount);
```

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_SemCount.jpg)

## 8.1 队列

1. 二进制信号量--添加槽位-->计数信号量--附带数据-->队列--长度为1-->邮箱
2. 创建队列时当任务堆内存大小不够时失败，返回NULL；占据的字节是QueueHead+QueueLen*ItemSize
3. 调用pdMS_TO_TICK的时候，转换的时间长度必须是tick的单位时间长度的整数倍
4. 通讯的守门员任务：
   1. 在多个任务直接接触到通讯外设时，如果在前一个任务BUSY的时候下一个任务使用外设传输，下一个任务直接报错
   2. 添加守门员任务：原本的任务添加队列，xQueueSend()发送到公共队列上；守门员任务xQueueReceive()从公共队列读取，随后控制外设发布
5. 通讯时按值拷贝和按引用拷贝
   1. 如果按值拷贝：每个Item都是一个完整的字符串，内存消耗大
   2. 按引用拷贝：每个Item都只是对应字符串的首地址
   3. Queue存储的Item不再是具体的值，而是指针；而传入参数是Item的地址，所以这种情况下是传入二级指针
   4. 需要原任务在Send的时候先malloc()一个内存，将其指针地址传给Queue；如果不malloc是用提前预留的缓冲区，存储在.bss段，地址不会变，但是需要给缓冲区上锁保护；buff和malloc都是整个字符串作为一个整体，首地址作为Item传入到Queue当中，但是前者是多个整个在一个任务中共用一个buff，不然内存崩溃，后者是不停分配但是也会在守门员任务中回收；
   5. malloc之后要strcpy(pmem,str)，然后send(&pmem)
   6. 在守门员任务中，先定义一个同样类型的指针，随后reverive(&pmem)，pmem的值是一样的，也就是说在堆内存的地址是一样的，然后free()释放

```c
QueueHandle_t hQ;
hQ=xQueueCreate(uxQueueLen,uxItemSize);//返回值为NULL为创建失败
xQueueReveive(hQ,&data,portMAX_DELAY);
```

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_QueueAPI.jpg)

## 8.2 邮箱

1. 邮箱和队列是一个内核对象
2. peek只窥探，不拿出数据
3. 邮箱的特点
   1. Queue是同步的，有严格的FIFO顺序，并且每次receive都是对一个不重复的Item操作；MailBox是异步的，只要有数据，就可以peek，多次peek可以对同一个Item
   2. 只看不取，适合广播，即多个Receiver都；获取同一个Item；信号量和队列允许多个Receiver但是不允许获取同一个Item
4. 应用：
   1. 前端AFE采集数据；但是保证其他任务处理的都是同一个数据？需要其他同步量优化
   2. 在二元信号量与中断应用的基础上添加一定的简短数值，但是send和receive
   3. 状态机，警告任务，等只需要获取最新状态的
   4. 高优先级任务先peek再决定是否reveive拿数据

```c
xMail=xQueueCreate(1,sizeof(Type));
BaseType_t xQueueOverWrite(hQ,&item);
BaseType_t xQueuePeek(hQ,&item,TickType_t xTicksToWait);//窥探不拿出数据
```

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_MailBoxAPI.jpg)

## 9.1 软件定时器

1. 周期性的执行一段代码：自动重装；
2. 延迟一段时间之后执行一段代码：不自动重装
3. 使能软件定时器之后，再vTaskSchduler()之后会创建软件定时器任务
4. 计时单位是configTICK_RATE_HZ
5. 使用API之后是向定时器任务队列发送一个Item；定时器任务从队列取出消息并执行
6. API的参数指定的是等待放进队列的最长时间
7. 定时器任务：
   1. 不是基于时间片的周期执行，而是基于systick中断，虽然时间片长度就是一个systick中断周期。在每个systick中断中，先incretick()，随后从delay列表的首元素获取其唤醒的时间（该列表式按照唤醒时间排序的，是block类列表），判断当前是否有任务超时需要唤醒；对于vTaskDelay()就是调用的任务，对于不同的定时器，定时器任务又本身只取超时时间最近的定时器，所以定时器任务对外暴露一个超时时间；如果有，将任务移进ready列表，如果优先级比当前任务高，置pendsv挂起位1；当systick中断结束，进入pendsv中断，执行任务切换
   2. 取出消息并执行
   3. 判断当前是否有到期任务，执行其回调函数

8. 回调函数：

   1. 不能带阻塞的API，定时器任务执行回调函数，如果阻塞，在下一个systick中断产生的时候就不能判断是否有超时需要放回ready列表；可以将阻塞前后的代码拆分到两个状态机，前者的周期是原本周期，后者周期是delay的时间，在本阶段执行完之后，修改period为下一阶段周期，并修改状态机
   1. 不能长时间霸占CPU，尽管是有systick中断打断，但是打断之后没有更高优先级任务执行，还是执行定时器任务

9. 和硬件定时器区别

   1. 硬件用于PWM IC OC等，微妙级周期，周期是中断触发，即中断回调开始执行的间隔，更加精细控制开始执行的周期
   2. 软件用于设置毫秒级周期，周期是两次回调函数之间的间隔，控制执行的间隔

10. SysTick 中断产生后，先递增 tick，检查是否有任务超时需要唤醒；如果有，就把它们放到就绪列表。然后判断最高优先级就绪任务是否高于当前任务，如果需要切换，就挂起 PendSV。等 SysTick 中断退出后，PendSV 执行真正的任务上下文切换。软件定时器服务任务如果因为等待下一个定时器到期而被唤醒，它会在任务上下文中运行，并执行软件定时器回调。

```c
//创建
TimerHandler_t xTimerCreate(pcTimername,//定时器名称
                           xTimerPeriod,//周期，单位是系统节拍configTICK_RATE_HZ
                           uxAUtoReload,//是否重装载
                           pvTimerID,//定时器ID，相当于回调函数参数
                           pxCallback);//回调函数
//创建之后需要start
BaseType_t xTimerStart(htimer,TickType_t Ticks_To_Wait);//如果队列满，等待的时间
```

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_TimerInit.jpg)

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_TimerAPI.jpg)

![](D:\coding_codes\stm32f407\learning_logs\resources\FreeRTOS_TimerQueue.jpg)

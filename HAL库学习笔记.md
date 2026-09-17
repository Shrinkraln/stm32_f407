## 1. 概述

1. HAL库使用句柄的方式实现对对象的操作，而不是传统c语言的对过程的操作
2. MSP函数指的是与MCU硬件相关的配置，HAL_UART_MSPInit()指对GPIO口配置，而普通的HAL_UART_Init()只是对于该外设的协议的设置
3. HAL_UART_IRQ_Handler()实现对中断类型的判断，随后执行相应的回调函数Callback()
4. HAL库对于外设进行了统一，支持三种编程方式
   1. 阻塞	HAL_I2C_Master_Transmit()
   2. 中断    HAL_I2C_Master_Transmit_IT()
   3. DMA   HAL_I2C_Master_Transmit_DMA()；

## 2. SPI读flash

### 1. SPI读写代码

1. 强制转换是直接截断，对于uint不影响，对于指针，是改变指向地址的对应解析方式（在生成汇编指令之前；不同类型的指针的p++的步长不一样，汇编指令不一样） 
2. 指针指向的数据类型的改变要满足内存对齐：如对于0x0001上的uint8，转化为uint32报错，只能是0x0000
3. 不能使用0x01，是立即数，存储在flash里面，没有地址
4. **不可以使用sizeof(*data)，data无论是想作为指向数组的指针，都是指向一个uint8，只是p++的方式刚好可以取到下一个元素的地址** ;而对于数组，汇编之后数组名编程标号，不可修改，并且确定，对于sizeof是整个数组的长度
5. BSP里面形参是指定数值但是APP调用可以是传入sizeof数组

```c
/*****************************************************************************
 * 文件名：w25.c
 * 说明：W25Q系列SPI Flash驱动 (基于STM32 HAL库)
 *
 * ======================== 注释索引表（原理与规则） =========================
 * [REF-01] CS控制原则：拉低CS代表"事务开始"，拉高CS代表"事务结束"。
 *          每一条完整的指令(Opcode + Address + Data)是一个事务单位。
 *          CS的拉高时机由"指令码"决定（查表）：纯命令(0x06)发完即拉高；
 *          带地址/数据的(0x02/0x03/0x20)必须在所有字节完成后拉高。
 *
 * [REF-02] 写使能(0x06)的特殊性：该指令无后续数据。
 *          Flash芯片在CS的上升沿锁存并执行该指令，因此发完0x06后必须立即拉高CS。
 *          若不拉高，芯片会一直等待后续字节，命令永不生效。
 *
 * [REF-03] 读状态寄存器(0x05)时序：必须作为一个独立事务。
 *          时序：CS低 -> 发0x05 -> 收1字节 -> CS高。
 *          不可与其他指令连续发送，否则读出的数据错位。
 *
 * [REF-04] 等待忙(BUSY)机制：通过读状态寄存器1的BIT 0轮询。
 *          芯片擦除/编程期间BUSY=1，完成后硬件自动清零。
 *          必须在每次写/擦除操作后调用，确保前序操作完成。
 *
 * [REF-05] 页编程(0x02)物理限制：单次最多256字节，且地址低8位 + 长度 <= 256。
 *          若跨页（地址回卷），数据会覆盖当前页首，必须拆分为多次写入。
 *
 * [REF-06] 数组传参的本质：函数参数 uint8_t *data 是指针变量（占4字节），
 *          并非原数组。sizeof(*data) == 1，无法获取数组长度。
 *          必须显式传入 len 参数，HAL库根据len搬运字节流。
 *					
 * =========================================================================
 ****************************************************************************/

#include "w25.h"

/* ---------------------------- 静态工具函数（内部调用） ----------------------------- */

// [REF-03] 读取状态寄存器1 (返回当前状态值)
static uint8_t w25_read_status_reg1(void) {
    uint8_t cmd = W25_READ_R1; // 0x05
    uint8_t status;
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_RESET); // [REF-01] 事务开始
    HAL_SPI_Transmit(&hspi1, &cmd, 1, HAL_MAX_DELAY);
    HAL_SPI_Receive(&hspi1, &status, 1, HAL_MAX_DELAY);
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_SET);   // [REF-01] 事务结束
    
    return status;
}

// [REF-04] 等待芯片内部操作完成 (BUSY位清除)
static void w25_wait_busy(void) {
    while (w25_read_status_reg1() & 0x01) {
        // 轮询等待，直到硬件清零BUSY位
    }
}

// [REF-02] 写使能 (含等待WEL位生效)
static void w25_write_enable(void) {
    uint8_t cmd = W25_WRITE_ENBALE; // 0x06
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_RESET); // [REF-01] 事务开始
    HAL_SPI_Transmit(&hspi1, &cmd, 1, HAL_MAX_DELAY);
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_SET);   // [REF-02] 立即拉高触发执行
    
    // 可选：等待WEL位(bit 1)置1，确保写使能已生效
    while ((w25_read_status_reg1() & 0x02) == 0);
}

/* ---------------------------- 对外API（用户调用） ----------------------------- */

// [REF-05] [REF-06] 页编程 (写入数据)
void spi_write(uint32_t addr, uint8_t *data, uint16_t len) {
    // 参数合法性检查
    if (data == NULL || len == 0 || len > 256) return;
    // [REF-05] 检查跨页 (页起始地址由低8位决定)
    if ((addr & 0xFF) + len > 256) {
        // 实际工程中此处应拆分为多次页编程，本示例暂做保护性退出
        return; 
    }
    
    w25_write_enable(); // 内部含CS控制及WEL等待
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_RESET); // [REF-01] 事务开始
    
    uint8_t temp;
    temp = W25_PAGE_PROMGRAM; // 0x02
    HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    // 发送24位地址 (大端模式，高位先发)
    temp = addr >> 16; HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr >> 8;  HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr;       HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    // [REF-06] 发送数据（必须用显式传入的len，不能用sizeof）
    HAL_SPI_Transmit(&hspi1, data, len, HAL_MAX_DELAY);
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_SET);   // [REF-01] 事务结束
    
    w25_wait_busy(); // [REF-04] 等待写入完成
}

// [REF-01] 标准读取数据 (无Dummy字节)
void spi_read(uint32_t addr, uint8_t *data, uint16_t len) {
    if (data == NULL || len == 0) return;
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_RESET); // [REF-01] 事务开始
    
    uint8_t temp = W25_READ; // 0x03
    HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    temp = addr >> 16; HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr >> 8;  HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr;       HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    HAL_SPI_Receive(&hspi1, data, len, HAL_MAX_DELAY);
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_SET);   // [REF-01] 事务结束
}

// [REF-01] 扇区擦除 (4KB对齐)
uint8_t spi_erase_sector(uint32_t addr) {
    // 扇区擦除要求地址低12位为0 (4KB对齐)
    if (addr & 0x0FFF) return 0;
    
    w25_write_enable();
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_RESET); // [REF-01] 事务开始
    
    uint8_t temp = W25_SECTOR_ERASE; // 0x20
    HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    temp = addr >> 16; HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr >> 8;  HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    temp = addr;       HAL_SPI_Transmit(&hspi1, &temp, 1, HAL_MAX_DELAY);
    
    HAL_GPIO_WritePin(GPIOG, GPIO_PIN_6, GPIO_PIN_SET);   // [REF-01] 事务结束
    
    w25_wait_busy(); // [REF-04] 等待擦除完成
    
    return 1;
}
```

### 2. SPI 驱动

1. 先配置后使能
2. SSM soft-select-manager 软件从机管理，片选信号源输入选择（选中的一对主机和从机的NSS都是0，表示当前两个在通讯）
   1. 0，读取硬件NSS引脚作为控制信号源
   2. 1，读取寄存器内部SSI作为控制信号，而SSI不受外部控制，只能是程序自己设置的或者是上电默认值；实际作用是在MSTR 1 “本分”地做主机，实际项目非常常见；当SII 0表示从机；SSI 1表示主机
3. SSOE 片选输出使能
   1. 0，禁用NSS自动输出，可以在NSS同一个物理引脚的GPIO实现模拟控制
   2. 1
4. SSM和SSOE正交
   1. 10软件管理NSS，且不使用自动输出，只使用同一硬件的GPIO；由于不会读取NSS，不适合释放控制权的主机，只适合单主机模式；不读取，不适合从机；**实际上最常见的组合**
   2. 11软件管理NSS，由于不读取NSS，又要强制输出NSS，不适合通讯，无法自检
   3. 00硬件管理NSS，要读取NSS，但不输出NSS，只做从机
   4. 01硬件管理NSS，要读取NSS，同时输出，
5. CPLO CPHA 时钟极性0低电平，采样边沿0奇数
6. LSBFIRST DFF 低位先行 数据长度
7. BR设置波特率
8. FRF1设置为TI模式满足德州仪器的SPI
9. MSTR 1 SPE 1 使能
10. TXE 1 可以进行下一次传输数据
11. EXNE 1进行读取
12. 对于全双工，必须是同时发，读，否者RDR会溢出
13. DR只有一个逻辑寄存器，但是RDR和TDR是两个物理寄存器公用一个逻辑地址
14. 寄存器结构体映射一定要声名__IO
15. SPI接受是由发送驱动的，一定要添加发送dummy
16. 寄存器写入之前设置其默认值
17. RX之后要手动读一次DR才能清理标志位，而不是直接操作标志位
18. 如果头文件定义了变量，其他源文件引用了头文件，会造成变量重复定义；规范做法是头文件extern 变量声名，只在一个源文件定义变量，其他源文件只引用，不定义，不声明
19. 使用UL表示数值是无符号长整型
20. 宏，结构体，枚举类型的使用场景
    1. 宏：寄存器地址，寄存器位掩码
    2. 结构体：句柄用于对对象的控制（**handletypedef第一个成员是typedef结构体实现寄存器映射，第二个成员是inittypedef结构体实现参数配置，inittypedef的结构体是可读性更强的配置**），寄存器映射其实就是对外设寄存器的控制，整合规范化（参数整合到一个结构体）
    3. 枚举：固定范围内互斥的值的表示，如状态机，返回值


| SSM            |  SSI | SSOE | 使用场景 |                                                              |
| -------------- | ---: | ---: | -------- | ------------------------------------------------------------ |
| 软件 NSS，主机 |    1 |    1 | 0        | 内部 NSS 保持高电平，避免主机 Mode Fault；NSS 引脚释放，从机 CS 用 GPIO 控制。**当前 W25Q 项目就是这种** |
| 软件 NSS，从机 |    1 |    0 | 0        | 内部 NSS 保持低电平，表示从机被软件选中；不使用 NSS 引脚     |
| 硬件 NSS 输入  |    0 | 忽略 | 0        | NSS 引脚作为输入；从机由外部硬件 NSS 选择；主机多主机模式下检测 Mode Fault |
| 硬件 NSS 输出  |    0 | 忽略 | 1        | NSS 引脚由 SPI 硬件自动驱动，作为硬件片选信号输出；具体时序需查 STM32F407 参考手册确认 |

```c
//.h文件中
#define SPI_1_BASE			0x40013000UL:
typedef struct{
	__IO	uint32_t cr1;
	__IO	uint32_t cr2;
	__IO	uint32_t sr;
	__IO	uint32_t dr;
	__IO	uint32_t crcpr;
	__IO	uint32_t rxcrcr;
	__IO	uint32_t txcrcr;
	__IO	uint32_t i2scfgr;
	__IO	uint32_t i2spr;
}spi_register_t;
extern spi_register_t *spi1_instance;
//.c文件中
void spi_send_1byte(spi_register_t* spix,uint8_t tx){
	while((spix->sr & 0x02) == 0x00);
	spix->dr = tx;
	while((spix->sr & 0x01) == 0x00);
	uint32_t dummy = spix->dr; // 读走并清除 RXNE
	(void)dummy;
}

void spi_receive(spi_register_t* spix,uint8_t *rx){
	while((spix->sr&0x02)==0x00);
	spix->dr=0xff;
	while((spix->sr&0x01)==0x00);
	*rx=spix->dr;
}
void spi_init(spi_register_t* spix, spi_typedef_t cfg) {
    spix->cr1 = 0;
    spix->cr2 = 0;

    spix->cr1 = (cfg.line << 14)     | /* BIDIMODE */
                (cfg.ssm_ssi << 8)   | /* SSM/SSI */
                (cfg.braude << 3)    | /* BR[2:0] */
                (cfg.clk << 0)       | /* CPOL/CPHA */
                (cfg.master << 2)    | /* MSTR */
                (cfg.lsbfirst << 7)  | /* LSBFIRST */
                (cfg.len << 11);       /* DFF */

    spix->cr2 = (cfg.ssoe << 2);       /* SSOE */

    spix->cr1 |= 0x40;                 /* SPE: SPI enable */
}
```

## 3. RCC时钟使能

### 3.1 基于HAL库实现

**所有外设都要使能时钟和配置引脚为复用，而引脚配置有需要使能时钟**

```c
//使能时钟 
__HAL_RCC_GPIOA_CLK_ENABLE();
//给到外设对应RCC一个先1再0的脉冲，实现对应外设寄存器的复位
__HAL_RCC_CAN1_FORCE_RESET();
__HAL_RCC_CAN1_RELEASE_RESET();
```

### 3.2 寄存器控制

```c
void my_rcc_apb2_enable(uint32_t periph){ RCC->APB2ENR |= periph; }
void my_rcc_apb2_disable(uint32_t periph){ RCC->APB2ENR &= ~periph; }

void my_rcc_apb2_reset(uint32_t periph){
    RCC->APB2RSTR |= periph;
    RCC->APB2RSTR &= ~periph;
}
```

## 4. GPIO配置

### 4.1 HAL库

对于其他外设需要使用引脚复用功能查看规格手册（硬件电气特征）

```c
void my_gpio_hal_init(GPIO_TypeDef *gpiox, uint16_t pin_mask,
                      uint32_t mode, uint32_t pull, uint32_t speed, uint32_t af)
{
    GPIO_InitTypeDef init = {0};
    init.Pin = pin_mask;
    init.Mode = mode;
    init.Pull = pull;
    init.Speed = speed;
    init.Alternate = af;
    HAL_GPIO_Init(gpiox, &init);
}
```

### 4.2 寄存器控制

```c
//.h文件
#define MY_GPIOA_BASE 0x40020000UL
#define MY_GPIOB_BASE 0x40020400UL
#define MY_GPIOC_BASE 0x40020800UL

#define MY_GPIOA ((gpio_register_t *)MY_GPIOA_BASE)
#define MY_GPIOB ((gpio_register_t *)MY_GPIOB_BASE)
#define MY_GPIOC ((gpio_register_t *)MY_GPIOC_BASE)

#define MY_GPIO_PIN0  0x0001UL
#define MY_GPIO_PIN1  0x0002UL
#define MY_GPIO_PIN2  0x0004UL
#define MY_GPIO_PIN3  0x0008UL
#define MY_GPIO_PIN4  0x0010UL
#define MY_GPIO_PIN5  0x0020UL
#define MY_GPIO_PIN6  0x0040UL
#define MY_GPIO_PIN7  0x0080UL
#define MY_GPIO_PIN8  0x0100UL
#define MY_GPIO_PIN9  0x0200UL
#define MY_GPIO_PIN10 0x0400UL
#define MY_GPIO_PIN11 0x0800UL
#define MY_GPIO_PIN12 0x1000UL
#define MY_GPIO_PIN13 0x2000UL
#define MY_GPIO_PIN14 0x4000UL
#define MY_GPIO_PIN15 0x8000UL

#define MY_GPIO_MODE_INPUT   0x00UL
#define MY_GPIO_MODE_OUTPUT  0x01UL
#define MY_GPIO_MODE_AF      0x02UL
#define MY_GPIO_MODE_ANALOG  0x03UL

#define MY_GPIO_OTYPER_PUSH_PULL   0x00UL
#define MY_GPIO_OTYPER_OPEN_DRAIN   0x10UL
#define MY_GPIO_MODE_OUTPUT_PP      (MY_GPIO_MODE_OUTPUT | MY_GPIO_OTYPER_PUSH_PULL)
#define MY_GPIO_MODE_OUTPUT_OD      (MY_GPIO_MODE_OUTPUT | MY_GPIO_OTYPER_OPEN_DRAIN)
#define MY_GPIO_MODE_AF_PP          (MY_GPIO_MODE_AF | MY_GPIO_OTYPER_PUSH_PULL)
#define MY_GPIO_MODE_AF_OD          (MY_GPIO_MODE_AF | MY_GPIO_OTYPER_OPEN_DRAIN)

#define MY_GPIO_SPEED_LOW        0x00UL
#define MY_GPIO_SPEED_MEDIUM     0x01UL
#define MY_GPIO_SPEED_HIGH       0x02UL
#define MY_GPIO_SPEED_VERY_HIGH  0x03UL

#define MY_GPIO_NOPULL    0x00UL
#define MY_GPIO_PULLUP    0x01UL
#define MY_GPIO_PULLDOWN  0x02UL

typedef struct {
    volatile uint32_t moder; 
    volatile uint32_t otyper;
    volatile uint32_t ospeedr;
    volatile uint32_t pupdr;
    volatile uint32_t idr;
    volatile uint32_t odr;
    volatile uint32_t bsrr;
    volatile uint32_t lckr;
    volatile uint32_t afr[2];   //高低两个寄存器用数组结合
} gpio_register_t;

void my_gpio_init(gpio_register_t *gpio, uint32_t pin_mask,
                  uint32_t mode, uint32_t speed, uint32_t pull, uint32_t af);
/////////////////////////////////////////////////////////////////////////////////////
//.c文件
//每个GPIO是二维外设，特殊使用pin_mask来表示引脚位置
static void my_gpio_config_2bit(volatile uint32_t *reg, uint32_t pin_mask, uint32_t value)
{
    //pos记录当前处理的引脚位置
    uint32_t pos = 0U;
    //pins记录当前处理的引脚掩码
    uint32_t pins = pin_mask;
    while (pins != 0U)
    {   
        //pins最低位是否为1，如果是，则表示当前处理的引脚需要配置
        if ((pins & 1U) != 0U)
        {
            //计算当前引脚在寄存器中的位置，每个引脚占用2位，因此需要左移2位
            uint32_t shift = pos * 2U;
            //3UL表示2位掩码，清除当前引脚的配置位，然后设置为新的值
            *reg = (*reg & ~(3UL << shift)) | ((value & 3U) << shift);
        }
        pins >>= 1U;
        pos++;
    }
}

void my_gpio_init(gpio_register_t *gpio, uint32_t pin_mask,
                  uint32_t mode, uint32_t speed, uint32_t pull, uint32_t af)
{
    uint32_t moder_mode;
    uint32_t output_type;

    if (gpio == 0U)
        return;

    /* 低2位是 MODER 模式，bit4 表示输出类型选择 */
    moder_mode  = mode & 0x03U;
    output_type = (mode >> 4U) & 0x01U;

    my_gpio_config_2bit(&gpio->moder, pin_mask, moder_mode);
    my_gpio_config_2bit(&gpio->ospeedr, pin_mask, speed);
    my_gpio_config_2bit(&gpio->pupdr, pin_mask, pull);

    /* 推挽/开漏选择：OTYPER 每引脚1位，1=开漏，0=推挽 */
    if (output_type != 0U)
        gpio->otyper |= pin_mask;
    else
        gpio->otyper &= ~pin_mask;

    if (moder_mode == MY_GPIO_MODE_AF)
    {
        uint32_t pos = 0U;
        uint32_t pins = pin_mask;
        while (pins != 0U)
        {
            if ((pins & 1U) != 0U)
            {
                uint32_t idx = pos >> 3U;
                uint32_t shift = (pos & 7U) * 4U;
                gpio->afr[idx] = (gpio->afr[idx] & ~(0xFUL << shift)) | ((af & 0xFUL) << shift);
            }
            pins >>= 1U;
            pos++;
        }
    }
}
```

## 5. UART

### 5.1 HAL库

```c
void my_uart_hal_init(UART_HandleTypeDef *huart, USART_TypeDef *instance, uint32_t baudrate)//波特率可以直接在init成员配置
{
    if (huart == NULL)
        return;
    //对一个句柄选择指定的寄存器映射基地址
    huart->Instance = instance;
    //对句柄的inittypedef结构体进行赋值
    //也可以直接定义一个结构体然后该结构体作为右值，直接赋值（字符串是不能作为右值直接赋值）
    //逐个赋值有助于跳过空白成员，而结构体赋值右值必须全部先初始化
    huart->Init.BaudRate = baudrate;
    huart->Init.WordLength = UART_WORDLENGTH_8B;
    huart->Init.StopBits = UART_STOPBITS_1;
    huart->Init.Parity = UART_PARITY_NONE;
    huart->Init.Mode = UART_MODE_TX_RX;
    huart->Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart->Init.OverSampling = UART_OVERSAMPLING_16;
    //hal_xx_init()内部回调hal_xx_mspinit，先完成协议层初始化，再完成硬件层，rcc时钟开启和底层gpio复用模式选择
    HAL_UART_Init(huart);
}
void my_uart_hal_send_string(UART_HandleTypeDef *huart, const uint8_t *buf, uint16_t len)
{
    if (huart == NULL || buf == NULL)
        return;
    HAL_UART_Transmit(huart, (uint8_t *)buf, len, HAL_MAX_DELAY);
}

void my_uart_hal_receive_string(UART_HandleTypeDef *huart, uint8_t *buf, uint16_t len)
{
    if (huart == NULL || buf == NULL)
        return;
    HAL_UART_Receive(huart, buf, len, HAL_MAX_DELAY);
}
```

### 5.2 寄存器控制

寄存器地址

SR:		/1. TXE 发送为空，允许写入，写入DR清理标志位	

​			/2. TC 传输完成，允许关闭串口，读SR写DE清理

​			/3. EXNE IDLE **PE**奇偶校验使能

**BRR**:	8+4 构成，数值逻辑是 PCLK/(16*braude)

CR1:	/1. **OVER8** 过采样位 UE使能 **M** 长度 

​			/2. PEIE **TEXIE** TCIE **RXNEIE** IDLEIE **TE**允许传输 **RE**允许接受 SBK发送break帧用于LIN通讯

CR2:	**STOP** 停止位长度

CR3:	DMA控制

![](D:\coding_codes\stm32f407\learning_logs\resources\UASRT_procedure.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\USART_TX1.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\USART_TX2.png)

```c
//.h文件
#define MY_USART1_BASE 0x40011000UL
#define MY_USART2_BASE 0x40004400UL
#define MY_USART6_BASE 0x40011400UL

#define MY_USART1 ((uart_register_t *)MY_USART1_BASE)
#define MY_USART2 ((uart_register_t *)MY_USART2_BASE)
#define MY_USART6 ((uart_register_t *)MY_USART6_BASE)

#define USART_SR_UE	0x00002000UL

typedef struct {
    volatile uint32_t sr;
    volatile uint32_t dr;
    volatile uint32_t brr;
    volatile uint32_t cr1;
    volatile uint32_t cr2;
    volatile uint32_t cr3;
    volatile uint32_t gtpr;
} uart_register_t;

//.c文件中
/* HAL mapping:
 *   BRR           <-> UART_InitTypeDef.BaudRate
 *   CR1.M         <-> UART_InitTypeDef.WordLength
 *   CR1.PCE/PS    <-> UART_InitTypeDef.Parity
 *   CR1.TE/RE     <-> UART_InitTypeDef.Mode
 *   CR1.OVER8     <-> UART_InitTypeDef.OverSampling
 *   CR2.STOP      <-> UART_InitTypeDef.StopBits
 *   CR3.CTSE/RTSE <-> UART_InitTypeDef.HwFlowCtl
 *   SR.TXE/RXNE   <-> HAL_UART_Transmit/Receive status polling
 */

static void my_uart_set_brr_16x(uart_register_t *uart, uint32_t pclk, uint32_t baudrate)
{
    uint64_t usartdiv;
    uint32_t mantissa;
    uint32_t fraction;

    /* USARTDIV = PCLK / (16 * baud)，放大 100 倍计算 */
    //小数部分uasrtdeiv是*100之后，对应转化存储到寄存器的位的单位就是1/16*100，那么久要转化进制，即*16U，再四舍五入+50U，最后是、100U取整数
    usartdiv = ((uint64_t)pclk * 25U) / (4U * (uint64_t)baudrate);
    mantissa = (uint32_t)(usartdiv / 100U);
    fraction = (uint32_t)((((usartdiv % 100U) * 16U) + 50U) / 100U);

    uart->brr = (mantissa << 4U) + (fraction & 0xF0U) + (fraction & 0x0FU);
}

static void my_uart_set_brr_8x(uart_register_t *uart, uint32_t pclk, uint32_t baudrate)
{
    uint64_t usartdiv;
    uint32_t mantissa;
    uint32_t fraction;

    /* USARTDIV = PCLK / (8 * baud)，放大 100 倍计算 */
    usartdiv = ((uint64_t)pclk * 25U) / (2U * (uint64_t)baudrate);
    mantissa = (uint32_t)(usartdiv / 100U);
    fraction = (uint32_t)((((usartdiv % 100U) * 8U) + 50U) / 100U);

    uart->brr = (mantissa << 4U) + ((fraction & 0xF8U) << 1U) + (fraction & 0x07U);
}

void my_uart_init(uart_register_t *uart, uint32_t pclk, uint32_t baudrate)
{
    /* 默认配置：8 位数据、1 停止位、无校验、TX+RX、16 倍过采样、无流控 */
    my_uart_init_full(uart, pclk, baudrate,
                      MY_UART_WORDLENGTH_8B,
                      MY_UART_STOPBITS_1,
                      MY_UART_PARITY_NONE,
                      MY_UART_MODE_TX_RX,
                      MY_UART_OVERSAMPLING_16,
                      MY_UART_HWCONTROL_NONE);
}

void my_uart_init_full(uart_register_t *uart,
                       uint32_t pclk,
                       uint32_t baudrate,
                       uint32_t word_length,
                       uint32_t stop_bits,
                       uint32_t parity,
                       uint32_t mode,
                       uint32_t oversampling,
                       uint32_t hw_flow)
{
    if (uart == 0U)
        return;

    /* 先清零配置寄存器 */
    uart->cr1 = 0U;
    uart->cr2 = 0U;
    uart->cr3 = 0U;

    /* BRR：对应 HAL 的 Init.BaudRate */
    if (oversampling == MY_UART_OVERSAMPLING_8)
        my_uart_set_brr_8x(uart, pclk, baudrate);
    else
        my_uart_set_brr_16x(uart, pclk, baudrate);

    /* CR1.M：对应 HAL 的 Init.WordLength */
    uart->cr1 |= word_length;

    /* CR1.PCE/PS：对应 HAL 的 Init.Parity */
    uart->cr1 |= parity;

    /* CR1.TE/RE：对应 HAL 的 Init.Mode */
    uart->cr1 |= mode;

    /* CR1.OVER8：对应 HAL 的 Init.OverSampling */
    if (oversampling == MY_UART_OVERSAMPLING_8)
        uart->cr1 |= 0x8000U;

    /* CR2.STOP：对应 HAL 的 Init.StopBits */
    uart->cr2 |= stop_bits;

    /* CR3.CTSE/RTSE：对应 HAL 的 Init.HwFlowCtl */
    uart->cr3 |= hw_flow;

    /* CR1.UE：最后使能 UART */
    uart->cr1 |= 0x2000U;
}

void my_uart_send_byte(uart_register_t *uart, uint8_t data)
{
    while ((uart->sr & 0x0080U) == 0U) /* TXE */
    {
    }
    uart->dr = data;
}

uint8_t my_uart_receive_byte(uart_register_t *uart)
{
    while ((uart->sr & 0x0020U) == 0U) /* RXNE */
    {
    }
    return (uint8_t)uart->dr;
}
```

## 6. IIC

### 6.1 HAL库

```c
void my_i2c_hal_init(I2C_HandleTypeDef *hi2c, I2C_TypeDef *instance, uint32_t speed)
{
    if (hi2c == NULL)
        return;
    hi2c->Instance = instance;
    hi2c->Init.ClockSpeed = speed;	//设置时钟为标准模式100K或者快速模式400k
    hi2c->Init.DutyCycle = I2C_DUTYCYCLE_2;	//设置快速模式下的占空比
    hi2c->Init.OwnAddress1 = 0U;	//配置本机地址
    hi2c->Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;	//配置地址长度
    hi2c->Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;	//是否使用双地址，如果是，OwnAddress2也启用
    hi2c->Init.OwnAddress2 = 0U;
    hi2c->Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;	//对于接受到的广播地址是否响应
    hi2c->Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;	//是否启用时钟延长，是低速从设备让主机等待的机制
    HAL_I2C_Init(hi2c);
}

void my_i2c_hal_write_reg(I2C_HandleTypeDef *hi2c, uint8_t dev_addr, uint8_t reg_addr,
                          uint8_t *buf, uint16_t len)
{
    uint8_t tmp[256];
    uint16_t i;
    //将目的地址和消息结合到缓冲区一起
    tmp[0] = reg_addr;
    for (i = 0U; i < len && (i + 1U) < sizeof(tmp); i++)
        tmp[i + 1U] = buf[i];
    HAL_I2C_Master_Transmit(hi2c, (uint16_t)(dev_addr << 1U), tmp, len + 1U, HAL_MAX_DELAY);
}
```

### 6.2 寄存器

CR1:	基本的信号的传输，默认是0x00 PEC 包错误检查 ENGC广播应答使能 ACK STOP START NOSTRETCH 禁止时钟延长 PE使能

CR2:	DMA IT FREQ外设时钟频率

OAR1:	OARMODE ADDR1

OAR2: 	ADDR2 ENDUAL使能双地址

SR1:	TXE RXNE STOPF ADD10 10bit地址发送完成（10bit是start之后第一个数据帧包含11110xx0其中xx是相较于8bit多出的两位地址） BTF 字节传输完成 ADDR7bit地址发送完成 SB start传输完成

SR2:	MSL主从模式	

![image-20260909025133535](D:\coding_codes\stm32f407\learning_logs\resources\IIC_SLAVE_T.png)

EV1 ADDR=1 地址传输完成，随后开始传输，而TXE允许写入，当TXE=1且shiftR空时（EV3-1），写入数据直接到SHIFTR，即开始传输数据，随后都是TDR空，但是SHIFTR非空（EV3），**收到NA之后（EV3-2）释放控制权，停止，而接受到主机的STOP之后清理（但是在slave r模式下，是接受到STOP作为停止信号而不是NA）**（因为是slave不允许发送NA，只是接受master的NA）；10bit模式下RS重启动实现传输方向改变，因为默认slave是r模式

![image-20260909030135899](D:\coding_codes\stm32f407\learning_logs\resources\IIC_SLAVE_R.png)

**EV2 RXNE不是只受DR控制，还有ACK确认之后，因为底层逻辑是允许读，前提是确认收到**；EV4对应接受到的帧不是data而是STOP，**实际上P的位置就是接收到STOP**

![](D:\coding_codes\stm32f407\learning_logs\resources\IIC_MASTER_T.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\IIC_MASTER_R.png)

EV7_1 因为是对于最后一个数据的NA帧编程，是在最后一个数据开始节后的时候触发



```c
//.h文件
typedef struct {
    volatile uint32_t cr1;
    volatile uint32_t cr2;
    volatile uint32_t oar1;
    volatile uint32_t oar2;
    volatile uint32_t dr;
    volatile uint32_t sr1;
    volatile uint32_t sr2;
    volatile uint32_t ccr;
    volatile uint32_t trise;
} i2c_register_t;

//.c文件
typedef struct {
    volatile uint32_t cr1;
    volatile uint32_t cr2;
    volatile uint32_t oar1;
    volatile uint32_t oar2;
    volatile uint32_t dr;
    volatile uint32_t sr1;
    volatile uint32_t sr2;
    volatile uint32_t ccr;
    volatile uint32_t trise;
} i2c_register_t;
typedef struct {
    uint32_t clock_speed;       /* HAL: Init.ClockSpeed */
    uint32_t duty_cycle;        /* HAL: Init.DutyCycle */
    uint16_t own_address1;      /* HAL: Init.OwnAddress1 */
    uint8_t  own_address2;      /* HAL: Init.OwnAddress2 */
    uint32_t addressing_mode;   /* HAL: Init.AddressingMode */
    uint32_t dual_address_mode; /* HAL: Init.DualAddressMode */
    uint32_t general_call_mode; /* HAL: Init.GeneralCallMode */
    uint32_t no_stretch_mode;   /* HAL: Init.NoStretchMode */
} my_i2c_config_t;
#include "my_i2c.h"
#include "my_rcc.h"

/*
 * HAL mapping:
 *   CR1.PE      <-> HAL_I2C_Init()
 *   CR1.NOSTRETCH <-> I2C_InitTypeDef.NoStretchMode
 *   CR1.ENGC    <-> I2C_InitTypeDef.GeneralCallMode
 *   CR2.FREQ    <-> I2C input clock PCLK1
 *   OAR1.ADDMODE/ADD <-> I2C_InitTypeDef.AddressingMode/OwnAddress1
 *   OAR2.ENDUAL/ADD <-> I2C_InitTypeDef.DualAddressMode/OwnAddress2
 *   CCR.FS/DUTY/CCR <-> I2C_InitTypeDef.ClockSpeed/DutyCycle
 *   TRISE       <-> I2C timing rise time
 */

void my_i2c_enable_clock(i2c_register_t *i2c)
{
    if (i2c == MY_I2C1)
        my_rcc_apb1_enable(MY_RCC_I2C1EN);
    else if (i2c == MY_I2C2)
        my_rcc_apb1_enable(0x00400000UL); /* I2C2EN */
    else if (i2c == MY_I2C3)
        my_rcc_apb1_enable(0x00800000UL); /* I2C3EN */
}

void my_i2c_init(i2c_register_t *i2c, uint32_t pclk1, uint32_t speed)
{
    my_i2c_config_t cfg;

    cfg.clock_speed = speed;
    cfg.duty_cycle = MY_I2C_DUTY_2;
    cfg.own_address1 = 0U;
    cfg.own_address2 = 0U;
    cfg.addressing_mode = MY_I2C_ADDRESSING_7BIT;
    cfg.dual_address_mode = MY_I2C_DUALADDR_DISABLE;
    cfg.general_call_mode = MY_I2C_GENERALCALL_DISABLE;
    cfg.no_stretch_mode = MY_I2C_NOSTRETCH_DISABLE;

    my_i2c_init_full(i2c, pclk1, &cfg);
}

void my_i2c_init_full(i2c_register_t *i2c, uint32_t pclk1, my_i2c_config_t *cfg)
{
    uint32_t freq;
    uint32_t ccr;
    uint32_t cr1;

    if (i2c == 0U || cfg == 0U)
        return;

    my_i2c_enable_clock(i2c);

    cr1 = i2c->cr1 & 0x00000001U; /* 保留 PE 状态 */
    i2c->cr1 = 0U;
    i2c->cr2 = 0U;
    i2c->oar1 = 0U;
    i2c->oar2 = 0U;

    /* CR2.FREQ：I2C 输入时钟 MHz */
    freq = pclk1 / 1000000U;
    if (freq == 0U) freq = 1U;
    i2c->cr2 = freq & 0x3FU;

    /* OAR1：自身地址 + 寻址模式 */
    if (cfg->addressing_mode == MY_I2C_ADDRESSING_10BIT)
    {
        i2c->oar1 |= 0x8000U; /* ADDMODE=1 */
        i2c->oar1 |= (uint32_t)cfg->own_address1 & 0x03FFU;
    }
    else
    {
        i2c->oar1 |= ((uint32_t)cfg->own_address1 & 0x007FU) << 1U;
    }

    /* OAR2：双地址模式 */
    if (cfg->dual_address_mode != 0U)
    {
        i2c->oar2 = 0x0001U; /* ENDUAL=1 */
        i2c->oar2 |= ((uint32_t)cfg->own_address2 & 0x007FU) << 1U;
    }

    /* CCR：时钟控制 */
    if (cfg->clock_speed <= 100000U)
    {
        ccr = pclk1 / (cfg->clock_speed * 2U);
        i2c->ccr = ccr & 0x0FFFU;      /* FS=0, standard mode */
    }
    else
    {
        i2c->ccr = 0x8000U; /* FS=1 */
        if (cfg->duty_cycle == MY_I2C_DUTY_16_9)
        {
            ccr = pclk1 / (cfg->clock_speed * 25U);
            i2c->ccr |= 0x4000U; /* DUTY=1 */
        }
        else
        {
            ccr = pclk1 / (cfg->clock_speed * 3U);
        }
        i2c->ccr |= ccr & 0x0FFFU;
    }

    if (ccr == 0U) ccr = 1U;
    i2c->ccr = (i2c->ccr & 0xF000U) | (ccr & 0x0FFFU);

    /* TRISE：上升时间 */
    if (cfg->clock_speed <= 100000U)
        i2c->trise = (freq + 1U) & 0x3FU;
    else
        i2c->trise = (((freq * 300U) / 1000U) + 1U) & 0x3FU;

    /* CR1：NOSTRETCH、ENGC、PE */
    if (cfg->no_stretch_mode != 0U)
        i2c->cr1 |= 0x0080U;
    if (cfg->general_call_mode != 0U)
        i2c->cr1 |= 0x0020U;
    i2c->cr1 |= (cr1 & 0x00000001U);
    i2c->cr1 |= 0x0001U; /* PE */
}

void my_i2c_start(i2c_register_t *i2c)
{
    i2c->cr1 |= 0x0100U; /* START */
    while ((i2c->sr1 & 0x0001U) == 0U) /* SB */
    {
    }
}

void my_i2c_stop(i2c_register_t *i2c)
{
    i2c->cr1 |= 0x0200U; /* STOP */
}

void my_i2c_send_addr(i2c_register_t *i2c, uint8_t addr, uint8_t rw)
{
    i2c->dr = (uint8_t)((addr << 1U) | (rw & 0x01U));
    while ((i2c->sr1 & 0x0002U) == 0U) /* ADDR */
    {
    }
    (void)i2c->sr2;
}

void my_i2c_send_byte(i2c_register_t *i2c, uint8_t data)
{
    while ((i2c->sr1 & 0x0080U) == 0U) /* TXE */
    {
    }
    i2c->dr = data;
    while ((i2c->sr1 & 0x0004U) == 0U) /* BTF */
    {
    }
}

uint8_t my_i2c_receive_byte(i2c_register_t *i2c, uint8_t nack)
{
    if (nack == 0U)
        i2c->cr1 |= 0x0400U;  /* ACK */
    else
        i2c->cr1 &= ~0x0400U; /* NACK */
    while ((i2c->sr1 & 0x0040U) == 0U) /* RXNE */
    {
    }
    return (uint8_t)i2c->dr;
}

void my_i2c_write_reg(i2c_register_t *i2c, uint8_t dev_addr, uint8_t reg_addr, uint8_t *buf, uint16_t len)
{
    uint16_t i;
    my_i2c_start(i2c);
    my_i2c_send_addr(i2c, dev_addr, 0U);
    my_i2c_send_byte(i2c, reg_addr);
    for (i = 0U; i < len; i++)
        my_i2c_send_byte(i2c, buf[i]);
    my_i2c_stop(i2c);
}

void my_i2c_read_reg(i2c_register_t *i2c, uint8_t dev_addr, uint8_t reg_addr, uint8_t *buf, uint16_t len)
{
    uint16_t i;
    my_i2c_start(i2c);
    my_i2c_send_addr(i2c, dev_addr, 0U);
    my_i2c_send_byte(i2c, reg_addr);
    my_i2c_start(i2c);
    my_i2c_send_addr(i2c, dev_addr, 1U);
    for (i = 0U; i < len; i++)
    {
        uint8_t nack = (i == (len - 1U)) ? 1U : 0U;
        buf[i] = my_i2c_receive_byte(i2c, nack);
    }
    my_i2c_stop(i2c);
}
```

## 7. CAN

### 7.1 HAL

| 模式           | 控制位状态 (`INRQ`, `SLEEP`) | 描述                                                         |
| :------------- | :--------------------------- | :----------------------------------------------------------- |
| **初始化模式** | `1`, `0`                     | - **用途**：进行CAN外设的配置，如位时序 (`CAN_BTR`) 和过滤器等。 - **特点**：**禁止报文的接收和发送**。 - **确认**：进入后，硬件会将 `CAN_MSR` 寄存器的 `INAK` 位置1作为确认。 |
| **正常模式**   | `0`, `0`                     | - **用途**：CAN控制器的**正常运行状态**，用于接收和发送报文。 - **确认**：当 `INAK` 和 `SLAK` 位都为0时，表示处于正常模式。 |
| **睡眠模式**   | 任意, `1`                    | - **用途**：**低功耗模式**，节省电能。 - **特点**：bxCAN的**时钟停止**，但软件仍可访问邮箱寄存器。 - **默认状态**：**硬件复位后，bxCAN默认进入睡眠模式**。 - **确认**：进入后，硬件会将 `CAN_MSR` 寄存器的 `SLAK` 位置1作为确认。 |

TEST MODE

​	/1. silent 回读 读总线 发1总线（1是隐形）

​	/2. koop-back 回读 不读总线 发总线

​	/3. 回读 不读总线 发1总线![](D:\coding_codes\stm32f407\learning_logs\resources\CAN_MODE.png)

MAILBOX_STATE

​	/1. TXRQ之后进入pending，不再对邮箱有写权利

​	/2. ABRQ之后都是回到empty，在transmit过程中

​	/3. TME transmit mailbox empty	RQCP 请求完成，是对于TXRQ和ABRQ

![](D:\coding_codes\stm32f407\learning_logs\resources\CAN_MAILBOX_STATE.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\CAN_FIFO_RECEIVE.png)



FILTER

​	/1. 28个过滤器各自两个寄存器：在mask模式下第二个作为mask，在id模式下作为双id

​	/2. 每个过滤器选择scale模式，全字为扩展帧，半字为标准帧，倍增过滤器

​	/3. mapping是在TIR的帧的地址，只是同样的地址的映射，不是存储原本的帧

![](D:\coding_codes\stm32f407\learning_logs\resources\CAN_FILTER.png)

FRAME

​	/1. RTR 遥控帧 IDE 扩展帧

![](D:\coding_codes\stm32f407\learning_logs\resources\CAN_FRAME.png)

```c
#include "my_can_hal.h"

void my_can_hal_init(CAN_HandleTypeDef *hcan, CAN_TypeDef *instance)
{
    CAN_FilterTypeDef filter = {0};

    if (hcan == NULL)
        return;
    hcan->Instance = instance;

    /* -------- CAN_InitTypeDef：位时序与工作模式 -------- */
    hcan->Init.Prescaler = 6U;
    // 性质：位时序数值  | 作用：波特率预分频，tq = APB1_CLK/Prescaler
    // 可选：1~1024；本例配合 BS1/BS2 在 APB1=42MHz 时约 500kbit/s

    hcan->Init.Mode = CAN_MODE_NORMAL;
    // 性质：枚举模式    | 作用：选择正常/回环/静默，决定是否对外收发
    // 可选：CAN_MODE_NORMAL / CAN_MODE_LOOPBACK / CAN_MODE_SILENT / CAN_MODE_SILENT_LOOPBACK
    // 本例：正常模式，非回环，非静默

    hcan->Init.SyncJumpWidth = CAN_SJW_1TQ;
    // 性质：位时序枚举  | 作用：同步跳转宽度，用于同步调节采样点
    // 可选：CAN_SJW_1TQ / CAN_SJW_2TQ / CAN_SJW_3TQ / CAN_SJW_4TQ

    hcan->Init.TimeSeg1 = CAN_BS1_13TQ;
    // 性质：位时序枚举  | 作用：时间段1，采样点之前的时间段
    // 可选：CAN_BS1_1TQ ~ CAN_BS1_16TQ

    hcan->Init.TimeSeg2 = CAN_BS2_2TQ;
    // 性质：位时序枚举  | 作用：时间段2，采样点之后；位时间 = 1+BS1+BS2 个 tq
    // 可选：CAN_BS2_1TQ ~ CAN_BS2_8TQ
    // 完整周期是 SJW 同步 + tq1 + tq2

    hcan->Init.TimeTriggeredMode = DISABLE;
    // 性质：功能开关    | 作用：时间触发通信模式(TTCM)
    // 可选：ENABLE / DISABLE；本例禁用

    hcan->Init.AutoBusOff = ENABLE;
    // 性质：功能开关    | 作用：自动总线关闭；发送错误计数超限后自动离线，总线空闲后自动恢复
    // 可选：ENABLE / DISABLE

    hcan->Init.AutoWakeUp = DISABLE;
    // 性质：功能开关    | 作用：检测到总线活动时自动退出睡眠
    // 可选：ENABLE / DISABLE；本例禁用

    hcan->Init.AutoRetransmission = ENABLE;
    // 性质：功能开关    | 作用：自动重传(对应 NART 取反)；仲裁失败/出错后是否重发
    // 可选：ENABLE / DISABLE

    hcan->Init.ReceiveFifoLocked = DISABLE;
    // 性质：功能开关    | 作用：接收 FIFO 锁定；满时覆盖旧帧(DISABLE)或丢弃新帧(ENABLE)
    // 可选：ENABLE / DISABLE；本例取消锁定，FIFO 满了直接覆盖

    hcan->Init.TransmitFifoPriority = DISABLE;
    // 性质：功能开关    | 作用：发送优先级；DISABLE 按 ID 仲裁，ENABLE 按请求先后(FIFO)
    // 可选：ENABLE / DISABLE；本例不按发送顺序而是按 ID 号仲裁

    HAL_CAN_Init(hcan);

    /* -------- CAN_FilterTypeDef：验收过滤器（收帧前必须配置） -------- */
    filter.FilterBank = 0U;
    // 性质：过滤器编号  | 作用：选用哪一组过滤器寄存器
    // 可选：CAN1 常用 0~13，与 CAN2 共享时 SlaveStartFilterBank 之后给 CAN2

    filter.FilterMode = CAN_FILTERMODE_IDMASK;
    // 性质：过滤算法    | 作用：掩码匹配(关心位由 Mask 决定) 或 列表匹配(精确 ID)
    // 可选：CAN_FILTERMODE_IDMASK / CAN_FILTERMODE_IDLIST

    filter.FilterScale = CAN_FILTERSCALE_32BIT;
    // 性质：位宽选择    | 作用：一组过滤器按 32bit 还是两个 16bit 使用
    // 可选：CAN_FILTERSCALE_16BIT / CAN_FILTERSCALE_32BIT
    //掩码模式下，id+mask 32bit，列表模式下，id1+id2 32bit，mask作为第二个 id
    filter.FilterIdHigh = 0U;
    // 性质：ID 寄存器高半字 | 作用：掩码模式下为期望 ID 高 16bit；列表模式下为 ID
    // 可选：0~0xFFFF；本例 0 配合 Mask=0 表示不关心任何位

    filter.FilterIdLow = 0U;
    // 性质：ID 寄存器低半字 | 作用：同上低 16bit（含 IDE/RTR 等位域布局）
    // 可选：0~0xFFFF

    filter.FilterMaskIdHigh = 0U;
    // 性质：掩码/列表高半字 | 作用：掩码模式 1=必须匹配，0=忽略；列表模式为第二个 ID
    // 可选：0~0xFFFF；全 0 = 接收全部帧

    filter.FilterMaskIdLow = 0U;
    // 性质：掩码/列表低半字 | 作用：同上
    // 可选：0~0xFFFF

    filter.FilterFIFOAssignment = CAN_RX_FIFO0;
    // 性质：路由选择    | 作用：匹配的帧进入 FIFO0 还是 FIFO1
    // 可选：CAN_RX_FIFO0 / CAN_RX_FIFO1

    filter.FilterActivation = ENABLE;
    // 性质：功能开关    | 作用：是否启用该过滤器组
    // 可选：ENABLE / DISABLE

    filter.SlaveStartFilterBank = 14U;
    // 性质：分区边界    | 作用：双 CAN 时从哪一组开始分给 CAN2
    // 可选：0~27；单 CAN1 常用 14

    HAL_CAN_ConfigFilter(hcan, &filter);
    HAL_CAN_Start(hcan); /* 退出初始化，进入 Normal，才能收发 */
}

uint8_t my_can_hal_send(CAN_HandleTypeDef *hcan, my_can_hal_msg_t *msg)
{
    CAN_TxHeaderTypeDef tx = {0};
    uint32_t mailbox = 0U;
    if (hcan == NULL || msg == NULL)
        return 0U;

    tx.StdId = msg->id;
    // 性质：标准帧 ID   | 作用：11bit 标识符；扩展帧时本字段不用
    // 可选：0~0x7FF

    tx.ExtId = msg->id;
    // 性质：扩展帧 ID   | 作用：29bit 标识符；标准帧时本字段不用
    // 可选：0~0x1FFFFFFF

    // 判断标准帧 / 扩展帧
    tx.IDE = (msg->ide != 0U) ? CAN_ID_EXT : CAN_ID_STD;
    // 性质：帧格式枚举  | 作用：选择 StdId 还是 ExtId
    // 可选：CAN_ID_STD / CAN_ID_EXT

    // 判断数据帧 / 远程帧
    tx.RTR = (msg->rtr != 0U) ? CAN_RTR_REMOTE : CAN_RTR_DATA;
    // 性质：帧类型枚举  | 作用：数据帧携带 data[]，远程帧请求对方数据
    // 可选：CAN_RTR_DATA / CAN_RTR_REMOTE

    // 数据长度
    tx.DLC = msg->dlc;
    // 性质：长度数值    | 作用：有效数据字节数
    // 可选：0~8

    if (HAL_CAN_AddTxMessage(hcan, &tx, msg->data, &mailbox) != HAL_OK)
        return 0U;
    return 1U;
}

uint8_t my_can_hal_receive(CAN_HandleTypeDef *hcan, my_can_hal_msg_t *msg)
{
    CAN_RxHeaderTypeDef rx = {0};
    if (hcan == NULL || msg == NULL)
        return 0U;
    if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &rx, msg->data) != HAL_OK)
        return 0U;
    msg->id = (rx.IDE == CAN_ID_STD) ? rx.StdId : rx.ExtId;
    msg->ide = (rx.IDE == CAN_ID_STD) ? 0U : 1U;
    msg->rtr = (rx.RTR == CAN_RTR_DATA) ? 0U : 1U;
    msg->dlc = rx.DLC;
    return 1U;
}

```

### 7.2 寄存器

```c
//.h文件

#define MY_CAN1_BASE 0x40006400UL
#define MY_CAN2_BASE 0x40006800UL

#define MY_CAN1 ((can_register_t *)MY_CAN1_BASE)
#define MY_CAN2 ((can_register_t *)MY_CAN2_BASE)

typedef struct {
    volatile uint32_t tir;
    volatile uint32_t tdtr;
    volatile uint32_t tdlr;
    volatile uint32_t tdhr;
} can_tx_mailbox_t;

typedef struct {
    volatile uint32_t rir;
    volatile uint32_t rdtr;
    volatile uint32_t rdlr;
    volatile uint32_t rdhr;
} can_rx_mailbox_t;

typedef struct {
    volatile uint32_t mcr;
    volatile uint32_t msr;
    volatile uint32_t tsr;
    volatile uint32_t rf0r;
    volatile uint32_t rf1r;
    volatile uint32_t ier;
    volatile uint32_t esr;
    volatile uint32_t btr;
    uint32_t          reserved0[88];
    can_tx_mailbox_t  tx_mailbox[3];
    can_rx_mailbox_t  rx_mailbox[2];
} can_register_t;

typedef struct {
    uint32_t id;     /* [ID] 标准0~0x7FF / 扩展0~0x1FFFFFFF */
    uint8_t  ide;    /* [帧格式] 0=标准帧，1=扩展帧 */
    uint8_t  rtr;    /* [帧类型] 0=数据帧，1=远程帧 */
    uint8_t  dlc;    /* [长度] 有效字节 0~8 */
    uint8_t  data[8];/* [载荷] 数据帧内容；远程帧可忽略 */
} my_can_msg_t;

/* HAL mapping constants */
#define MY_CAN_MODE_NORMAL         0U
#define MY_CAN_MODE_LOOPBACK       1U
#define MY_CAN_MODE_SILENT         2U
#define MY_CAN_MODE_SILENT_LOOPBACK 3U

#define MY_CAN_DISABLE 0U
#define MY_CAN_ENABLE  1U

typedef struct {
    uint32_t prescaler;           /* [位时序] 预分频 1~1024；波特率=APB1/(Prescaler*(1+BS1+BS2)) */
    uint32_t mode;                /* [枚举] NORMAL/LOOPBACK/SILENT/SILENT_LOOPBACK */
    uint32_t sync_jump_width;     /* [位时序] SJW 1~4，同步调节采样点 */
    uint32_t time_seg1;           /* [位时序] BS1 1~16，采样点前 */
    uint32_t time_seg2;           /* [位时序] BS2 1~8，采样点后 */
    uint32_t time_triggered;      /* [开关] TTCM 时间触发；ENABLE/DISABLE */
    uint32_t auto_bus_off;        /* [开关] ABOM 自动总线关闭恢复 */
    uint32_t auto_wake_up;        /* [开关] AWUM 自动唤醒 */
    uint32_t auto_retransmit;     /* [开关] 自动重传；DISABLE 则置 NART */
    uint32_t receive_fifo_locked; /* [开关] RFLM；DISABLE=FIFO满覆盖旧帧 */
    uint32_t tx_fifo_priority;    /* [开关] TXFP；DISABLE=按ID仲裁，ENABLE=按请求顺序 */
} my_can_config_t;

//.c文件
#include "my_can.h"
#include "my_rcc.h"

/*
 * HAL mapping:
 *   MCR.INRQ      <-> HAL_CAN_Init() initialization mode
 *   MCR.SLEEP     <-> leave sleep to enter Normal (HAL clears SLEEP, waits SLAK)
 *   MCR.TXFP      <-> CAN_InitTypeDef.TransmitFifoPriority
 *   MCR.RFLM      <-> CAN_InitTypeDef.ReceiveFifoLocked
 *   MCR.NART      <-> CAN_InitTypeDef.AutoRetransmission
 *   MCR.AWUM      <-> CAN_InitTypeDef.AutoWakeUp
 *   MCR.ABOM      <-> CAN_InitTypeDef.AutoBusOff
 *   MCR.TTCM      <-> CAN_InitTypeDef.TimeTriggeredMode
 *   BTR.BRP       <-> CAN_InitTypeDef.Prescaler
 *   BTR.TS1/TS2   <-> CAN_InitTypeDef.TimeSeg1/TimeSeg2
 *   BTR.SJW       <-> CAN_InitTypeDef.SyncJumpWidth
 *   BTR.LBKM/SILM <-> CAN_InitTypeDef.Mode
 *   TSR.TME0/1/2  <-> HAL_CAN_GetTxMailboxesFreeLevel()  (bits 26/27/28)
 *   Filter banks  <-> HAL_CAN_ConfigFilter()
 *   TxMailbox     <-> CAN_TxHeaderTypeDef + HAL_CAN_AddTxMessage()
 *   RxMailbox     <-> CAN_RxHeaderTypeDef + HAL_CAN_GetRxMessage()
 */

/* Filter register block starts at CAN base + 0x200 */
#define MY_CAN_FMR_OFF   0x200UL
#define MY_CAN_FM1R_OFF  0x204UL
#define MY_CAN_FS1R_OFF  0x20CUL
#define MY_CAN_FFA1R_OFF 0x214UL
#define MY_CAN_FA1R_OFF  0x21CUL
#define MY_CAN_FR1_OFF   0x240UL
#define MY_CAN_FR2_OFF   0x244UL

#define MY_CAN_REG(can, off) (*(volatile uint32_t *)((uint32_t)(can) + (off)))

#define MY_CAN_TSR_TME0  0x04000000UL
#define MY_CAN_TSR_TME1  0x08000000UL
#define MY_CAN_TSR_TME2  0x10000000UL

void my_can_enable_clock(can_register_t *can)
{
    if (can == MY_CAN1)
        my_rcc_apb1_enable(MY_RCC_CAN1EN);
    else if (can == MY_CAN2)
        my_rcc_apb1_enable(0x04000000UL); /* CAN2EN */
}

void my_can_filter_accept_all(can_register_t *can)
{
    if (can == 0U)
        return;

    /* Enter filter init mode */
    MY_CAN_REG(can, MY_CAN_FMR_OFF) |= 0x00000001UL; /* FINIT */

    /* Deactivate filter bank 0, then configure as 32-bit ID mask, FIFO0 */
    //是类似GPIO的二维外设，使用掩码模式，32bit ID+Mask，Mask=0表示不关心任何位
    MY_CAN_REG(can, MY_CAN_FA1R_OFF) &= ~0x00000001UL;  /* deactivate bank 0 */
    MY_CAN_REG(can, MY_CAN_FS1R_OFF) |= 0x00000001UL;  /* 32-bit scale */
    MY_CAN_REG(can, MY_CAN_FM1R_OFF) &= ~0x00000001UL; /* identifier mask mode */
    MY_CAN_REG(can, MY_CAN_FR1_OFF) = 0U;              /* ID = 0 */
    MY_CAN_REG(can, MY_CAN_FR2_OFF) = 0U;              /* Mask = 0 => accept all */
    MY_CAN_REG(can, MY_CAN_FFA1R_OFF) &= ~0x00000001UL; /* 选择分配对象FIFO0 */
    MY_CAN_REG(can, MY_CAN_FA1R_OFF) |= 0x00000001UL;  /* activate bank 0 */

    /* Leave filter init mode */
    MY_CAN_REG(can, MY_CAN_FMR_OFF) &= ~0x00000001UL;
}

void my_can_init(can_register_t *can, uint32_t prescaler, uint32_t bs1, uint32_t bs2)
{
    my_can_config_t cfg;
    /* -------- 简易入口默认填充 my_can_config_t（同 HAL CAN_InitTypeDef） -------- */
    cfg.prescaler = prescaler;
    // 性质：位时序数值  | 作用：预分频，写入 BTR.BRP+1；波特率=APB1/(Prescaler*(1+BS1+BS2))
    // 可选：1~1024
    cfg.mode = MY_CAN_MODE_NORMAL;
    // 性质：枚举模式    | 作用：正常收发；可选 NORMAL / LOOPBACK / SILENT / SILENT_LOOPBACK
    cfg.sync_jump_width = 1U;
    // 性质：位时序数值  | 作用：同步跳转宽度 SJW，调节采样点；可选 1~4
    //整个采样周期长度时squence = 1 + BS1 + BS2，采样点在 BS1 末尾，BS2 之后是总线空闲期。同步跳转宽度 SJW 是为了应对总线抖动，允许采样点前后各 SJW 个 tq 的时间内调整采样点位置。
    cfg.time_seg1 = bs1;
    // 性质：位时序数值  | 作用：时间段1(BS1)，采样点前；可选 1~16
    cfg.time_seg2 = bs2;
    // 性质：位时序数值  | 作用：时间段2(BS2)，采样点后；可选 1~8
    cfg.time_triggered = MY_CAN_DISABLE;
    // 性质：功能开关    | 作用：时间触发 TTCM；可选 ENABLE / DISABLE
    cfg.auto_bus_off = MY_CAN_ENABLE;
    // 性质：功能开关    | 作用：自动总线关闭 ABOM；错误离线后自动恢复
    cfg.auto_wake_up = MY_CAN_DISABLE;
    // 性质：功能开关    | 作用：自动唤醒 AWUM
    cfg.auto_retransmit = MY_CAN_ENABLE;
    // 性质：功能开关    | 作用：自动重传；DISABLE 则置 NART
    cfg.receive_fifo_locked = MY_CAN_DISABLE;
    // 性质：功能开关    | 作用：FIFO 锁定 RFLM；DISABLE=满则覆盖旧帧
    cfg.tx_fifo_priority = MY_CAN_DISABLE;
    // 性质：功能开关    | 作用：发送优先级 TXFP；DISABLE=按 ID 仲裁，ENABLE=按请求顺序

    my_can_init_full(can, &cfg);
}

void my_can_init_full(can_register_t *can, my_can_config_t *cfg)
{
    uint32_t mcr;

    if (can == 0U || cfg == 0U)
        return;

    my_can_enable_clock(can);

    /* Exit sleep first (reset leaves SLEEP=1), then enter init — same as HAL_CAN_Init */
    // MCR.SLEEP=0退出SLEEP，等待SLAK=0确定已退出
    can->mcr &= ~0x00000002U; /* SLEEP */
    while ((can->msr & 0x00000002U) != 0U) /* SLAK */
    {
    }
    // MCR.INRQ=1进入初始化模式，等待INAK=1确认已进入
    can->mcr |= 0x00000001U; /* INRQ */
    while ((can->msr & 0x00000001U) == 0U) /* INAK */
    {
    }

    /* MCR options while INRQ remains set */
    mcr = can->mcr;
    mcr &= ~(0x00000004U | 0x00000008U | 0x00000010U |
             0x00000020U | 0x00000040U | 0x00000080U);
    mcr |= 0x00000001U; /* keep INRQ */

    if (cfg->tx_fifo_priority != 0U)
        mcr |= 0x00000004U; /* TXFP */
    if (cfg->receive_fifo_locked != 0U)
        mcr |= 0x00000008U; /* RFLM */
    if (cfg->auto_retransmit == MY_CAN_DISABLE)
        mcr |= 0x00000010U; /* NART */
    if (cfg->auto_wake_up != 0U)
        mcr |= 0x00000020U; /* AWUM */
    if (cfg->auto_bus_off != 0U)
        mcr |= 0x00000040U; /* ABOM */
    if (cfg->time_triggered != 0U)
        mcr |= 0x00000080U; /* TTCM */
    can->mcr = mcr;

    /* BTR: bit timing (only writable in init mode) */
    //配置采样周期以及采样点位置
    can->btr = 0U;
    can->btr |= ((cfg->prescaler - 1U) & 0x000003FFUL) << 0U;
    can->btr |= (((cfg->time_seg1 - 1U) & 0x0FU) << 16U);
    can->btr |= (((cfg->time_seg2 - 1U) & 0x07U) << 20U);
    can->btr |= (((cfg->sync_jump_width - 1U) & 0x03U) << 24U);
    //testmode 是静默还是回环
    if (cfg->mode == MY_CAN_MODE_LOOPBACK)
        can->btr |= 0x40000000UL; /* LBKM */
    else if (cfg->mode == MY_CAN_MODE_SILENT)
        can->btr |= 0x80000000UL; /* SILM */
    else if (cfg->mode == MY_CAN_MODE_SILENT_LOOPBACK)
        can->btr |= 0xC0000000UL;

    /* Leave initialization mode -> Normal */
    can->mcr &= ~0x00000001U; /* INRQ */
    while ((can->msr & 0x00000001U) != 0U)
    {
    }

    /* Default accept-all filter so RX demos can receive frames */
    my_can_filter_accept_all(can);
}

uint8_t my_can_send(can_register_t *can, my_can_msg_t *msg, uint32_t timeout)
{
    uint32_t mailbox;
    uint32_t tick = 0U;
    uint32_t tir;

    if (can == 0U || msg == 0U)
        return 0U;

    /* TME0/1/2 are bits 26/27/28 — NOT bits 0/1/2 (those are RQCP*) */
    while (((can->tsr & MY_CAN_TSR_TME0) == 0U) &&
           ((can->tsr & MY_CAN_TSR_TME1) == 0U) &&
           ((can->tsr & MY_CAN_TSR_TME2) == 0U))
    {
        tick++;
        if ((timeout != 0U) && (tick >= timeout))
            return 0U;
    }
    //选择空闲的邮箱，优先选择 mailbox0
    if ((can->tsr & MY_CAN_TSR_TME0) != 0U)
        mailbox = 0U;
    else if ((can->tsr & MY_CAN_TSR_TME1) != 0U)
        mailbox = 1U;
    else
        mailbox = 2U;
    //TIR先保存 ID 和 IDE、RTR 等信息，后续再写入数据和请求发送
    if (msg->ide == 0U)
        tir = (msg->id & 0x7FFUL) << 21U;
    else
        tir = ((msg->id & 0x1FFFFFFFUL) << 3U) | 0x00000004UL; /* IDE */

    if (msg->rtr != 0U)
        tir |= 0x00000002UL; /* RTR */

    can->tx_mailbox[mailbox].tdtr = msg->dlc & 0x0FU;   // 数据长度，是报文的有效字节数，0~8
    can->tx_mailbox[mailbox].tdlr = (uint32_t)msg->data[0] |
                                    ((uint32_t)msg->data[1] << 8U) |
                                    ((uint32_t)msg->data[2] << 16U) |
                                    ((uint32_t)msg->data[3] << 24U);
    can->tx_mailbox[mailbox].tdhr = (uint32_t)msg->data[4] |
                                    ((uint32_t)msg->data[5] << 8U) |
                                    ((uint32_t)msg->data[6] << 16U) |
                                    ((uint32_t)msg->data[7] << 24U);

    can->tx_mailbox[mailbox].tir = tir | 0x00000001UL; /* TXRQ */
    return 1U;
}

uint8_t my_can_receive(can_register_t *can, my_can_msg_t *msg, uint32_t timeout)
{
    uint32_t tick = 0U;
    can_rx_mailbox_t *rx;

    if (can == 0U || msg == 0U)
        return 0U;
    //等待 FIFO0 有数据接收，RF0N=0 表示 FIFO0 空
    while ((can->rf0r & 0x00000003UL) == 0U)
    {
        tick++;
        if ((timeout != 0U) && (tick >= timeout))
            return 0U;
    }
    //因为 FIFO0 只有一个接收邮箱，所以直接使用 rx_mailbox[0]，rx_mailbox[1] 是 FIFO1 的邮箱
    //之前说的有三个独立的发送邮箱，有两个三极的FIFO接收邮箱，FIFO是将三个邮箱统一为一个，所以寄存器上只有两个RIR
    //每个FIFO邮箱是三极的，是物理地址深度，逻辑地址完全重合，是硬件设计的三维地址
    rx = &can->rx_mailbox[0];
    msg->ide = ((rx->rir & 0x00000004UL) != 0U) ? 1U : 0U;
    msg->rtr = ((rx->rir & 0x00000002UL) != 0U) ? 1U : 0U;
    msg->id = (msg->ide == 0U) ? (rx->rir >> 21U) : (rx->rir >> 3U);
    msg->dlc = rx->rdtr & 0x0FU;
    msg->data[0] = (uint8_t)rx->rdlr;
    msg->data[1] = (uint8_t)(rx->rdlr >> 8U);
    msg->data[2] = (uint8_t)(rx->rdlr >> 16U);
    msg->data[3] = (uint8_t)(rx->rdlr >> 24U);
    msg->data[4] = (uint8_t)rx->rdhr;
    msg->data[5] = (uint8_t)(rx->rdhr >> 8U);
    msg->data[6] = (uint8_t)(rx->rdhr >> 16U);
    msg->data[7] = (uint8_t)(rx->rdhr >> 24U);

    can->rf0r |= 0x00000020UL; /* RFOM0 */
    return 1U;
}
```

## 8. ADC

### 8.1 HAL

1. 修改规则组和注入组的寄存器会打断
2. 规则通道有16个外部通道和3个内部通道
3. 注入组的长度和顺序是提前配置的，类似于中断的优先级，注入组的通道一直有信号，但是等到软件，EXTI，定时器的时候触发采样
4. 连续模式和扫描模式是可正交的
   1. scan,cont=00 只读第一个通道，结束之后停止
   2. 01，只读第一个通道，结束之后继续读
   3. 10，读整个组，读完停止
   4. 11，读整个组，读完后继续读
5. discont 和 cont 是独立的，discont是间断跳越
6. 一个ADC有两个分组，规则组有19个硬件确定的通道，但是可以编程哪些通道进入规则组，以及规则组长度
7. EXTI有多个中断向量，一个中断向量对应一个中断服务函数，而一个中断向量可以对应多个EXTI通道，而一个EXTI通道可以对应多个GPIO引脚
8. 可以通过降低adc的分辨率实现fast mdoe
9. 多adc同步模式下（规则组和注入组）：各adc转换的组长度一样，或者大于最长的adc长度；同时开始采样转换；不能同时对同一通道采样
10. 多adc交错模式下（规则组）：先后开始采样转换，但是转换不能在采样还在进行的时候执行；间隔max(DELAY,sampletime+2adcclk)，sampletime是前一个在采样的adc的时间，间隔指的是两个adc采样开始之间的时间间隔，同理rtos的时间间隔；只对于规则组，常见只是对一个通道采样
11. 多adc交叉复用模式下（注入组）：在cont下，有触发源就转换第一个adc实体组的所有channel，有第二个触发源转换第二个adc实体；在discont，有触发源转换第一个实体的所有ch，有第二个转换第二个实体
12. 多adc模式下是共用common寄存器，同时作为主adc1的寄存器，在规则组的结构是都存放在一个cdr下，所以需要dma，但是注入组的结果放在四个各自的寄存器下

```c
#include "my_adc_hal.h"

void my_adc_hal_msp_init(ADC_HandleTypeDef *hadc)
{
    GPIO_InitTypeDef gpio = {0};
    if (hadc->Instance == ADC1)
    {
        __HAL_RCC_ADC1_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();

        gpio.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3 |
                   GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7;
        // 性质：引脚掩码    | 作用：选择要初始化的引脚
        // 可选：GPIO_PIN_0 ~ GPIO_PIN_15 的按位或

        gpio.Mode = GPIO_MODE_ANALOG;
        // 性质：模式枚举    | 作用：模拟输入，供 ADC 采样
        // 可选：INPUT / OUTPUT_PP/OD / AF_PP/OD / ANALOG / IT_* / EVT_*

        gpio.Pull = GPIO_NOPULL;
        // 性质：上下拉枚举  | 作用：模拟脚通常无上下拉
        // 可选：GPIO_NOPULL / GPIO_PULLUP / GPIO_PULLDOWN

        HAL_GPIO_Init(GPIOA, &gpio);
    }
}

void my_adc_hal_init(ADC_HandleTypeDef *hadc, ADC_TypeDef *instance)
{
    hadc->Instance = instance;
    // 性质：硬件实例    | 作用：选择 ADC1/2/3
    // 可选：ADC1 / ADC2 / ADC3

    /* -------- ADC_InitTypeDef -------- */
    hadc->Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
    // 性质：时钟分频枚举 | 作用：ADC 时钟相对 PCLK2 的分频
    // 可选：ADC_CLOCK_SYNC_PCLK_DIV2 / DIV4 / DIV6 / DIV8

    hadc->Init.Resolution = ADC_RESOLUTION_12B;
    // 性质：分辨率枚举  | 作用：转换结果位数
    // 可选：ADC_RESOLUTION_12B / 10B / 8B / 6B

    hadc->Init.ScanConvMode = DISABLE;
    // 性质：功能开关    | 作用：扫描模式；多通道规则组时 ENABLE
    // 可选：ENABLE / DISABLE

    hadc->Init.ContinuousConvMode = DISABLE;
    // 性质：功能开关    | 作用：连续转换；DISABLE 则每次需软件/触发启动
    // 可选：ENABLE / DISABLE

    hadc->Init.DiscontinuousConvMode = DISABLE;
    // 性质：功能开关    | 作用：间断模式，每次只转部分通道
    // 可选：ENABLE / DISABLE（与 Continuous 互斥）

    hadc->Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
    // 性质：触发边沿    | 作用：无外部触发则软件启动
    // 可选：NONE / RISING / FALLING / RISINGFALLING

    hadc->Init.ExternalTrigConv = ADC_SOFTWARE_START;
    // 性质：触发源枚举  | 作用：选择外部触发事件；软件启动时填 SOFTWARE_START
    // 可选：ADC_SOFTWARE_START 或各 TIMx_CCx / EXTI 触发宏

    hadc->Init.DataAlign = ADC_DATAALIGN_RIGHT;
    // 性质：对齐方式    | 作用：结果在 DR 中左对齐或右对齐
    // 可选：ADC_DATAALIGN_RIGHT / ADC_DATAALIGN_LEFT

    hadc->Init.NbrOfConversion = 1;
    // 性质：序列长度    | 作用：规则组通道个数，写入 SQR1.L+1
    // 可选：1~16

    hadc->Init.DMAContinuousRequests = DISABLE;
    // 性质：功能开关    | 作用：DMA 连续请求(DDS)；单次 DMA 用 DISABLE
    // 可选：ENABLE / DISABLE

    hadc->Init.EOCSelection = ADC_EOC_SINGLE_CONV;
    // 性质：EOC 行为    | 作用：每次转换结束置 EOC，或整序列结束才置
    // 可选：ADC_EOC_SINGLE_CONV / ADC_EOC_SEQ_CONV

    my_adc_hal_msp_init(hadc);
    HAL_ADC_Init(hadc);
}

uint16_t my_adc_hal_read_channel(ADC_HandleTypeDef *hadc, uint32_t channel, uint32_t sample_time)
{
    ADC_ChannelConfTypeDef cfg = {0};

    cfg.Channel = channel;
    // 性质：通道号      | 作用：选择 ADCx_INn
    // 可选：ADC_CHANNEL_0 ~ ADC_CHANNEL_18（含内部通道）

    cfg.Rank = 1;
    // 性质：序列位置    | 作用：该通道在规则组中的次序
    // 可选：1~16（需 ≤ NbrOfConversion）

    cfg.SamplingTime = sample_time;
    // 性质：采样时间枚举 | 作用：采样电容充电时间，越大越稳但更慢
    // 可选：ADC_SAMPLETIME_3CYCLES ~ ADC_SAMPLETIME_480CYCLES

    HAL_ADC_ConfigChannel(hadc, &cfg);
    HAL_ADC_Start(hadc);
    if (HAL_ADC_PollForConversion(hadc, HAL_MAX_DELAY) != HAL_OK)
        return 0U;
    uint16_t value = (uint16_t)HAL_ADC_GetValue(hadc);
    HAL_ADC_Stop(hadc);
    return value;
}
```

### 8.2 寄存器

1. common 是muti多寄存器模式下主ADC的寄存器，共用
2. DOFR 是注入组的结果的偏移量，需要减去偏移量得到最后的值

![](D:\coding_codes\stm32f407\learning_logs\resources\ADC_R_MAP0.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\ADC_R_MAP1.png)

![](D:\coding_codes\stm32f407\learning_logs\resources\ADC_R_MAP2.png)

```c
//.h文件
typedef struct {
    volatile uint32_t sr;
    volatile uint32_t cr1;
    volatile uint32_t cr2;
    volatile uint32_t smpr1;
    volatile uint32_t smpr2;
    volatile uint32_t jofr[4];
    volatile uint32_t htr;
    volatile uint32_t ltr;
    volatile uint32_t sqr1;
    volatile uint32_t sqr2;
    volatile uint32_t sqr3;
    volatile uint32_t jsqr;
    volatile uint32_t jdr[4];
    volatile uint32_t dr;
} adc_register_t;

typedef struct {
    volatile uint32_t csr;
    volatile uint32_t ccr;
    volatile uint32_t cdr;
} adc_common_register_t;

/* HAL mapping constants */
#define MY_ADC_RESOLUTION_12B  0U
#define MY_ADC_RESOLUTION_10B  1U
#define MY_ADC_RESOLUTION_8B   2U
#define MY_ADC_RESOLUTION_6B   3U

#define MY_ADC_SCAN_DISABLE    0U
#define MY_ADC_SCAN_ENABLE     1U

#define MY_ADC_DATAALIGN_RIGHT 0U
#define MY_ADC_DATAALIGN_LEFT  1U

#define MY_ADC_EOC_SEQ         0U  /* EOCS=0, EOC at end of sequence */
#define MY_ADC_EOC_SINGLE      1U  /* EOCS=1, EOC after each conversion */

#define MY_ADC_CONT_DISABLE    0U
#define MY_ADC_CONT_ENABLE     1U

#define MY_ADC_DISC_DISABLE    0U
#define MY_ADC_DISC_ENABLE     1U

#define MY_ADC_DMA_DISABLE     0U
#define MY_ADC_DMA_ENABLE      1U

#define MY_ADC_DDS_DISABLE     0U
#define MY_ADC_DDS_ENABLE      1U

#define MY_ADC_SAMPLETIME_3CYCLES     0U
#define MY_ADC_SAMPLETIME_15CYCLES    1U
#define MY_ADC_SAMPLETIME_28CYCLES    2U
#define MY_ADC_SAMPLETIME_56CYCLES    3U
#define MY_ADC_SAMPLETIME_84CYCLES    4U
#define MY_ADC_SAMPLETIME_112CYCLES   5U
#define MY_ADC_SAMPLETIME_144CYCLES   6U
#define MY_ADC_SAMPLETIME_480CYCLES   7U

#define MY_ADC_EXT_TRIG_EDGE_NONE      0x00000000UL
#define MY_ADC_EXT_TRIG_EDGE_RISING    0x10000000UL
#define MY_ADC_EXT_TRIG_EDGE_FALLING   0x20000000UL
#define MY_ADC_EXT_TRIG_EDGE_BOTH      0x30000000UL

#define MY_ADC_CLOCK_DIV2   0x00000000UL
#define MY_ADC_CLOCK_DIV4   0x00010000UL
#define MY_ADC_CLOCK_DIV6   0x00020000UL
#define MY_ADC_CLOCK_DIV8   0x00030000UL

typedef struct {
    uint32_t clock_prescaler;  /* [分频] DIV2/4/6/8，ADC 时钟相对 PCLK2 */
    uint32_t resolution;       /* [分辨率] 12B/10B/8B/6B */
    uint32_t data_align;       /* [对齐] RIGHT / LEFT */
    uint32_t scan_mode;        /* [开关] 多通道扫描 SCAN */
    uint32_t eoc_selection;    /* [EOC] SINGLE=每次转换；SEQ=整序列结束 */
    uint32_t continuous_mode;  /* [开关] 连续转换 CONT */
    uint32_t dma_mode;         /* [开关] CR2.DMA 请求 */
    uint32_t dma_dds;          /* [开关] DDS 连续 DMA 请求 */
    uint32_t discontinuous_mode; /* [开关] 间断模式 DISCEN */
    uint32_t nbr_disc;         /* [数值] 间断子组长度 1~8 */
    uint32_t external_trigger_edge; /* [边沿] NONE/RISING/FALLING/BOTH */
    uint32_t external_trigger;      /* [触发源] EXTSEL；软件启动可 0 */
} my_adc_config_t;

//.c
#include "my_adc.h"
#include "my_rcc.h"

/*
 * HAL mapping:
 *   ADC_CCR.ADCPRE <-> ADC_InitTypeDef.ClockPrescaler
 *   CR1.RES        <-> ADC_InitTypeDef.Resolution
 *   CR1.SCAN       <-> ADC_InitTypeDef.ScanConvMode
 *   CR1.DISCEN/DISC <-> ADC_InitTypeDef.DiscontinuousConvMode/NbrOfDiscConversion
 *   CR2.CONT       <-> ADC_InitTypeDef.ContinuousConvMode
 *   CR2.ALIGN      <-> ADC_InitTypeDef.DataAlign
 *   CR2.EOCS       <-> ADC_InitTypeDef.EOCSelection (SINGLE sets EOCS)
 *   CR2.DMA/DDS    <-> ADC_InitTypeDef.DMAContinuousRequests
 *   CR2.EXTEN/EXTSEL <-> ADC_InitTypeDef.ExternalTrigConvEdge/ExternalTrigConv
 *   SMPR/SQR       <-> ADC_ChannelConfTypeDef.SamplingTime/Rank
 *   SQR1.L         <-> NbrOfConversion - 1
 *   SR.EOC/DR      <-> HAL_ADC_PollForConversion/HAL_ADC_GetValue
 */

void my_adc_init(adc_register_t *adc, my_adc_config_t *cfg)
{
    if (adc == 0U || cfg == 0U)
        return;
    /* 时钟预分频在 ADC 公共 CCR 寄存器 */
    MY_ADC_COMMON->ccr = (MY_ADC_COMMON->ccr & ~0x00030000UL) |
                         (cfg->clock_prescaler & 0x00030000UL);
    /* 先清零关键寄存器 */
    adc->cr1 = 0U;
    adc->cr2 = 0U;
    /* CR1.RES: resolution */
    adc->cr1 |= (cfg->resolution & 0x03U) << 24U;
    /* CR1.SCAN */
    if (cfg->scan_mode != 0U)
        adc->cr1 |= 0x00000100U; /* SCAN */
    /* CR1.DISCEN + DISC */
    if (cfg->discontinuous_mode != 0U)
    {
        adc->cr1 |= 0x00000800U; /* DISCEN */
        adc->cr1 |= (cfg->nbr_disc & 0x07U) << 13U;
    }
    /* CR2.CONT */
    if (cfg->continuous_mode != 0U)
        adc->cr2 |= 0x00000002U;
    /* CR2.ALIGN */
    if (cfg->data_align != 0U)
        adc->cr2 |= 0x00000800U;
    /* CR2.EOCS: MY_ADC_EOC_SINGLE=1 sets EOCS (matches HAL) */
    if (cfg->eoc_selection == MY_ADC_EOC_SINGLE)
        adc->cr2 |= 0x00000400U;
    /* CR2.DMA / DDS */
    if (cfg->dma_mode != 0U)
        adc->cr2 |= 0x00000100U;
    if (cfg->dma_dds != 0U)
        adc->cr2 |= 0x00000200U;    //DDS DMA 连续请求
    /* CR2.EXTEN/EXTSEL: 规则通道外部触发 */
    adc->cr2 |= (cfg->external_trigger_edge & 0x30000000UL);
    if (cfg->external_trigger_edge != 0U)
        adc->cr2 |= (cfg->external_trigger & 0x0F000000UL);
}

void my_adc_enable(adc_register_t *adc)
{
    if (adc == 0U)
        return;
    adc->cr2 |= 0x00000001U; /* ADON */
}

void my_adc_set_nbr_of_conversion(adc_register_t *adc, uint32_t nbr)
{
    if (adc == 0U)
        return;
    if (nbr < 1U)
        nbr = 1U;
    if (nbr > 16U)
        nbr = 16U;
    adc->sqr1 = (adc->sqr1 & ~(0x0FUL << 20U)) | (((nbr - 1U) & 0x0FU) << 20U);
}

void my_adc_config_channel(adc_register_t *adc, uint32_t channel,
                           uint32_t rank, uint32_t sample_time)
{
    uint32_t shift;
    uint32_t rank_in;
    uint32_t l;
    volatile uint32_t *sqr;

    if (adc == 0U)
        return;
    if (rank < 1U)
        rank = 1U;
    if (rank > 16U)
        rank = 16U;
    rank_in = rank;

    /* 采样时间 SMPR1/SMPR2 */
    shift = (channel % 10U) * 3U;
    if (channel <= 9U)
        adc->smpr2 = (adc->smpr2 & ~(7UL << shift)) | ((sample_time & 7U) << shift);
    else
        adc->smpr1 = (adc->smpr1 & ~(7UL << shift)) | ((sample_time & 7U) << shift);

    /* 规则序列 SQR1/SQR2/SQR3 */
    if (rank <= 6U)
        sqr = &adc->sqr3;
    else if (rank <= 12U)
        sqr = &adc->sqr2;
    else
        sqr = &adc->sqr1;

    if (rank > 6U)  rank -= 6U;
    if (rank > 6U)  rank -= 6U;
    *sqr = (*sqr & ~(0x1FUL << ((rank - 1U) * 5U))) |
           ((channel & 0x1FU) << ((rank - 1U) * 5U));

    /* Extend SQR1.L so sequence length covers this rank */
    l = (adc->sqr1 >> 20U) & 0x0FU;
    if (rank_in > (l + 1U))
        my_adc_set_nbr_of_conversion(adc, rank_in);
}

void my_adc_start_software_conversion(adc_register_t *adc)
{
    if (adc == 0U)
        return;
    adc->cr2 |= 0x40000000U; /* SWSTART */
}

uint16_t my_adc_wait_conversion(adc_register_t *adc)
{
    if (adc == 0U)
        return 0U;
    while ((adc->sr & 0x00000002U) == 0U) /* EOC */
    {
    }
    return (uint16_t)adc->dr;
}

uint16_t my_adc_read_channel(adc_register_t *adc, uint32_t channel, uint32_t sample_time)
{
    my_adc_config_channel(adc, channel, 1U, sample_time);
    /* Single-channel poll: force L=0 (1 conversion) */
    my_adc_set_nbr_of_conversion(adc, 1U);
    my_adc_start_software_conversion(adc);
    return my_adc_wait_conversion(adc);
}

void my_adc1_enable_clock(void)
{
    my_rcc_apb2_enable(MY_RCC_ADC1EN);
}

void my_adc1_init(void)
{
    my_adc_config_t cfg;

    my_adc1_enable_clock();

    /* -------- my_adc_config_t：对应 HAL ADC_InitTypeDef -------- */
    cfg.clock_prescaler = MY_ADC_CLOCK_DIV4;
    // 性质：时钟分频    | 作用：ADC 公共时钟相对 PCLK2；可选 DIV2 / DIV4 / DIV6 / DIV8

    cfg.resolution = MY_ADC_RESOLUTION_12B;
    // 性质：分辨率枚举  | 作用：转换位数；可选 12B / 10B / 8B / 6B

    cfg.data_align = MY_ADC_DATAALIGN_RIGHT;
    // 性质：对齐方式    | 作用：DR 右对齐或左对齐；可选 RIGHT / LEFT

    cfg.scan_mode = MY_ADC_SCAN_DISABLE;
    // 性质：功能开关    | 作用：多通道扫描；单通道轮询用 DISABLE；可选 ENABLE / DISABLE

    cfg.eoc_selection = MY_ADC_EOC_SINGLE;
    // 性质：EOC 行为    | 作用：每次转换结束置 EOC(SINGLE) 或整序列结束(SEQ)
    // 可选：MY_ADC_EOC_SINGLE / MY_ADC_EOC_SEQ（与 HAL 一致：SINGLE 置 EOCS）

    cfg.continuous_mode = MY_ADC_CONT_DISABLE;
    // 性质：功能开关    | 作用：连续转换；DISABLE 则每次需 SWSTART；可选 ENABLE / DISABLE

    cfg.dma_mode = MY_ADC_DMA_DISABLE;
    // 性质：功能开关    | 作用：是否产生 DMA 请求(CR2.DMA)

    cfg.dma_dds = MY_ADC_DDS_DISABLE;
    // 性质：功能开关    | 作用：DMA 连续请求 DDS；单次传输用 DISABLE

    cfg.discontinuous_mode = MY_ADC_DISC_DISABLE;
    // 性质：功能开关    | 作用：间断模式；与连续模式互斥

    cfg.nbr_disc = 0U;
    // 性质：间断通道数  | 作用：每次触发转换的子组长度；DISC 使能时 1~8

    cfg.external_trigger_edge = MY_ADC_EXT_TRIG_EDGE_NONE;
    // 性质：触发边沿    | 作用：无边沿=软件启动；可选 NONE / RISING / FALLING / BOTH

    cfg.external_trigger = 0U;
    // 性质：触发源编码  | 作用：EXTSEL 字段；软件启动时可为 0；外部触发时填 TIM/EXTI 编码

    my_adc_init(MY_ADC1, &cfg);
    my_adc_enable(MY_ADC1);
}

uint16_t my_adc1_read_channel(uint32_t channel, uint32_t sample_time)
{
    return my_adc_read_channel(MY_ADC1, channel, sample_time);
}
```

## 9. DMA

### 9.1 HAL

1. 在F407是有8个stream，每个stream有8个ch，ch是硬件设计，不可编程，但是有可能出现同一个物理ch在不同的逻辑ch出现
2. 优先级仲裁出现在stream层面，包括软件寄存器配置仲裁，如果同软件优先级，硬件序号小的优先级高
3. stream mode :常规模式；双缓冲区模式是只对于内存地址有双缓冲区，对于外设没有
4. 地址增加：可以通过PSIZE MSIZE 修改增加的地址长度，也可以通过PINCOS指定为AHB总线长度32bit
5. 直接模式和FIFO模式对于数据长度的处理：
   1. FIFO模式允许长度不一致，由MSIZE和PSIZE配置，会自己在FIFO里面进行拆合；不一致的时候数据长度以PSIZE为单位，计算数据数量；拆合的过程又叫做打包解包，打包解包只支持小端序（低字节低地址）；打包解包的过程中可能被中断打断，所以配置burst保护为一个整体
   2. 直接模式下，不允许打包解包，不允许长度不一致，由PSIZE配置
6. single 和 burst：在burst下，读取数据时不是只读一个带宽，而是连续读多个带宽（需配置），并且地址递增（和前面配置的地址增量是一致的）；写数据时，只有达到FIFO的阈值才触发，随后是按照目的地址的数据长度传输指定的数据项

```c
#include "my_dma_hal.h"

void my_dma_hal_init(DMA_HandleTypeDef *hdma, DMA_Stream_TypeDef *stream,
                     uint32_t direction, uint32_t priority)
{
    if (hdma == NULL)
        return;

    hdma->Instance = stream;
    // 性质：硬件实例    | 作用：绑定 DMA1/2 的某个 Stream
    // 可选：DMA1_Stream0~7 / DMA2_Stream0~7（M2M 只能用 DMA2）
    /* -------- DMA_InitTypeDef -------- */
    hdma->Init.Channel = DMA_CHANNEL_0;
    // 性质：通道枚举    | 作用：选择外设请求通道；请按实际外设映射修改
    // 可选：DMA_CHANNEL_0 ~ DMA_CHANNEL_7
    hdma->Init.Direction = direction;
    // 性质：传输方向    | 作用：P2M / M2P / M2M
    // 可选：DMA_PERIPH_TO_MEMORY / DMA_MEMORY_TO_PERIPH / DMA_MEMORY_TO_MEMORY
    hdma->Init.PeriphInc = DMA_PINC_DISABLE;
    // 性质：地址增量开关 | 作用：外设地址是否自增；固定外设寄存器通常 DISABLE
    // 可选：DMA_PINC_ENABLE / DMA_PINC_DISABLE
    hdma->Init.MemInc = DMA_MINC_ENABLE;
    // 性质：地址增量开关 | 作用：内存缓冲区地址是否自增
    // 可选：DMA_MINC_ENABLE / DMA_MINC_DISABLE
    hdma->Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    // 性质：数据宽度    | 作用：外设侧一次传输宽度，需匹配外设寄存器
    // 可选：DMA_PDATAALIGN_BYTE / HALFWORD / WORD
    hdma->Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
    // 性质：数据宽度    | 作用：内存侧一次传输宽度
    // 可选：DMA_MDATAALIGN_BYTE / HALFWORD / WORD
    hdma->Init.Mode = DMA_NORMAL;
    // 性质：工作模式    | 作用：单次或循环；M2M 不可用 Circular
    // 可选：DMA_NORMAL / DMA_CIRCULAR

    hdma->Init.Priority = priority;
    // 性质：仲裁优先级  | 作用：Stream 之间优先级
    // 可选：DMA_PRIORITY_LOW / MEDIUM / HIGH / VERY_HIGH

    hdma->Init.FIFOMode = DMA_FIFOMODE_DISABLE;
    // 性质：FIFO 开关   | 作用：直通或 FIFO；M2M 必须 ENABLE
    // 可选：DMA_FIFOMODE_DISABLE / DMA_FIFOMODE_ENABLE
    // 注：若开 FIFO，还需配置 FIFOThreshold / MemBurst / PeriphBurst
    DMA_InitStructure.DMA_FIFOMode           = DMA_FIFOMode_Enable;
    DMA_InitStructure.DMA_FIFOThreshold      = DMA_FIFOThreshold_HalfFull;
    /* 突发配置 */
    DMA_InitStructure.DMA_MemoryBurst        = DMA_MemoryBurst_INC4;
    DMA_InitStructure.DMA_PeripheralBurst    = DMA_PeripheralBurst_Single;
    HAL_DMA_Init(hdma);
}

HAL_StatusTypeDef my_dma_hal_start(DMA_HandleTypeDef *hdma,
                                   uint32_t src, uint32_t dst, uint16_t len)
{
    if (hdma == NULL)
        return HAL_ERROR;
    return HAL_DMA_Start(hdma, src, dst, len);
}
```

### 8.3 寄存器

```c
//.h
typedef struct {
    dma_stream_register_t *stream; /* [实例] DMA1/2 Stream0~7；M2M 仅 DMA2 */
    uint32_t channel;       /* [通道] 0~7，外设请求映射；M2M 常填 0 */
    uint32_t direction;     /* [方向] P2M / M2P / M2M */
    uint32_t periph_inc;    /* [开关] PAR 侧地址自增；INC_DISABLE / ENABLE */
    uint32_t mem_inc;       /* [开关] M0AR 侧地址自增 */
    uint32_t periph_size;   /* [宽度] BYTE / HALFWORD / WORD */
    uint32_t mem_size;      /* [宽度] BYTE / HALFWORD / WORD */
    uint32_t mode;          /* [模式] NORMAL / CIRCULAR（M2M 禁 CIRCULAR） */
    uint32_t priority;      /* [优先级] LOW / MEDIUM / HIGH / VERY_HIGH */
    uint32_t fifo_mode;     /* [开关] FIFO；M2M 必须 ENABLE */
    uint32_t fifo_threshold;/* [阈值] 1_4 / 1_2 / 3_4 / FULL */
} my_dma_config_t;

typedef struct {
    volatile uint32_t cr;
    volatile uint32_t ndtr;
    volatile uint32_t par;
    volatile uint32_t m0ar;
    volatile uint32_t m1ar;
    volatile uint32_t fcr;
} dma_stream_register_t;

//.h
#include "my_dma.h"
#include "my_rcc.h"

/*
 * HAL mapping:
 *   CR.CHSEL <-> DMA_InitTypeDef.Channel
 *   CR.DIR   <-> DMA_InitTypeDef.Direction
 *   CR.PINC  <-> DMA_InitTypeDef.PeriphInc
 *   CR.MINC  <-> DMA_InitTypeDef.MemInc
 *   CR.PSIZE <-> DMA_InitTypeDef.PeriphDataAlignment
 *   CR.MSIZE <-> DMA_InitTypeDef.MemDataAlignment
 *   CR.CIRC  <-> DMA_InitTypeDef.Mode
 *   CR.PL    <-> DMA_InitTypeDef.Priority
 *   FCR.DMDIS/FTH <-> DMA_InitTypeDef.FIFOMode/FIFOThreshold
 *   NDTR/PAR/M0AR <-> HAL_DMA_Start() parameters
 */

static void my_dma_enable_clock(dma_stream_register_t *stream)
{
    if (stream == 0U)
        return;
    if (((uint32_t)stream & 0x00000400UL) != 0U)
        my_rcc_ahb1_enable(MY_RCC_DMA2EN);
    else
        my_rcc_ahb1_enable(MY_RCC_DMA1EN);
}

void my_dma_init(my_dma_config_t *cfg)
{
    dma_stream_register_t *s;

    if (cfg == 0U || cfg->stream == 0U)
        return;

    s = cfg->stream;
    my_dma_enable_clock(s);

    /* Disable stream and wait until EN clears before configuring */
    s->cr &= ~0x00000001U;
    while ((s->cr & 0x00000001U) != 0U)
    {
    }

    s->cr = 0U;
    s->fcr = 0U;

    /* CR.CHSEL: channel */
    s->cr |= (cfg->channel & 0x07U) << 25U;
    /* CR.DIR */
    s->cr |= (cfg->direction & 0x03U) << 6U;
    /* CR.PINC / MINC */
    if (cfg->periph_inc != 0U) s->cr |= 0x00000200U;
    if (cfg->mem_inc     != 0U) s->cr |= 0x00000400U;
    /* CR.PSIZE / MSIZE */
    s->cr |= (cfg->periph_size & 0x03U) << 11U;
    s->cr |= (cfg->mem_size     & 0x03U) << 13U;
    /* CR.CIRC: mode */
    if (cfg->mode == MY_DMA_MODE_CIRCULAR)
        s->cr |= 0x00000100U;
    /* CR.PL */
    s->cr |= (cfg->priority & 0x03U) << 16U;
    /* FCR.FTH / DMDIS: FIFO */
    s->fcr |= (cfg->fifo_threshold & 0x03U) << 0U;
    if (cfg->fifo_mode == MY_DMA_FIFOMODE_ENABLE)
        s->fcr |= 0x00000004U;
}

void my_dma_start(dma_stream_register_t *stream, uint32_t src, uint32_t dst, uint16_t len)
{
    uint32_t dir;

    if (stream == 0U)
        return;

    stream->cr &= ~0x00000001U;
    while ((stream->cr & 0x00000001U) != 0U)
    {
    }

    /* Match HAL_DMA_Start: M2P swaps PAR/M0AR */
    dir = (stream->cr >> 6U) & 0x03U;
    if (dir == MY_DMA_DIR_M2P)
    {
        stream->par = dst;
        stream->m0ar = src;
    }
    else
    {
        stream->par = src;
        stream->m0ar = dst;
    }

    stream->ndtr = len;
    stream->cr |= 0x00000001U; /* EN */
}

uint8_t my_dma_is_transfer_complete(dma_stream_register_t *stream)
{
    if (stream == 0U)
        return 0U;
    /* F4 Normal mode: EN clears when transfer finishes */
    return ((stream->ndtr == 0U) && ((stream->cr & 0x00000001U) == 0U)) ? 1U : 0U;
}

void my_dma_wait_complete(dma_stream_register_t *stream)
{
    if (stream == 0U)
        return;
    while ((stream->cr & 0x00000001U) != 0U)
    {
    }
}
```


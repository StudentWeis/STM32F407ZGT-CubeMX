STM32F407ZGTx CubeMX 练习仓库

CubeMX Firmware 版本：1.26.2

https://gitee.com/studentwei/STM32F407ZGT

https://github.com/StudentWeis/STM32F407ZGT

---

本仓库以进阶 STM32 学习为目的，主要完成了以下几个任务：

### 任务0

测试 UART，将此文件作为后续任务的模板。

### 任务1

测试 SDIO

参考：https://blog.csdn.net/qq_42039294/article/details/112045786

一次成功。

### 任务2

测试 FatFs

参考：https://www.cnblogs.com/showtime20190824/p/11523402.html

第一次失败：因为之前 SDIO 的学习中破坏了 SD 卡格式，所以需要先格式化再进行 FatFs 文件系统调用。格式化的文件系统是 FAT32（FatFs 默认）。

### 任务3

测试 RTC

核心板硬件上装有低速 32kHz 晶振，且装有纽扣电池，可以进行测试。

主要目标：

1. 基本时钟使用；
2. 掉电继续运行；
3. 关闭掉电继续运行模式；

参考：https://blog.csdn.net/as480133937/article/details/105741893?spm=1001.2014.3001.5501

- 时钟选择要选择 LSE，不要用默认的 LSI，这样掉电之后才有时钟。

- 分频寄存器是自动配置的，为了凑够 1s 的中断。

- 写的时候 BCD 写，读的时候 BIN 读，方便后续的显示。

- 重启之后 RTC 也重启了，如何避免？

  核心板需要连接 VBAT 备用电源；然后使用备份寄存器（Backup Registers）做记录，上电之后检查备份寄存器，然后跳过 RTC 初始化。

    ```c
    if (HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR1) != 0x5051)
    {
        HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR1, 0x5051);
        //....
    }
    else
    {
    }
    ```
  
- 掉电之后的机制到底是什么？

  VBat：数据手册 P31

  Power supply overview：参考手册 P116

  LSE：参考手册 P156

  RTC：参考手册 P221

  RCC Backup domain control register (RCC_BDCR)：参考手册 P199，这些不会被 Reset 复位。

  断电之后其他时钟就停了，而备用纽扣电池从 Vbat 口供电，维持 BKP 寄存器的数据、维持 LSE 时钟，为完全独立的 RTC 提供频率和电源，维持时钟继续。

- 如何关闭电池？为了节能。

  换回 LSI 时钟就行，默认不开启 LSE 即可。
  

### 任务4

测试 UART+DMA。

参考：https://blog.csdn.net/as480133937/article/details/104827639

问题：CubeMX 生成的初始化函数顺序有问题，DMA 应该在 UART 之前。

原因剖析：https://shequ.stmicroelectronics.cn/forum.php?mod=viewthread&tid=622385

是 CubeMX 生成函数的问题，`MX_DMA_Init()` 并没有初始化 DMA 数据；`MX_USART1_UART_Init();` 中才初始化。而不开启 DMA 的时钟是没办法初始化 DMA 数据的。

如何测试 DMA 是否真的不占用 CPU？

串口传送 1940 个字节，波特率 115200，即 1s 发送 14400 个字节，串口发数据会占用 0.135s。

对比使用 DMA 和不使用 DMA 的 UART 发送 10s（每一次加一个 500ms 软件延时），根据发送的字节数量做对比。

- DMA：37520B
- 无 DMA：31040B

### 任务5

测试低功耗模式 Sleep

参考：https://blog.csdn.net/qq_36347513/article/details/114323123

要注意 systick 中断问题，否则会直接激活睡眠。

### 任务6

测试低功耗模式 Standby，通过 RTC 唤醒。


```mermaid
flowchart LR
    id[小灯亮灭亮 3s] --> id2[待机 7s] --> id3[自动唤醒] --> id
```

问题1：无法 Wakeup。因为跳过了 RTC init，解决了好久这个问题。

问题2：唤醒之后持续被唤醒。因为没有清除唤醒标志位（RM 140P）：

```c
__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU);
```

参考手册有 Safe RTC alternate function wakeup flag clearing sequence，应该仔细查看。

### 任务7

测试 SPI 读写 Flash W25Q16

下载 [W25Q16](https://www.winbond.com/hq/product/code-storage-flash-memory/serial-nor-flash/?__locale=en&partNo=W25Q16JV) 数据手册；

1. 首先读取 ID 测试 SPI 通信；

​	问题1：CubeMX 默认使用的 SPI 引脚与开发板上的连接不同 ，更换引脚后正常。

​	读取 ID=9Fh 之后会先收到 Winbond Serial Flash：EFh，然后收到 4015h。

2. 然后测试读写数据。

   参考：

   https://www.pianshen.com/article/2084546465

   https://blog.csdn.net/qq_36347513/article/details/113252061

   问题：全片擦除之后写不进去；

3. 完成封装

   问题：写和读的地址有问题：从0x000000开始读写都没有问题，从0x000000开始写，然后从0x000001开始读就不行了。
   
   解决方案：👇
   
   ```c
   // 下面的发地址方式总是不行
   uint32_t WriteAddrt = WriteAddr << 8;
   FLASH_SPI_TRANSMIT((uint8_t *)&WriteAddrt, 3);
   
   // 换成下面这个就好了
   cmd = (WriteAddr & 0xFF0000) >> 16;
   FLASH_SPI_TRANSMIT(&cmd, 1);
   cmd = (WriteAddr & 0xFF00) >> 8;
   FLASH_SPI_TRANSMIT(&cmd, 1);
   cmd = (WriteAddr & 0xFF);
   ```
   

### 任务8

测试 SPI DMA 读写 FLASH 

问题1：可以 SPI DMA 回环操作，但是无法驱动 FLASH。

暂且搁置该任务，换任务为 SPI DMA 回环实验。

连接 MOSI、MISO，使用 DMA 发送信息，然后通过串口打印出来。

### 任务9

双片 SPI DMA 通信 

主机向从机发送数据，从机将接收到的数据 printf 出来。

难点在于：从机如何向主机发送数据？

测试 DMA 效率：重点在于不占用 CPU，所以要测试 CPU 完成时间；

### 任务10

MCU >> PC USB 通信

```c
CDC_Transmit_FS(Buf, *Len);
```

问题1：识别不到设备

解决方法：增加 CubeMX 中的堆栈的大小 0x1200；

参考：https://blog.csdn.net/13011803189/article/details/108669947

https://blog.csdn.net/qq_16597387/article/details/93094052


# Direct Memory Access Controller

* TOC
{:toc}

The devices feature two general-purpose dual-port DMAs (DMA1 and DMA2) with 8
streams each. They are able to manage memory-to-memory, peripheral-to-memory and
memory-to-peripheral transfers. They feature dedicated FIFOs for APB/AHB peripherals,
support burst transfer and are designed to provide the maximum peripheral bandwidth
(AHB/APB).
The two DMA controllers support circular buffer management, so that no specific code is
needed when the controller reaches the end of the buffer. The two DMA controllers also
have a double buffering feature, which automates the use and switching of two memory
buffers without requiring any special code.
Each stream is connected to dedicated hardware DMA requests, with support for software
trigger on each stream. Configuration is made by software and transfer sizes between
source and destination are independent. DMA controller is an AMBA advanced high performance bus (AHB) master module present in the Microcontroller


The DMA can be used with the main peripherals:
* SPI and I2S
* I2C
* USART
* General-purpose, basic and advanced-control timers TIMx
* DAC
* SDIO
* Camera interface (DCMI)
* ADC
* SAI1/SAI2
* SPDIF Receiver (SPDIFRx)
* QuadSPI


### Direct Memory Access Block Diagram

![] ({{ site.url }}/assets/dma-controller/block-diagram-dma.png)

|:--:|
| *DMA1 Request mapping*|

### Multi-AHB bus matrix
|![img] ({{ site.url }}/assets/dma-controller/ahb-bus-matrix.png)|
|:--:|
| *DMA1 Request mapping*|

The 32-bit multi-AHB bus matrix interconnects all the masters (CPU, DMAs, USB HS) and
the slaves Flash memory, RAM, QuadSPI, FMC, AHB and APB peripherals and ensures a
seamless and efficient operation even when several high-speed peripherals work
simultaneously.



### Master and Slave Ports in DMA
DMA controller features three AHB ports:
	1. A Slave Port for DMA programming
	2. Two Master Pordt for Peripherals and Memory respectively
             This allows DMA  to initiate data transfer between different slave modules

### Streams in DMA
Each DMA controller has 8 Streams. All of 8 streams provides unidirectional transfer link between a source and a destination.
A Stream can be configured to perform transactions as follows -
* Peripheral to memory
* Memory to peripheral
* Memory to Memory

Each Stream is associated with a DMA request that can be selected out of 8 possible channel requests

### Channel Mapping in DMA
|![img] ({{site.url}}/assets/dma-controller/channel-selection-dma.png)|
|:--:|
| *Channel Selection Internals*|

Register needed to configure this is DMA_SxCR  ()where x is the number from 0 to 7

|![img] ({{ site.url }}/assets/dma-controller/request-mapping-dma1.png)|
|:--:|
| *DMA1 Request mapping*|

|![img] ({{ site.url }}/assets/dma-controller/request-mapping-dma2.png)|
|:--:|
|*DMA2 Request mapping*|

### Arbiter in DMA

### Peripheral to Memory Data Transfer

|![Peripheral to Memory Mode] (/assets/dma-controller/peripheral-to-mem-dma.png "Peripheral Mode")|
|:--:|
|*Peripheral to Memory Mode*|


Each time a peripheral request occurs, data is moved from peripheral data register to memory. There are two modes of data transfer as follows-

* _Direct Mode_

	In Direct Mode, the threshold level of the FIFO is not used. After each data transfer from the 	peripheral to FIFO, the corresponding data are immediately moved and stored into the destination memory. In this case, FIFO threshold is not needed.

* _FIFO Mode_

	In FIFO mode, if the threshold level of the FIFO is reached, the contents of the FIFO are flushed to be stored into the destination. In this case, FIFO threshold settings in needed. FIFO in stm32f446xx is 4 bytes. Threshold settings can be done for 1 byte (1/4), 2 bytes (2/4), 3 bytes (3/4) and 4 bytes (4/4).
Register used to set FIFO Threshold Settings is *DMA_SxFCR* (x=0..7)

Advantages for using FIFO mode are as follows- 
1) Memory port is less accessed allowing other pending DMA requests to access the memory port
2) Allowing other bus masters to access the memory, thus decreasing the wait time for other masters


### Memory to Peripheral Data Transfer

|![Memory to Peripheral Mode] ({{ site.url }}/assets/dma-controller/mem-to-peripheral-dma.png )|
|:--:|
|*Memory to Peripheral Mode*|

Each time a peripheral request occurs, data is moved from memory to peripheral data register to memory.

* _Direct Mode_

	In Direct Mode, the threshold level of the FIFO is not used. Once the Stream is enabled, the DMA preloads the first data to transfer into an internal FIFO. As soon as the peripheral requests a data transfer, the DMA transfer the preloaded value into the configured destination. DMA again reloads the empty FIFO with next data to be transferred. In this case, FIFO threshold is not needed.

* _FIFO Mode_

	In FIFO mode, once the Stream is enabled, the FIFO will be preloaded fully from memory. When peripheral requests for a data transfer the contents of the FIFO are flushed and stored into the destination.When is level if the FIFO is lower than or equal to the predefined threshold level, the FIFO is fully reloaded with data from the memory. In this case, FIFO threshold settings in needed. FIFO in stm32f446xx is 4 bytes. Threshold settings can be done for 1 byte (1/4), 2 bytes (2/4), 3 bytes (3/4) and 4 bytes (4/4).
Register used to set FIFO Threshold Settings is *DMA_SxFCR* (x=0..7)



### Current Requirement from DMA1 and DMA2


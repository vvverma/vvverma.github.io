
# Table of Contents

- [Modes in 16X2 LCD](#modes-in-16x2-lcd)
- [Types of Memory in LCD](#types-of-memory-in-lcd)
- [Pin Description](#pin-description)
- [Reference](#reference)
- [Table of Contents](ReadMd.md)

<!-- toc -->

## Modes in 16X2 LCD

LCD operates in two modes, which are as follows

1) **4-bit mode**

    4-bit mode is used to save GPIO Interfacing pins on the host side controller.
    Drawback for 4-bit mode is slow data or command transfer to the LCD controller.
    In order to sent 1byte data it takes double time than 8-bit mode

2) **8-bit mode**

    8-bit mode is used to for faster GPIO data/commad transfer. 
    Drawback for 8-bit mode is more number of GPIO pins are needed.

## Types of Memory in LCD

1) **Display Data RAM (DDRAM)**

    DDRAM is used to display character on the LCD. It decodes instruction or command
    from IR of the LCD Controller

2) **Character Generator ROM (CGROM)**

    CGROM is responsible to convert ASCII character Hex to 5x7 or 5x8 dot matrix pattern.
    It has all pre-programmed supported character in the CGROM.

3) **Character Generator RAM (CGRAM)**

    CGRAM is used if we want a customized font the LCD Display.
    We just have to load the array to the CGRAM address. 

## Pin Description

|Pin#	|Label		|Value		|Description|
|----------|----------------|----------------|--------------|
|16 		|BL- 			|0 V 			|Back light power supply voltage( - )|
|15		|BL+ 		|4.2 V		|Back light power supply voltage( + )|
|14 		|DB7 		|H/L			|Data bit7|		
|13		|DB6			|H/L 		|Data bit6|
|12		|DB5 		|H/L 		|Data bit5|
|11 		|DB4 		|H/L 		|Data bit4|
|10 		|DB3 		|H/L 		|Data bit3		not needed in 4 bit mode|
|9 		|DB2 		|H/L 		|Data bit2		not needed in 4 bit mode|
|8 		|DB1 		|H/L 		|Data bit1		not needed in 4 bit mode|
|7 		|DB0 		|H/L	 		|Data bit0		not needed in 4 bit mode|
|6 		|E 			|H/L 		|Starts data read/write|
|5 		|R/W 		|H/L 		|Selects read or write|
|4 		|RS 			|H/L 		|Selects registers(H: Data L: Instruction)|
|3 		|VEE 		|0.3V 		|Power supply voltage LCD(-)|
|2 		|VCC 		|5.0V 		|Power supply voltage for logic and LCD(+) |
|1 		|VSS 		|0V 			|Ground|


## Reference

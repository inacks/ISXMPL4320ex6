# IS4320 I2C Modbus RTU Master example on STM32 (ISXMPL4320ex6)

Coding with STM32 for the IS4320 is very simple. It does not require any INACKS-specific library—just the standard HAL I2C functions: ```HAL_I2C_Mem_Read()``` and ```HAL_I2C_Mem_Write()```.

This STM32 CubeIDE project is based on the **Nucleo-C071** evaluation board from ST, which features an STM32C071RBT microcontroller, and the **Kappa4320Ard Evaluation Board**, which features the IS4320 I2C Modbus RTU Master chip. The Kappa4320Ard has an Arduino form factor and directly fits into the Nucleo-C071 board, so no additional connections between the microcontroller and the IS4320 are required, making it very easy to test the example.
 
The project demonstrates how to use the STM32 microcontroller to communicate with the IS4320 over I2C. In this example, the microcontroller instructs the IS4320 to read Holding Register 0 using the Function Code 3 from a Modbus Slave, and prints the result via the STLink Virtual COM Port. You can view the output using any serial terminal software, such as CoolTerm.

To test the example, you will need a Modbus Slave. You can use the **pyModSlave** software, which is a Modbus TCP/RTU Slave simulator.  Configure the Slave with these values: Slave Address 1, 19200 baud, Even parity, and 1 Stop bit.

- **Kappa4320Ard Evaluation Board:** [www.inacks.com/kappa4320ard](https://www.inacks.com/kappa4320ard)  
- **IS4320 Datasheet:** [www.inacks.com/is4320](https://www.inacks.com/is4320)
- **pyModSlave** [www.sourceforge.net/projects/pymodslave/](https://sourceforge.net/projects/pymodslave/)
- **Nucleo-C071 Information:** [www.st.com/en/evaluation-tools/nucleo-c071rb.html](https://www.st.com/en/evaluation-tools/nucleo-c071rb.html)  
- **CoolTerm** [https://freeware.the-meiers.org/](https://freeware.the-meiers.org/)

For more information, visit [www.inacks.com](https://www.inacks.com/is4320)

For clarity purposes, the code below excludes all extra HAL code and only shows the parts relevant to the IS4320. You can download the full STM32 project example as a .rar file from this repo.

```c
#include <stdio.h>

// IS4320 Memory Map Addresses:
#define CFG_MBBDR		0
#define CFG_MBPAR		1
#define CFG_MBSTP		2
#define REQ_EXECUTE 	6
#define REQ_SLAVE		7
#define REQ_FC			8
#define REQ_STARTING	9
#define REQ_QTY			10
#define RES_STATUS		138
#define RES_DATA1		139

/**
 * @brief	Reads a single register from the IS4320 memory map.
 * @param	registerAdressToRead: Address in the IS4320 memory map to be read.
 * @retval	Value stored at the registerAdressToRead register address.
 */
uint16_t readIS4320Register(uint16_t registerAdressToRead) {
    uint8_t IS4320_I2C_Chip_Address; // This variable stores the I2C chip address of the IS4320.
    IS4320_I2C_Chip_Address = 0x14; // The IS4320's I2C address is 0x14.
    // The STM32 HAL I2C library requires the I2C address to be shifted left by one bit.
    // Let's shift the IS4320 I2C address accordingly:
    IS4320_I2C_Chip_Address = IS4320_I2C_Chip_Address << 1;

    // The following array will store the read data.
    // Since each holding register is 16 bits long, reading one register requires reading 2 bytes.
    uint8_t readResultArray[2];

    // This variable will contain the final result:
    uint16_t readResult;

    /*
     * This is the HAL function to read from an I2C memory device. The IS4320 is designed to operate as an I2C memory.
     *
     * HAL_I2C_Mem_Read parameters explained:
     * 1. &hi2c1: This is the name of the I2C that you're using. You set this in the CubeMX. Don't forget the '&'.
     * 2. IS4320_I2C_Chip_Address: The I2C address of the IS4320 (must be left-shifted).
     * 3. registerAdressToRead: The holding register address to read from the IS4320.
     * 4. I2C_MEMADD_SIZE_16BIT: You must indicate the memory addressing size. The IS4320 memory addressing is 16-bits.
     *    This keyword is an internal constant of HAL libraries. Just write it.
     * 5. readResultArray: An 8-bit array where the HAL stores the read data.
     * 6. 2: The number of bytes to read. Since one holding register is 16 bits, we need to read 2 bytes.
     * 7. 1500: Timeout in milliseconds. IMPORTANT, this timeout must be higher than the timeout specified in CFG_MB_TIMEOUT.
     *    1500 is a good default value.
     */
    HAL_I2C_Mem_Read(&hi2c1, IS4320_I2C_Chip_Address, registerAdressToRead, I2C_MEMADD_SIZE_16BIT, readResultArray, 2, 1500);

    // Combine two bytes into a 16-bit result:
    readResult = readResultArray[0];
    readResult = readResult << 8;
    readResult = readResult | readResultArray[1];

    return readResult;
}

/**
 * @brief	Reads a single register from the IS4320 memory map.
 * @param	registerAdressToRead: Address in the IS4320 memory map to be read.
 * @retval	Value stored at the registerAdressToRead register address.
 */
void writeIS4320Register(uint16_t registerAdressToWrite, uint16_t value) {
    uint8_t IS4320_I2C_Chip_Address;  // I2C address of IS4320 chip (7-bit).
    IS4320_I2C_Chip_Address = 0x14; // IS4320 I2C address is 0x14 (7-bit).
    // STM32 HAL expects 8-bit address, so shift left by 1:
    IS4320_I2C_Chip_Address = IS4320_I2C_Chip_Address << 1;

    // The HAL library to write I2C memories needs the data to be in a uint8_t array.
    // So, lets put our uint16_t data into a 2 registers uint8_t array.
    uint8_t writeValuesArray[2];
    writeValuesArray[0] = (uint8_t) (value >> 8);
    writeValuesArray[1] = (uint8_t) value;

    /*
     * This is the HAL function to write to an I2C memory device. To be simple and easy to use, the IS4320 is designed to operate as an I2C memory.
     *
     * HAL_I2C_Mem_Write parameters explained:
     * 1. &hi2c1: This is the name of the I2C that you're using. You set this in the CubeMX. Don't forget the '&'.
     * 2. IS4320_I2C_Chip_Address: The I2C address of the IS4320 (must be left-shifted).
     * 3. registerAdressToWrite: The holding register address of the IS4320 we want to write to.
     * 4. I2C_MEMADD_SIZE_16BIT: You must indicate the memory addressing size. The IS4320 memory addressing is 16-bits.
     * This keyword is an internal constant of HAL libraries. Just write it.
     * 5. writeValuesArray: An 8-bit array where we store the data to be written by the HAL function.
     * 6. 2: The number of bytes to write. Since one holding register is 16 bits, we need to write 2 bytes.
     * 7. 1500: IMPORTANT, this timeout must be higher than the timeout specified in CFG_MB_TIMEOUT.
     *    1500 is a good default value.
     */
    HAL_I2C_Mem_Write(&hi2c1, IS4320_I2C_Chip_Address, registerAdressToWrite, I2C_MEMADD_SIZE_16BIT, writeValuesArray, 2, 1500);
}

// This function is required to make printf work over the Serial port.
// It is never called directly in the code — printf internally uses it
int _write(int file, char *ptr, int len) {
	HAL_UART_Transmit(&huart2, (uint8_t*)ptr, len, HAL_MAX_DELAY);
    return len;
}

int main(void) {
	/* First, configure the Modbus communication parameters
	 * to match the slave characteristics.
	 * This only needs to be done once before communicating
	 * with a Modbus device that has a different configuration. */

	// Set baud rate to 19200:
	uint16_t baudRate = 113;
	writeIS4320Register(CFG_MBBDR, baudRate);

	// Set parity to Even:
	uint16_t parityBit = 122;
	writeIS4320Register(CFG_MBPAR, parityBit);

	// Set stop bits to 1:
	uint16_t stopBits = 131;
	writeIS4320Register(CFG_MBSTP, stopBits);

 while (1) {
	/* Example: Read Holding Register 0 and print its value
	* to the PC via the Serial port */

	// Set the Modbus Slave ID:
	uint16_t modbusSlaveId = 1;
	writeIS4320Register(REQ_SLAVE, modbusSlaveId);

	// Set the Function Code. For reading Holding Registers, use FC = 3:
	uint16_t functionCode = 3;
	writeIS4320Register(REQ_FC, functionCode);

	// Set the Starting Register address. Here we want to read Holding Register 0:
	uint16_t startingRegister = 0;
	writeIS4320Register(REQ_STARTING, startingRegister);

	// Set the number of Holding Registers to read (minimum is 1):
	uint16_t quantity = 1;
	writeIS4320Register(REQ_QTY, quantity);

	// Send the request to the Modbus Slave:
	writeIS4320Register(REQ_EXECUTE, 1);

	// Read back the result/status of the operation:
	uint16_t status = readIS4320Register(RES_STATUS);

	if (status == 2) {
	  // OK! The request was sent and a response was received
	  // Read the data:
	  uint16_t holdingRegisterData = readIS4320Register(RES_DATA1);
	  printf("Holding Register 0 content = %d\r\n", holdingRegisterData);
	}
	else if (status == 3) {
	  // Timeout: no response from the Modbus Slave
	  printf("Timeout, the slave did not answer.\n");
	}
	else if (status == 4) {
	  printf("Broadcast message sent.\n");
	}
	else if (status == 5) {
	  printf("You configured wrongly the REQ_SLAVE register.\n");
	}
	else if (status == 6) {
	  printf("You configured wrongly the REQ_FC register.\n");
	}
	else if (status == 7) {
	  printf("You configured wrongly the REQ_QTY register.\n");
	}
	else if (status == 8) {
	  printf("There was a Frame Error.\n");
	}
	else if (status == 201) {
	  printf("Modbus Exception Code 1: Illegal Function.\n");
	}
	else if (status == 202) {
	  printf("Modbus Exception Code 2: Illegal Data Address.\n");
	}
	else if (status == 203) {
	  printf("Modbus Exception Code 3: Illegal Data Value.\n");
	}
	else if (status == 204) {
	  printf("Modbus Exception Code 4: Server Device Failure.\n");
	}
	else {
	  printf("Unkown Error");
	}
	
	// Add a delay to give the Modbus Slave time to respond and avoid stressing it: 
	HAL_Delay(1000);
}
```





# MCP2515_Arduino


## Hardware:
# CAN Shield
The following code samples uses the CAN-BUS Shield, wired up as shown:

![This is an image](https://github.com/deepalarm/MCP2515_Arduino/blob/main/examples/wiring.png)


# If you want to make your own CAN board on [JLCPCB](https://jlcpcb.com)  
![This is an image](https://github.com/deepalarm/MCP2515_Arduino/blob/main/examples/wiring-diy.png)

<br>
CAN-BUS is a common industrial bus because of its long travel distance, medium communication speed and high reliability. It is commonly found on modern machine tools and as an automotive diagnostic bus. This CAN-BUS Shield adopts MCP2515 CAN Bus controller with SPI interface and MCP2551 CAN transceiver to give your Arduino/Seeeduino CAN-BUS capibility. With an OBD-II converter cable added on and the OBD-II library imported, you are ready to build an onboard diagnostic device or data logger.

- Implements CAN V2.0B at up to 1 Mb/s
- SPI Interface up to 10 MHz
- Standard (11 bit) and extended (29 bit) data and remote frames
- Two receive buffers with prioritized message storage




## Library Installation

1. Download the ZIP file from [MCP2515_Arduino](https://github.com/deepalarm/MCP2515_Arduino/archive/refs/heads/main.zip)
2. From the Arduino IDE: Sketch -> Include Library... -> Add .ZIP Library...
3. Restart the Arduino IDE to see the new "mcp2515" library with examples



# Usage:



## 1. Initialization

To create connection with MCP2515 provide pin number where SPI CS is connected (10 by default), baudrate and mode

The available modes are listed as follows:
```C++
mcp2515.setNormalMode();
mcp2515.setLoopbackMode();
mcp2515.setListenOnlyMode();
```
The available baudrates are listed as follows:
```C++
enum CAN_SPEED {
    CAN_5KBPS,
    CAN_10KBPS,
    CAN_20KBPS,
    CAN_31K25BPS,
    CAN_33KBPS,
    CAN_40KBPS,
    CAN_50KBPS,
    CAN_80KBPS,
    CAN_83K3BPS,
    CAN_95KBPS,
    CAN_100KBPS,
    CAN_125KBPS,
    CAN_200KBPS,
    CAN_250KBPS,
    CAN_500KBPS,
    CAN_1000KBPS
};
```


Example of initialization

```C++
MCP2515 mcp2515(10);
mcp2515.reset();
mcp2515.setBitrate(CAN_125KBPS);
mcp2515.setLoopbackMode();
```

<br>

<br>
You can also set oscillator frequency for module when setting bitrate:

```C++
mcp2515.setBitrate(CAN_125KBPS, MCP_8MHZ);
```

<br>
The available clock speeds are listed as follows:

```C++
enum CAN_CLOCK {
    MCP_20MHZ,
    MCP_16MHZ,
    MCP_8MHZ
};
```

Default value is MCP_16MHZ
<br>

Note: To transfer data on high speed of CAN interface via UART dont forget to update UART baudrate as necessary.

## 2. Frame data format

Library uses Linux-like structure to store can frames;

```C++
struct can_frame {
    uint32_t can_id;  /* 32 bit CAN_ID + EFF/RTR/ERR flags */
    uint8_t  can_dlc;
    uint8_t  data[8];
};
```

For additional information see [SocketCAN](https://www.kernel.org/doc/Documentation/networking/can.txt)

## 3. Send Data

```C++
MCP2515::ERROR sendMessage(const MCP2515::TXBn txbn, const struct can_frame *frame);
MCP2515::ERROR sendMessage(const struct can_frame *frame);
```

This is a function to send data onto the bus. 

For example, In the 'send' example, we have:

```C++
struct can_frame frame;
frame.can_id = 0x000;
frame.can_dlc = 4;
frame.data[0] = 0xFF;
frame.data[1] = 0xFF;
frame.data[2] = 0xFF;
frame.data[3] = 0xFF;

/* send out the message to the bus and 
tell other devices this is a standard frame from 0x00. */
mcp2515.sendMessage(&frame);
```

```C++
struct can_frame frame;
frame.can_id = 0x12345678 | CAN_EFF_MASK;
frame.can_dlc = 2;
frame.data[0] = 0xFF;
frame.data[1] = 0xFF;

/* send out the message to the bus using second TX buffer and 
tell other devices this is a extended frame from 0x12345678. */
mcp2515.sendMessage(MCP2515::TXB1, &frame);
```



## 4. Receive Data

The following function is used to receive data on the 'receive' node:

```C++
MCP2515::ERROR readMessage(const MCP2515::RXBn rxbn, struct can_frame *frame);
MCP2515::ERROR readMessage(struct can_frame *frame);
```

In conditions that masks and filters have been set. This function can only get frames that meet the requirements of masks and filters.

You can choise one of two method to receive: interrupt-based and polling

Example of poll read

```C++
struct can_frame frame;

void loop() {
    if (mcp2515.readMessage(&frame) == MCP2515::ERROR_OK) {
        // frame contains received message
    }
}
```

Example of interrupt based read

```C++
bool interrupt = false;
struct can_frame frame;
	
void irqHandler() {
    interrupt = true;
}
	
void setup() {
    ...
    attachInterrupt(0, irqHandler, FALLING);
}
	
void loop() {
    if (interrupt) {
        interrupt = false;
       
        uint8_t irq = mcp2515.getInterrupts();
        
        if (irq & MCP2515::CANINTF_RX0IF) {
            if (mcp2515.readMessage(MCP2515::RXB0, &frame) == MCP2515::ERROR_OK) {
                // frame contains received from RXB0 message
            }
        }
            
        if (irq & MCP2515::CANINTF_RX1IF) {
            if (mcp2515.readMessage(MCP2515::RXB1, &frame) == MCP2515::ERROR_OK) {
                // frame contains received from RXB1 message
            }
        }
    }
}
```


## 5. Set Receive Mask and Filter

There are 2 receive mask registers and 5 filter registers on the controller chip that guarantee you get data from the target device. They are useful especially in a large network consisting of numerous nodes.

We provide two functions for you to utilize these mask and filter registers. They are:

```C++
MCP2515::ERROR setFilterMask(const MASK mask, const bool ext, const uint32_t ulData)
MCP2515::ERROR setFilter(const RXF num, const bool ext, const uint32_t ulData)
```

**MASK mask** represents one of two mask **MCP2515::MASK0** or **MCP2515::MASK1**

**RXF num** represents one of six acceptance filters registers from **MCP2515::RXF0** to **MCP2515::RXF5**

**ext** represents the status of the frame. **false** means it's a mask or filter for a standard frame. **true** means it's for a extended frame.

**ulData** represents the content of the mask of filter.




## 6. Examples

[Usage MCP2515](https://github.com/deepalarm/Can_Hacker_Arduino.git) 
    
----

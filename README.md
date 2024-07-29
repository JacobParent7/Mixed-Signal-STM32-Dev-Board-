# Mixed-Signal-STM32-Dev-Board

This project follows the  [Mixed-Signal Hardware Design with KiCad](https://fedevel.com/courses/mixed-signal-hardware-design-with-kicad) 4-layer PCB design course instructed by Philip Salmony of the Phil's Lab youtube channel.  

## System Level Description
### Basic Requirements
    - USB powered, single channel, low frequency analysis and generation capabilities
    - Perform frequency analysis and arbitrary signal generation
### Feature Specific Requirements
    - USB - <500mA current consumption 
    - Low Frequency - 20Hz - 20kHz (audio band)
    - ADC and DAC (or codec) for analysis and generation, single channel 
### Simple Block Diagram  
![SimpleBlockDiagram](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/2797847b-17f6-4e3c-a1bc-c3d00f2641d9)
### Refine requirements
    - USB - low bandwidth and transfer requirements
    - Sample at Nyquist limit (40kHz)
    - Choose higher resolution ADC to minimize quantisation error
![datarate](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/e2f4f8e3-7ad7-4880-bcec-dd1402deb6ed)
### USB Standard
    - Full Speed - Up to 12 Mbits/s (this is plenty enough)
    - High Speed - Up to 480 Mbits/s
    - System is bus powered (not a host) and will most likely require <500mA
    - Use type C, new universal standard
### Processor
    - Low bandwidth and storage requirements so FPGA unnecessary
    - Use MCU
        - C easy compared to Verilog
        - SWD or JTAG protocols available 
    - Peripherals
        - USB for data stream
        - SPI for ADC and DAC
    - Required Power?
        - Cortex M0 should be adequate 
        - Avoid BGA or QFN packages (too expensive and benefits not necessary for our use case) 
        - I know STM32 platform well
### PSU
    - Digital Circuitry Supply
        - Switch mode supply is noisy but more efficient than LDO. Works for resillient digital circuitry
    - Analog Circuitry Supply
        - LDO for low noise
        - Less current required compared to digital circuitry typically
    - Power from USB
        - Must regulate due to noise
        - +5V nominal, can drop to +4.5V
        - 500mA can be negotiated, but 150mA is default so design for that

### Simple Converter Theory
    - Higher bits gives better signal to noise ratio
    - Higher sampling rate provides less aliasing with a higher Nyquist limit, however this produces more data to store, process, and stream
    - Consider using oversampling and antialiasing filters on the convert ICs themselves
    - Interface with SPI since data rate is low (typical for microcontrollers)
    
### Analog Circuitry for Converters
    - ADC
        - High-impedance (don't attenuate original signal), ESD protection, RF filtering
        - Antialiasing filter at 1/2 the sampling frequency (Nyquist Rate)
        - Single ended vs differential compatibility
    - DAC
        - Low-impedance with output buffer for driving a load
        - Antialiasing filter at 1/2 the sampling frequency (Nyquist Rate) 
        - Single ended vs differential compatibility

### Miscellaneous Requirements
    - Mechanical Constraints
        - PCB dimensions
        - Mounting Holes
        - Connector Placement
        - Conductive Case?
    - Connectors
        - Debugging interface
        - IO types (SMA, BNC)
    - Peripherals
        - LEDs?
    - Other
        - Timing
        - ESD protection
        - EMC filtering
        - Biasing for analog circuitry 
### Final Block Diagram
![finalblockdiagram](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/48f4dbe4-663c-48d4-8cd2-4369597c9597)

## Schematic Design - Power
![1](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/603f80e3-1b72-454e-8f95-f3b2b249c0d8)

### Input Pi Filter
    - The input RLC Pi filter from the USB-C has an inductor with a DC resistance of 0.15 Ohms and max DC of 0.5A
    - http://sim.okawa-denshi.jp/en/Fkeisan.htm -> Tools for filter design 
    - Using the website, we can see the Pi Filter has a cutoff in the 1.6 MHz range. For PSU, we want to keep the cutoff high to dampen high frequency noise
    - Be careful; IC transients are hungry for high frequency current and you don't want to starve them. Cutoff only what is needed to pass EMI tests. 
    
![Screenshot 2024-06-25 211942](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/de1b405d-80ce-4dce-917f-c72f35ba7fe9)

### Analog LDO regulator - HT75xx
    - Datasheet says to use at least 10uF for input and output caps. We use 22uF to avoid issues due to tolerances and voltage derating. 
    - Series Input resistor with at least 1/8W power rating creates low-pass filter RC with 22uF input capacitor
    - In general, RC filters have a much shallower roll-off compared to the RLC Pi Filter. They also have significant Johnson noise. 
    - Input low-pass has cutoff at 723 Hz. This is low but okay because it feeds a low freq analog supply. 

### Buck Converter
    - Main external circuitry
        - Input caps
        - Output LC filter - use below formula to calculate inductor value
        - Feedback network (formula straight from datasheet and also in schematic) 
        - Sometimes external FETs and diodes
#### [Choosing Inductor Value]
        - Delta IL is ripple current - try to get 20% - 30% of max output current
        - Design for worst case
            - Larger Vin
            - Low current loads
        - Choosing larger inductor than initially calculated
            - +- 20% typical tolerance
            - Max DC and sat current need to be higher than what we require in our worst peak current case (250mA)
            - Low current loads (down to 50mA) will require larger inductors
            - Is shielding important to us?
        - Plotting below function in DESMOS we see lighter loads require much larger inductance (6x) what we calculated, so we chose a 68 uF inductor
        - Although datasheet says we can use 4.7uF minimum caps, we want to consolidate our BOM, so choose 22uF as required by LDO
#### [DNP???]
    - We have a DNP (Do not place) cap because the datasheet recommends it for stability and speed
    - We still want the option to place it, so we will put pads on the PCB for it. 
![Screenshot 2024-06-25 214346](https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/91e6cef9-52f3-44ba-89f5-1ca48ed08062)

### Bias Generator
    - We have a single supply 0V -> 3.3 V
    - Bias an input to 1/2 VA to give max swing potential for superimposed AC signal 
    - Provide bypass cap as recommended by datasheet (place physically close on board)
    - In high freq designs, the voltage and ground planes seperated by a dielectric board material create a low value high speed capacitor. 

## Schematic Design - MCU

### MCU Digital Power Decoupling
    - As recommended by the datasheet, we have a 100uF cap for each digital power input for a total of four caps
    - We also have on 10uF bulk decoupling cap, also recommended by the datasheet
<img width="339" alt="Screenshot 2024-07-08 205031" src="https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/9264ed34-b8d7-43bb-84d7-65af0aa09342">

### MCU Analog Power Decoupling
    - We do not need to do this because we are using external DAC and ADC
    - Below shows the datasheet implementation if this were not the case
<img width="250" alt="Screenshot 2024-07-08 205635" src="https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/e4ca3d28-7c24-4d12-b280-546f959446f2">

### Config pins and Crystal Oscillator
    - NRST typically pulled high by internal 40k resistor. Route to SWD header for more control
    - BOOT0 on most STM32 uC. Tells chip to either run program flashed (low) or to listen for debug signals on UART or USB (high)
    - HSE_IN and HSE_OUT are our STM32 chips pins for an external crystal oscillator. Internal osc is okay for some applications, but not really accurate enough for UART or USB communication. 
    - The load capacitance (10p), ESR, and frequency of the crystal can all be found in the datasheet. 
    - For more info on calculating load caps, find AN2867 design guide for crystal oscillators. 

<img width="340" alt="Screenshot 2024-07-08 211210" src="https://github.com/JacobParent7/Mixed-Signal-STM32-Dev-Board/assets/105901480/64625600-abec-4d33-b82f-11d7694531cc">

## Microcontroller Pin Layout
    - The datasheet for any given uC will specify what functions the uC pins can serve. However, we can take advantage of the STM32 Cube IDE to do this for us and save time
### Pin layout description
    - Under SYS, set up serial wire debug for communication. Choose trace asynchronous Sw for live plotting or monitoring.
    - Under RCC, choose HSE to be crystal resonator for external circuitry
    - Under TIM, we can choose timer channels for PWM generation for LEDs. 
    - Under connectivity, set up receiving and transmitting master SPI channels for ADC and DAC. Choose check box under USB for full speed USB.
    
![image](https://github.com/user-attachments/assets/78c875ef-ba2b-4a0c-9837-d6d11c709275)

### Moving to KiCAD
    - Once you choose the desired pin configuration in Cube IDE, move back to KiCAD and map pins to schematic symbol shown below
    - Add pull up resistors on SPI hardware and chip select lines (not necessary if driven by GPIO), however, we are using hardware chip select pin from SPI drivers
    - On USB diff pair, name + and - to tell KiCAD to map as diff pair.
    - Check application note for STM32F1 to see if embededded pullup is included on D+ line. When we check our application note, we can see that an external 1.5 KOhm pullup is required.
![image](https://github.com/user-attachments/assets/e7230750-cc78-4dd9-8f6e-afe2b30f4e17)

## USB, RGB LED, and SWD header

### RGB LED
    - simple 1k current limiting resistors, chosen mostly for BOM consolidation since we are driving using PWM
    - decoupling capacitor since the LEDs are driven by PWM

![image](https://github.com/user-attachments/assets/1c62febd-3adf-40a8-999b-5aa16a218081)

### USB connector
    - Power and data connections are main 
    - USB C has more pins than typical USB A
    - Read AN4879 application note for specefics on USB layout
    - Supply is noisy, so use a filter like a Pi Filter and RLC filter (seen in PSU section above) 
    - Add ESD protection with low capacitance since we are using a relatively high bandwidth 
    - Data line filtering via common-mode choke will reduce signal integrity but improve EMC performance (not really needed for USB 2.0)
#### USB Type C specifics
    - TA0357 application note from ST
    - Instead of drawing 5V at 5mA, we can use a type C power delivery IC and get 100W (5A at 20 V). This is not required for our project.
    - Tell host we want to draw power by having seperate 5.1k pulldown resistors on CC1 and CC2 pins, where CC stands for configuration channel (Make sure to use two seperate resistors!)
#### Common mode choke specifics
    - Choke must be rated for USB 2 frequency range and not impede too much
    - Choke will have a differential impedance which must match the USB bus impedance (90 Ohm)
#### Specify signal path for readability 
    - Connector -> ESD -> CM Choke -> Pull-Up -> MCU

![image](https://github.com/user-attachments/assets/f449183e-5e5d-4832-922a-a74a124008ef)

### SWD Header
    - Look up SWD pinout on google for easy specification
    - SWD is a simple in-circuit debugging and programming method that requires only data and clock lines
    - ST uses ST-Link to make debugging very easy 
    - Include bi-directional TVS diodes on signal lines for protection 
    - Low-pass filter included on NRST line to prevent unwanted resets due to high frequency noise
#### SWD signals explained
    - Main signals are SWDIO and SWCLK
    - SWDIO is bidirectional data
    - SWCLK is clock signal 
    - SWO is an optional signal (trace line from earlier) which allows us to trace output in real time
    - NRST is a pin that if pulled low triggers a hard reset

![image](https://github.com/user-attachments/assets/768faf53-2171-47f7-aa8c-a10c581739b3)


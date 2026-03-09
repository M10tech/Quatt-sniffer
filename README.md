# Quatt-sniffer  (c) 2024-2025 M10tech

Purpose of this repo is to sniff info from the Quatt modbus interface into HomeAssistant

Reusing the framework of modbus and modbus_controller we introduce the role `sniffer`  
By using the `register_count` concept, we create the same register-ranges as are present on the bus.  
For Quatt, there are four write commands (FC=6) and one read command (FC=3)

## Setup
- disable the physical TX hardware in case your RS485-board does not have an explicit TX pin (aka auto TX mode)!!!
- for best performance, remove a 120 ohm resistor between A and B that might be present in the sniffer board
- likewise, remove a pull-up to Vcc and pull down resistor to GND which might decrease reliability of the CiC-Quatt communication!
- start with creating (_but do not install!_) a new device named `quatt-sniffer`  
- this will have a default template content with secrets in it
- capture the various secrets in the secrets.yaml file as QMB_api, QMB_ota and QMB_AP
- replace the entire content with the `quatt-sniffer.yaml` file in this repo.
- select the ESP32 board model in the packages section by uncommenting one line
- verify GPIO pin to be used for UART-RX-pin
- if you have a Quatt-DUO, un-comment the line for `HP2`
- save the config and install, BUT choose manual download and abort that if you are not ready yet
- if the compiling went well, upload to your device via USB, else debug
- carefully, first switch off Quatt by lowering the thermostat, then unplug Quatt and then CiC
- connect the A, B and G from CiC to the device, in the same connector that goes to the Quatt
- preferrably use cables less than a meter long and try to twist them together..
- power up ESP, CiC and Quatt and check that Quatt-app/json interface works properly for half an hour
- meanwhile, select QMBus-device in history and check that info comes in every second (or at throttle speed)
- NOW, if you feel like tuning, you can change as you like
- if a new version of the repo is available, you can re-copy or update the `quatt-sniffer.yaml` and recompile + install

#### example setup
![](https://github.com/M10tech/Quatt-sniffer/blob/main/Quatt-Sniffer-Ybema-foto.jpg)  
Red arrows: removal of 120 ohm (R2) and pull-up/down (R1/R3)  
Green arrow: RX will light every second if a message comes in. TX should NOT light!  
Yellow line: board model [available here](https://www.tindie.com/products/thehognl/esp32-c3-with-rs485-modbus-and-optional-touch-tft/)  
<img src="https://github.com/M10tech/Quatt-sniffer/blob/main/Quatt-Sniffer-Ybema-noTX.jpg" width="200" />  
Cut this trace to disable TX  
Note that not removing the 3 resistors is not breaking the system right away, but it is easy enough, so just do it.  
If you don't have a solder iron, just cut a gap in them with a cutter, with care.

ESP32 with [RS485 board](https://www.aliexpress.com/item/1005006727769995.html) also works...

### What does it all mean
There are 40 registers that get read and 4 registers that send orders to the Quatt.  
Not all read registers are known for their meaning, and of those many are static over all time observed.

A _mode_ is either on or off  
A _level_ can take discrete values, but is not directly a physical unit  
A _demand_ is the translation of a level to a physical unit  
And then there is the _actual_ measurement of the device delivering that unit  

| Register | Sensor name                  | Register | Sensor name                  |
| :---     | :--------------------------- | :---     | :--------------------------- |
| R3999    | Working Mode set by CiC      | R2010    | Pump Mode set by CiC         |
| R1999    | Compressor Level set by CiC  | R2015    | Pump Level set by CiC        |
| R2006    | Silent mode set by CiC       | R2116    | Evaporator Pressure          |
| R2099    | Working Mode Actual          | R2117    | Condenser Pressure           |
| R2100    | Compressor AC Voltage        | R2118b0  | Defrost Mode                 |
| R2101    | Compressor AC Current        | R2119b0  | Alarm - Main Line Current    |
| R2102    | Compressor Frequency Demand  | R2119b3  | Info - Compressor Oil Return |
| R2103    | Compressor Frequency Actual  | R2119b4  | Alarm - High Pressure Switch |
| R2104    | Fan Speed Maximum            | R2119b6  | Alarm - 1st Start Pre-heat   |
| R2105    | Fan Speed Actual             | R2119b9  | Alarm - AC High/Low Voltage  |
| R2107    | Electric Expansion Valve     | R2119b12 | Alarm - Low Pressure Switch  |
| R2108b2  | Bottom Heater                | R2119    | Status bits R2119            |
| R2108b3  | Crankcase Heater             | R2120    | Status bits R2120            |
| R2108b0  | Fan Low Speed Mode           | R2121    | Status bits R2121            |
| R2108b4  | Fan Defrost Speed Mode       | R2122/23 | Firmware        Version      |
| R2108b5  | Fan High Speed Mode          | R2123    | EEPROM Version               |
| R2108b6  | 4way Valve                   | R2127    | Main control board item No   |
| R2108b11 | Pump Relay                   | R2131    | Condensing Temperature       |
| R2108bx  | Other bits R2108             | R2132    | Evaporating Temperature      |
| R2110    | Outside Temperature          | R2133    | Water In Temperature         |
| R2111    | Evaporator Coil Temperature  | R2134    | Water Out Temperature        |
| R2112    | Gas Discharge Temperature    | R2135    | Condenser Coil Temperature   |
| R2113    | Gas Return Temperature       | R2137    | Pump Power                   |
| R2115    | DIPswitch readout            | R2138    | Pump Flow                    |

<img src="https://github.com/M10tech/Quatt-sniffer/blob/main/Quatt-Concepts.png" width="600" />  

This is a schematic diagram that tries to provide a guide to the various items mentioned above.  
The Condensing and Evaporating Temperature are derived from the respective pressures and
are a [physical property of the R32 cooling agent](https://www.vymeniky-tepla.cz/files/pdf/r32-pt-chart.pdf). They are not physical temperature sensors.

The Status bits are uncertain, and there is no easy way to find out the more obscure ones.

### Coding strategy
The idea is to override `modbus` and make it catch all messages on the bus, both commands and responses.  
By flipping between actual_role of `client` and `server` we connect the register range available in the
command with the response.  
Any reponse that passes CRC is sent up to `modbus_controller` in `on_modbus_message(FC,start_addr,reg_count,data)`  
The override of `modbus_controller` creates a `ModbusCommandItem` in the `incoming_queue_` and the standard
processing takes care of sending the selected registers into HA.

These two override components are not compatible with the originals anymore,  
however, the intent is that it would be possible to merge them again.  
As such, this would then become an extension of the originals.  
If time and demand come together, this might happen.

There was an issue that did not allow high volume sensor updates without missing data.
This was solved in ESPhome since 2025.3.0

### Use this for other modbus machines
Say you want to use this for some other bus, not Quatt.  
Remove (or comment) all of the register definitions and enable verbose logging  
If your bus is active you could see messages like this:
```
[18:20:59][D][modbus:141]: good CRC as server for address=1     with FC=6 , offset=2 and len=4   => start@2015 #1
[18:20:59][D][modbus:141]: good CRC as client for address=1     with FC=6 , offset=2 and len=4   => start@2015 #1
[18:20:59][D][modbus:141]: good CRC as server for address=1     with FC=3 , offset=2 and len=4   => start@2099 #40
[18:20:59][D][modbus:141]: good CRC as client for address=1     with FC=3 , offset=3 and len=80  => start@2099 #40
```
Use all this information to set up the registers in the yaml to match what happens on the bus.  
Do not forget to disable verbose logging once you have an idea of the registers...

### History

#### 2.0.0 matching CiC firmware 4.2.0+
- CiC now uses 7 blocks for reading instead of one and functioncode 16 instead of 6
- added command 2006 for silent mode (whatever that is)
- added DIPswitch register 2115
- added Main control board item No register 2127
- added registers 11160x20 and 11219x20 as ignored
  (you could add those if you know what they mean and your Quatt supports them)
- fixed a '/' in a name to '\' to prevent future error
- updated modbus.cpp to translate a function code 16 request in a code 6 response
  (only works if number of registers is 1, which it is (for now))
- tested with 2026.1.3 but initial syncing can take a long time (even an hour)
  (just give it time, usually it works fine once it syncs (and blame arduino loop architecture))

#### 1.1.0 sync with 2025.5.0 situation
- removed PR7547 reference since that is now part of release 2025.3.0 and later
- updated components to match changes up to release 2025.5.0

#### 1.0.0 official release
- components got version 1.0.X but have not changed

=================

#### 0.9.1 release candidate two
- fixed typo in R2119b4

#### 0.9.0 release candidate one
- updated sensor names in yaml files
- added EEPROM version
- documentation up to date
- component versions left at 0.8.X

#### 0.8.2++++ more description fine tuning

#### 0.8.2+++ description of sensor names draft
- to receive feedback before applying to actual sensors

#### 0.8.2++ final cosmetics
- verified there are no changes between esphome versions 2024.12.2 and 2024.12.4 for the local components

#### 0.8.2+ cosmetic updates and instructions

#### 0.8.2 fine tuning the reported registers
- board type is now a package which includes the right GPIO pin
- version of firmware R2122 is now reported
- renamed R2108b3 to crankcase heater (it is not bit4)
- temperature clamping always minimum -30
- now minor version will only be updated if the C++ code is updated

#### 0.8.1 hide support yaml files
- starting with dot makes them invisible to esphome
- only relevant to the developer

#### 0.8.0 Apply ESPHome packages for easier user configurations and updates
- minor improvements as well, e.g.
- throttling_value applied to all values from main substitution value
- incorporate pull request PR#7547 which solves missing updates

#### 0.7.3 merged changes from 2024.10.3 - 2024.12.2
- affects the modbus_controller component
- no overlap with the sniffer functions
- into branch to2024_12_2

#### 0.7.2 new bit in R2108
- split off bit3 from R2108
- fixed other_bits concept
- not sure anymore of bit4 is crankcase heater
- cosmetic changes as suggested by PR#2

#### 0.7.1 Report ranges, other_bits and logging levels
- dump_config now shows configured modbus_controller register ranges
- Register2108 unidentified bits now a separate sensor
- speculative descriptive names for bits (verify...)
- logging levels to WARN or DEBUG for better feedback

#### 0.7.0 Updated Readme with instructions and overview
- version now 0.7.0 to reflect that we are on our way to 1.0.0

#### 0.1.4 Added register 2108 full next to bits
- full register and bits can exist side by side
- added HP2 as a clone of HP1, but commented
- zapped some gremlins

#### Update quatt-sniffer.yaml (#1)
- Defrost to binary sensor
- Some HA icons added
- Range of Evaporator pressure increased
- Add some binary sensors (protections) for register 2119

#### 0.1.2 added expansion of bits in register 2108
- bit2 is floor_heating
- bit11 is pump_running
- throtteling example

#### 0.1.1 updated YAML to meaningful register names
- also updated min and max values which were too restrictive
- ToDo: split 2108 in separate bits, and maybe also 2119-2121 Status bits

#### 0.1.0 sniffing seems to work
- rudimentary sniff for fc 3 and 6
- ToDo: throttle, inventory, error, style guidelines

#### 0.0.6 introduced sniffer as a role
- this will steer behavior of the underlying systems

#### 0.0.5 use optimized register range
- makes a single read-range that matches the quatt read command
- using git works now as expected in 0.0.4

#### 0.0.4 renamed my_components to components
- seems to allow the use of the external components of source type git
- to be tested after this is committed

#### 0.0.3 Work in sniffer mode and detect commands and responses
- switches between command and response on every good CRC
- this provides the start address and number of registers for each response
- the parsing of messages sent up is not working yet

#### 0.0.2 Rename back to modbus and modbus_controller
- renaming is a non-trivial exercise
- will depend on local components blocking-out built-in ones

#### 0.0.1 Clone modbus and modbus_controller
- they provide a reasonable startpoint
- and a esphome yaml file to match

### Sample
```
01 06 07cf 0000 b881
01 06 07cf 0000 b881
01 06 07da 0000 a945
01 06 07da 0000 a945
01 06 07df 0000 b944
01 06 07df 0000 b944
01 06 0f9f 0000 baf0
01 06 0f9f 0000 baf0
01 03 0833 0028 b7bb
01 03 50 0000 00ea 0000 0000 0000 0000 0000 0000 0064 0000 01f4 191b 15d0 14ef 151a 0000 1800 0094 0093 0000 0000 0000 0000 011e 0072 0002 2e78 0000 0e37 0bb8 0bb8 0000 14a7 14d1 15a2 17f5 160a 0bb8 0000 0000 c907
```

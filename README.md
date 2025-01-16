# Quatt-sniffer  (c) 2024 M10tech

Purpose of this repo is to sniff info from the Quatt modbus interface into HomeAssistant

Reusing the framework of modbus and modbus_controller we introduce the role `sniffer`  
By using the `register_count` concept, we create the same register-ranges as are present on the bus.  
For Quatt, there are four write commands (FC=6) and one read command (FC=3)

## Setup
- disable the physical TX hardware in case your RS485-board does not have an explicit TX pin (aka auto TX mode)!!!
- for best preformance, remove a 120 ohm resistor between A and B that might be present in the sniffer board
- likewise, a pull-up to Vcc and pull down resistor to GND will decrease reliability of the CiC-Quatt communication!
- start with creating (_but do not install!_) a new device named `quatt-sniffer`  
- this will have a default template content with secrets in it
- capture the various secrets in the secrets.yaml file as QMB_api, QMB_ota and QMB_AP
- copy the ESP32 board model string
- replace the entire content with the `quatt-sniffer.yaml` file in this repo.
- set back the ESP32 board model and verify GPIO pin to be used for UART-RX-pin
- if you have a Quatt-DUO, un-comment all the registers for `HP2` (three sections!)
- install, BUT choose manual download and abort that if you are not ready yet
- if the compiling went well, upload to your device, else debug
- carefully, first switch off Quatt, then unplug Quatt and then CiC
- connect the A, B and G from CiC to the device
- power up ESP, CiC and Quatt and check that Quatt-app/json interface works properly for half an hour
- meanwhile, select QMBus-device in history and check that info comes in every second
- NOW, if you feel like tuning, you can change as you like
- if a new version of the repo is available, you can re-copy or update the yaml and recompile + install

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

#### 0.8 Packages for simple user config
- Apply ESPHome packages for easier user configurations and updates

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

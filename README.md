# ESPHome-Navien
ESPHome configuration for RS-485 control of Navien Hot Water Heaters


## Common Fields

The first 6 bytes appear to be a packet header. The first two bytes are common to all packets. The next three uniquely identify each of our three packet types. Byte index 5 is the length of the data, not including the header (6 bytes) or the final byte, could be a checksum. In other words, byte index 5 is the length of the packet minus 7 bytes.

| Index |	Value |	Notes |
|-------|---------|-------|
| 0 	| f7 	  | Common to all packets |
| 1 	| 05 	  | Common to all packets |
| 2 	|	      | Packet id byte 0 |
| 3 	|	      | Packet id byte 1 |
| 4 	|	      | Packet id byte 2 |
| 5 	|	      | Data length |
| … 	|	      | Data |
| N 	|	      | Checksum |

## Packet A (from heater)

This packet seems to have information related to water.

Here is a full example packet:

```
f7 05 50 50 90 22 42 00 00 05 14 72 37 2e 00 00 00 00 00 00 f8 8e 00 00 02 00 00 00 05 00 07 00 00 02 00 00 00 00 00 00 67
```

And the fields decoded so far:
| Index | Value | Notes                                                                                       |
|-------|-------|---------------------------------------------------------------------------------------------|
| 0     | f7    | Common to all packets                                                                       |
| 1     | 05    | Common to all packets                                                                       |
| 2     | 50    | Packet id byte 0                                                                            |
| 3     | 50    | Packet id byte 1                                                                            |
| 4     | 90    | Packet id byte 2                                                                            |
| 5     | 22    | Data length                                                                                 |
| 8     | 00    | Related to recirculation status 0x00, 0x08, 0x20 observed                                   |
| 9     | 05    | System power: high nibble: unknown (values 0 and 0x20 observed); low nibble: 0=off 0x5=on   |
| 11    | 72    | Set temp - measured in 0.5 degrees C. 57C in this case                                      |
| 12    | 37    | Heat Exchanger Outlet temp - measured in 0.5 degrees C. 27.5C in this case                  |
| 13    | 2e    | Heat Exchanger Inlet temp - measured in 0.5 degrees C. 27.5C in this case                   |
| 18    | 00    | Flow rate - measured in 0.1 liters per minute (divide by 10 to get LPM)                     |
| 24    | 02    | System status. Probably a bitwise field. Partially decoded: display units: 0x08 position: 1=metric 0=imperial. 0x02 position: 1=weekly 0=hotbutton   |
| 33    | 02    | If recirculation is enabled/disabled 0x02/0x00                                              |


## Packet B (from heater)

This packet seems to have information related to gas.

Here is a full example packet:

```
f7 05 50 0f 90 2a 45 00 0b 01 0c 03 17 00 72 6d 23 00 00 00 00 00 00 00 61 00 00 00 0b 00 1d 00 f6 34 00 00 05 00 00 00 00 00 aa 48 00 00 01 00 b3
```

And the fields decoded so far:
| Index | Value | Notes                                                                           |
|-------|-------|---------------------------------------------------------------------------------|
| 0     | f7    | Common to all packets                                                           |
| 1     | 05    | Common to all packets                                                           |
| 2     | 50    | Packet id byte 0                                                                |
| 3     | 0f    | Packet id byte 1                                                                |
| 4     | 90    | Packet id byte 2                                                                |
| 5     | 2a    | Data length                                                                     |
| 14    | 72    | Set temp - measured in 0.5 degrees C. 57C in this case                          |
| 15    | 6d    | Outlet temp - measured in 0.5 degrees C. 27.5C in this case                     |
| 16    | 23    | Inlet temp - measured in 0.5 degrees C. 27.5C in this case                      |
| 20    | 00    | almost identical to high byte kcal                                              |
| 22    | 00    | Low byte current gas usage in kcal                                              |
| 23    | 00    | High byte current gas usage in kcal                                             |
| 24    | 61    | Total gas usage in 0.1m³ (divide by 10 to get m³; 9.7 cubic meters in this case)|


## Packet C (from navilink)

This appears to be an announcement packet. It tells the heater there is a navilink attached. The heater uses this to disable local setting of the weekly schedule and instead require setting the schedule from the app.

Here is a full example packet:

```
f7 05 0f 50 10 03 4a 00 01 55
```

## Packet D (from navilink)

This is the command packet. tested each of the following commands: power off, power on, set temp, hotbutton. Each type of command uses a different field. The navilink app send each command independently, but the format suggests that more than one command could be combined into a single packet. The navilink repeats the on/off/set commands multiple times. For hotbutton presses, it sends the on command twice followed by the off command once.

Here is a full example:
```
f7 05 0f 50 10 0c 4f 00 0b 00 00 00 00 00 00 00 00 00 0a
```

| Index | Value | Notes                                                  |
|-------|-------|--------------------------------------------------------|
| 0     | f7    | Common to all packets                                  |
| 1     | 05    | Common to all packets                                  |
| 2     | 0f    | Packet id byte 0                                       |
| 3     | 50    | Packet id byte 1                                       |
| 4     | 10    | Packet id byte 2                                       |
| 5     | 0c    | Data length                                            |
| 8     | 0b    | Power: 0x0a=on 0x0b=off                                |
| 9     | 00    | Set temperature (units of 0.5C: 0x74 is 58C)           |
| 11    | 00    | Hotbutton: 0x01=press 0x00=release                     |


Current list of full commands sent from navilink (so you can see the checksums):

```
  f7 05 0f 50 10 0c 4f 00 0b 00 00 00 00 00 00 00 00 00 0a : off
  f7 05 0f 50 10 0c 4f 00 0a 00 00 00 00 00 00 00 00 00 ce : on
  f7 05 0f 50 10 0c 4f 00 00 74 00 00 00 00 00 00 00 00 ea : set to 58c
  f7 05 0f 50 10 0c 4f 00 00 72 00 00 00 00 00 00 00 00 c4 : set to 57c
  f7 05 0f 50 10 0c 4f 00 00 00 00 01 00 00 00 00 00 00 6a : hotbutton press
  f7 05 0f 50 10 0c 4f 00 00 00 00 00 00 00 00 00 00 00 2a : hotbutton release
  F7,05,0F,50,10,0C,4F,00,00,00,00,10,DF,00,00,00,00,00,3E : recirculation off from app
  F7,05,0F,50,10,0C,4F,00,00,00,00,00,DF,00,00,00,00,00,D4 : recirc off ack?
  F7,05,0F,50,10,0C,4F,00,00,00,00,08,D9,00,00,00,00,00,D0 : recirculation on from app
  F7,05,0F,50,10,0C,4F,00,00,00,00,00,D9,00,00,00,00,00,14 : recirc on ack?
```

## Checksum

The last byte sent could be a checksum value, but the algorithm is currently unknown
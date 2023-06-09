# screentronic

This python library allows for interfacing with ScreenTronic blinds by CONTROLtronic via IP. This can be either dirctly to a specific ST2134 or though the same smartcontrol API used by the app.

## Installation

Use pip

`pip install screentronic`

## Usage

### SmartControl

This uses the same API used to control the blinds as via the ScreenTronic APP.

```python
from screentronic.smartControl

smartControl = screentronic.smartControl.SmartControl()
blind = smartControl.blind( (1,1,1,5) )
blind.up()
blind.down()
blind.set_position( height=128, angle=192 )
print( blind.position() )
```

### ST2134

Interfacing directly with a specific ST2134 is also possible in a similar way to how the CTS (Configuration Tool Software for Blinds by CONTROLtronic) works. Here the ST2134 is used as a gateway to talk directly to  (activated) the blinds themselfs allowing for lower level access.

```python
import time
from screentronic.directControl import ST2134

module = next( screentronic.directControl.s.ST2134.discover(), None )
print(f"Found the following device: {module=}")
module.identify()
module.info( 1 ) # Print the most important blind info

blind = module.blind( 1 )
blind.up()
time.sleep(5)
blind.stop()
blind.set_position( height=128, angle=192 )
print( blind.position() )
```

### Custom packet handler

If you want to create a simple service to interact with the ScreenLine blindes, and snoop used UDP multicast address you can create a custom listner as follows;

```python
import asyncio
import screentronic.Client

class MyClient(screentronic.Client):
  async def handle( self, pkt ):
    print(f"Received: {pkt}")

myclient = MyClient()
loop = asyncio.get_event_loop()
loop.run_until_complete( myclient.loop() )
```

## ScreenLine protocol operations

### SmartControl UDP Multicast

All packets start with b'CT' and end with b'\n'. By default all communication happens on the multicast IP 224.0.43.54 with dst port 43541. The smartcontrol (app) packets are not encrypted and have the following general format;

```
'C',                  # Constant
'T',                  # Constant
17,                   # 17 = ControlGroup packet type
1,                    # ???
controlGroup[0],      # Control group ID Byte 1
controlGroup[1],      # Control group ID Byte 2
packetId,             # Packet counter, to keep track of request and reponses that belong together ?
duplicate,            # Packet repeat counter, sometimes the same packet is sent multiple times (sine udp is unreliable). The only difference between duplicate packets is this counter that is incremented. New packets always start at 1
frame_len,            # Total frame length in #bytes
controlGroup[2],      # Control group ID Byte 3
controlGroup[3],      # Control group ID Byte 4
payload,              # Command type
payload,              # Control type
payload,              # Control command
payload,              # Data byte 1
payload,              # Data byte 2
crc,                  # Frame CRC (sum of all (header+payload) & 0xff)
'\n'                  # End-of-frame char 
```

### ST2134 UDP Multicast

All packets start with b'CT' and end with b'\n'. By default all communication happens on the multicast IP 224.0.43.54 with dst port 43541. The packets are encrypted with a basic scheme, where the bytes values are offset by a fixed secret (except for payload bytes 5 & 11 which have their encryption in the packet itself).

```
'C',              # Constant
'T',              # Constant
0xff,             # Packet type
1,                # ???
1,                # ???
1,                # ???
packetId,         # Packet counter, to keep track of request and reponses that belong together ?
1,                # Packet repeat counter, sometimes the same packet is sent multiple times (sine udp is unreliable). The only difference between duplicate packets is this counter that is incremented. New packets always start at 1
frame_len,        # Total frame length in #bytes (header (15b) + payload + crc(1b) + end-of-frame(1b))
mac[0],           # Destination device mac address
mac[1],           # Destination device mac address
mac[2],           # Destination device mac address
mac[3],           # Destination device mac address
mac[4],           # Destination device mac address
mac[5],           # Destination device mac address
... payload ...
crc,              # Frame CRC (sum of all (header+payload) & 0xff)
'\n'              # End-of-frame char 
```

### ScreenLine phyisical interface

Three wires, GND, +24V, and data. 

When operating in DC (or dumb mode) simply applying power on GND & +24V will make the blind travel (untill the endstops). Reversing poarity on the two lines makes the blind travel in the opposite direction. The data line us unused. There is no position data or feedback.

When operating in SC mode, the blind is always powered through GND & +24V. The data line is used to issue commands from the gateway (or serial interface) to the blinds. This happens through the data wire which is a ground referenced, single-ended CAN interface running on 24V at 20000bps.

Some more background info can be found in; https://patents.google.com/patent/EP2755318B1/en

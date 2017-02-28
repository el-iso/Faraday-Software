
# Tutorial - Local Telemetry Parsing

In this tutorial Telemetry packets from Faraday are queried, parsed, and displayed. Information about current system status, debugging information, and general telemetry are available.

##Telemetry Packet Types

* **System Operation Information**
  * Current operation details (frequency, power levels, etc...)
* **Device Debug Information**
  * Non-volatile system failure/reset counters useful for debugging
  * Firmware Revision
* **Main Faraday Telemetry**
  * Faraday telemetry that include all peripheral data (i.e GPS, ADC's, etc...) 

#Running The Tutorial Example Script

### Prerequisites
* Properly configured and connected proxy
  * 1x Faraday

# Running The Tutorial Example Script

## Configuration

* Open `telemetry_parsing.sample.ini` with a text editor
* Update `REPLACEME` from `CALLSIGN` to match the callsign of the Faraday unit **as assigned** in proxy
* Update `REPLACEME` from `NODEID` to match the callsign node ID of the Faraday unit **as assigned** in proxy
* Save the file as `telemetry_parsing.ini`
## Tutorial Output Examples

below is an example of the output of the tutorial script when run in a python interpreter (PyCharm).

```python
--- Telemetry Packet #1 ---
Index[0]: RF Freq 2 35
Index[1]: RF Freq 1 44
Index[2]: RF Freq 0 78
Index[3]: RF Power Bitmask 145
Faraday's Current Frequency: 914.499 MHz


18
--- Telemetry Packet #2 ---
Index[0]: Boot Count 106
Index[1]: Reset Count 0
Index[2]: Brownout reset counter 0
Index[3]: Reset / Non-maskable Interrupt counter 0
Index[4]: PMM Supervisor Low counter 0
Index[5]: PMM Supervisor High counter 0
Index[6]: PMM Supervisor Low - OVP counter 0
Index[7]: PMM Supervisor High - OVP counter 0
Index[8]: Watchdog timeout counter 0
Index[9]: Flash key violation counter 0
Index[10]: FLL Unlock counter 0
Index[11]: Peripheral / Config counter 0
Index[12]: Access violation counter 0
Index[13]: Firmware Revision 232278577 232278577


--- Telemetry Packet #3 ---
Source Callsign KB1LQD
Source Callsign Length 6
Source Callsign ID 1
Destination Callsign KB1LQD
Destination Callsign Length 6
Destination Callsign ID 1
RTC Second 19
RTC Minute 0
RTC Hour 0
RTC Day 1
RTC Day Of Week 4
RTC Month 1
Year 45575
GPS Lattitude 000000000
GPS Lattitude Direction 0
GPS Longitude 0000000000
GPS Longitude Direction 0
GPS Altitude 00000000
GPS Altitude Units 0
GPS Speed 00000
GPS Fix 0
GPS HDOP 0000
GPIO State Telemetry 192
IO State Telemetry 7
RF State Telemetry 0
ADC 0 139
ADC 1 88
ADC 2 1194
ADC 3 106
ADC 4 88
ADC 5 85
VCC 0
CC430 Temperature 26
ADC 8 2852
HAB Automatic Cutdown Timer State Machine State 0
HAB Cutdown Event State Machine State 0
HAB Automatic Cutdown Timer Trigger Time 7200
HAB Automatic Cutdown Timer Current Time 0
EPOCH 1488187725.92
Parsed packet dictionary: {'RFSTATE': 0, 'GPIOSTATE': 192, 'VCC': 0, 'RTCMIN': 0, 'SOURCECALLSIGN': 'KB1LQD', 'GPSLONGITUDE': '0000000000', 'SOURCEID': 1, 'RTCYEAR': 45575, 'ADC1': 88, 'BOARDTEMP': 26, 'DESTINATIONID': 1, 'EPOCH': 1488187725.92, 'RTCSEC': 19, 'GPSLATITUDE': '000000000', 'IOSTATE': 7, 'GPSSPEED': '00000', 'RTCDAY': 1, 'RTCMONTH': 1, 'GPSFIX': '0', 'RTCHOUR': 0, 'HABTIMER': 0, 'GPSHDOP': '0000', 'ADC3': 106, 'HABTRIGGERTIME': 7200, 'DESTINATIONCALLSIGNLEN': 6, 'GPSLONGITUDEDIR': '0', 'DESTINATIONCALLSIGN': 'KB1LQD', 'ADC4': 88, 'ADC5': 85, 'SOURCECALLSIGNLEN': '6', 'GPSALTITUDE': '00000000', 'ADC0': 139, 'GPSLATITUDEDIR': '0', 'ADC2': 1194, 'HABTIMERSTATE': 0, 'RTCDOW': 4, 'HABCUTDOWNSTATE': 0, 'ADC8': 2852, 'GPSALTITUDEUNITS': '0'}

Process finished with exit code 0
```

# Code Overview

Faraday may be configured to automatically send UART telemetry data in specific intervals (telemetry beacon) but each telemetry packet can be queried through a command sent to the unit. The example script provided performs the following steps to retrieve and display a telemetry packet:

* `FlushRxPort()`
  * Remove all prior data in the buffer in the respective Proxy port
  * Flushing all prior data to commanding the packet to be sent ensures within reason that the next retrieved data from the proxy FIFO is the intended packet type. This simplifies the tutorial.
* `POST()` Command to unit
  * Command unit to send the specified telemetry packet over UART
* The data packet(s) are retrieved from Proxy using `GetWait()`
  * `GetWait()` Allows the program to block until timeout while waiting for an empty FIFO in proxy to be filled with new data
* The first (only likely only) data packet returned from Proxy is BASE64 decoded
* The telemetry application frame is parsed using `UnpackDatagram()`
* All telemetry frames are fixed length, `ExtractPaddedPacket()` is used to extract the expected (smaller) fixed length Systems Settings packet from the received data
* The telemetry packet is parsed and displayed




## Code - Parse Telemetry Packet Type #1 (System Settings)

The System Settings packet is defined as telemetry packet type #1 and is shown below to be queried, decoded, and displayed.

**Notable Parsing Operations**
* The Systems Settings packet (Packet Type #1) is parsed using `UnpackPacket_1()`
  * `debug=True` causes the parsing function to display information about the packet as it is decoded
* `freq0_reverse_carrier_calculation()` calculate the frequency of the CC430 radio in MHz.



```python
############
## System Settings
############

#Flush old data from UART service port
faraday_1.FlushRxPort(local_device_callsign, local_device_node_id, faraday_1.TELEMETRY_PORT)

#Command UART Telemetry Update NOW
faraday_1.POST(local_device_callsign, local_device_node_id, faraday_1.CMD_UART_PORT, faraday_cmd.CommandLocalSendTelemDeviceSystemSettings())

#Wait up to 1 second for the unit to respond to the command. NOTE: GETWait will return ALL packets received if more than 1 packet (likley not in THIS case)
rx_settings_data = faraday_1.GETWait(local_device_callsign, local_device_node_id,faraday_1.TELEMETRY_PORT, 1, False) #Will block and wait for given time until a packet is recevied

#Decode the first packet in list from BASE 64 to a RAW bytesting
rx_settings_pkt_decoded = faraday_1.DecodeRawPacket(rx_settings_data[0]['data'])

#Unpack the telemetry datagram containing the standard "Telemetry Packet #3" packet
rx_settings_datagram = faraday_parser.UnpackDatagram(rx_settings_pkt_decoded, debug = False) #Debug is ON
rx_settings_packet = rx_settings_datagram['PayloadData']

#Extract the exact debug packet from longer datagram payload (Telemetry Packet #2)
rx_settings_pkt_extracted = faraday_parser.ExtractPaddedPacket(rx_settings_packet, faraday_parser.packet_1_len)

#Parse the Telemetry #3 packet
rx_settings_parsed = faraday_parser.UnpackPacket_1(rx_settings_pkt_extracted, debug = True) #Debug ON

# Print current Faraday radio frequency
faraday_freq_mhz = cc430radioconfig.freq0_reverse_carrier_calculation(rx_settings_parsed['RF_Freq_2'], rx_settings_parsed['RF_Freq_1'], rx_settings_parsed['RF_Freq_0'])
print "Faraday's Current Frequency:", str(faraday_freq_mhz)[0:7], "MHz"
```


## Code - Parse Telemetry Packet Type #2 (Device Debug)

The actions needed to flush, command, unpack from the telemetry datagram, and parse telemetry packet type #2 are the same as previously exampled except that parsing routines for packet type #2 are specifically used.  Notably `CommandLocalSendTelemDeviceDebugFlash()` and `UnpackPacket_2()` command the sending / parsing of telemetry packet type #2 respectively.

```python
############
## Debug
############

#Flush old data from UART service port
faraday_1.FlushRxPort(local_device_callsign, local_device_node_id, faraday_1.TELEMETRY_PORT)

#Command UART Telemetry Update NOW
faraday_1.POST(local_device_callsign, local_device_node_id, faraday_1.CMD_UART_PORT, faraday_cmd.CommandLocalSendTelemDeviceDebugFlash())

#Wait up to 1 second for the unit to respond to the command. NOTE: GETWait will return ALL packets received if more than 1 packet (likley not in THIS case)
rx_debug_data = faraday_1.GETWait(local_device_callsign, local_device_node_id,faraday_1.TELEMETRY_PORT, 1, False) #Will block and wait for given time until a packet is recevied

#Decode the first packet in list from BASE 64 to a RAW bytesting
rx_debug_data_pkt_decoded = faraday_1.DecodeRawPacket(rx_debug_data[0]['data'])

#Unpack the telemetry datagram containing the standard "Telemetry Packet #3" packet
rx_debug_data_datagram = faraday_parser.UnpackDatagram(rx_debug_data_pkt_decoded, False) #Debug is ON
rx_debug_data_packet = rx_debug_data_datagram['PayloadData']

#Extract the exact debug packet from longer datagram payload (Telemetry Packet #2)
rx_debug_data_pkt_extracted = faraday_parser.ExtractPaddedPacket(rx_debug_data_pkt_decoded, faraday_parser.packet_2_len)

#Parse the Telemetry #3 packet
rx_debug_data_parsed = faraday_parser.UnpackPacket_2(rx_debug_data_pkt_extracted, debug = True) #Debug ON
```


## Code - Parse Telemetry Packet Type #3 (Main Faraday Telemetry)

The actions needed to flush, command, unpack from the telemetry datagram, and parse telemetry packet type #3 are the same as previously exampled except that parsing routines for packet type #2 are specifically used.  Notably `CommandLocalUARTFaradayTelemetry()` and `UnpackPacket_3()` command the sending / parsing of telemetry packet type #3 respectively.

`rx_telemetry_packet_parsed` is printed after reception and parsing to example the raw parsed dictionary item returned from the parsing function(s).

```python
############
## Telemetry
############

#Flush old data from UART service port
faraday_1.FlushRxPort(local_device_callsign, local_device_node_id, faraday_1.TELEMETRY_PORT)

#Command UART Telemetry Update NOW
faraday_1.POST(local_device_callsign, local_device_node_id, faraday_1.CMD_UART_PORT, faraday_cmd.CommandLocalUARTFaradayTelemetry())

#Wait up to 1 second for the unit to respond to the command. NOTE: GETWait will return ALL packets received if more than 1 packet (likley not in THIS case)
rx_telem_data = faraday_1.GETWait(local_device_callsign, local_device_node_id,faraday_1.TELEMETRY_PORT, 1) #Will block and wait for given time until a packet is recevied

#Decode the first packet in list from BASE 64 to a RAW bytesting
rx_telem_pkt_decoded = faraday_1.DecodeRawPacket(rx_telem_data[0]['data'])

#Unpack the telemetry datagram containing the standard "Telemetry Packet #3" packet
rx_telemetry_datagram = faraday_parser.UnpackDatagram(rx_telem_pkt_decoded) #Debug is OFF
rx_telemetry_packet = rx_telemetry_datagram['PayloadData']

#Extract the exact debug packet from longer datagram payload (Telemetry Packet #3)
rx_telemetry_datagram_extracted = faraday_parser.ExtractPaddedPacket(rx_telemetry_packet, faraday_parser.packet_3_len)

#Parse the Telemetry #3 packet
rx_telemetry_packet_parsed = faraday_parser.UnpackPacket_3(rx_telemetry_datagram_extracted, debug = True) #Debug ON

print "Parsed packet dictionary:", rx_telemetry_packet_parsed

```
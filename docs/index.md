# arkIQ System Documentation

### TTN to AWS
    

![TTN AWS Integration Diagram](https://arkiq-open-images.s3.us-east-1.amazonaws.com/ttn+aws+integration+diagram.png)
    

1. **Uplink sent from TTN to AWS**
    

2. **AWS IoT Core** receives the uplinks.  
   A separate **IoT Core Rule** is defined for each device type, extracting the necessary information from each uplink.
    

3. The extracted data is sent to a **dedicated queue** for each device type.
    

4. Each queue triggers a **separate Lambda function**, which processes the received data and follows a distinct path depending on the device.
    

5. Alerts  
If necessary, alerts are sent via **email** and **SMS**.
    

6. Database Storage  
Processed data is stored in the **database**.
    

**From now on (September, 2025) the new devices should be onboarded on AWS IoT Core**

---
    

---
    

---
    

### Device Types:
    

arkIQ has different types of devices
    

1- IQWL: Leak Sensor  
2- IQSL: Leak Sensor + Toilet Sensor  
3- IQWM: Water Meter  
4- IQSV: Smart Valve  
    

### IQWL
    

Attributes:
Water Leak Detected: boolean  
Tamper Detected: boolean  
Button Pressed: boolean  
Temperature (Celsius): decimal  
Humidity (%): decimal  
Battery (V): decimal  
    

Payload:
    

Bytes:
00 08 d2 00 3e
    

Criptographic:
AAjSAD4=
    

Decoded:
```json
{
  "battery_volt": 2.9,
  "button": 0,
  "humi": 62,
  "tamper": 0,
  "temperature": 21,
  "water": 0
}
```
    

Payload Formatter (TTN):
```js
function hex2bin(hex){
  return (parseInt(hex, 16).toString(2)).padStart(8, '0');
}
    

//IQWL sensor
function decodeUplink(input) {
	let fPort = input.fPort;
	let payloadlens = input.bytes.length;
	if(fPort==126 && payloadlens==5){
		let intput_list = input.bytes;
		let battery_int=intput_list[1];// battery calculate
		battery_volt = (21+battery_int)/10;
		temperature_hex= (intput_list[3].toString(16).padStart(2, '0'))+(intput_list[2].toString(16).padStart(2, '0'));  //temperature calculate
		if((parseInt(temperature_hex, 16))>1250){
			temperature = ((parseInt(temperature_hex, 16))-65536)/10;
		}
		else{
			temperature = (parseInt(temperature_hex, 16))/10;
		}
		
		humi =  intput_list[4]; //Humidity calculate
		let water_hex = intput_list[0].toString(16).padStart(2, '0'); // Sensor Status calculate
		let water_binary = hex2bin(water_hex); 
		let water_st = water_binary.substring(7, 8); 
		let button_st = water_binary.substring(6, 7); 
		let tamper_st = water_binary.substring(5, 6); 

		water = parseInt(water_st); // water status
		button = parseInt(button_st); // Button pressed
		tamper = parseInt(tamper_st); // Tamper detected
		  
		return {
		data: {
		battery_volt,
		temperature,
		humi,
		water,
		button,
		tamper
		},
		};
	}
	else if (fPort==126 && payloadlens==4){
		let intput_list = input.bytes;
		let battery_int=intput_list[1];// battery calculate
		battery_volt = (21+battery_int)/10;
		temperature_int= intput_list[2]; //temperature calculate
		if(temperature_int>125){
			temperature = temperature_int-256;
		}
		else{
			temperature = temperature_int;
		}
		humi =  intput_list[3]; //Humidity calculate
    

		let water_hex = intput_list[0].toString(16).padStart(2, '0'); // Sensor Status calculate
		let water_binary = hex2bin(water_hex); 
		let water_st = water_binary.substring(7, 8); 
		let button_st = water_binary.substring(6, 7); 
		let tamper_st = water_binary.substring(5, 6); 
    

		water = parseInt(water_st); // water status
		button = parseInt(button_st); // Button pressed
		tamper = parseInt(tamper_st); // Tamper detected
    

		return {
		data: {
		battery_volt,
		temperature,
		humi,
		water,
		button,
		tamper
		},
		};
	}
	//Fireware Info
	else if (fPort==204 && payloadlens==9){
    let message = 'This is fireware info.';
    byteArray = input.bytes;
    let payload = byteArray.map(byte => byte.toString(16).padStart(2, '0')).join('');
    return {
      data: {
			  fPort,
				message,
				payload
			  },
			};
	}
	//Configuration Info
	else if (fPort==204 && payloadlens==11){
    let message = 'This is configuration info.';
    byteArray = input.bytes;
    let payload = byteArray.map(byte => byte.toString(16).padStart(2, '0')).join('');
    return {
      data: {
			  fPort,
				message,
				payload
			  },
			};
	}
	else{
			let fPort = input.fPort;
			let payloadlength = input.bytes.length;
			let message = 'Invalid fPort or payload length';
			return {
			  data: {
				fPort,
				payloadlength,
				message,
			  },
			};
	}
}
```
    

### IQWL Dual Chip
    

Some IQWL has dual chip:  
1. LoRaWAN  
2. Amazon Sidewalk  
    

By default, when it's turned ON, it will try to connect to LoRaWAN.  
    

If after 3 attempts, it doesn't connect to LoRaWAN, it will connect to Amazon Sidewalk.  
    

#### What are the difference between IQWL chip LoRaWAN and IQWL chip Sidewalk
    

* LoRaWAN needs to be connected to a network and be cover by a LoRaWAN gateway. Sidewalk doesn't need Gateways.
* Sidewalk uses the devices from Amazon to spread the connection
* Sidewalk nowadays (year 2025) only work in United States, but soon it will be active to other countries as well ([coverage.sidewalk.amazon](https://coverage.sidewalk.amazon/))
    

**In September 2025, the Sidewalk devices are not in arkIQ cloud. It's in another cloud. The uplinks are forwarded via a Lambda that is connected to an AWS IoT Core Rule from the other AWS Account. We are working on it to bring all the Sidewalk chips to our cloud**
    

The Lambda that receives the messages from Amazon Sidewalk is called ReceiveUplinkBrowanSidewalkLambda. This is the payload structure of every payload:
    

```json
{
    "MessageId": "48a0c3cb-7dc6-4e03-b738-9f3565808bd0",
    "WirelessDeviceId": "cda427e6-1f7c-45bb-aeb2-7324852a191f",
    "PayloadData": "N2UwMDBmY2YwMDQw",
    "WirelessMetadata": {
        "Sidewalk": {
            "CmdExStatus": "COMMAND_EXEC_STATUS_UNSPECIFIED",
            "MessageType": "CUSTOM_COMMAND_ID_NOTIFY",
            "NackExStatus": [],
            "Seq": 20,
            "SidewalkId": "B659A2463A",
            "Timestamp": "2025-09-22T14: 47: 18.911Z"
        }
    }
}
```
    

This payoload doesn't contain the DEVEUI or the Serial Number of the device. Rather, it contains the **WirelessDeviceId**
To assign which device is related to this particular uplink, the **WirelessDeviceId** need to be used.   
There is a table in the database that contains the **serial-number WirelessDeviceId** relationship: **iqwl_sidewalk_deviceid_connector**  
    

**In the future, when all Amazon Sidewalk chips will be onboarded on our cloud, then we'll implement a more efficient and less manual way to assign the devices with the uplinks.**
    

    

### IQSL
    

This device is in the same time a **Leak Sensor** and a **Transmitter**:  
    

1. Leak Sensor: detects leak using its probs (Similar to IQWL devices).  
    

2. Transmitter: it's possible to connect a headphone jack wth two functionalities:  
    

* Extends Leak Sensor capacity. Another device can be connected to the IQSL that detects Water Leaks. So, using the Headphone Jack, the other device will let the IQSL know that there is an external Leak. **Attribute externalLeakDetectionEnabled = TRUE to extend the Leak Detection to the headphone jack. Attribute externalLeakDetected = TRUE if the extensor detects leak.**  
    

* Connect a Toilet Sensor. The Toilet Sensor is a device that can be connected directly on a Toilet Pipe and it will give information about flushes and water usage. It can be connected using the Headphone Jack and will send toilet information to the transmitter. **attribute externalLeakDetected = FALSE to make Toilet Sensor works.**  
    

**The platform need to show if the IQSL is connected to a Leak Sensor, or to a Toilet Sensor, or if there's no headphone jack connected**  
    

**1- Heartbeat Packet (fPort = 0x01)**  
Attributes:  
* **Battery Voltage (V)**  
* **batteryPercentage (%)**  
* **Temperature (Celsius)**  
* **Humidity (%)**  
* **pinLeakDetectionEnabled: boolean (default: true)**
* **externalLeakDetectionEnabled: boolean** (TRUE to work the Leak Sensor extensor. FALSE to work the Toilet Sensor)  
* **tamperDetectionEnabled: boolean**  
* **pinLeakDetected: boolean** (if it's occuring a leak right now)  
* **externalLeakDetected: boolean** (if Toilet Sensor connected, then always FALSE)  
* **powerDetected: boolean** (No idea what is this)  
* **jackDetected: boolean** (True, if connected to a Toilet Sensor)  
* **magnetDetected: boolean** (Tamper detection using a magnet)  
* **Water Leak Detected: boolean**  
* **Tamper Detected: boolean**  
* **Button Pressed: boolean**  
* **Temperature (Celsius): decimal**  
* **Humidity (%): decimal**  
    

Payload:
    

Bytes:
01B06408D614720BFFB405
    

Criptographic:
AbBkCNYUcgv/tAU=
    

Decoded:
```json
        "data": {
          "batteryPercentage": 100,
          "batteryVoltage": 3.52,
          "enabledAlerts": {
            "externalLeakDetectionEnabled": false,
            "pinLeakDetectionEnabled": true,
            "tamperDetectionEnabled": true
          },
          "humidity": 52.34,
          "lastRssi": -76,
          "lastSnr": 11,
          "state": {
            "externalLeakDetected": false,
            "jackDetected": false,
            "magnetDetected": false,
            "pinLeakDetected": false,
            "powerDetected": false
          },
          "stateAsUint8": 5,
          "temperature": 22.62
        },
        "payloadType": "Heartbeat",
        "payloadTypeId": 1
```
    

**2- Alert Packets (fPort = 0x02)**
Triggered when a specific event occurs.  
The first byte (bytes[0]) defines the alert type:  
    

**Case 1: External Leak (via headphone jack)** - not applied, because the Toilet Sensor will be connected to the headphone Jack, not a Leak Sensor Extensor.  
Attributes:  
* **triggerValue:** (No idea what is it)
* **currentValue: int** (how moisture is it. Can define the level to consider it a leak detected)
* **temperature** 
* **humidity** 
* **delayCount** (No idea what is it)
* **currentState: boolean** (true = leak detected. false = leak cleared) 
    

(IMPORTANT)
Case 2: Local Leak (via bottom contact pins) **Only activate if the attribute pinLeakDetectionEnabled = true**
Attributes:
triggerValue: (No idea what is it)
currentValue: int (how moisture is it. Can define the level to consider it a leak detected)
temperature 
humidity 
delayCount (No idea what is it)
currentState: boolean (true = leak detected. false = leak cleared) 
    

Case 3: Tamper (magnet detected) - Activate when the magnet touch the device, simulating a tamper. **Only activate if the attribute tamperDetectionEnabled = true**
  

Case 4: Push button pressed - Activated when button is pressed
  

Case 5: Temperature High. Defined by downlink th hreshold
Case 6: Temperature Low.
Case 7: Humidity High.
Case 8: Humidity Low.
  From 5 to 8: Each includes current value, trigger threshold, guardband (safety margin), delay, and state.
  

(IMPORTANT)
Case 9: Total Events (cumulative counter): **Toilet Sensor Heartbeat**
Attributes:
longEventTriggered: boolean (is it occuring a Long Flush right now? True = yes. False = No)
alertInterval (no Idea what is it)
deviceTypeId (no Idea what is it)
totalEvents: int (How many flushes occured since the last heartbeat)
totalLiters: decimal (How many litters were spent since the last heartbeat) -> **Convert to Gallons (litters ** 0.264172)**
totalPulses: decimal (How many pulses were spent since the last heartbeat)
  

  

Payload:
  

Bytes:
0900001661000101000005
  

Criptographic:
CQAAFmEAAQEAAAU=
  

Decoded:
```json
{
        "alertType": "Pulse Counting (Total Events)",
        "data": {
          "longEventTriggered": 0,
          "settings": {
            "alertInterval": 500,
            "deviceTypeId": 1
          },
          "totalEvents": 1,
          "totalLiters": 5.319405756731662,
          "totalPulses": 5729
        },
        "payloadType": "Alert",
        "payloadTypeId": 9
      }
```
  

  

(IMPORTANT)
Case 10: Long Event Alert (long-duration events): **Toilet Sensor Long Event detected**
  

Attriubtes:
currentState: boolean (True = started. False: Ended)
durationTrigger: int (How many time flushing is consider a Long Flush and it will trigger this payload - in milliseconds)
  

Note: this payload will only be triggered after the pulse_duration_trigger interval. So we don't know when the flush started, but we know when the flush is running for a long time. So in order to identify when the flush started, it has to subtract the pulse_duration_trigger value.
  

Payload:
  

Bytes:
-
  

Criptographic:
-
  

Decoded:
```json
{
        "alertType": "Pulse Counting (Long Event Alert)",
        "data": {
          "currentState": 1,
          "settings": {
            "durationTrigger": 180000
          },
        },
        "payloadType": "Alert",
        "payloadTypeId": 10
      }
```
  

(IMPORTANT)
Case 11: Last Event Usage (usage of the last event): **Toilet Sensor water usage when a flush (normal or long) ends**
  

Attriubtes:
totalLiters: decimal (how many litters was spent in the last flush)  -> **Convert to Gallons (litters x 0.264172)**
totalPulses: decimal (how many pulses was spent in the last flush)
humidity (%)
temperature (Celsius)
  

Note: this payload will be triggered only after a flush (normal or long). 
- If this flush was a normal flush, we don't know how much time this flush was running. However, we know by this payload when it finished.
- If this flush was a long flush, we know when it started, because the Long Flush payload (type: 10) was triggered before this payload of Last Event Usage (type: 11).
- One challenge is to assign the Last Event Usage payload (type: 11) with the Long Flush payload (type: 10), because the payloads will be triggered in different moments. Also, the Last Event Usage payload doesn't indicate if the last event usage was long or a normal flush.
- If the last flush spent less then 1 gallon. So this is consider a **Scape**, not a normal flush 
  

Payload:
  

Bytes:
0B00001CCC01088115FF
  

Criptographic:
CwAAHMwBCIEV/w==
  

Decoded:
```json
{
  "alertType": "Pulse Counting Event (Last Event Usage)",
  "data": {
    "humidity": 56.31,
    "settings": {
      "deviceTypeId": 1
    },
    "temperature": 21.77,
    "totalLiters": 6.84493964716806,
    "totalPulses": 7372
  },
  "payloadType": "Alert",
  "payloadTypeId": 11
}
```
  

  

(IMPORTANT)
Case 12: Jack connected / disconnected
Attriubtes:
jackConnected: Boolean (true: yes. false: no)
  

Payload:
  

Bytes:
-
  

Criptographic:
-
  

**3- System Packets (fPort = 0x03)**
Maintenance/system packets.
  

Case 0x01: Network Test Packet → reports SNR and RSSI.
  

Case 0x02: Joined Uplink Packet → confirms joining the network and sends configs:
  

firmwareVersion, heartbeatInterval, sensorCheckInterval.
  

Which alerts are enabled (bitmasks in bytes[7] and bytes[8]).
  

Used for network testing and initial configuration after join.
  

**4- Parameter Values (fPort = 0x04)**
  

Here we have many subtypes (dozens). These are configuration/parameterization packets.
The first byte (bytes[0]) indicates which parameter. Examples:
Identifiers
* Case 0: devEui.
* Case 1: appEui.
* Case 2: appKey (disabled for security reasons).  
Network and operation configs
* Case 3–9: retry, confirm mode, join mode, ADR, device class, duty cycle, datarate.
* Case 10–15: RX1/RX2 delays, datarate, TX power.
* Case 16–19: region, channel mask, network mode, mode.  
Operation groups
* Case 21–26: heartbeat interval, join group, network test group, battery group, no-ack recovery, heartbeat group.
Event acknowledgements
* Case 27–37: Individual acks for external leak, pin leak, tamper, button, temp high/low, humidity high/low, pulse count, etc.  
Detailed sensor configs  
* Cases 38–46: Leak detection (pin/external) + buzzer + all-in-one parameters.
* Cases 47–59: Button, high/low temperature, high/low humidity, all with buzzer/guardband/delay.
* Case 60–61: temperature/humidity offsets.
* Case 62: sensing interval.
* Case 63–64: pulse counting configs.
* Case 65–69: Jack detection (enabled, buzzer, ack, etc.) and global buzzer control.
  

Summary of the Payload Packets:
  

* Heartbeat (0x01): general status (battery, SNR, sensors, flags).
* Alerts (0x02): triggered events (leak, tamper, button, temperature, humidity, pulses, jack).
* System (0x03): network/system packets (tests, join, firmware).
* Parameters (0x04): massive set of configuration parameters (network, ADR, alerts, delays, buzzer, offsets, etc.).
  

### IQSL Payload Formatter (TTN):
  

```js
function DoDecode(fPort, bytes) {
    var decoded = { data: {} };
    switch (fPort) {
        case 0x01: { // HeartBeat Packet
  

            decoded.data.state = {};
            decoded.data.enabledAlerts = {};
            decoded.payloadType = "Heartbeat";
            decoded.payloadTypeId = bytes[0];
            decoded.data.batteryVoltage = (bytes[1] / 100) * 2;
            decoded.data.batteryPercentage = bytes[2];
            decoded.data.temperature = UInt16(bytes[3] << 8 | bytes[4]) / 100;
            decoded.data.humidity = (bytes[5] << 8 | bytes[6]) / 100;
            decoded.data.lastSnr = Int8(bytes[7]);
            decoded.data.lastRssi = Int16(bytes[8] << 8 | bytes[9]);
            decoded.data.stateAsUint8 = bytes[10];
            decoded.data.enabledAlerts.pinLeakDetectionEnabled = ((bytes[10] & 0b00000001) > 0);
            decoded.data.enabledAlerts.externalLeakDetectionEnabled = ((bytes[10] & 0b00000010) > 0);
            decoded.data.enabledAlerts.tamperDetectionEnabled = ((bytes[10] & 0b00000100) > 0);
            decoded.data.state.pinLeakDetected = ((bytes[10] & 0b00001000) > 0);
            decoded.data.state.externalLeakDetected = ((bytes[10] & 0b00010000) > 0);
            decoded.data.state.powerDetected = ((bytes[10] & 0b00100000) > 0);
            decoded.data.state.jackDetected = ((bytes[10] & 0b01000000) > 0);
            decoded.data.state.magnetDetected = ((bytes[10] & 0b10000000) > 0);
        }
            break;
        case 0x02: { // Alert Packet
  

            decoded.payloadType = "Alert";
            decoded.payloadTypeId = bytes[0];
  

            switch (bytes[0]) {
                case 1: {
                    decoded.data.settings = {};
                    decoded.alertType = "External Leak Detection (Via Headphone Jack)";
                    decoded.data.settings.triggerValue = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.currentValue = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.temperature = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.humidity = UInt16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 2: {
                    decoded.data.settings = {};
                    decoded.alertType = "Local Leak Detection (Via Bottom Contact Pins)";
                    decoded.data.settings.triggerValue = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.currentValue = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.temperature = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.humidity = UInt16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 3: {
                    decoded.alertType = "Tamper Detection (Magnet Presence)";
                    decoded.data.currentValue = bytes[1];
                } break;
                case 4: {
                    decoded.alertType = "Push Button Pressed";
                    decoded.data.duration = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.turningOff = bytes[3];
                } break;
                case 5: {
                    decoded.data.settings = {};
                    decoded.alertType = "Temperature Alert (High)";
                    decoded.data.temperature = Int16(bytes[1] << 8 | bytes[2]) / 100;
                    decoded.data.settings.triggerValue = Int16(bytes[3] << 8 | bytes[4]) / 100;
                    decoded.data.settings.guardbandValue = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.humidity = UInt16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 6: {
                    decoded.data.settings = {};
                    decoded.alertType = "Temperature Alert (Low)";
                    decoded.data.temperature = Int16(bytes[1] << 8 | bytes[2]) / 100;
                    decoded.data.settings.triggerValue = Int16(bytes[3] << 8 | bytes[4]) / 100;
                    decoded.data.settings.guardbandValue = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.humidity = UInt16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 7: {
                    decoded.data.settings = {};
                    decoded.alertType = "Humidity Alert (High)";
                    decoded.data.humidity = UInt16(bytes[1] << 8 | bytes[2]) / 100;
                    decoded.data.settings.triggerValue = Int16(bytes[3] << 8 | bytes[4]) / 100;
                    decoded.data.settings.guardbandValue = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.temperature = Int16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 8: {
                    decoded.data.settings = {};
                    decoded.alertType = "Humidity Alert (Low)";
                    decoded.data.humidity = UInt16(bytes[1] << 8 | bytes[2]) / 100;
                    decoded.data.settings.triggerValue = Int16(bytes[3] << 8 | bytes[4]) / 100;
                    decoded.data.settings.guardbandValue = Int16(bytes[5] << 8 | bytes[6]) / 100;
                    decoded.data.temperature = Int16(bytes[7] << 8 | bytes[8]) / 100;
                    decoded.data.settings.delayCount = bytes[9];
                    decoded.data.currentState = bytes[10];
                } break;
                case 9: {
                    decoded.data.settings = {};
                    decoded.alertType = "Pulse Counting (Total Events)";
  

                    decoded.data.totalPulses = (bytes[1] << 24) + (bytes[2] << 16) + (bytes[3] << 8) + bytes[4];
                    decoded.data.totalEvents = UInt16(bytes[5] << 8 | bytes[6]);
                    decoded.data.settings.deviceTypeId = bytes[7];
                    decoded.data.longEventTriggered = bytes[8];
                    decoded.data.settings.alertInterval = UInt16(bytes[9] << 8 | bytes[10]) * 100;
  

                    switch (decoded.data.settings.deviceTypeId) {
                        // Generic Device Type, no usage calculations
                        case 0x00: {
  

                        } break;
                        // Toilet Flow Sensor
                        case 0x01: {
                            // F=18*Q-3 = 1077 pulses, Q(L/s) = f/1077, Q(L/min) = f*60/1077 = f/18*Q-3, Q(L/hour) = f*60*60/1077 = f*60/18*Q-3  
                            decoded.data.totalLiters = decoded.data.totalPulses / 1077;
                        } break;
                    }
  

                } break;
                case 10: {
                    decoded.data.settings = {};
                    decoded.alertType = "Pulse Counting (Long Event Alert)";
                    decoded.data.currentState = bytes[1];
                    decoded.data.settings.durationTrigger = UInt16(bytes[2] << 8 | bytes[3]) * 100;
                } break;
                case 11: {
                    decoded.data.settings = {};
                    decoded.alertType = "Pulse Counting Event (Last Event Usage)";
                    decoded.data.totalPulses = (bytes[1] << 24) + (bytes[2] << 16) + (bytes[3] << 8) + bytes[4];
                    decoded.data.settings.deviceTypeId = bytes[5];
                    decoded.data.temperature = Int16(bytes[6] << 8 | bytes[7]) / 100;
                    decoded.data.humidity = UInt16(bytes[8] << 8 | bytes[9]) / 100;
                    switch (decoded.data.settings.deviceTypeId) {
                        // Generic Device Type, no usage calculations
                        case 0x00:
                        case 0x01: {
                            decoded.data.totalLiters = decoded.data.totalPulses / 1077;
                        } break;
                    }
  

                } break;
                case 12: {
                    decoded.alertType = "Headphone Jack Alert";
                    decoded.data.currentValue = bytes[1];
                } break;
            }
        } break;
        case 0x03: { // System Packets
  

            decoded.payloadType = "System";
            decoded.payloadTypeId = bytes[0];
            switch (bytes[0]) {
                // "Network Test Packet"
                case 0x01: {
                    decoded.systemType = "Network Test Packet";
                    decoded.data.snr = Int8(bytes[1]);
                    decoded.data.rssi = Int16(bytes[2] << 8 | bytes[3]);
                } break;
                // "Joined Uplink Packet"
                case 0x02: {
                    decoded.data.settings = {};
                    decoded.data.settings.enabledAlerts = {};
                    decoded.systemType = "Joined Uplink Packet";
                    decoded.data.firmwareVersion = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.settings.heartbeatInterval = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.settings.sensorCheckInterval = UInt16(bytes[5] << 8 | bytes[6]);
                    decoded.data.settings.enabledAlerts.pinLeakDetection = ((bytes[7] & 0b00000001) > 0);
                    decoded.data.settings.enabledAlerts.externalLeakDetection = ((bytes[7] & 0b00000010) > 0);
                    decoded.data.settings.enabledAlerts.tamperDetection = ((bytes[7] & 0b00000100) > 0);
                    decoded.data.settings.enabledAlerts.pushButton = ((bytes[7] & 0b00001000) > 0);
                    decoded.data.settings.enabledAlerts.temperatureHigh = ((bytes[7] & 0b00010000) > 0);
                    decoded.data.settings.enabledAlerts.temperatureLow = ((bytes[7] & 0b00100000) > 0);
                    decoded.data.settings.enabledAlerts.humidityHigh = ((bytes[7] & 0b01000000) > 0);
                    decoded.data.settings.enabledAlerts.humidityLow = ((bytes[7] & 0b10000000) > 0);
                    decoded.data.settings.enabledAlerts.pulseCountingAlert = ((bytes[8] & 0b00000001) > 0);
                    decoded.data.settings.enabledAlerts.pulseCountingEventLongAlert = ((bytes[8] & 0b00000010) > 0);
  

                } break;
            }
  

        }
            break;
        case 0x04: { // Parameter Values
  

            decoded.payloadType = "Parameter Values";
            decoded.payloadTypeId = bytes[0];
  

            switch (bytes[0]) {
                // devEui Parameter
                case 0: {
                    let devEuiTemp = "";
                    devEuiTemp += bytes[1].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[2].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[3].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[4].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[5].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[6].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[7].toString(16).padStart(2, '0').toUpperCase();
                    devEuiTemp += bytes[8].toString(16).padStart(2, '0').toUpperCase();
  

                    decoded.parameterType = "devEui";
                    decoded.data.devEui = devEuiTemp;
                } break;
                // appEui Parameter
                case 1: {
                    let appEuiTemp = "";
                    appEuiTemp += bytes[1].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[2].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[3].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[4].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[5].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[6].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[7].toString(16).padStart(2, '0').toUpperCase();
                    appEuiTemp += bytes[8].toString(16).padStart(2, '0').toUpperCase();
  

                    decoded.parameterType = "appEui";
                    decoded.data.appEui = appEuiTemp;
                } break;
                // appKey Parameter
                case 2: {
                    decoded.parameterType = "appKey";   // This is not enabled for security reasons
                } break;
                // Retry Parameter
                case 3: {
                    decoded.parameterType = "retry";
                    decoded.data.retry = bytes[1];
                } break;
                // Confirm Mode Parameter
                case 4: {
                    decoded.parameterType = "Confirm Mode";
                    decoded.data.confirmMode = bytes[1];
                } break;
                // Join Mode Parameter
                case 5: {
                    decoded.parameterType = "Join Mode";
                    decoded.data.joinMode = bytes[1];
                } break;
                // ADR Parameter
                case 6: {
                    decoded.parameterType = "ADR";
                    decoded.data.adr = bytes[1];
                } break;
                // Device Class Parameter
                case 7: {
                    decoded.parameterType = "Device Class";
                    decoded.data.deviceClass = bytes[1];
                } break;
                // Duty Cycle Parameter
                case 8: {
                    decoded.parameterType = "Duty Cycle";
                    decoded.data.dutyCycle = bytes[1];
                } break;
                // Datarate Parameter
                case 9: {
                    decoded.parameterType = "Datarate";
                    decoded.data.datarate = bytes[1];
                } break;
                // Join Delay RX1 Parameter
                case 10: {
                    decoded.parameterType = "Join Delay RX1";
                    decoded.data.joinDelayRx1 = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // Join Delay RX2 Parameter
                case 11: {
                    decoded.parameterType = "Join Delay RX2";
                    decoded.data.joinDelayRx2 = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // Public Network Mode Parameter
                case 12: {
                    decoded.parameterType = "Public Network Mode";
                    decoded.publicNetworkMode = bytes[1];
                } break;
                // RX1 Delay Parameter
                case 13: {
                    decoded.parameterType = "RX1 Delay";
                    decoded.data.rx1Delay = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // RX2 Delay Parameter
                case 14: {
                    decoded.parameterType = "RX2 Delay";
                    decoded.data.rx2Delay = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // RX2 Datarate Parameter
                case 15: {
                    decoded.parameterType = "RX2 Datarate";
                    decoded.data.rx2Datarate = bytes[1];
                } break;
                // TX Power Parameter
                case 16: {
                    decoded.parameterType = "TX Power";
                    decoded.data.txPower = bytes[1];
                } break;
                // Region Parameter
                case 17: {
                    decoded.parameterType = "Region";
                    decoded.data.region = bytes[1];
                } break;
                // Channel Mask Parameter
                case 18: {
                    let channelMaskTemp = "";
                    channelMaskTemp += bytes[1].toString(16).padStart(2, '0').toUpperCase();
                    channelMaskTemp += bytes[2].toString(16).padStart(2, '0').toUpperCase();
  

                    decoded.parameterType = "Channel Mask";
                    decoded.data.channelMask = channelMaskTemp;
                } break;
                // Network Mode Parameter
                case 19: {
                    decoded.parameterType = "Network Mode";
                    decoded.data.networkMode = bytes[1];
                } break;
                // Mode Parameter
                case 20: {
                    decoded.parameterType = "Mode";
                    decoded.data.mode = bytes[1];
                } break;
                // Heartbeat Interval Parameter
                case 21: {
                    decoded.parameterType = "Heartbeat Interval";
                    decoded.data.heartbeatInterval = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // Join Group Parameter
                case 22: {
                    decoded.parameterType = "Join Group";
                    decoded.data.delayBetweenIterations = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.delayBetweenSequence = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.failCountToTriggerLongDelay = bytes[5];
                    decoded.data.longDelayBetweenSequence = UInt16(bytes[6] << 8 | bytes[7]);
                    
  

                } break;
                // Network Test Group Parameter
                case 23: {
                    decoded.parameterType = "Network Test Group";
                    decoded.data.interval = UInt16(bytes[1] << 8 | bytes[2]);
                    decoded.data.adr = bytes[3];
                    decoded.data.datarate = bytes[4];
                    decoded.data.txPower = bytes[5];
                } break;
                // Battery Group Parameter
                case 24: {
                    decoded.parameterType = "Battery Group";
                    decoded.data.batteryMin = (bytes[1] * 2) / 100;
                    decoded.data.batteryMax = (bytes[2] * 2) / 100;
                    decoded.data.batteryType = bytes[3];
                } break;
                // No ACK Recovery Group Parameter
                case 25: {
                    decoded.parameterType = "No ACK Recovery Group";
                    decoded.data.noAckRecoveryMode = bytes[1];
                    decoded.data.adrDisabled = bytes[2];
                    decoded.data.delay = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.retryCount = bytes[5];
                    decoded.data.Retrydelay = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Heartbeat Group Parameter
                case 26: {
                    decoded.parameterType = "Heartbeat Group";
                    decoded.data.heartbeatId = bytes[1];
                    decoded.data.ack = bytes[2];
                    decoded.data.ackRetryCount = bytes[3];
                } break;
                // External Leak ACK Parameter
                case 27: {
                    decoded.parameterType = "External Leak ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Pin Leak ACK Parameter
                case 28: {
                    decoded.parameterType = "Pin Leak ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Tamper Detection ACK Parameter
                case 29: {
                    decoded.parameterType = "Tamper Detection ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Button Press ACK Parameter
                case 30: {
                    decoded.parameterType = "Button Press ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Temperature High ACK Parameter
                case 31: {
                    decoded.parameterType = "Temperature High ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Temperature Low ACK Parameter
                case 32: {
                    decoded.parameterType = "Temperature Low ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Humidity High ACK Parameter
                case 33: {
                    decoded.parameterType = "Humidity High ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Humidity Low ACK Parameter
                case 34: {
                    decoded.parameterType = "Humidity Low ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Pulse Count ACK Parameter
                case 35: {
                    decoded.parameterType = "Pulse Count ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Pulse Count Long Event ACK Parameter
                case 36: {
                    decoded.parameterType = "Pulse Count Long Event ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Parameter Uplink ACK Parameter
                case 37: {
                    decoded.parameterType = "Parameter Uplink ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                // Pin Leak Detection Parameter
                case 38: {
                    decoded.parameterType = "Pin Leak Detection";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = UInt16(bytes[2] << 8 | bytes[3]);
                    decoded.data.triggerDelay = bytes[4];
                } break;
                // Pin Leak Detection Buzzer Parameter
                case 39: {
                    decoded.parameterType = "Pin Leak Detection Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
  

                } break;
                // Pin Leak Detection All Parameter
                case 40: {
                    decoded.parameterType = "Pin Leak Detection All";
                    decoded.data.pinLeakDetectionEnabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = UInt16(bytes[2] << 8 | bytes[3]);
                    decoded.data.triggerDelay = bytes[4];
                    decoded.data.silenceCounter = UInt16(bytes[5] << 8 | bytes[6]);
                    decoded.data.beepCount = bytes[7];
                    decoded.data.beepDuration = UInt16(bytes[8] << 8 | bytes[9]);
  

                } break;
                // External Leak Detection Parameter
                case 41: {
                    decoded.parameterType = "External Leak Detection";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = UInt16(bytes[2] << 8 | bytes[3]);
                    decoded.data.triggerDelay = bytes[4];
                } break;
                // External Leak Detection Buzzer Parameter
                case 42: {
                    decoded.parameterType = "External Leak Detection Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // External Leak Detection All Parameter
                case 43: {
                    decoded.parameterType = "External Leak Detection All";
                    decoded.data.pinLeakDetectionEnabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = UInt16(bytes[2] << 8 | bytes[3]);
                    decoded.data.triggerDelay = bytes[4];
                    decoded.data.silenceCounter = UInt16(bytes[5] << 8 | bytes[6]);
                    decoded.data.beepCount = bytes[7];
                    decoded.data.beepDuration = UInt16(bytes[8] << 8 | bytes[9]);
                } break;
                // Tamper Detection Parameter
                case 44: {
                    decoded.parameterType = "Tamper Detection";
                    decoded.data.enabled = bytes[1];
                } break;
                // Tamper Detection Buzzer Parameter
                case 45: {
                    decoded.parameterType = "Tamper Detection Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Tamper Detection All Parameter
                case 46: {
                    decoded.parameterType = "Tamper Detection All";
                    decoded.data.enabled = bytes[1];
                    decoded.data.buzzerEnabled = bytes[2];
                    decoded.data.silenceEnabled = bytes[3];
                    decoded.data.silenceCounter = UInt16(bytes[4] << 8 | bytes[5]);
                    decoded.data.beepCount = bytes[6];
                    decoded.data.beepDuration = UInt16(bytes[7] << 8 | bytes[8]);
                } break;
                // Button Pressed Parameter
                case 47: {
                    decoded.parameterType = "Button Pressed";
                    decoded.data.enabled = bytes[1];
                } break;
                // Temperature High Parameter
                case 48: {
                    decoded.parameterType = "Temperature High";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                } break;
                // Temperature High Buzzer Parameter
                case 49: {
                    decoded.parameterType = "Temperature High Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Temperature High All Parameter
                case 50: {
                    decoded.parameterType = "Temperature High All";
                    decoded.data.enabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                    decoded.data.silenceCounter = bytes[7];
                    decoded.data.beepCount = bytes[8];
                    decoded.data.beepDuration = UInt16(bytes[9] << 8 | bytes[10]);
  

                } break;
                // Temperature Low Parameter
                case 51: {
                    decoded.parameterType = "Temperature Low";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                } break;
                // Temperature Low Buzzer Parameter
                case 52: {
                    decoded.parameterType = "Temperature Low Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Temperature Low All Parameter
                case 53: {
                    decoded.parameterType = "Temperature Low All";
                    decoded.data.enabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                    decoded.data.silenceCounter = bytes[7];
                    decoded.data.beepCount = bytes[8];
                    decoded.data.beepDuration = UInt16(bytes[9] << 8 | bytes[10]);
                } break;
                // Humidity High Parameter
                case 54: {
                    decoded.parameterType = "Humidity High";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                } break;
                // Humidity High Buzzer Parameter
                case 55: {
                    decoded.parameterType = "Humidity High Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Humidity High All Parameter
                case 56: {
                    decoded.parameterType = "Humidity High All";
                    decoded.data.enabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                    decoded.data.silenceCounter = bytes[7];
                    decoded.data.beepCount = bytes[8];
                    decoded.data.beepDuration = UInt16(bytes[9] << 8 | bytes[10]);
                } break;
                // Humidity Low Parameter
                case 57: {
                    decoded.parameterType = "Humidity Low";
                    decoded.data.enabled = bytes[1];
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                } break;
                // Humidity Low Buzzer Parameter
                case 58: {
                    decoded.parameterType = "Humidity Low Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                // Humidity Low All Parameter
                case 59: {
                    decoded.parameterType = "Humidity Low All";
                    decoded.data.enabled = ((bytes[1] & 0b00000001) > 0);
                    decoded.data.buzzerEnabled = ((bytes[1] & 0b00000010) > 0);
                    decoded.data.silenceEnabled = ((bytes[1] & 0b00000100) > 0);
                    decoded.data.triggerValue = (Int16(bytes[2] << 8 | bytes[3])) / 100;
                    decoded.data.guardbandValue = (Int16(bytes[4] << 8 | bytes[5])) / 100;
                    decoded.data.delayCount = bytes[6];
                    decoded.data.silenceCounter = bytes[7];
                    decoded.data.beepCount = bytes[8];
                    decoded.data.beepDuration = UInt16(bytes[9] << 8 | bytes[10]);
                } break;
                // Temperature Offset Parameter
                case 60: {
                    decoded.parameterType = "Temperature Offset";
                    decoded.data.offset = (Int16(bytes[1] << 8 | bytes[2])) / 100;
                } break;
                // Humidity Offset Parameter
                case 61: {
                    decoded.parameterType = "Humidity Offset";
                    decoded.data.offset = (Int16(bytes[1] << 8 | bytes[2])) / 100;
                } break;
                // Sense Interval Parameter
                case 62: {
                    decoded.parameterType = "Sense Interval";
                    decoded.data.interval = UInt16(bytes[1] << 8 | bytes[2]);
                } break;
                // Pulse Counting Parameter
                case 63: {
                    decoded.parameterType = "Pulse Counting";
                    decoded.data.enabled = bytes[1];
                    decoded.data.deviceTypeId = bytes[2];
                    decoded.data.reportingInterval = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.eventDelayValue = UInt16(bytes[5] << 8 | bytes[6]);
  

                } break;
                // Pulse Counting Long Event Parameter
                case 64: {
                    decoded.parameterType = "Pulse Counting Long Event";
                    decoded.data.enabled = bytes[1];
                    decoded.data.eventLongDelayValue = UInt16(bytes[2] << 8 | bytes[3]);
                } break;
                case 65: {
                    decoded.parameterType = "Jack Detected Event ACK";
                    decoded.data.ack = bytes[1];
                    decoded.data.ackRetryCount = bytes[2];
                } break;
                case 66: {
                    decoded.parameterType = "Jack Detected Event";
                    decoded.data.enabled = bytes[1];
                    decoded.data.delayCount = bytes[2];
                } break;
                case 67: {
                    decoded.parameterType = "Jack Detected Buzzer";
                    decoded.data.enabled = bytes[1];
                    decoded.data.silenceEnabled = bytes[2];
                    decoded.data.silenceCounter = UInt16(bytes[3] << 8 | bytes[4]);
                    decoded.data.beepCount = bytes[5];
                    decoded.data.beepDuration = UInt16(bytes[6] << 8 | bytes[7]);
                } break;
                case 68: {
                    decoded.parameterType = "Jack Detected All";
                    decoded.data.enabled = bytes[1];
                    decoded.data.delayCount = bytes[2];
                    decoded.data.buzzerEnabled = bytes[3];
                    decoded.data.silenceEnabled = bytes[4];
                    decoded.data.silenceCounter = UInt16(bytes[5] << 8 | bytes[6]);
                    decoded.data.beepCount = bytes[7];
                    decoded.data.beepDuration = UInt16(bytes[8] << 8 | bytes[9]);
                } break;
                case 69: {
                    decoded.parameterType = "Buzzer Disabled";
                    decoded.data.disabled = bytes[1];
                } break;
            }
  

  

        } break;
        default:
            decoded.unknown = "Unknown data format";
            break;
    }
    return decoded;
}
  

// For TTN
function Decoder(bytes, fPort) {
    return DoDecode(fPort, bytes);
}
  

// For Chirpstack
function Decode(fPort, bytes, variables) {
    return DoDecode(fPort, bytes);
}
  

// Chirpstack v3 to v4 compatibility wrapper
function decodeUplink(input) {
    return {
        data: Decode(input.fPort, input.bytes, input.variables)
    };
}
  

var UInt4 = function (value) {
    return (value & 0xF);
};
  

var Int4 = function (value) {
    var ref = UInt4(value);
    return (ref > 0x7) ? ref - 0x10 : ref;
};
  

var UInt8 = function (value) {
    return (value & 0xFF);
};
  

var Int8 = function (value) {
    var ref = UInt8(value);
    return (ref > 0x7F) ? ref - 0x100 : ref;
};
  

var UInt16 = function (value) {
    return (value & 0xFFFF);
};
  

var Int16 = function (value) {
    var ref = UInt16(value);
    return (ref > 0x7FFF) ? ref - 0x10000 : ref;
};
```
  

## IQSV
  

IQSV devices are installed directly in the pipe and measure the water flow, temperature, pressure and contain a valve that can open and close the flow.
  

**There are 2 types of IQSV**
  

* IQSV V1 and V2 (older ones - until Sep/2025):
  

**Attributes:**
BatteryAlarm: bool  
BurstAlarm: bool (when pipe is damaged)  
CumulativeConsumption: decimal (water flow since the moment the device is turned ON) -> * 264.172052 = value in gallons
EEPROMERROR: bool (no idea what is it)  
EmptyPipeAlarm: bool (when no flow)  
InstantConsumption: decimal (water flow in this moment PER HOUR) -> * 264.172052 = value in gallons
LeakageAlarm: bool (when there is a leak in the device/pipe)
LowBatteryAlarm: bool (when the battery is low)
OverRangeAlarm: bool (no idea what is it)
OverTemperatureAlarm: bool (temperature defined in the firmware)
PreviousDayConsumption: decimal (water flow yesterday)  -> * 264.172052 = value in gallons
ReverseCumulativeConsumption: decimal (if there is a flow in reverse) ->  * 264.172052 = value in gallons
ReverseFlowAlarm: bool (when reverse flow)
ValveStatus: open/close
WaterPressure: decimal (in MPA) -> * 145.038 to convert to PSI
WaterTemperature: decimal (in Celsius) -> Need to convert to °F
  

Payload:
  

Bytes:
FEFE68201335465201740081259090002B612461002B812155002B000000003500000000652300810248091122092520000013161F…
  

Criptographic:
/v5oIBM1RlIBdACBJZCQACthJGEAK4EhVQArAAAAADUAAAAAZSMAgQJICREiCSUgAAATFh8=
  

Decoded:
```json
{
    "BatteryAlarm": false,
    "BurstAlarm": false,
    "CumulativeConsumption": "612.461",
    "EEPROMERROR": false,
    "EmptyPipeAlarm": false,
    "InstantConsumption": "0.0000",
    "LeakageAlarm": false,
    "LowBatteryAlarm": false,
    "OverRangeAlarm": false,
    "OverTemperatureAlarm": false,
    "PreviousDayConsumption": "552.181",
    "ReverseCumulativeConsumption": "0.000",
    "ReverseFlowAlarm": false,
    "ValveStatus": "open",
    "WaterPressure": "0.281",
    "WaterTemperature": "23.65"
}
```
  

Payload Formatter (TTN):
```js
function decodeUplink(input)
{
	var bytes = input.bytes;
	
    var bitst1, bitst1L, bitt, bittL, Bat, Battery, Code, Len, ID, Count, Unit, decimals, raw, output, Data1, Data2, Data3, Data4, Data5, Data6, Data7, Status, Err, Concat, Concat1, Concat2, Concat3, Concat4, Concat5, Concat6;
    var st1error5, st1error4, st1error3, st1error2, error7, error6, error5, error4, error3, error2, error1, error0;
    var Downlink;
    var interval, inte_unit;
    var deff = "null";
  

    //METER SN DECODE
    if (bytes[0] == "131")
    {
        //control code
        if (bytes[0] == "131") { Code = bytes[0].toString(16); }
  

        //data length
        if (bytes[1] == "10") { Len = "Length of the Data Frame : " + bytes[1]; }
  

        //data identification for meter SN uplink
        if (bytes[2] == "129" & bytes[3] == "10") { ID = "Read Meter Serial Number"; }
  

        //uplink data counter
        Count = "Uplink Data Counter : " + bytes[4];
  

        //meter SN indicator
        Data1 = bytes[5];
        Data2 = bytes[6];
        Data3 = bytes[7];
        Data4 = bytes[8];
        Data5 = bytes[9];
        Data6 = bytes[10];
        Data7 = bytes[11];
        if (Data1 < 10) { Data1 = '0' + Data1.toString(16); } else { Data1 = Data1.toString(16); }
        if (Data2 < 10) { Data2 = '0' + Data2.toString(16); } else { Data2 = Data2.toString(16); }
        if (Data3 < 10) { Data3 = '0' + Data3.toString(16); } else { Data3 = Data3.toString(16); }
        if (Data4 < 10) { Data4 = '0' + Data4.toString(16); } else { Data4 = Data4.toString(16); }
        if (Data5 < 10) { Data5 = '0' + Data5.toString(16); } else { Data5 = Data5.toString(16); }
        if (Data6 < 10) { Data6 = '0' + Data6.toString(16); } else { Data6 = Data6.toString(16); }
        if (Data7 < 10) { Data7 = '0' + Data7.toString(16); } else { Data7 = Data7.toString(16); }
        Concat1 = Data7.concat(Data6);
        Concat2 = Data5.concat(Data4);
        Concat3 = Data3.concat(Data2);
        Concat4 = Concat1.concat(Concat2);
        Concat5 = Concat3.concat(Data1);
        Concat = Concat4.concat(Concat5);
  

        return {
        Command_Code: Code,
                Data_Length: Len,
                Data_Type: ID,
                Count: Count,
                Meter_Addr: Concat
        };
    }
  

    //METER TOTALIZER DECODE
    else if (bytes[0] == 0x81)
    {
        console.log(bytes[0])
        //control code
        if (bytes[0] == 0x81) { Code = bytes[0].toString(16); }
  

        //data length(not include battery indicator byte)
        if (bytes[1] == 0x0A) { Len = "Length of the Data Frame : " + bytes[1]; }
  

        //data identification for totalizer uplink
        if (bytes[2] == 0x144 & bytes[3] == 0x31) { ID = "Read Meter Cumulative Value"; }
  

        //uplink data counter
        Count = "Uplink Data Counter : " + bytes[4];
  

        //decimals
        if (bytes[5] == 0x2B) { Unit = "Cubic Meter"; decimals = 1000; }
        else if (bytes[5] == 0x2C) { Unit = "Cubic Meter"; decimals = 100; }
  

        //totalizer
        a = bytes[6];
        b = bytes[7];
        c = bytes[8];
        d = bytes[9];
        if (a < 0x10) { a = '0' + a.toString(16); } else { a = a.toString(16); }
        if (b < 0x10) { b = '0' + b.toString(16); } else { b = b.toString(16); }
        if (c < 0x10) { c = '0' + c.toString(16); } else { c = c.toString(16); }
        if (d < 0x10) { d = '0' + d.toString(16); } else { d = d.toString(16); }
        concat1 = d.concat(c);
        concat2 = b.concat(a);
        raw = concat1.concat(concat2);
        output = raw / decimals;
  

  

        //valve indicator
        if (bytes[10] == 0x01) { Status = "Valve Control : Available"; } else { Status = "Valve Control : Not Available"; }
  

        //error bits ST1
        bitst1 = bytes[10];
        bitst1 = bitst1.toString(2);
        bitst1 = ('000000000' + bitst1).slice(-8);
        if (bitst1[5] == "1") { st1error5 = "Leakage Alert, "; } else { st1error5 = ""; }
        if (bitst1[4] == "1") { st1error4 = "Burst Alert, "; } else { st1error4 = ""; }
        if (bitst1[3] == "1") { st1error3 = "Tamper Alert, "; } else { st1error3 = ""; }
        if (bitst1[2] == "1") { st1error2 = "Freezing Alert, "; } else { st1error2 = ""; }
        bitst1L = st1error5 + st1error4 + st1error3 + st1error2;
        bitst1L = bitst1L.substr(0, bitst1L.length - 2);
        if (bitst1L.length == 0) { bitst1L = "No st1error"; }
        //if (bytes[11] == "8") { Err = "Transducer Outlet Error"; }
        //if (bytes[11] == "7") { Err = "Transducer Inlet Error"; }
        //if (bytes[11] == "6") { Err = "EEPROM Error"; }
        //if (bytes[11] == "5") { Err = "Temperature Alert"; }
        //if (bytes[11] == "4") { Err = "Over Range/Flow Alert"; }
        //if (bytes[11] == "3") { Err = "Reverse Flow Alert"; }
        //if (bytes[11] == "2") { Err = "Empty Pipe Alert"; }
        //if (bytes[11] == "1") { Err = "Battery Alert"; }
        //if (bytes[11] == "0") { Err = "No Error"; }
  

  

        //error bits ST2
        bitt = bytes[11];
        bitt = bitt.toString(2);
        bitt = ('000000000' + bitt).slice(-8);
        if (bitt[7] == "1") { error7 = "Battery Alert, "; } else { error7 = ""; }
        if (bitt[6] == "1") { error6 = "Empty Pipe Alert, "; } else { error6 = ""; }
        if (bitt[5] == "1") { error5 = "Reverse Flow Alert, "; } else { error5 = ""; }
        if (bitt[4] == "1") { error4 = "Over Range/Flow Alert, "; } else { error4 = ""; }
        if (bitt[3] == "1") { error3 = "Temperature Alert, "; } else { error3 = ""; }
        if (bitt[2] == "1") { error2 = "EEPROM Error, "; } else { error2 = ""; }
        if (bitt[1] == "1") { error1 = "Transducer Inlet Error, "; } else { error1 = ""; }
        if (bitt[0] == "1") { error0 = "Transducer Outlet Error, "; } else { error0 = ""; }
        bittL = error7 + error6 + error5 + error4 + error3 + error2 + error1 + error0;
        bittL = bittL.substr(0, bittL.length - 2);
        if (bittL.length == 0) { bittL = "No Error"; }
        //if (bytes[11] == "8") { Err = "Transducer Outlet Error"; }
        //if (bytes[11] == "7") { Err = "Transducer Inlet Error"; }
        //if (bytes[11] == "6") { Err = "EEPROM Error"; }
        //if (bytes[11] == "5") { Err = "Temperature Alert"; }
        //if (bytes[11] == "4") { Err = "Over Range/Flow Alert"; }
        //if (bytes[11] == "3") { Err = "Reverse Flow Alert"; }
        //if (bytes[11] == "2") { Err = "Empty Pipe Alert"; }
        //if (bytes[11] == "1") { Err = "Battery Alert"; }
        //if (bytes[11] == "0") { Err = "No Error"; }
  

        //battery indicator
        Bat = (bytes[12] - 1) / 253;
        Battery = (Bat * 100).toFixed(2);
        return {
        Command_Code: Code,
                Data_Length: Len,
                Data_Type: ID,
                Count: Count,
                Unit: Unit,
                Water_Flow_in_Cubic_Meter: output,
                Valve_Status: Status,
                Error_Status_ST2: bittL,
                Error_Status_ST1: bitst1L,
                Battery: Battery,
                Raw_Data: raw
                };
    }
  

    //SENDING INTERVAL DECODE
    else if (bytes[0] == "34")
    {
        //control code
        if (bytes[0] == "34") { Code = bytes[0].toString(16); }
        ID = "Modify Meter Uplink Interval";
  

  

        //data length(not include battery indicator byte)
        if (bytes[1] == "10") { Len = "Length of the Data Frame : " + bytes[1]; }
  

        //interval
        a = bytes[5].toString(16);
        b = bytes[6].toString(16);
        concat2 = b.concat(a);
        interval = parseInt(concat2, 16);
        inte_unit = "minutes";
  

        return {
        Command_Code: Code,
                Data_Length: Len,
                Data_Type: ID,
                Uplink_Interval: interval,
                Interval_Unit: inte_unit
                };
    }
  

    else if(bytes[0] == "81"){
  

        if (bytes.substring(10, 12) == "2B") 
        { 
            Unit = "Cubic Meter"; 
            decimals = 1000; 
        }
        var unit = Unit;
  

        a = bytes.substring(12, 14);
        b = bytes.substring(14, 16);
        c = bytes.substring(16, 18);
        d = bytes.substring(18, 20);
  

        a = a.toString(10);
        b = b.toString(10);
        c = c.toString(10);
        d = d.toString(10);
  

        concat1 = d.concat(c);
        concat2 = b.concat(a);
        raw = concat1.concat(concat2);
        var waterFlowInCubicMeter = raw / decimals;
  

        console.log(bytes.substring(24, 26).toString(10))
  

        Bat = (bytes.substring(24, 26).toString(10) - 1) / 253;
        Battery = (Bat * 100).toFixed(2);
        var battery = Battery; 
  

        return {
            data: {
                Unit: unit,
                WaterFlowInCubicMeter: waterFlowInCubicMeter,
                Battery: battery
            },
            };
    }
  

    else if(bytes[0].toString(16) == "fe"){
        
        //Cumulative Consumption
        var unitCumulativeConsuption = bytes[16];
        var base16UCC = unitCumulativeConsuption.toString(16)
        var doubleUCC = 0.001
        var decimalsToFixed = 3;
        if(base16UCC == "2b"){
            doubleUCC = 0.001
            decimalsToFixed = 3
        }
        else if(base16UCC == "2c"){
            doubleUCC = 0.01
            decimalsToFixed = 2
        }
        else if(base16UCC == "2e"){
            doubleUCC = 1
            decimalsToFixed = 0
        }
        else if(base16UCC == "35"){
            doubleUCC = 0.0001
            decimalsToFixed = 4
        }
  

        var cumulativeConsuption1 = bytes[20];
        var cumulativeConsuption2 = bytes[19];
        var cumulativeConsuption3 = bytes[18];
        var cumulativeConsuption4 = bytes[17];
  

        var base16CC1 = cumulativeConsuption1.toString(16).padStart(2, '0');
        var base16CC2 = cumulativeConsuption2.toString(16).padStart(2, '0');
        var base16CC3 = cumulativeConsuption3.toString(16).padStart(2, '0');
        var base16CC4 = cumulativeConsuption4.toString(16).padStart(2, '0');
  

        var base16CC = base16CC1 + base16CC2 + base16CC3 + base16CC4
        var floatCC = parseFloat(base16CC)
        var realCumulativeConsumption = (floatCC * doubleUCC).toFixed(decimalsToFixed)
  

        //Previous day Consumption
        var unitPreviousDayConsuption = bytes[21];
        var base16UPDC = unitPreviousDayConsuption.toString(16)
        var doubleUDPC = 0.001
        if(base16UPDC == "2b"){
            doubleUDPC = 0.001
            decimalsToFixed = 3
        }
        else if(base16UPDC == "2c"){
            doubleUDPC = 0.01
            decimalsToFixed = 2
        }
        else if(base16UPDC == "2e"){
            doubleUDPC = 1
            decimalsToFixed = 0
        }
        else if(base16UPDC == "35"){
            doubleUDPC = 0.0001
            decimalsToFixed = 4
        }
  

  

        var prevDayConsuption1 = bytes[25];
        var prevDayConsuption2 = bytes[24];
        var prevDayConsuption3 = bytes[23];
        var prevDayConsuption4 = bytes[22];
  

        var base16PDC1 = prevDayConsuption1.toString(16).padStart(2, '0');
        var base16PDC2 = prevDayConsuption2.toString(16).padStart(2, '0');
        var base16PDC3 = prevDayConsuption3.toString(16).padStart(2, '0');
        var base16PDC4 = prevDayConsuption4.toString(16).padStart(2, '0');
  

        var base16PDC = base16PDC1 + base16PDC2 + base16PDC3 + base16PDC4
        var floatPDC = parseFloat(base16PDC)
        var realPrevDayConsumption = (floatPDC * doubleUDPC).toFixed(decimalsToFixed)
  

        //Reverse Cumulative Consumption
        var unitReverseCumulativeConsumption = bytes[26];
        var base16URCC = unitReverseCumulativeConsumption.toString(16)
        var doubleURCC = 0.001
        if(base16URCC == "2b"){
            doubleURCC = 0.001
            decimalsToFixed = 3
        }
        else if(base16URCC == "2c"){
            doubleURCC = 0.01
            decimalsToFixed = 2
        }
        else if(base16URCC == "2e"){
            doubleURCC = 1
            decimalsToFixed = 0
        }
        else if(base16URCC == "35"){
            doubleURCC = 0.0001
            decimalsToFixed = 4
        }
  

        var reverseCumulativeConsumption1 = bytes[30];
        var reverseCumulativeConsumption2 = bytes[29];
        var reverseCumulativeConsumption3 = bytes[28];
        var reverseCumulativeConsumption4 = bytes[27];
  

        var base16RCC1 = reverseCumulativeConsumption1.toString(16).padStart(2, '0');
        var base16RCC2 = reverseCumulativeConsumption2.toString(16).padStart(2, '0');
        var base16RCC3 = reverseCumulativeConsumption3.toString(16).padStart(2, '0');
        var base16RCC4 = reverseCumulativeConsumption4.toString(16).padStart(2, '0');
  

        var base16RCC = base16RCC1 + base16RCC2 + base16RCC3 + base16RCC4
        var floatRCC = parseFloat(base16RCC)
        var realReverseCumulativeConsumption = (floatRCC * doubleURCC).toFixed(decimalsToFixed)
  

        //Instant Consumption
        var unitInstantConsumption = bytes[31];
        var base16UIC = unitInstantConsumption.toString(16)
        var doubleUIC = 0.001
        if(base16UIC == "2b"){
            doubleUIC = 0.001
            decimalsToFixed = 3
        }
        else if(base16UIC == "2c"){
            doubleUIC = 0.01
            decimalsToFixed = 2
        }
        else if(base16UIC == "2e"){
            doubleUIC = 1
            decimalsToFixed = 0
        }
        else if(base16UIC == "35"){
            doubleUIC = 0.0001
            decimalsToFixed = 4
        }
  

        var instanceConsumption1 = bytes[35];
        var instanceConsumption2 = bytes[34];
        var instanceConsumption3 = bytes[33];
        var instanceConsumption4 = bytes[32];
  

        var base16IC1 = instanceConsumption1.toString(16).padStart(2, '0');
        var base16IC2 = instanceConsumption2.toString(16).padStart(2, '0');
        var base16IC3 = instanceConsumption3.toString(16).padStart(2, '0');
        var base16IC4 = instanceConsumption4.toString(16).padStart(2, '0');
  

        var base16IC = base16IC1 + base16IC2 + base16IC3 + base16IC4
        var floatIC = parseFloat(base16IC)
        var realInstantConsumption = (floatIC * doubleUIC).toFixed(decimalsToFixed)
  

        //Water Temperature
        var waterTemperature1 = bytes[38];
        var waterTemperature2 = bytes[37];
        var waterTemperature3 = bytes[36];
  

        var base16WT1 = waterTemperature1.toString(16).padStart(2, '0');
        var base16WT2 = waterTemperature2.toString(16).padStart(2, '0');
        var base16WT3 = waterTemperature3.toString(16).padStart(2, '0');
  

        var base16WT = base16WT1 + base16WT2 + base16WT3
        var floatWT = parseFloat(base16WT)
        var realWaterTemperature = (floatWT * 0.01).toFixed(2)
  

        //Water Pressure
        var waterPressure1 = bytes[40];
        var waterPressure2 = bytes[39];
  

        var base16WP1 = waterPressure1.toString(16).padStart(2, '0');
        var base16WP2 = waterPressure2.toString(16).padStart(2, '0');
  

        var base16WP = base16WP1 + base16WP2
        var floatWP = parseFloat(base16WP)
        var realWaterPressure = (floatWP * 0.001).toFixed(3)
  

  

        var ST1 = bytes[48];
        var ST1Binary = ST1.toString(2).padStart(8, '0');
  

        valveStatusBinary = ST1Binary.substring(6)
        var valveStatus = "open"
        if(valveStatusBinary == "00"){
            valveStatus = "open"
        } else if(valveStatusBinary == "01"){
            valveStatus = "close"
        } else if(valveStatusBinary == "11"){
            valveStatus = "abnormal"
        }
  

        var batteryAlarmBinary = ST1Binary.substring(5, 6)
        var batteryAlarm = batteryAlarmBinary == "0" ? false : true
  

        var ST2 = bytes[49];
        var ST2Binary = ST2.toString(2).padStart(8, '0');
  

        var lowBatteryAlarmBinary = ST2Binary.substring(7)
        var lowBatteryAlarm = lowBatteryAlarmBinary == "0" ? false : true
  

        var emptyPipeAlarmBinary = ST2Binary.substring(6,7)
        var emptyPipeAlarm = emptyPipeAlarmBinary == "0" ? false : true
  

        var reverseFlowAlarmBinary = ST2Binary.substring(5,6)
        var reverseFlowAlarm = reverseFlowAlarmBinary == "0" ? false : true
  

        var overRangeAlarmBinary = ST2Binary.substring(4,5)
        var overRangeAlarm = overRangeAlarmBinary == "0" ? false : true
  

        var overTemperatureAlarmBinary = ST2Binary.substring(3,4)
        var overTemperatureAlarm = overTemperatureAlarmBinary == "0" ? false : true
  

        var EEPROMErrorBinary = ST2Binary.substring(2,3)
        var EEPROMError = EEPROMErrorBinary == "0" ? false : true
  

        var leakageAlarmBinary = ST2Binary.substring(1,2)
        var leakageAlarm = leakageAlarmBinary == "0" ? false : true
  

        var burstAlarmBinary = ST2Binary.substring(0,1)
        var burstAlarm = burstAlarmBinary == "0" ? false : true
  

        return {
            data: {
                CumulativeConsumption: realCumulativeConsumption,
                PreviousDayConsumption: realPrevDayConsumption,
                ReverseCumulativeConsumption: realReverseCumulativeConsumption,
                InstantConsumption: realInstantConsumption,
                WaterTemperature: realWaterTemperature,
                WaterPressure: realWaterPressure,
                ValveStatus: valveStatus,
                BatteryAlarm: batteryAlarm,
                LowBatteryAlarm: lowBatteryAlarm,
                EmptyPipeAlarm: emptyPipeAlarm,
                ReverseFlowAlarm: reverseFlowAlarm,
                OverRangeAlarm: overRangeAlarm,
                OverTemperatureAlarm: overTemperatureAlarm,
                EEPROMERROR: EEPROMError,
                LeakageAlarm: leakageAlarm,
                BurstAlarm: burstAlarm
            },
        };
    }
}
```
  

* IQSV DC and BP (new ones - from Sep/2025):
  

**Attributes:**
BatteryAlarm: bool  
CumulativeConsumption: decimal (water flow since the moment the device is turned ON) -> * 264.172052 = value in gallons
EEPROMERROR: bool (no idea what is it)  
EmptyPipeAlarm: bool (when no flow)  
InstantConsumption: decimal (water flow in this moment PER HOUR) -> * 264.172052 = value in gallons
LeakageAlarm: bool (when there is a leak in the device/pipe)
LowBatteryAlarm: bool (when the battery is low)
OverRangeAlarm: bool (no idea what is it)
OverTemperatureAlarm: bool (temperature defined in the firmware)
PreviousDayConsumption: decimal (water flow yesterday)  -> * 264.172052 = value in gallons
ReverseCumulativeConsumption: decimal (if there is a flow in reverse) ->  * 264.172052 = value in gallons
ReverseFlowAlarm: bool (when reverse flow)
ValveStatus: open/close/ **half-open (25%, 50% or 75%)**
WaterPressure: decimal (in MPA) -> * 145.038 to convert to PSI
WaterTemperature: decimal (in Celsius) -> Need to convert to °F
**PowerSupply: Battery/External Power Supply**
  

Payload:
  

Bytes:
FEFE68100000000000000000000000002324000000000200C11615
  

Criptographic:
/v5oEAAAAAAAAAAAAAAAACMkAAAAAAIAwRYV
  

Decoded:
```json
{
    "BatteryAlarm": false,
    "BurstAlarm": false,
    "CumulativeConsumption": "612.461",
    "EEPROMERROR": false,
    "EmptyPipeAlarm": false,
    "InstantConsumption": "0.0000",
    "LeakageAlarm": false,
    "LowBatteryAlarm": false,
    "OverRangeAlarm": false,
    "OverTemperatureAlarm": false,
    "PreviousDayConsumption": "552.181",
    "ReverseCumulativeConsumption": "0.000",
    "ReverseFlowAlarm": false,
    "ValveStatus": "open",
    "WaterPressure": "0.281",
    "WaterTemperature": "23.65"
}
```
  

Payload Formatter (TTN):
```js
function decodeUplink(input) {
  const bytes = input.bytes;
  const toLEHex = (arr) =>
    arr.map(b => b.toString(16).padStart(2, "0")).reverse().join("").toUpperCase();
  

  const applyScale = (digitStr, scale) => {
    const s = digitStr.replace(/[^0-9]/g, "");
    let core = s.replace(/^0+/, "");
    if (core.length === 0) core = "0";
  

    if (core.length <= scale) core = core.padStart(scale + 1, "0");
  

    const p = core.length - scale;
    const withPoint = core.slice(0, p) + "." + core.slice(p);
    return withPoint.startsWith(".") ? "0" + withPoint : withPoint;
  };
  

  let off = 4;
  

  const cumulativeHex = toLEHex(bytes.slice(off, off + 4)); off += 4;
  const cumulativeStr = applyScale(cumulativeHex, 3);
  

  const reverseHex = toLEHex(bytes.slice(off, off + 4)); off += 4;
  const reverseStr = applyScale(reverseHex, 3);
  

  const flowHex = toLEHex(bytes.slice(off, off + 4)); off += 4;
  const flowStr = applyScale(flowHex, 4);
  

  const tempHex = toLEHex(bytes.slice(off, off + 3)); off += 3;
  const tempStr = applyScale(tempHex, 2);
  

  const pressureHex = toLEHex(bytes.slice(off, off + 2)); off += 2;
  const pressureStr = applyScale(pressureHex, 3);
  

  const st1 = bytes[off++];
  const valveBits = st1 & 0b11;
  const ValveStatus = (valveBits === 0b00) ? "open"
                    : (valveBits === 0b01) ? "close"
                    : (valveBits === 0b10) ? "half-open"
                    : "abnormal";
  const BatteryAlarm = (st1 & 0b00000100) !== 0;
  

  const st2 = bytes[off++];
  const LowBatteryAlarm    = (st2 & 0b00000001) !== 0;
  const EmptyPipeAlarm     = (st2 & 0b00000010) !== 0;
  const ReverseFlowAlarm   = (st2 & 0b00000100) !== 0;
  const OverRangeAlarm     = (st2 & 0b00001000) !== 0;
  const OverTemperatureAlarm = (st2 & 0b00010000) !== 0;
  const EEPROMERROR        = (st2 & 0b00100000) !== 0;
  const LeakageAlarm       = (st2 & 0b01000000) !== 0;
  

  const powerSupplyByte = bytes[off++];
  const PowerSupply = (powerSupplyByte === 0x00) ? "Battery" : "Connected";
  

  

  return {
    data: {
      CumulativeConsumption: cumulativeStr,         
      ReverseCumulativeConsumption: reverseStr,
      InstantConsumption: flowStr,
      WaterTemperature: tempStr,
      WaterPressure: pressureStr,
      LowBatteryAlarm,
      EmptyPipeAlarm,
      ReverseFlowAlarm,
      OverRangeAlarm,
      OverTemperatureAlarm,
      EEPROMERROR,
      LeakageAlarm,
      ValveStatus,
      BatteryAlarm,
      PowerSupply
    }
  }
}
```
  

## IQWM
  

IQWM devices are installed directly in the pipe and measure the water flow. It sends one uplink every 6 hours and has the history of water consumption in the last 12 hours.  
If one uplink failes, the consumption can be recovered in the next uplink, since the next one will be sent after 6 hours and it contains the last 12 hours of consumption history 
  

  

**Attributes:**
batteryStatus: decimal
clockDate: 09/22 (date - example: Sep 22th)
clockTime: 11:16 (time - example 11h16m)
currentReading: decimal (water flow since the moment the device is turned ON) -> * 264.172052 = value in gallons
burstAlarm: bool (same as IQSV)
eepromError: bool (same as IQSV)
emptyPipeAlarm: bool (same as IQSV)
freezingAlarm: bool (when freezed)
leakageAlarm: bool (same as IQSV)
lowBatteryAlarm: bool (same as IQSV)
overRangeAlarm: bool  (same as IQSV)
overTemperatureAlarm: bool  (same as IQSV)
reverseFlowAlarm: bool  (same as IQSV)
tamperAlarm: bool (when moved) 
log-1: decimal (* 264.172052 = consumption gallons)
log-1-Time: "23:00 00:00" (initial and final time reading)
...
log-12: decimal,
log-12-Time: "10:00 11:00" (initial and final time reading)
  

Payload:
  

Bytes:
FEFEBA9003006D000F00080012000600000008001D00EF001B00040007000006092211167BFE
  

Criptographic:
/v66kAMAbQAPAAgAEgAGAAAACAAdAO8AGwAEAAcAAAYJIhEWe/4=
  

Decoded:
```json
{
    "batteryStatus": 100,
    "clockDate": "09/22",
    "clockTime": "11:16",
    "currentReading": 233.658,
    "meterStatus": {
      "flags": {
        "burstAlarm": false,
        "eepromError": false,
        "emptyPipeAlarm": true,
        "freezingAlarm": false,
        "leakageAlarm": false,
        "lowBatteryAlarm": false,
        "overRangeAlarm": false,
        "overTemperatureAlarm": false,
        "reserved1": false,
        "reserved2": false,
        "reserved3": false,
        "reserved4": false,
        "reserved5": false,
        "reserved6": false,
        "reverseFlowAlarm": true,
        "tamperAlarm": false
      },
      "raw": 1536
    },
    "parameters": [
      {
        "log-1": 0.109,
        "log-1-Time": "23:00 00:00"
      },
      {
        "log-2": 0.015,
        "log-2-Time": "00:00 01:00"
      },
      {
        "log-3": 0.008,
        "log-3-Time": "01:00 02:00"
      },
      {
        "log-4": 0.018,
        "log-4-Time": "02:00 03:00"
      },
      {
        "log-5": 0.006,
        "log-5-Time": "03:00 04:00"
      },
      {
        "log-6": 0,
        "log-6-Time": "04:00 05:00"
      },
      {
        "log-7": 0.008,
        "log-7-Time": "05:00 06:00"
      },
      {
        "log-8": 0.029,
        "log-8-Time": "06:00 07:00"
      },
      {
        "log-9": 0.239,
        "log-9-Time": "07:00 08:00"
      },
      {
        "log-10": 0.027,
        "log-10-Time": "08:00 09:00"
      },
      {
        "log-11": 0.004,
        "log-11-Time": "09:00 10:00"
      },
      {
        "log-12": 0.007,
        "log-12-Time": "10:00 11:00"
      }
    ]
}
```
  

Payload Formatter (TTN):
```js
function decodeUplink(input) {
  const bytes = input.bytes;
  

  // Fixed sizes
  const startBytesSize = 4; // 4 bytes
  const logEntriesSize = 12 * 2; // 12 logs, each 2 bytes
  const meterStatusSize = 2; // 2 bytes
  const timestampSize = 4; // 4 bytes (clock format)
  const batteryStatusSize = 1; // 1 byte (last byte)
  

  const fixedSize = startBytesSize + logEntriesSize + meterStatusSize + timestampSize + batteryStatusSize;
  

  // Validate byte array length
  if (bytes.length <= fixedSize) {
    return { data: {}, warnings: [], errors: ["Invalid byte array length. Expected fixed size."] };
  }
  

  

  let hexString = bytes.map((byte) => byte.toString(16).padStart(2, '0')).join('');
  hexString = hexString.substring(4); // Remove the first 2 bytes
  

  // Extract Current Reading (first 4 bytes) and convert to integer (Little Endian)
  const currentReadingHex = hexString.substring(0, 8);
  const currentReadingLittleEndian =
    currentReadingHex.substring(6, 8) +
    currentReadingHex.substring(4, 6) +
    currentReadingHex.substring(2, 4) +
    currentReadingHex.substring(0, 2);
  const currentReading = (parseInt(currentReadingLittleEndian, 16) / 1000).toFixed(3);
  

  // Extract 12 log entries (convert each to Little Endian signed 16-bit integer)
  const parameters = [];
  let index = 8; // Start after currentReading
  for (let i = 0; i < 12; i++) {
    const paramHex = hexString.substring(index, index + 4);
    const littleEndianHex = paramHex.substring(2, 4) + paramHex.substring(0, 2);
    const paramInt = toSignedInt16(parseInt(littleEndianHex, 16))/1000;
    parameters.push(paramInt);
    index += 4;
  }
  

  // Extract Meter Status (2 bytes) and parse flags
  const meterStatusHex = hexString.substring(index, index + 4);
  const littleEndianMeterStatus =
    meterStatusHex.substring(2, 4) + meterStatusHex.substring(0, 2);
  const meterStatus = parseInt(littleEndianMeterStatus, 16);
  

  const meterStatusFlags = {};
  const firstByte = parseInt(littleEndianMeterStatus.substring(0, 2), 16);
  meterStatusFlags.lowBatteryAlarm = Boolean(firstByte & 0b00000001);
  meterStatusFlags.emptyPipeAlarm = Boolean(firstByte & 0b00000010);
  meterStatusFlags.reverseFlowAlarm = Boolean(firstByte & 0b00000100);
  meterStatusFlags.overRangeAlarm = Boolean(firstByte & 0b00001000);
  meterStatusFlags.overTemperatureAlarm = Boolean(firstByte & 0b00010000);
  meterStatusFlags.eepromError = Boolean(firstByte & 0b00100000);
  meterStatusFlags.reserved1 = Boolean(firstByte & 0b01000000);
  meterStatusFlags.reserved2 = Boolean(firstByte & 0b10000000);
  

  const secondByte = parseInt(littleEndianMeterStatus.substring(2, 4), 16);
  meterStatusFlags.leakageAlarm = Boolean(secondByte & 0b00000100);
  meterStatusFlags.burstAlarm = Boolean(secondByte & 0b00001000);
  meterStatusFlags.tamperAlarm = Boolean(secondByte & 0b00010000);
  meterStatusFlags.freezingAlarm = Boolean(secondByte & 0b00100000);
  meterStatusFlags.reserved3 = Boolean(secondByte & 0b00000001);
  meterStatusFlags.reserved4 = Boolean(secondByte & 0b00000010);
  meterStatusFlags.reserved5 = Boolean(secondByte & 0b01000000);
  meterStatusFlags.reserved6 = Boolean(secondByte & 0b10000000);
  

  index += 4;
  

  // Extract Clock Time (4 bytes)
  const clockHex = hexString.substring(index, index + 8);
  
  const hours = parseInt(clockHex.substring(4, 6), 10); // First 2 hex digits represent hours
  const minutes = parseInt(clockHex.substring(6, 8), 10); // Last 2 hex digits represent minutes
  const clockTime = `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`;
  
  const day = parseInt(clockHex.substring(0, 2), 10); // Last 2 hex digits represent minutes
    
  const month = parseInt(clockHex.substring(2, 4), 10); // Last 2 hex digits represent minutes
  const clockDate = `${day.toString().padStart(2, '0')}/${month.toString().padStart(2, '0')}`;
  

  
  index += 8;
  
  var hrs = hours;
  
// Initialize the combined array
var combined_time_array = [];
  

// Generate the past 12 entries
for (let i = 0; i < 12; i++) {
  // Format the current hour and minute as HH:MM
  let currentTime = `${hrs.toString().padStart(2, '0')}:00`;
  

  // Calculate the previous hour
  let previousHrs = hrs - 1;
  if (previousHrs < 0) {
    previousHrs = 23; // Handle rollover to the previous day
  }
  

  // Format the previous hour and minute as HH:MM
  let previousTime = `${previousHrs.toString().padStart(2, '0')}:00`;
  

  // Combine the two times and add to the array
  combined_time_array.push(`${previousTime} ${currentTime}`);
  

  // Decrement the hour
  hrs -= 1;
  

  // Handle rollover to the previous day
  if (hrs < 0) {
    hrs = 23;
  }
}
  

  // Extract Battery Status (1 byte, last byte of the array) and divide by 2.54
  const batteryStatusRaw = parseInt(hexString.substring(hexString.length - 2), 16);
  const batteryStatus = (batteryStatusRaw / 2.54).toFixed(2);
  

  // Prepare the response
  return {
    data: {
      currentReading: parseFloat(currentReading), // Convert back to number for output
      parameters: parameters.map((param, i) => ({ [`log-${i + 1}`]: param , [`log-${i + 1}-Time`]: combined_time_array[11-i]})), // Signed 16-bit integers
      meterStatus: {
        raw: meterStatus,
        flags: meterStatusFlags
      },
      clockTime, // Time as HH:MM
      clockDate, // Month/Day 
      batteryStatus: parseFloat(batteryStatus), // Convert back to number for output
    },
    warnings: [],
    errors: []
  };
}
  

// Helper function to convert int16_t to signed number
function toSignedInt16(value) {
  value = value & 0xFFFF; // Ensure 16 bits
  if (value & 0x8000) { // Check MSB for negative
    return value - 0x10000; // Convert to signed
  }
  return value; // Positive values remain unchanged
}
```
  

  

## Lambdas for example
  

### Lambda Functions
  

The Lambda functions are the **core** of the system.  
They handle all calculations and business logic decisions.
  

# Lambda Function: `Process_iqsl`
  

## Purpose
  

Only for Water Leak Devices of type IQSL. Doesn't include Toilet Sensors
  

## Main Operations and Algorithms:
  

* Temperature: from Celcius to Fahrenheit
  

* Saves all connected gateways in a list and saves it in the database.
  

* Saves data in the iqwl_data table and updates the latest data in the devices_last_data table.
  

Note: Alerts are not sent by this Lambda. This Lambda sends data to another Lambda (SendAlertNotification) and that one will decide whether to send an alert.
  

---
  

## Input Source
  

Receives messages from the following SQS queue:  
**`iqsl_uplink_queue`**
  

## Output Destinations
  

- Sends data to Lambda: **`SendAlertNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `iqsl_data`
  - `devices_last_data`
  

---
  

## Input Format (JSON)
  

```json
{
    "device_id": "123",
    "application_id": "app1",
    "dev_eui": "1234",
    "read_time": "2025-07-02T19:16:38.982846975Z",
    "payload_type": "Heartbeat",
    "payload_type_id": 1,
    "battery_percentage": 38,
    "battery_voltage": 3.16,
    "external_leak_detection_enabled": true,
    "pin_leak_detection_enabled": true,
    "tamper_detection_enabled": true,
    "humidity": 54.2,
    "last_rssi": -42,
    "last_snr": 13,
    "external_leak_detected": false,
    "jack_detected": false,
    "magnet_detected": false,
    "pin_leak_detected": false,
    "power_detected": false,
    "state_as_uint8": 7,
    "temperature": 24.33,
    "frame_count": 2677,
    "metadata": [
        {
            "gateway_ids": {
                "gateway_id": "eui-58a0cbfffe8040c1",
                "eui": "58A0CBFFFE8040C1"
            },
            "time": "2025-07-02T19:16:38.982846975Z",
            "timestamp": 2143403483,
            "rssi": -63,
            "channel_rssi": -63,
            "snr": 9.25,
            "uplink_token": "CiIKIAoUZXVpLTU4YTBjYmZmZmU4MDQwYzESCFigy//+gEDBENv7hv4HGgwIl4uWwwYQyZ28jAEg+L625rA+",
            "received_at": "2025-07-02T19:16:39.241239981Z"
        },
        {
            "gateway_ids": {
                "gateway_id": "iqgw-tk-20",
                "eui": "647FDAFFFE01F36D"
            },
            "timestamp": 1245444315,
            "rssi": -78,
            "channel_rssi": -78,
            "snr": 9.5,
            "uplink_token": "ChgKFgoKaXFndy10ay0yMBIIZH/a//4B820Q2/Hv0QQaDAiXi5bDBhCLqciXASD4rpDSn+UU",
            "received_at": "2025-07-02T19:16:39.125916168Z"
        }
    ]
}
```
  

  

# Lambda Function: `Process_long_flush_iqsl`
  

## Purpose
  

Only for Toilet Sensor Devices from type IQSL filtered by the uplinks of type LONG_FLUSH (payload_type_id = 10).
  

## Main Operations and Algorithms:
  

### How Does Long Flush Work?
  

The device only sends the uplink **after 3 minutes (default)** of continuous flushing.  
In other words, when the uplink is received, the flushing has already been occurring for 3 minutes.  
Because of that, the Lambda must register that the flush **started 3 minutes earlier**.
  

---
  

### End of Flush
  

When a flush ends, a different type of uplink is sent:  
**Last Event Water Usage** (`payload_type_id = 11`).  
However, this uplink is processed by **another Lambda**:  
`Process_water_usage_iqsl`.
  

---
  

### Custom Long Flush Duration
  

The user can request the device to register a flush **only after 4 or more minutes**. In this case:
  

1. An external API (not the Lambda) stores on the device the setting that long flush should be recorded after 4+ minutes.
2. The Lambda checks the device settings in the database.
3. Based on the setting:
   - **If it's set to 3 minutes (default):**  
     The long flush is recorded normally.
   - **If it's set to more than 3 minutes (late long flush):**  
     - The long flush is saved to the database.  
     - The property `long_event_hide_on_dashboard` is set to `False`, so it **won't appear on the Dashboard**.  
     - A **schedule is created in EventBridge** to trigger the long flush later via the `LateLongFlush` Lambda.
  

---
  

### Notes
  

- There is **no information** about gallons used in the flush in this uplink.  
  This data will arrive in a separate uplink of type **Last Event Water Usage** (`payload_type_id = 11`).
  

- The device is renamed with a `-flow` suffix to distinguish Toilet Sensor data from Leak Sensor data,  
  since **IQSL includes both sensor types** in the same transmitter.
---
  

## Input Source
  

Receives messages from the following SQS queue:  
**`iqsl_long_flush_uplink_queue`**
  

## Output Destinations
  

- Sends data to Lambda: **`SendAlertToiletNotification`**, **`LateLongFlush`**
- Stores data in the following tables:
  - `devices_details`
  - `iqsl_alerts_data`
  - `devices_last_data`
  

---
  

## Input Format (JSON)
  

```json
{
    "device_id": "123",
    "application_id": "app2",
    "dev_eui": "1234",
    "read_time": "2025-07-02T19:11:57.283046007Z",
    "payload_type": "Alert",
    "payload_type_id": 10,
    "alert_type": "Pulse Counting (Long Event Alert)",
    "current_status": 1,
    "pulse_duration_trigger": 180000,
    "frame_count": 3642
}
```
  

  

# Lambda Function: `Process_water_usage_iqsl`
  

## Purpose
  

Only for Toilet Sensor Devices from type IQSL filtered by the uplinks of type WATER_USAGE (payload_type_id = 11).
  

## Main Operations and Algorithms:
  

### How Does Water Usage Work?
  

Gallons_spent = Pulse_total_pulses * 0.264172
  

Water usage can occur after a normal flush or after a Long Flush.
  

If it is after a normal flush, then it simply records in the database that a flush occurred without needing to calculate when the flush started. However, there can be two types:
  

- Above 1 gallon: **"normal flush"**
- Equal to or less than 1 gallon: **"escape"** (value too low to be a real flush)
  

If it is after a Long Flush, then:
  

- If the Long Flush has already been registered, then this Water Usage must be recorded at the moment the Long Flush started. The type of this flush must be **"long"**.
- If the Long Flush has not yet been registered on the dashboard [see late long flush], then the Long Flush should not appear on the dashboard. Instead, it should appear as a normal flush, without the need to calculate when the flush started.
  

---
  

## Input Source
  

Receives messages from the following SQS queue:  
**`iqsl_water_usage_uplink_queue`**
  

## Output Destinations
  

- Sends data to Lambda: **`SendAlertToiletNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `iqsl_alerts_data`
  - `devices_last_data`
  

---
  

## Input Format (JSON)
  

```json
{
    "device_id": "123",
    "application_id": "app1",
    "dev_eui": "1234",
    "read_time": "2025-07-02T20:13:31.984311103Z",
    "payload_type": "Alert",
    "payload_type_id": 11,
    "alert_type": "Pulse Counting Event (Last Event Usage)",
    "humidity": 54.33,
    "temperature": 23.64,
    "pulse_device_type_id": 1,
    "pulse_total_pulses": 2902,
    "pulse_total_liters": 2.694521819870009,
    "frame_count": 24465
}
```
  

  

  

# Lambda Function: `Process_toilet_heartbeat_iqsl`
  

## Purpose
  

Only for Toilet Sensor Devices from Type IQSL filtered by the uplinks of type HEARTBEAT (payload_type_id = 9).
  

## Main Operations and Algorithms:
  

### How Does the Heartbeat Work?
  

The heartbeat shows what has occurred since the last heartbeat:
  

- Number of flushes  
- Gallons spent  
  

---
  

**Note:**  
If a long flush is currently happening and has not yet finished, and a heartbeat is sent,  
then the gallons spent during this ongoing long flush are **pre-calculated**.
---
  

## Input Source
  

Receives messages from the following SQS queue:  
**`IQWLWaterLeakUplinkQueue`**
  

## Output Destinations
  

- Sends data to Lambda: **`iqsl_toilet_heartbeat_uplink_queue`**
- Stores data in the following tables:
  - `devices_details`
  - `iqsl_alerts_data`
  - `devices_last_data`
  

---
  

## Input Format (JSON)
  

```json
{
    "device_id": "123",
    "application_id": "app1",
    "dev_eui": "1234",
    "read_time": "2025-07-02T20:18:57.869425058Z",
    "payload_type": "Alert",
    "payload_type_id": 9,
    "alert_type": "Pulse Counting (Total Events)",
    "pulse_long_event_triggered": 0,
    "pulse_alert_interval": 500,
    "pulse_device_type_id": 1,
    "pulse_total_events": 11,
    "pulse_total_pulses": 11,
    "pulse_total_liters": 0.01021355617455896,
    "frame_count": 325
}
```
  

  

# Lambda Function: `Process_alerts_iqsl`
  

## Purpose
  

Only for Leak Sensor and Toilet Sensor Devices of type IQSL filtered by the uplinks of payload_type_id = 1,2,3,4,5,6,7,8 and 12.
  

## Main Operations and Algorithms:
  

### Processing Based on `payload_type_id`
  

Depending on the `payload_type_id`, the processing is different:
  

- **payload_type_id = 1** → Leak Detected External (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 2** → Leak Detected Local (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 3** → Tamper (LEAK SENSOR)  
  Saves the event. This uplink is sent when the device comes into contact with the magnet (not actual movement).
  

- **payload_type_id = 4** → Button Pressed (LEAK SENSOR)  
  Saves the event.
  

- **payload_type_id = 5** → High Temperature (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 6** → Low Temperature (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 7** → High Humidity (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 8** → Low Humidity (LEAK SENSOR)  
  Saves the event and calls the Lambda **SendAlertNotification** to send an alert.
  

- **payload_type_id = 12** → Headphone Jack Alert (LEAK SENSOR)  
  Saves the event.
---
  

## Input Source
  

Receives messages from the following SQS queue:  
**`iqsl_alerts_uplink_queue`**
  

## Output Destinations
  

- Sends data to Lambda: **`SendAlertNotification`**, **`SendAlertToiletNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `iqsl_data`
  - `iqsl_alerts_data`
  - `devices_last_data`
  

---
  

## Input Format (JSON) - example only for payload_type_id = 2
  

```json
{
    "device_id": "123",
    "application_id": "app4",
    "dev_eui": "1234",
    "read_time": "2025-07-02T18:46:19.129642963Z",
    "payload_type": "Alert",
    "payload_type_id": 2,
    "alert_type": "Local Leak Detection (Via Bottom Contact Pins)",
    "current_status": 0,
    "current_value": 3044,
    "humidity": 49.23,
    "temperature": 23.68,
    "delay_count": 1,
    "trigger_value": 5000,
    "frame_count": 2789
}
```
  

## Downlinks
  

Tip: use this site to facilitate the convert from base to hex [base64-to-hex](https://cryptii.com/pipes/base64-to-hex)
  

  

### Working on...
Downlinks
Alerts

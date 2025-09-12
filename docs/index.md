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

### From now on (September, 2025) the new devices should be onboarded on AWS IoT Core


---

### Device Types:

ArkIQ has different types of devices

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

//MerryIot Leak Detection sensor
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

### IQSL

This device has two parts: the leak sensor, that it's also a transmitter. If connected to a jack, it can extend the Leak detection probs, or can also receive information from the *Toilet Sensor*

*By default at arkIQ, all IQSL is connected to a Toilet Sensor*

*1- Heartbeat Packet (fPort = 0x01)*
Attributes:
Battery Voltage (V)
batteryPercentage (%)
Temperature (Celsius)
Humidity (%)
pinLeakDetectionEnabled: boolean (default: true)
externalLeakDetectionEnabled: boolean (default: false in order to work the Toilet Sensor)
tamperDetectionEnabled: boolean
pinLeakDetected: boolean (if it's occuring a leak right now)
externalLeakDetected: boolean (always false)
powerDetected: boolean (No idea what is this)
jackDetected: boolean (True, if connected to a Toilet Sensor)
magnetDetected: boolean (Tamper detection using a magnet)



Water Leak Detected: boolean
Tamper Detected: boolean
Button Pressed: boolean
Temperature (Celsius): decimal
Humidity (%): decimal

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

*2- Alert Packets (fPort = 0x02)*
Triggered when a specific event occurs.
The first byte (bytes[0]) defines the alert type:

Case 1: External Leak (via headphone jack) - not applied, because the Toilet Sensor will be connected to the Jack, not a Leak Sensor Extensor.
Attributes:
triggerValue: (No idea what is it)
currentValue: int (how moisture is it. Can define the level to consider it a leak detected)
temperature 
humidity 
delayCount (No idea what is it)
currentState: boolean (true = leak detected. false = leak cleared) 

(IMPORTANT)
Case 2: Local Leak (via bottom contact pins) *Only activate if the attribute pinLeakDetectionEnabled = true*
Attributes:
triggerValue: (No idea what is it)
currentValue: int (how moisture is it. Can define the level to consider it a leak detected)
temperature 
humidity 
delayCount (No idea what is it)
currentState: boolean (true = leak detected. false = leak cleared) 

Case 3: Tamper (magnet detected) - Activate when the magnet touch the device, simulating a tamper. *Only activate if the attribute tamperDetectionEnabled = true*

Case 4: Push button pressed - Activated when button is pressed

Case 5: Temperature High. Defined by downlink th hreshold
Case 6: Temperature Low.
Case 7: Humidity High.
Case 8: Humidity Low.
  From 5 to 8: Each includes current value, trigger threshold, guardband (safety margin), delay, and state.

(IMPORTANT)
Case 9: Total Events (cumulative counter): *Toilet Sensor Heartbeat*
Attributes:
longEventTriggered: boolean (is it occuring a Long Flush right now? True = yes. False = No)
alertInterval (no Idea what is it)
deviceTypeId (no Idea what is it)
totalEvents: int (How many flushes occured since the last heartbeat)
totalLiters: decimal (How many litters were spent since the last heartbeat) -> *Convert to Gallons (litters * 0.264172)*
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
Case 10: Long Event Alert (long-duration events): *Toilet Sensor Long Event detected*

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
Case 11: Last Event Usage (usage of the last event): *Toilet Sensor water usage when a flush (normal or long) ends*

Attriubtes:
totalLiters: decimal (how many litters was spent in the last flush)  -> *Convert to Gallons (litters * 0.264172)*
totalPulses: decimal (how many pulses was spent in the last flush)
humidity (%)
temperature (Celsius)

Note: this payload will be triggered only after a flush (normal or long). 
- If this flush was a normal flush, we don't know how much time this flush was running. However, we know by this payload when it finished.
- If this flush was a long flush, we know when it started, because the Long Flush payload (type: 10) was triggered before this payload of Last Event Usage (type: 11).
- One challenge is to assign the Last Event Usage payload (type: 11) with the Long Flush payload (type: 10), because the payloads will be triggered in different moments. Also, the Last Event Usage payload doesn't indicate if the last event usage was long or a normal flush.
- If the last flush spent less then 1 gallon. So this is consider a *Scape*, not a normal flush 

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

*3- System Packets (fPort = 0x03)*
Maintenance/system packets.

Case 0x01: Network Test Packet → reports SNR and RSSI.

Case 0x02: Joined Uplink Packet → confirms joining the network and sends configs:

firmwareVersion, heartbeatInterval, sensorCheckInterval.

Which alerts are enabled (bitmasks in bytes[7] and bytes[8]).

Used for network testing and initial configuration after join.

*4- Parameter Values (fPort = 0x04)*

Here we have many subtypes (dozens). These are configuration/parameterization packets.
The first byte (bytes[0]) indicates which parameter. Examples:
Identifiers
Case 0: devEui.
Case 1: appEui.
Case 2: appKey (disabled for security reasons).
Network and operation configs
Case 3–9: retry, confirm mode, join mode, ADR, device class, duty cycle, datarate.
Case 10–15: RX1/RX2 delays, datarate, TX power.
Case 16–19: region, channel mask, network mode, mode.
Operation groups
Case 21–26: heartbeat interval, join group, network test group, battery group, no-ack recovery, heartbeat group.
Event acknowledgements
Case 27–37: Individual acks for external leak, pin leak, tamper, button, temp high/low, humidity high/low, pulse count, etc.
Detailed sensor configs
Cases 38–46: Leak detection (pin/external) + buzzer + all-in-one parameters.
Cases 47–59: Button, high/low temperature, high/low humidity, all with buzzer/guardband/delay.
Case 60–61: temperature/humidity offsets.
Case 62: sensing interval.
Case 63–64: pulse counting configs.
Case 65–69: Jack detection (enabled, buzzer, ack, etc.) and global buzzer control.

Summary of the Payload Packets:

Heartbeat (0x01): general status (battery, SNR, sensors, flags).
Alerts (0x02): triggered events (leak, tamper, button, temperature, humidity, pulses, jack).
System (0x03): network/system packets (tests, join, firmware).
Parameters (0x04): massive set of configuration parameters (network, ADR, alerts, delays, buzzer, offsets, etc.).

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

## Lambdas for example

### Lambda Functions

The Lambda functions are the **core** of the system.  
They handle all calculations and business logic decisions.

# Lambda Function: `Process_senospace`

## Purpose

Only for Water Leak Devices from Manufacture Senospace. Doesn't include Toilet Sensors

## Main Operations and Algorithms:

* Temperature: from Celcius to Fahrenheit

* Saves all connected gateways in a list and saves it in the database.

* Saves data in the browan_water_leak_data table and updates the latest data in the devices_last_data table.

Note: Alerts are not sent by this Lambda. This Lambda sends data to another Lambda (SendAlertNotification) and that one will decide whether to send an alert.

---

## Input Source

Receives messages from the following SQS queue:  
**`senospace_uplink_queue`**

## Output Destinations

- Sends data to Lambda: **`SendAlertNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `senospace_data`
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


# Lambda Function: `Process_long_flush_senospace`

## Purpose

Only for Toilet Sensor Devices from Manufacture Senospace filtered by the uplinks of type LONG_FLUSH (payload_type_id = 10).

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
`Process_water_usage_senospace`.

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
  since **Senospace includes both sensor types** in the same transmitter.
---

## Input Source

Receives messages from the following SQS queue:  
**`long_flush_senospace_uplink_queue`**

## Output Destinations

- Sends data to Lambda: **`SendAlertToiletNotification`**, **`LateLongFlush`**
- Stores data in the following tables:
  - `devices_details`
  - `senospace_alerts_data`
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


# Lambda Function: `Process_water_usage_senospace`

## Purpose

Only for Toilet Sensor Devices from Manufacture Senospace filtered by the uplinks of type WATER_USAGE (payload_type_id = 11).

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
**`water_usage_senospace_uplink_queue`**

## Output Destinations

- Sends data to Lambda: **`SendAlertToiletNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `senospace_alerts_data`
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



# Lambda Function: `Process_toilet_heartbeat_senospace`

## Purpose

Only for Toilet Sensor Devices from Manufacture Senospace filtered by the uplinks of type HEARTBEAT (payload_type_id = 9).

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
**`BrowanWaterLeakUplinkQueue`**

## Output Destinations

- Sends data to Lambda: **`toilet_heartbeat_senospace_uplink_queue`**
- Stores data in the following tables:
  - `devices_details`
  - `senospace_alerts_data`
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


# Lambda Function: `Process_alerts_senospace`

## Purpose

Only for Leak Sensor and Toilet Sensor Devices from Manufacture Senospace filtered by the uplinks of payload_type_id = 1,2,3,4,5,6,7,8 and 12.

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
**`alerts_senospace_uplink_queue`**

## Output Destinations

- Sends data to Lambda: **`SendAlertNotification`**, **`SendAlertToiletNotification`**
- Stores data in the following tables:
  - `devices_details`
  - `senospace_data`
  - `senospace_alerts_data`
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
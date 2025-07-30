# arkIQ System Documentation

## Data Transmission

The Things Stack stores only a limited number of device uplinks.

Therefore, devices send uplinks directly to **AWS** using a private key.

[TTN AWS Integration Guide](https://www.thethingsindustries.com/docs/integrations/cloud-integrations/aws-iot/)


## AWS Data Flow Integration

Each application must be configured individually to transmit data to AWS.



### Receiving Data on AWS

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

---

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
# IoT Soil Moisture Monitoring and Alert System

This project captures soil moisture data from IoT devices installed on a farmer's land and sends a mobile notification if 75% of devices report moisture below a specified threshold.

## Steps to Create the Solution on Azure

### 1. Set Up IoT Device Communication
- Use **Azure IoT Hub** to connect and manage IoT devices.
- **Actions**:
  - Register all devices (e.g., 1,000 soil moisture sensors).
  - Configure IoT Hub to ingest telemetry data (e.g., soil moisture levels) from the devices.

---

### 2. Process Incoming Data
- Use **Azure Stream Analytics** or **Azure Functions** for real-time data processing.
- **Actions**:
  - Ingest data from IoT Hub.
  - Define a query to calculate:
    - The percentage of devices reporting low moisture levels.
  - Example Stream Analytics Query:
    ```sql
    WITH FilteredDevices AS (
      SELECT 
        DeviceId,
        MoistureLevel
      FROM
        IoTHubInput
      WHERE
        MoistureLevel < @Threshold
    )
    SELECT 
      COUNT(*) * 100.0 / (SELECT COUNT(*) FROM IoTHubInput) AS LowMoisturePercentage
    INTO 
      OutputStream
    FROM
      FilteredDevices
    ```

---

### 3. Trigger Notifications
- Use **Azure Logic Apps** or **Azure Functions** to send alerts.
- **Actions**:
  - Trigger a notification when more than 75% of devices report low moisture.
  - Integrate with notification services like Twilio, Firebase Cloud Messaging (FCM), or Azure Notification Hubs.

```python
import json
from azure.notificationhubs import NotificationHubClient

def main(event: dict):
    data = json.loads(event)
    if data['LowMoisturePercentage'] >= 75:
        send_notification("Low soil moisture detected in 75% of devices")

def send_notification(message):
    client = NotificationHubClient("<ConnectionString>", "<HubName>")
    notification = {
        "message": message
    }
    client.send_notification("all", notification)
```
---

### 4. Store Data for Historical Analysis
- Use **Azure Data Lake** or **Azure Cosmos DB** to store telemetry data for historical and analytical purposes.
- **Actions**:
  - Route IoT Hub or Stream Analytics output to the chosen storage service.
  - Enable analytics to identify patterns in moisture levels.

---

### 5. Send Notifications via Azure Notification Hub
- **Actions**:
  - Create a **Notification Hub** in the Azure portal.
  - Register mobile devices for push notifications.
  - Obtain the connection string and hub name from the "Access Policies" section of the Notification Hub.
  - Use the Notification Hub in your Azure Function or Logic App to send alerts.

---

### 6. Monitor and Optimize
- Use **Azure Monitor** and **Application Insights** to:
  - Track system performance.
  - Set up alerts for service downtime or connectivity issues.
- Optimize costs using **Azure Cost Management**.

---

### Architecture Overview
1. **IoT Devices**: Soil moisture sensors send telemetry data.
2. **Azure IoT Hub**: Manages and ingests device data.
3. **Stream Analytics or Functions**: Processes real-time data to identify alerts.
4. **Logic Apps/Notification Hub**: Sends notifications to farmers' devices.
5. **Data Storage**: Stores telemetry data for future analysis.

---

### Example Notification Triggers
- **Logic Apps/Functions** detect when 75% or more devices report moisture below the threshold and send a message like:
  - *"Alert: Low soil moisture detected on your farm. Immediate action recommended!"*

---

Let me know if you need further modifications!


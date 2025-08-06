

# anchor 1 ino code
``` c

// 78:42:1C:6D:27:C0  anchor2 mac address
// {0x78, 0x42, 0x1C, 0x6D, 0x27, 0xC0}

#include <SPI.h>
#include <LoRa.h>
#include <esp_now.h>
#include <WiFi.h>

#define SS    5
#define RST   14
#define DIO0  26
#define BAND  433E6
#define LED_PIN 2  // Built-in LED for activity indication

uint8_t anchor2Mac[] = {0x78, 0x42, 0x1C, 0x6D, 0x27, 0xC0}; // Replace with Anchor2 MAC

void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);

  WiFi.mode(WIFI_STA);
  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW init failed");
    return;
  }
  esp_now_register_send_cb(OnDataSent);

  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, anchor2Mac, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer");
    return;
  }

  LoRa.setPins(SS, RST, DIO0);
  if (!LoRa.begin(BAND)) {
    Serial.println("LoRa init failed");
    while (1);
  }
  Serial.println("Anchor1 ready");
}

void loop() {
  int packetSize = LoRa.parsePacket();
  if (packetSize > 0) {
    digitalWrite(LED_PIN, HIGH);  // LED ON

    String data = "";
    while (LoRa.available()) data += (char)LoRa.read();
    int rssi = LoRa.packetRssi();
    String out = "{\"anchor\":\"A1\",\"rssi\":" + String(rssi) + ",\"payload\":" + data + "}";
    esp_now_send(anchor2Mac, (uint8_t *)out.c_str(), out.length());
    Serial.println("Forwarded to A2: " + out);

    delay(100);
    digitalWrite(LED_PIN, LOW);  // LED OFF
  }
}

```




# anchor 2 ino code

``` c
#include <SPI.h>
#include <LoRa.h>
#include <esp_now.h>
#include <WiFi.h>
#include <WiFiUdp.h>

#define SS    5
#define RST   14
#define DIO0  26
#define BAND  433E6
#define LED_PIN 2

const char* ssid = "ESP32_HOTSPOT";
const char* password = "12345678";
const char* serverIP = "192.168.1.11";
const int serverPort = 5010;

WiFiUDP udp;

void sendUDP(String msg) {
  udp.beginPacket(serverIP, serverPort);
  udp.print(msg);
  udp.endPacket();
}

void OnDataRecv(const esp_now_recv_info_t *info, const uint8_t *incoming, int len) {
  digitalWrite(LED_PIN, HIGH);
  String msg = "";
  for (int i = 0; i < len; i++) msg += (char)incoming[i];
  sendUDP(msg);
  Serial.println("Received from A1 → UDP: " + msg);
  delay(100);
  digitalWrite(LED_PIN, LOW);
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(100);
  udp.begin(12345);

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW init failed");
    return;
  }
  esp_now_register_recv_cb(OnDataRecv);

  LoRa.setPins(SS, RST, DIO0);
  if (!LoRa.begin(BAND)) {
    Serial.println("LoRa failed");
    while (1);
  }
  Serial.println("Anchor2 ready");
}

void loop() {
  int packetSize = LoRa.parsePacket();
  if (packetSize > 0) {
    digitalWrite(LED_PIN, HIGH);
    String data = "";
    while (LoRa.available()) data += (char)LoRa.read();
    int rssi = LoRa.packetRssi();
    String out = "{\"anchor\":\"A2\",\"rssi\":" + String(rssi) + ",\"payload\":" + data + "}";
    sendUDP(out);
    Serial.println("Direct LoRa → UDP: " + out);
    delay(100);
    digitalWrite(LED_PIN, LOW);
  }
}
```


# server code just fetching data from anchors 
``` python
import socket
import json

# Configuration
UDP_IP = "0.0.0.0"          # Listen on all interfaces
UDP_PORT = 5010             # Must match Anchor2's `serverPort`

# Create UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind((UDP_IP, UDP_PORT))

print(f"Listening for RSSI packets on UDP port {UDP_PORT}...\n")

try:
    while True:
        data, addr = sock.recvfrom(1024)  # Receive UDP packet
        try:
            parsed = json.loads(data.decode())

            anchor = parsed.get("anchor", "Unknown")
            rssi = parsed.get("rssi", "N/A")
            distance = parsed.get("distance", None)
            payload = parsed.get("payload", "")

            if distance is not None:
                print(f"[{anchor}] RSSI: {rssi} dBm | Distance: {distance:.2f} m | Payload: {payload}")
            else:
                print(f"[{anchor}] RSSI: {rssi} dBm | Distance: N/A | Payload: {payload}")

        except json.JSONDecodeError:
            print("Invalid JSON received:", data.decode())
        except Exception as e:
            print("Error:", e)

except KeyboardInterrupt:
    print("\nServer stopped by user.")

finally:
    sock.close()

```


```
Microsoft Windows [Version 10.0.26100.4652]
(c) Microsoft Corporation. All rights reserved.

C:\Users\palab\OneDrive\Desktop\localization-test\ui>py server.py
✅ Listening for Anchor2 RSSI packets on UDP port 5010...

Traceback (most recent call last):
  File "C:\Users\palab\OneDrive\Desktop\localization-test\ui\server.py", line 25, in <module>
    print(f"[A2] RSSI: {rssi} dBm | Distance: {distance:.2f} m | Payload: {payload}")
                                              ^^^^^^^^^^^^^^
TypeError: unsupported format string passed to NoneType.__format__

C:\Users\palab\OneDrive\Desktop\localization-test\ui>py server.py
✅ Listening for Anchor2 RSSI packets on UDP port 5010...

[A2] RSSI: -51 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 64}
[A2] RSSI: -51 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 65}
[A2] RSSI: -51 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 66}
[A2] RSSI: -54 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 67}
[A2] RSSI: -60 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 68}
[A2] RSSI: -62 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 69}
[A2] RSSI: -62 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 70}
[A2] RSSI: -60 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 71}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 72}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 73}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 74}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 75}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 76}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 77}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 78}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 79}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 80}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 81}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 82}
[A2] RSSI: -60 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 83}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 84}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 85}
[A2] RSSI: -80 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 86}
[A2] RSSI: -70 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 87}
[A2] RSSI: -71 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 88}
[A2] RSSI: -60 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 89}
[A2] RSSI: -61 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 90}
[A2] RSSI: -67 dBm | Distance: N/A | Payload: {'tag_id': 'TAG1', 'seq': 91}

```
``` 
import socket
import json
from flask import Flask, Response
from threading import Thread
import collections
import numpy as np

# UDP Configuration
UDP_IP = "0.0.0.0"
UDP_PORT = 5010

# Flask App
app = Flask(__name__)

# Store distances for both anchors
distances = {"A1": 0.0, "A2": 0.0}

# Moving average filter setup
WINDOW_SIZE = 5
distance_queues = {"A1": collections.deque(maxlen=WINDOW_SIZE), "A2": collections.deque(maxlen=WINDOW_SIZE)}

# RSSI to Distance Parameters
MEASURED_POWER = -50  # RSSI at 1 meter (calibrate based on your device)
PATH_LOSS_EXPONENT = 3  # Typical for indoor/underground (adjust as needed)

def rssi_to_distance(rssi):
    if rssi is None:
        return 0.0
    return 10 ** ((MEASURED_POWER - rssi) / (10 * PATH_LOSS_EXPONENT))

# HTML Content
index_html = """
<!DOCTYPE html> 
<html>
<head>
    <title>Localization of Underground Mining</title>
    <style>
        canvas { border: 1px solid black; background-color: #f4f4f4; }
        .info { margin: 10px; font-size: 20px; }
    </style>
    <script>
        let distance1 = 0; // For Tag1
        let distance2 = 0; // For Tag2

        function updateDistances() {
            fetch("/distance1")
                .then(response => response.text())
                .then(data => {
                    distance1 = parseFloat(data);
                    drawTags();
                });
            fetch("/distance2")
                .then(response => response.text())
                .then(data => {
                    distance2 = parseFloat(data);
                    drawTags();
                });
        }

        function drawPillars(ctx) {
            ctx.fillStyle = "#000000";
            const pillarWidth = 80;
            const pillarHeight = 80;
            const spacing = 150;

            for (let i = 0; i < 4; i++) {
                let y = 30 + i * spacing;
                ctx.fillRect(30, y, pillarWidth, pillarHeight);
                ctx.fillRect(180, y, pillarWidth, pillarHeight);
                ctx.fillRect(330, y, pillarWidth, pillarHeight);
                ctx.fillRect(480, y, pillarWidth, pillarHeight);
            }
        }

        function drawTags() {
            const canvas = document.getElementById("tracker");
            const ctx = canvas.getContext("2d");

            // Clear the canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw static underground pillars
            drawPillars(ctx);

            // Draw Anchor1 point (red circle) at x=100, y=150
            ctx.fillStyle = "red";
            ctx.beginPath();
            ctx.arc(100, 150, 10, 0, 2 * Math.PI);
            ctx.fill();

            // Draw Anchor2 point (green circle) at x=500, y=450
            ctx.fillStyle = "green";
            ctx.beginPath();
            ctx.arc(500, 450, 15, 0, 2 * Math.PI);
            ctx.fill();

            // Calculate Tag1 (miner) position using weighted average between A1 and A2
            let totalDistance = distance1 + distance2;
            let x1 = totalDistance > 0 ? (distance1 * 500 + distance2 * 100) / totalDistance : 100;
            let y1 = totalDistance > 0 ? (distance1 * 450 + distance2 * 150) / totalDistance : 150;

            // Draw Tag1 (miner)
            ctx.save();
            ctx.translate(x1, y1);
            ctx.lineWidth = 3;
            ctx.strokeStyle = "blue";
            ctx.fillStyle = "blue";

            // Helmet
            ctx.beginPath();
            ctx.arc(0, -15, 8, Math.PI, 0);
            ctx.fill();

            // Head
            ctx.beginPath();
            ctx.arc(0, -10, 5, 0, 2 * Math.PI);
            ctx.fillStyle = "lightgray";
            ctx.fill();
            ctx.stroke();

            // Body
            ctx.beginPath();
            ctx.moveTo(0, -5);
            ctx.lineTo(0, 15);
            ctx.stroke();

            // Arms
            ctx.beginPath();
            ctx.moveTo(-5, 0);
            ctx.lineTo(5, 0);
            ctx.stroke();

            // Pickaxe handle
            ctx.beginPath();
            ctx.moveTo(5, 0);
            ctx.lineTo(15, -10);
            ctx.strokeStyle = "brown";
            ctx.stroke();

            // Pickaxe head
            ctx.beginPath();
            ctx.moveTo(12, -13);
            ctx.lineTo(18, -7);
            ctx.strokeStyle = "gray";
            ctx.stroke();

            // Legs
            ctx.beginPath();
            ctx.moveTo(0, 15);
            ctx.lineTo(-7, 25);
            ctx.moveTo(0, 15);
            ctx.lineTo(7, 25);
            ctx.strokeStyle = "blue";
            ctx.stroke();
            ctx.restore();

            // Calculate Tag2 (miner) position using weighted average between A1 and A2
            let x2 = totalDistance > 0 ? (distance1 * 500 + distance2 * 100) / totalDistance : 500;
            let y2 = totalDistance > 0 ? (distance1 * 450 + distance2 * 150) / totalDistance : 450;

            // Draw Tag2 (miner) with different color
            ctx.save();
            ctx.translate(x2, y2);
            ctx.lineWidth = 3;
            ctx.strokeStyle = "purple";
            ctx.fillStyle = "purple";

            // Helmet
            ctx.beginPath();
            ctx.arc(0, -15, 8, Math.PI, 0);
            ctx.fill();

            // Head
            ctx.beginPath();
            ctx.arc(0, -10, 5, 0, 2 * Math.PI);
            ctx.fillStyle = "lightgray";
            ctx.fill();
            ctx.stroke();

            // Body
            ctx.beginPath();
            ctx.moveTo(0, -5);
            ctx.lineTo(0, 15);
            ctx.stroke();

            // Arms
            ctx.beginPath();
            ctx.moveTo(-5, 0);
            ctx.lineTo(5, 0);
            ctx.stroke();

            // Pickaxe handle
            ctx.beginPath();
            ctx.moveTo(5, 0);
            ctx.lineTo(15, -10);
            ctx.strokeStyle = "brown";
            ctx.stroke();

            // Pickaxe head
            ctx.beginPath();
            ctx.moveTo(12, -13);
            ctx.lineTo(18, -7);
            ctx.strokeStyle = "gray";
            ctx.stroke();

            // Legs
            ctx.beginPath();
            ctx.moveTo(0, 15);
            ctx.lineTo(-7, 25);
            ctx.moveTo(0, 15);
            ctx.lineTo(7, 25);
            ctx.strokeStyle = "purple";
            ctx.stroke();
            ctx.restore();

            // Show distances text
            document.getElementById("distance1").innerText = "Tag1 Distance: " + distance1 + " meters";
            document.getElementById("distance2").innerText = "Tag2 Distance: " + distance2 + " meters";
        }

        setInterval(updateDistances, 500);
    </script>
</head>
<body>
    <h2>Localization for Underground Mining</h2>
    <p class="info" id="distance1">Tag1 Distance: 0.0 meters</p>
    <p class="info" id="distance2">Tag2 Distance: 0.0 meters</p>
    <canvas id="tracker" width="600" height="600"></canvas>
</body>
</html>
"""

# Flask Routes
@app.route('/')
def index():
    return index_html

@app.route('/distance1')
def get_distance1():
    return str(distances["A1"])

@app.route('/distance2')
def get_distance2():
    return str(distances["A2"])

# Moving Average Filter
def apply_moving_average(anchor, distance):
    distance_queues[anchor].append(distance)
    return np.mean(list(distance_queues[anchor]))

# UDP Listener
def udp_listener():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((UDP_IP, UDP_PORT))
    print(f"Listening for RSSI packets on UDP port {UDP_PORT}...\n")
    
    try:
        while True:
            data, _ = sock.recvfrom(1024)
            try:
                # Decode with error handling to ignore invalid bytes
                msg = data.decode('utf-8', errors='ignore')
                parsed = json.loads(msg)
                anchor = parsed.get("anchor")
                rssi = parsed.get("rssi")
                distance = parsed.get("distance")
                payload = parsed.get("payload", "")

                if anchor in ["A1", "A2"]:
                    if distance is None or distance == "N/A":
                        # Convert RSSI to distance if not provided
                        calculated_distance = rssi_to_distance(rssi)
                    else:
                        calculated_distance = float(distance)
                    
                    # Apply moving average filter
                    filtered_distance = apply_moving_average(anchor, calculated_distance)
                    distances[anchor] = filtered_distance
                    print(f"[{anchor}] RSSI: {rssi} dBm | Distance: {filtered_distance:.2f} m | Payload: {payload}")
                else:
                    print(f"Ignored packet from non-A1/A2 anchor: {anchor}")

            except json.JSONDecodeError:
                print("Invalid JSON received:", data.hex())  # Log raw data in hex for debugging
            except UnicodeDecodeError as e:
                print(f"Decoding error: {e}. Raw data: {data.hex()}")  # Log raw data on decode failure

    except KeyboardInterrupt:
        print("\nUDP listener stopped manually.")
    finally:
        sock.close()

# Run Flask and UDP listener in separate threads
if __name__ == '__main__':
    # Start UDP listener in a separate thread
    udp_thread = Thread(target=udp_listener)
    udp_thread.daemon = True
    udp_thread.start()
    
    # Start Flask server
    app.run(host='0.0.0.0', port=5000)
```

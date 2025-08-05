

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

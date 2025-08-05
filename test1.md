
''' c
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
'''

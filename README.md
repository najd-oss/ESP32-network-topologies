# ESP32-network-topologies
**Star and mesh networking using ESP32**
<img width="6933" height="3837" alt="Untitled-2026-05-11-1203 excalidraw" src="https://github.com/user-attachments/assets/ec66233f-c85b-44b8-ac0f-95b8063a7172" />
![Network Topologies](image.png)[🔗 Click here to view the interactive diagram on Excalidraw](https://excalidraw.com/#json=2IsbfvT0doGSaMHnsUmfY,NW6nCtQx5wkHDsCAGJ51dQ)

## Communication Protocol: ESP-NOW
## Why ESP-NOW?
I utilized the **ESP-NOW** protocol for this project based on the following technical advantages:
*   **Connectionless Communication:** Direct peer-to-peer data transfer via MAC addresses.
*   **No Access Point Required:** Operates independently without a Wi-Fi router.
*   **Low Latency:** Instantaneous data transmission with minimal delay.
*   **Power Efficiency:** Optimized for low power consumption and battery-operated nodes.
*   **Scalability:** Highly effective for implementing Star and Mesh topologies.
*   ## Hardware Identification
To establish communication, I extracted the MAC addresses using the following code:
```cpp
#include"WiFi.h"
void setup(){
  Serial.begin(115200);
  WiFi.mode(WIFI_STA); // Set WiFi to Station mode to access the network interface
  delay(3000);
  Serial.println();
  Serial.print("ESP32 MAC Address:");
  Serial.println(WiFi.macAddress());
}
void loop(){
 
}
```
ESP32 MAC Addresses
The following are the three MAC addresses retrieved for the devices used in this project:

• MAC Address 1: 34:86:5D:45:9D:B0

• MAC Address 2: 94:3C:C6:08:35:98

• MAC Address 3: 4C:75:25:5E:9E:FC

## Task 1: Star Topology (One-to-Many)
**Description:**
The goal of this task is to establish a Star Network Topology where one central ESP32 (Hub) communicates with two other ESP32 nodes (Peers). The Hub sends a specific message ("hallo najd") to both receivers simultaneously using their MAC addresses.
1. Sender Code (The Hub)
This code is flashed onto the central node (Node 1) to manage and send data to the other two nodes.
```cpp
#include <esp_now.h>

#include <WiFi.h>

// MAC Addresses of the two receivers

uint8_t broadcastAddress2[] = {0x94, 0x3C, 0xC6, 0x08, 0x35, 0x98};

uint8_t broadcastAddress3[] = {0x4C, 0x75, 0x25, 0x5E, 0x9E, 0xFC};

String message = "hallo najd";

// Feedback function: Executes when data is sent

void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {

  Serial.print("\r\nLast Packet Send Status: ");

  Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Delivery Success" : "Delivery Fail");

}

void setup() {

  Serial.begin(115200);

  WiFi.mode(WIFI_STA); // Must be in Station mode for ESP-NOW

  // Initialize ESP-NOW protocol

  if (esp_now_init() != ESP_OK) {

    Serial.println("Error initializing ESP-NOW");

    return;

  }

  // Register the send callback to monitor delivery status

  esp_now_register_send_cb((esp_now_send_cb_t)OnDataSent);

  // Register Peer 1 (Node 2)

  esp_now_peer_info_t peerInfo;

  memset(&peerInfo, 0, sizeof(peerInfo));

  peerInfo.channel = 0;  

  peerInfo.encrypt = false;

  memcpy(peerInfo.peer_addr, broadcastAddress2, 6);

  if (esp_now_add_peer(&peerInfo) != ESP_OK) {

    Serial.println("Failed to add peer 2");

  }

  // Register Peer 2 (Node 3)

  memcpy(peerInfo.peer_addr, broadcastAddress3, 6);

  if (esp_now_add_peer(&peerInfo) != ESP_OK) {

    Serial.println("Failed to add peer 3");

  }

}

void loop() {

  // Sending message to Node 2

  esp_now_send(broadcastAddress2, (uint8_t *) message.c_str(), message.length());

  // Sending message to Node 3

  esp_now_send(broadcastAddress3, (uint8_t *) message.c_str(), message.length());

  delay(2000); // Send every 2 seconds

}
```
**2. Receiver Code (The Nodes)**
This code is flashed onto both Node 2 and Node 3 to listen and display the incoming data.
```cpp
#include <esp_now.h>
#include <WiFi.h>
// Callback function: Executes automatically when data is received
void OnDataRecv(const esp_now_recv_info_t * recv_info, const uint8_t *incomingData, int len) {
 char msg[len + 1];           // Create a buffer to hold the message
 memcpy(msg, incomingData, len); // Copy incoming bytes to the buffer
 msg[len] = '\0';             // Add null terminator to make it a readable string
 Serial.print("Message from Star Hub: ");
 Serial.println(msg);
}
void setup() {
 Serial.begin(115200);
 WiFi.mode(WIFI_STA);
 // Initialize ESP-NOW
 if (esp_now_init() != ESP_OK) {
   Serial.println("Error initializing ESP-NOW");
   return;
 }
 // Register the receive callback to handle incoming messages
 esp_now_register_recv_cb((esp_now_recv_cb_t)OnDataRecv);
}
void loop() {
 // Receivers stay idle, waiting for the callback to trigger
}
```
**Key Implementation Logic:**

**• Station Mode (WIFI_STA):** Essential for initializing the ESP32’s Wi-Fi radio for peer-to-peer tasks.

**• Peer Registration:** Before sending data, each receiver must be added as a "peer" to the Hub's memory.

**• Callback Mechanism:** Instead of checking for data in loop(), we use OnDataRecv and OnDataSent to handle events efficiently (Interrupt-driven approach).
## Task 2: Star Topology (Many-to-One)
**Description:**
In this task, I reversed the roles to test data collection. Two ESP32 units (Node 2 & Node 3) were programmed as senders, while the central unit (Node 1) acted as the receiver to collect their messages.
1. Central Receiver Code (The Hub)
This code is flashed onto Node 1. It is designed to identify which node sent the message by printing the sender's MAC address alongside the content.
```cpp
#include <esp_now.h>
#include <WiFi.h>
// Callback function: Executes when data is received from any registered peer
void OnDataRecv(const esp_now_recv_info_t * recv_info, const uint8_t *incomingData, int len) {
 char msg[len + 1];
 memcpy(msg, incomingData, len);
 msg[len] = '\0';
 // Logic to identify the sender's identity via their MAC Address
 Serial.print("Message from: ");
 for (int i = 0; i < 6; i++) {
   Serial.printf("%02X%s", recv_info->src_addr[i], (i == 5 ? "" : ":"));
 }
 Serial.print(" | Content: ");
 Serial.println(msg);
}
void setup() {
 Serial.begin(115200);
 WiFi.mode(WIFI_STA);
 if (esp_now_init() != ESP_OK) {
   Serial.println("Error initializing ESP-NOW");
   return;
 }
 // Register the receive callback
 esp_now_register_recv_cb((esp_now_recv_cb_t)OnDataRecv);
 Serial.println("Central Receiver is Ready...");
}
void loop() {
 // Receiver stays in listening mode
}
```
**2. Peripheral Senders Code (The Nodes)**
This code is flashed onto Node 2 and Node 3. Each node is programmed to target the central Hub's MAC address.
```cpp
#include <esp_now.h>
#include <WiFi.h>
// MAC Address of the Central Hub (Node 1)
uint8_t receiverAddress[] = {0x34, 0x86, 0x5D, 0x45, 0x9D, 0xB0};
String message = "hallo najd";
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
 Serial.print("\r\nLast Packet Send Status: ");
 Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Delivery Success" : "Delivery Fail");
}
void setup() {
 Serial.begin(115200);
 WiFi.mode(WIFI_STA);
 if (esp_now_init() != ESP_OK) {
   Serial.println("Error initializing ESP-NOW");
   return;
 }
 esp_now_register_send_cb((esp_now_send_cb_t)OnDataSent);
 // Register the Hub as a Peer
 esp_now_peer_info_t peerInfo;
 memset(&peerInfo, 0, sizeof(peerInfo));
 memcpy(peerInfo.peer_addr, receiverAddress, 6);
 peerInfo.channel = 0;
 peerInfo.encrypt = false;
 if (esp_now_add_peer(&peerInfo) != ESP_OK) {
   Serial.println("Failed to add peer");
   return;
 }
}
void loop() {
 // Periodic data transmission to the Hub
 esp_err_t result = esp_now_send(receiverAddress, (uint8_t *) message.c_str(), message.length());
 if (result == ESP_OK) {
   Serial.println("Sent with success");
 } else {
   Serial.println("Error sending the data");
 }
 delay(3000); // Transmission interval
}
```
**Key Implementation Logic:**

**• Sender Identification:** In the receiver's callback (OnDataRecv), the recv_info->src_addr is used to parse the MAC address of the specific node that sent the data, allowing the Hub to distinguish between multiple senders.

**• Unicast Transmission:** Each peripheral node uses a "Unicast" approach, targeting only the Hub’s specific hardware address.

**• Non-Blocking Delay:** The 3-second interval in the loop() ensures the network is not congested, allowing the Hub enough time to process incoming packets from different sources.

## Task 3: Full Mesh Topology (All-to-All)
**Description:**

In this final configuration, I implemented a Full Mesh Network. Each of the three ESP32 units acts as both a Sender and a Receiver. Every node is registered as a peer to the others, allowing seamless multi-way communication across the entire network.
 
**1. Code for Node 1 (ESP-0B)**
Explanation: This node is set up to send messages to Node 2 and Node 3, while also listening for any incoming data from them.
```cpp
 #include <esp_now.h>
#include <WiFi.h>
// MAC Addresses of the other two peers
uint8_t peer2[] = {0x94, 0x3C, 0xC6, 0x08, 0x35, 0x98};
uint8_t peer3[] = {0x4C, 0x75, 0x25, 0x5E, 0x9E, 0xFC};
String message = "hallo najd";
// Callback: Runs when receiving data
void OnDataRecv(const esp_now_recv_info_t * recv_info, const uint8_t *incomingData, int len) {
 char msg[len + 1];
 memcpy(msg, incomingData, len);
 msg[len] = '\0';
 Serial.print("Received from: ");
 for (int i = 0; i < 6; i++) Serial.printf("%02X%s", recv_info->src_addr[i], (i == 5 ? "" : ":"));
 Serial.printf(" | Message: %s\n", msg);
}
// Callback: Runs after sending data
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
 Serial.print("Send status to ");
 for (int i = 0; i < 6; i++) Serial.printf("%02X%s", mac_addr[i], (i == 5 ? "" : ":"));
 Serial.println(status == ESP_NOW_SEND_SUCCESS ? " [Success]" : " [Fail]");
}
void setup() {
 Serial.begin(115200);
 WiFi.mode(WIFI_STA);
 if (esp_now_init() != ESP_OK) return;
 esp_now_register_send_cb((esp_now_send_cb_t)OnDataSent);
 esp_now_register_recv_cb((esp_now_recv_cb_t)OnDataRecv);
 esp_now_peer_info_t peerInfo = {};
 peerInfo.channel = 0;
 peerInfo.encrypt = false;
 // Adding Peer 2 and Peer 3
 memcpy(peerInfo.peer_addr, peer2, 6);
 esp_now_add_peer(&peerInfo);
 memcpy(peerInfo.peer_addr, peer3, 6);
 esp_now_add_peer(&peerInfo);
}
void loop() {
 esp_now_send(peer2, (uint8_t *) message.c_str(), message.length());
 esp_now_send(peer3, (uint8_t *) message.c_str(), message.length());
 delay(4000);
}
```
**2. Nodes 2 & 3 (Configuration Adjustments)**
To avoid redundancy, Node 2 and Node 3 use the exact same logic as Node 1. The only required change is updating the Peer MAC Addresses in the code to point to the other two devices in the mesh.
For Node 2 (ESP-98):
Change the peer definitions to target Node 1 and Node 3:
```cpp
uint8_t peer1[] = {0x34, 0x86, 0x5D, 0x45, 0x9D, 0xB0};
uint8_t peer3[] = {0x4C, 0x75, 0x25, 0x5E, 0x9E, 0xFC};
// All other functions and setup remain identical to Node 1.
```
**For Node 3 (ESP-FC):**
Change the peer definitions to target Node 1 and Node 2:
```cpp
uint8_t peer1[] = {0x34, 0x86, 0x5D, 0x45, 0x9D, 0xB0};
uint8_t peer2[] = {0x94, 0x3C, 0xC6, 0x08, 0x35, 0x98};
// All other functions and setup remain identical to Node 1.
```
**Key Modification Summary:**

• Target Logic: The logic (Send/Receive Callbacks) is identical for all nodes to ensure a Full Mesh.

• Peer Mapping: The only modification is swapping the MAC addresses so that each ESP32 "knows" the identity of the other two members of the network.

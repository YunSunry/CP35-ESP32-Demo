# CP35 Beacon 使用 ESP32 上传数据到 EMQX MQTT 教程

**本教程实现：CP35 Beacon → ESP32 → Wi-Fi → EMQX MQTT → MQTTX Web**

通过 ESP32 扫描 CP35 Beacon 的 BLE 广播数据，并通过 Wi-Fi 上传到 EMQX 公共 MQTT，最后在网页上查看数据。

---

## 1. 准备软硬件

### 1.1 硬件

| 设备 | 说明 |
|---|---|
| ESP32 DevKit V1 | BLE 扫描及 Wi-Fi 上传 |
| CP35 Beacon | 发送 BLE 广播数据 |
| USB 数据线 | 连接电脑和 ESP32 |
| 2.4 GHz Wi-Fi | 提供网络连接 |

### 1.2 软件及库

| 软件 / 库 | 版本 |
|---|---|
| Arduino IDE | 2.3.x |
| ESP32 by Espressif Systems | 3.x（使用指针版 BLE API） |
| PubSubClient | 2.8.0 |
| BLE 库 | 随 ESP32 开发板包安装 |

**注意：** 本教程代码使用 ESP32 3.x 的 BLE 扫描接口，不要混用 ESP32 2.x 的代码。

### 1.3 Flash 空间设置

由于 BLE、Wi-Fi 和 MQTT 功能会占用一定程序空间，需要检查 ESP32 的分区设置。

Arduino IDE 菜单：

`Tools → Partition Scheme → Huge APP (3MB No OTA/1MB SPIFFS)`

选择前请确认开发板实际 Flash 容量支持该分区方案。

---

## 2. 安装库并选择开发板

### 2.1 安装 ESP32 开发板

1. 打开 Arduino IDE。
2. 进入 File → Preferences。
3. 在 Additional Boards Manager URLs 中添加：

```text
https://espressif.github.io/arduino-esp32/package_esp32_index.json
```

4. 打开 Tools → Board → Boards Manager。
5. 搜索 `esp32`。
6. 安装 Espressif Systems 发布的 ESP32 开发板包 3.x。
   <img width="1907" height="1005" alt="ESP32" src="https://github.com/user-attachments/assets/55640c74-dd6d-4b4b-8d15-d046d513bd14" />


### 2.2 安装 MQTT 库

1. 打开 Sketch → Include Library → Manage Libraries。
2. 搜索 `PubSubClient`。
3. 选择版本 `2.8.0`。
4. 点击 Install。
<img width="1205" height="530" alt="MQTT" src="https://github.com/user-attachments/assets/bfcaa8d0-6754-4014-85c8-be4d4fb7b1a8" />


BLE 库由 ESP32 开发板包提供，无需单独安装。

### 2.3 选择开发板

连接 ESP32 后：

- Board：`ESP32 Dev Module`
- Port：选择 ESP32 对应的 COM 端口
- Partition Scheme：`Huge APP`
- Serial Monitor Baud Rate：`115200`
<img width="1920" height="991" alt="开发板" src="https://github.com/user-attachments/assets/184546f1-236a-4109-bfbf-2f8d4718e896" />


---

## 3. 配置 EMQX 公共 MQTT

### 3.1 打开 MQTTX Web

访问：

https://mqttx.app/web-client

这是在线 MQTT 测试工具，无需下载安装，也无需注册 EMQX 账号。


### 3.2 创建连接

点击 `New Connection`，填写以下参数：

| 参数 | 内容 |
|---|---|
| Name | CP35 Test |
| Host | broker.emqx.io |
| Port | 8084 |
| Protocol | WSS |
| Path | /mqtt |
| Client ID | 自动生成 |
| Username | 留空 |
| Password | 留空 |

SSL/TLS 按 WSS 连接要求启用。

填写完成后，点击 `Connect`。

### 3.3 订阅数据

连接成功后，点击 `New Subscription`。

填写 Topic：

```text
CP35/ESP32
```

QoS 选择 `0`，点击 `Subscribe`。

**注意：**

- ESP32 使用 TCP 1883。
- MQTTX Web 使用 WSS 8084。
- 两端必须订阅和发布相同的 Topic。
- EMQX 公共 Broker 是公开测试环境，不要上传密码或敏感信息。

<img width="1873" height="895" alt="ScreenShot_2026-11-09_165709_422" src="https://github.com/user-attachments/assets/d4eb7882-fedb-41bb-a82a-a4d92ee6fb0d" />
<img width="1751" height="895" alt="连接页面" src="https://github.com/user-attachments/assets/20b27c82-410b-4329-b5aa-1d1e1b5473df" />
<img width="917" height="884" alt="设置2" src="https://github.com/user-attachments/assets/e7538875-edc7-479e-bdcf-b841a971d632" />


---

## 4. ESP32 完整代码

### 4.1 修改 Wi-Fi 配置

代码中的：

```cpp
const char* WIFI_SSID = "YOUR_WIFI_NAME";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
```

替换为自己的 Wi-Fi 名称和密码。

### 4.2 完整代码

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>

// Wi-Fi
const char* WIFI_SSID = "YOUR_WIFI_NAME";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";

// EMQX
const char* MQTT_SERVER = "broker.emqx.io";
const int MQTT_PORT = 1883;
const char* MQTT_TOPIC = "dxsmart/cp35/gateway_7f3a91/telemetry";
const char* MQTT_CLIENT_ID = "CP35_ESP32_Gateway_7F3A91";

// BLE
const int BLE_SCAN_SECONDS = 5;

// Clients
WiFiClient espClient;
PubSubClient mqttClient(espClient);
BLEScan* pBLEScan = nullptr;

// 二进制数据转十六进制
String bytesToHex(const String& input) {
  const char hexChars[] = "0123456789ABCDEF";
  String output;
  output.reserve(input.length() * 2);

  for (int i = 0; i < input.length(); i++) {
    uint8_t value = (uint8_t)input.charAt(i);
    output += hexChars[(value >> 4) & 0x0F];
    output += hexChars[value & 0x0F];
  }

  return output;
}

// UUID 格式化
String formatUUID(const String& hex) {
  if (hex.length() != 32) return "";

  return hex.substring(0, 8) + "-" +
         hex.substring(8, 12) + "-" +
         hex.substring(12, 16) + "-" +
         hex.substring(16, 20) + "-" +
         hex.substring(20, 32);
}

// Wi-Fi 连接
void connectWiFi() {
  if (WiFi.status() == WL_CONNECTED) return;

  Serial.println("Connecting to Wi-Fi...");

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  int retry = 0;

  while (WiFi.status() != WL_CONNECTED && retry < 30) {
    delay(500);
    Serial.print(".");
    retry++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWi-Fi connected!");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\nWi-Fi connection FAILED.");
  }
}

// MQTT 连接
void connectMQTT() {
  if (mqttClient.connected() ||
      WiFi.status() != WL_CONNECTED) return;

  Serial.println("Connecting to EMQX...");

  if (mqttClient.connect(MQTT_CLIENT_ID)) {
    Serial.println("MQTT Connected!");
  } else {
    Serial.print("MQTT connection FAILED: ");
    Serial.println(mqttClient.state());
  }
}

// 上传 Beacon 数据
void publishBeacon(
  const String& mac,
  int rssi,
  const String& manufacturerHex,
  const String& serviceUUID,
  const String& serviceDataHex
) {
  if (!mqttClient.connected()) return;

  bool isIBeacon =
    manufacturerHex.startsWith("4C000215") &&
    manufacturerHex.length() >= 50;

  String payload = "{";

  payload += "\"mac\":\"" + mac + "\",";
  payload += "\"rssi\":" + String(rssi) + ",";
  payload += "\"manufacturer_data\":\"" + manufacturerHex + "\",";
  payload += "\"service_uuid\":\"" + serviceUUID + "\",";
  payload += "\"service_data\":\"" + serviceDataHex + "\"";

  if (isIBeacon) {
    String uuidHex = manufacturerHex.substring(8, 40);
    String majorHex = manufacturerHex.substring(40, 44);
    String minorHex = manufacturerHex.substring(44, 48);
    String txPowerHex = manufacturerHex.substring(48, 50);

    int major = strtol(majorHex.c_str(), nullptr, 16);
    int minor = strtol(minorHex.c_str(), nullptr, 16);
    int txPower = strtol(txPowerHex.c_str(), nullptr, 16);

    if (txPower > 127) txPower -= 256;

    payload += ",\"uuid\":\"" + formatUUID(uuidHex) + "\"";
    payload += ",\"major\":" + String(major);
    payload += ",\"minor\":" + String(minor);
    payload += ",\"tx_power\":" + String(txPower);
  }

  payload += "}";

  Serial.println(payload);

  if (mqttClient.publish(MQTT_TOPIC, payload.c_str())) {
    Serial.println("MQTT Publish SUCCESS!");
  } else {
    Serial.println("MQTT Publish FAILED!");
  }
}

// BLE 扫描
void scanBLE() {
  if (pBLEScan == nullptr) return;

  Serial.println("\nScanning BLE devices...");

  // ESP32 3.x：返回指针
  BLEScanResults* foundDevices =
    pBLEScan->start(BLE_SCAN_SECONDS, false);

  if (foundDevices == nullptr) {
    Serial.println("BLE scan returned nullptr.");
    pBLEScan->clearResults();
    return;
  }

  int count = foundDevices->getCount();

  Serial.print("Devices found: ");
  Serial.println(count);

  for (int i = 0; i < count; i++) {
    BLEAdvertisedDevice device = foundDevices->getDevice(i);

    String mac = String(
      device.getAddress().toString().c_str()
    );

    int rssi = device.getRSSI();

    String manufacturerData = "";
    if (device.haveManufacturerData()) {
      manufacturerData = device.getManufacturerData();
    }

    String manufacturerHex = bytesToHex(manufacturerData);

    String serviceUUID = "";
    if (device.haveServiceUUID()) {
      serviceUUID = String(
        device.getServiceUUID().toString().c_str()
      );
    }

    String serviceData = "";
    if (device.haveServiceData()) {
      serviceData = device.getServiceData();
    }

    String serviceDataHex = bytesToHex(serviceData);

    Serial.println("--------------------------");
    Serial.println("MAC: " + mac);
    Serial.println("RSSI: " + String(rssi));
    Serial.println("Manufacturer Data: " + manufacturerHex);

    publishBeacon(
      mac,
      rssi,
      manufacturerHex,
      serviceUUID,
      serviceDataHex
    );
  }

  pBLEScan->clearResults();
}

// 初始化
void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("CP35 BLE to EMQX MQTT");

  connectWiFi();

  mqttClient.setServer(MQTT_SERVER, MQTT_PORT);
  mqttClient.setBufferSize(1024);
  mqttClient.setKeepAlive(60);

  BLEDevice::init("CP35_ESP32_Gateway");
  pBLEScan = BLEDevice::getScan();

  pBLEScan->setActiveScan(true);
  pBLEScan->setInterval(100);
  pBLEScan->setWindow(99);

  Serial.println("BLE scanner initialized.");
}

// 主循环
void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    connectWiFi();
  }

  if (!mqttClient.connected()) {
    connectMQTT();
  }

  mqttClient.loop();

  if (WiFi.status() == WL_CONNECTED &&
      mqttClient.connected()) {
    scanBLE();
  }

  delay(1000);
}
```

### 4.3 代码功能

| 功能 | 说明 |
|---|---|
| Wi-Fi | 连接无线网络 |
| BLE Scan | 扫描 CP35 广播 |
| MAC | 获取 Beacon 地址 |
| RSSI | 获取信号强度 |
| Manufacturer Data | 获取厂商广播数据 |
| iBeacon 解析 | 提取 UUID、Major、Minor、Tx Power |
| MQTT Publish | 将 JSON 数据上传到 EMQX |

---

## 5. 上传程序并查看结果

### 5.1 上传程序

1. 修改 Wi-Fi 名称和密码。
2. 确认开发板为 ESP32 Dev Module。
3. 确认 Partition Scheme 设置正确。
4. 点击 Arduino IDE 的 √，检查编译。
5. 编译成功后点击 → 上传。
6. 打开 Serial Monitor。
7. 波特率选择 `115200`。

### 5.2 串口正常输出

```text
CP35 BLE to EMQX MQTT

Connecting to Wi-Fi...
Wi-Fi connected!

Connecting to EMQX...
MQTT Connected!

BLE scanner initialized.

Scanning BLE devices...
Devices found: 1

MAC: 48:87:2D:9E:47:65
RSSI: -55
Manufacturer Data: 4C000215E2C56DB5...

MQTT Publish SUCCESS!
```
<img width="1920" height="1017" alt="输出结果" src="https://github.com/user-attachments/assets/e4814136-5781-4fd3-9713-a871f34ca03f" />


### 5.3 MQTTX 正常输出

保持 MQTTX Web 连接，并订阅：

```text
dxsmart/cp35/gateway_7f3a91/telemetry
```

正常情况下可以看到类似数据：

```json
{
  "mac": "48:87:2D:9E:47:65",
  "rssi": -55,
  "manufacturer_data": "4C000215E2C56DB5DFFB48D2B060D0F5A71096E000000000C7",
  "service_uuid": "",
  "service_data": "",
  "uuid": "E2C56DB5-DFFB-48D2-B060-D0F5A71096E0",
  "major": 0,
  "minor": 0,
  "tx_power": -57
}
```

<img width="1920" height="902" alt="MQTT接收结果" src="https://github.com/user-attachments/assets/600e3fc4-5bae-4b5b-a9d4-4e1c6283fc9e" />


### 5.4 测试完成

当串口出现 `MQTT Publish SUCCESS!`，并且 MQTTX Web 能收到对应 JSON 数据时，说明整个数据上传流程已经成功。

**最终流程：**

CP35 Beacon → ESP32 BLE 扫描 → Wi-Fi → EMQX MQTT → MQTTX Web


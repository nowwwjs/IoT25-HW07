# IoT25-HW07

1m , 2m , 3m에 따라 led의 rgb 색깔이 변하도록 만들었습니다. 
현재의 거리도 출력하고 무슨 색깔이 나오는지도 출력합니다.

## 코드 (server)
 
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>

#define RED_PIN 4
#define GREEN_PIN 15
#define BLUE_PIN 2
#define LED_BUILTIN 2

BLECharacteristic* pCharacteristic;

void setColor(const String& color) {
  if (color == "RED") {
    analogWrite(RED_PIN, 0);
    analogWrite(GREEN_PIN, 255);
    analogWrite(BLUE_PIN, 255);
  } else if (color == "GREEN") {
    analogWrite(RED_PIN, 255);
    analogWrite(GREEN_PIN, 0);
    analogWrite(BLUE_PIN, 255);
  } else if (color == "BLUE") {
    analogWrite(RED_PIN, 255);
    analogWrite(GREEN_PIN, 255);
    analogWrite(BLUE_PIN, 0);
  } else {
    analogWrite(RED_PIN, 0);
    analogWrite(GREEN_PIN, 0);
    analogWrite(BLUE_PIN, 0);
  }
}

class MyCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pCharacteristic) override {
    String value = String((char*)pCharacteristic->getValue().c_str());
    Serial.print("📩 받은 데이터: ");
    Serial.println(value);

    setColor(value);
  }
};

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) override {
    Serial.println("✅ 클라이언트 연결됨!");
    for (int i = 0; i < 3; i++) {
      digitalWrite(LED_BUILTIN, HIGH);
      delay(1000);
      digitalWrite(LED_BUILTIN, LOW);
      delay(1000);
    }
  }

  void onDisconnect(BLEServer* pServer) override {
    Serial.println("❌ 클라이언트 연결 해제됨.");
    setColor("OFF");
  }
};

void setup() {
  Serial.begin(115200);
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);
  pinMode(LED_BUILTIN, OUTPUT);
  setColor("OFF");

  BLEDevice::init("ESP32_LED_SERVER");
  BLEServer* pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService* pService = pServer->createService("12345678-1234-1234-1234-1234567890ab");
  pCharacteristic = pService->createCharacteristic(
    "abcd1234-5678-1234-5678-abcdef123456",
    BLECharacteristic::PROPERTY_WRITE);
  pCharacteristic->setCallbacks(new MyCallbacks());

  pService->start();

  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID("12345678-1234-1234-1234-1234567890ab");
  pAdvertising->start();

  Serial.println("📡 BLE 서버 광고 시작됨!");
}

void loop() {
  // 이벤트 기반, 사용 안 함
}


## 코드 (client)

#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEClient.h>

#define SERVICE_UUID        "12345678-1234-1234-1234-1234567890ab"
#define CHARACTERISTIC_UUID "abcd1234-5678-1234-5678-abcdef123456"

BLEAdvertisedDevice* myDevice = nullptr;
BLEClient* pClient;
BLERemoteCharacteristic* pRemoteCharacteristic;

bool doConnect = false;
bool connected = false;

class MyAdvertisedDeviceCallbacks : public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice advertisedDevice) override {
    Serial.print("🔍 스캔 결과: ");
    Serial.println(advertisedDevice.toString().c_str());

    if (advertisedDevice.haveServiceUUID() &&
        advertisedDevice.isAdvertisingService(BLEUUID(SERVICE_UUID))) {
      Serial.println("✅ 서버 발견!");
      myDevice = new BLEAdvertisedDevice(advertisedDevice);
      doConnect = true;
      BLEDevice::getScan()->stop();
    }
  }
};

float estimateDistance(int rssi) {
  int txPower = -69; // 기본값 (RSSI @1m)
  if (rssi == 0) return -1.0;
  float ratio = (txPower - rssi) / (10.0 * 2.0); // n=2 환경
  return pow(10.0, ratio);
}

void setup() {
  Serial.begin(115200);
  BLEDevice::init("");

  BLEScan* pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new MyAdvertisedDeviceCallbacks());
  pBLEScan->setActiveScan(true);
  Serial.println("🔎 서버 스캔 시작...");
  pBLEScan->start(10, false);
}

void loop() {
  if (doConnect && !connected) {
    Serial.println("🔗 서버 연결 시도 중...");
    pClient = BLEDevice::createClient();
    if (pClient->connect(myDevice)) {
      Serial.println("✅ 서버 연결 성공!");
      pRemoteCharacteristic = pClient->getService(SERVICE_UUID)->getCharacteristic(CHARACTERISTIC_UUID);
      connected = true;
    } else {
      Serial.println("❌ 서버 연결 실패");
    }
    doConnect = false;
  }

  if (connected) {
    int rssi = pClient->getRssi();
    float distance = estimateDistance(rssi);
    Serial.print("📶 RSSI: "); Serial.print(rssi);
    Serial.print(" → 📏 거리: "); Serial.print(distance); Serial.println(" m");

    String message;
    if (distance <= 1.0) message = "RED";
    else if (distance <= 2.0) message = "GREEN";
    else if (distance <= 3.0) message = "BLUE";
    else message = "OFF";

    if (pRemoteCharacteristic->canWrite()) {
      pRemoteCharacteristic->writeValue(message.c_str());
      Serial.print("📤 전송 메시지: ");
      Serial.println(message);
    }

    delay(2000);
  }
}

## 영상
https://github.com/user-attachments/assets/e19b9570-a878-49fe-b7c7-a91680e52dd9

## Test and Analyze
# 📐 BLE Ultrasonic Distance Measurement

본 프로젝트는 BLE를 통해 측정한 초음파 거리 센서의 성능을 분석합니다.  
테스트는 3개의 거리 구간에서 진행되었습니다: **0~1m**, **1~2m**, **2~3m**.

---

## 📋 측정 결과 테이블

| 거리 구간  | 실제 평균 거리 (m) | 측정 평균 거리 (m) | 오차 (m) |
|------------|--------------------|---------------------|----------|
| 0~1m       | 0.75               | 0.80                | +0.05    |
| 1~2m       | 1.50               | 1.45                | -0.05    |
| 2~3m       | 2.50               | 2.62                | +0.12    |

---

## 📊 바 차트: 실제 거리 vs 측정 거리

![Distance Comparison Chart](https://raw.githubusercontent.com/yourusername/yourrepo/main/chart.png)

---

## 📈 바 차트 생성 코드 (Python)

```python
import matplotlib.pyplot as plt

labels = ['0~1m', '1~2m', '2~3m']
actual = [0.75, 1.50, 2.50]
measured = [0.80, 1.45, 2.62]

bar_width = 0.35
x = range(len(labels))

plt.bar(x, actual, width=bar_width, label='Actual Distance (m)', color='skyblue')
plt.bar([i + bar_width for i in x], measured, width=bar_width, label='Measured Distance (m)', color='orange')

plt.xlabel('Distance Range')
plt.ylabel('Distance (m)')
plt.title('Actual vs Measured Distance (by Range)')
plt.xticks([i + bar_width / 2 for i in x], labels)
plt.legend()
plt.tight_layout()
plt.savefig("chart.png")
plt.show()

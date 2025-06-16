# 🤖 Board-A Arduino 프로젝트

Arduino + MQTT + OLED + Servo 제어를 포함한 스마트 로봇 프로젝트입니다.

---

## 🔧 주요 기능
- WiFi 연결 상태 OLED에 출력
- MQTT 명령으로 움직임 제어 (전진, 후진, 정지, 걷기 등)
- Servo 모터와 OLED 연동
- 상태 주기적 MQTT 전송

---

## 🧠 사용된 기술
| 항목        | 내용                      |
|-------------|---------------------------|
| MCU         | ESP32                     |
| 통신 방식   | MQTT (broker.emqx.io)     |
| 센서/모터   | OLED, 180도, 360도 서보모터 (2개씩)      |
| IDE         | Arduino IDE               |

## 🔌 핀 연결 요약

| 🧩 부품   | 📌 핀 번호 | 📝 역할 설명                     |
|-----------|------------|----------------------------------|
| 🦿 LF     | GPIO 7     | 왼쪽 바퀴 제어 (360° 서보)         |
| 🦿 RF     | GPIO 5     | 오른쪽 바퀴 제어 (360° 서보)       |
| 🦿 LL     | GPIO 6     | 왼쪽 다리 각도 조절 (180° 서보)     |
| 🦿 RL     | GPIO 4     | 오른쪽 다리 각도 조절 (180° 서보)   |
| 📟 OLED   | GPIO 14(SCL), GPIO 15(SDA) | 디스플레이 출력 (I2C) |


<details>
<summary>🔧 전체 Arduino 코드 보기 (클릭)</summary>

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <ESP32Servo.h>

// Wi-Fi 정보
const char* ssid = "Arnoldroom";
const char* password = "Fanta!1600";

// MQTT 정보
const char* mqtt_server = "broker.emqx.io";
const int mqtt_port = 1883;
const char* intopic = "i2r/warnold2114@gmail.com/in";
const char* sensortopic = "i2r/sensor";
const char* outtopic = "i2r/warnold2114@gmail.com/out";

WiFiClient espClient;
PubSubClient client(espClient);

// 서보모터 핀 설정
Servo servoLL, servoRL, servoLF, servoRF;
const int pinLL = 6, pinRL = 4, pinLF = 7, pinRF = 5;

bool autoMode = false;
float lastTemp = 0, lastHum = 0;
int lastLight = 0;

unsigned long lastStatusTime = 0;

void walkMotion(int steps = 2) {
  for (int i = 0; i < steps; i++) {
    servoLL.write(60); servoRL.write(100);
    delay(300); servoLL.write(90); servoRL.write(90); delay(200);
    servoRL.write(120); servoLL.write(80);
    delay(300); servoRL.write(90); servoLL.write(90); delay(200);
  }
}

void executeCommand(int order) {
  switch (order) {
    case 1: // 전진
      servoLL.write(165); servoRL.write(15);
      delay(400); servoLF.write(120); servoRF.write(65);
      break;
    case 2: // 후진
      servoLL.write(165); servoRL.write(15);
      delay(400); servoLF.write(65); servoRF.write(120);
      break;
    case 0: // 정지
      servoLL.write(90); servoRL.write(90); servoLF.write(90); servoRF.write(90);
      break;
    case 3: // 춤
      servoLL.write(30); delay(400); servoLL.write(90); delay(200);
      servoRL.write(150); delay(400); servoRL.write(90); delay(200);
      break;
    case 6: // 걷기
      walkMotion(2);
      break;
  }
}

void callback(char* topic, byte* payload, unsigned int length) {
  StaticJsonDocument<256> doc;
  DeserializationError err = deserializeJson(doc, payload, length);
  if (err) return;

  if (String(topic) == intopic) {
    if (doc.containsKey("mode")) {
      autoMode = (String(doc["mode"]) == "auto");
    }
    if (!autoMode && doc.containsKey("order")) {
      int order = doc["order"];
      executeCommand(order);
    }
  }

  else if (String(topic) == sensortopic) {
    if (!autoMode) return;
    lastTemp = doc["temp"];
    lastHum = doc["hum"];
    lastLight = doc["light"];

    if (lastLight > 3000) executeCommand(3);
    else if (lastTemp > 30) executeCommand(1);
    else if (lastTemp < 20) executeCommand(2);
    else if (lastLight < 1000) executeCommand(6);
    else executeCommand(0);
  }
}

void reconnect() {
  while (!client.connected()) {
    String clientId = "OttoBoardA-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      client.subscribe(intopic);
      client.subscribe(sensortopic);
    } else {
      delay(3000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); }

  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);

  servoLL.attach(pinLL); servoLL.write(90);
  servoRL.attach(pinRL); servoRL.write(90);
  servoLF.attach(pinLF); servoLF.write(90);
  servoRF.attach(pinRF); servoRF.write(90);
}

void loop() {
  if (!client.connected()) {
    Serial.println("🔌 MQTT 재연결 시도...");
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastStatusTime > 5000) {
    StaticJsonDocument<100> statusDoc;
    statusDoc["status"] = "connected";
    statusDoc["id"] = "boardA";
    char buffer[128];
    serializeJson(statusDoc, buffer);
    client.publish(outtopic, buffer);
    Serial.print("📤 상태 전송: "); Serial.println(buffer);
    lastStatusTime = now;
    }
  }


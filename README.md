
---

## Board-A code

`README.md`에 아래처럼 그대로 붙여넣으세요:

<details>
<summary>🔧 전체 Arduino 코드 보기 (클릭)</summary>

```cpp
#include <Wire.h>
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <ESP32Servo.h>
#include <U8g2lib.h>

// OLED 디스플레이 설정
U8G2_SH1106_128X64_NONAME_F_HW_I2C oled(U8G2_R0, U8X8_PIN_NONE);

// Wi-Fi 정보
const char* ssid = "Arnoldroom";
const char* password = "Fanta!1600";

// MQTT 브로커 정보
const char* mqtt_server = "broker.emqx.io";
const int mqtt_port = 1883;
const char* intopic = "i2r/warnold2114@gmail.com/in";
const char* outtopic = "i2r/warnold2114@gmail.com/out";
const char* statustopic = "i2r/status";

WiFiClient espClient;
PubSubClient client(espClient);

// 서보모터 세팅
Servo servoLL, servoRL, servoLF, servoRF;
const int pinLL = 6, pinRL = 4, pinLF = 7, pinRF = 5;

// OLED 상태 출력 함수
void showStatus(const char* text) {
  oled.clearBuffer();
  oled.setFont(u8g2_font_ncenB08_tr);
  oled.drawStr(0, 32, text);
  oled.sendBuffer();
}

// 걷기 모션 함수
void walkMotion(int steps = 2) {
  for (int i = 0; i < steps; i++) {
    servoLL.write(60); servoRL.write(100); delay(300);
    servoLL.write(90); servoRL.write(90); delay(200);
    servoRL.write(120); servoLL.write(80); delay(300);
    servoRL.write(90); servoLL.write(90); delay(200);
  }
}

// MQTT 메시지 수신 처리
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("📩 수신 메시지: ");
  String message;
  for (unsigned int i = 0; i < length; i++) message += (char)payload[i];
  Serial.println(message);

  StaticJsonDocument<200> doc;
  DeserializationError error = deserializeJson(doc, payload, length);
  if (error) {
    Serial.print("❌ JSON 파싱 오류: ");
    Serial.println(error.c_str());
    return;
  }

  int order = doc["order"];
  Serial.print("🛠 명령 코드: ");
  Serial.println(order);

  switch (order) {
    case 1: showStatus("Moving Forward...");
            servoLL.write(165); servoRL.write(15); delay(400);
            servoLF.write(120); servoRF.write(65); break;
    case 2: showStatus("Reversing...");
            servoLL.write(165); servoRL.write(15); delay(400);
            servoLF.write(65); servoRF.write(120); break;
    case 0: showStatus("Stopped");
            servoLL.write(90); servoRL.write(90);
            servoLF.write(90); servoRF.write(90); break;
    case 3: showStatus("Dancing!");
            servoLL.write(30); delay(400); servoLL.write(90); delay(200);
            servoRL.write(150); delay(400); servoRL.write(90); delay(200); break;
    case 4: showStatus("LL Test");
            servoLL.write(30); delay(500); servoLL.write(90); break;
    case 5: showStatus("RL Test");
            servoRL.write(150); delay(500); servoRL.write(90); break;
    case 6: showStatus("Walking...");
            walkMotion(2); showStatus("Done"); break;
    default: showStatus("Unknown Cmd"); break;
  }
}

// MQTT 재연결
void reconnect() {
  while (!client.connected()) {
    Serial.print("🔌 MQTT 연결 시도...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("✅ MQTT 연결 성공");
      client.subscribe(intopic);
    } else {
      Serial.print("❌ 실패, 상태: ");
      Serial.print(client.state());
      Serial.println(" → 5초 후 재시도");
      delay(5000);
    }
  }
}

// MQTT 전송 함수
void sendOrder(int order) {
  StaticJsonDocument<100> doc;
  doc["order"] = order;
  char buffer[128];
  size_t n = serializeJson(doc, buffer);
  client.publish(outtopic, buffer, n);
  Serial.printf("📤 명령 전송: %s\n", buffer);
}

// 설정
void setup() {
  Serial.begin(115200);
  Wire.begin(15, 14); // OLED I2C 핀
  oled.begin();
  showStatus("Robot Active...");

  servoLL.attach(pinLL); servoLL.write(90);
  servoRL.attach(pinRL); servoRL.write(90);
  servoLF.attach(pinLF); servoLF.write(90);
  servoRF.attach(pinRF); servoRF.write(90);

  WiFi.begin(ssid, password);
  Serial.print("📶 WiFi 연결 중");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500); Serial.print(".");
  }
  Serial.println("\n✅ WiFi 연결 완료: " + WiFi.localIP().toString());

  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);
  client.setKeepAlive(60);
}

// 반복
unsigned long lastStatusTime = 0;

void loop() {
  if (!client.connected()) reconnect();
  client.loop();

  unsigned long now = millis();
  if (now - lastStatusTime > 5000) {
    StaticJsonDocument<100> statusDoc;
    statusDoc["status"] = "connected";
    statusDoc["id"] = "boardA";
    char buffer[128];
    serializeJson(statusDoc, buffer);
    Serial.print("📤 상태 전송: ");
    Serial.println(buffer);
    client.publish(statustopic, buffer);
    lastStatusTime = now;
  }
}

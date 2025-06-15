# Board-A Arduino 프로젝트

```cpp
// 예시: main loop
void setup() {
  Serial.begin(9600);
}

void loop() {
  Serial.println("Hello from Board-A");
  delay(1000);
}

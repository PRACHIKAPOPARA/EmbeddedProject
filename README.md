# EmbeddedProject
Smart Kitchen Arduino 
#include <DHT.h>
#include <SoftwareSerial.h>

// DHT setup
#define DHTPIN 6
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// Bluetooth on pins 12 (RX), 13 (TX)
SoftwareSerial BT(12, 13);

// Sensor pins
const int pirPin = 2;
const int flamePin = 3;
const int gasPin = A0;
const int buzzerPin = 4;
const int ledPin = 5;

// Threshold values
int gasThreshold = 400;
float tempThreshold = 35.0;

void sendToAll(String msg) {
  Serial.println(msg);
  BT.println(msg);
}

void setup() {
  pinMode(pirPin, INPUT);
  pinMode(flamePin, INPUT);
  pinMode(gasPin, INPUT);
  pinMode(buzzerPin, OUTPUT);
  pinMode(ledPin, OUTPUT);

  Serial.begin(9600);
  BT.begin(9600);
  dht.begin();

  sendToAll("Enter gas threshold (0-1023): ");
  while (Serial.available() == 0 && BT.available() == 0) {}

  if (Serial.available()) gasThreshold = Serial.parseInt();
  else if (BT.available()) gasThreshold = BT.parseInt();

  sendToAll("Gas threshold set to: " + String(gasThreshold));
  delay(1000);

  sendToAll("Enter temperature threshold (in Celsius): ");
  while (Serial.available() == 0 && BT.available() == 0) {}

  if (Serial.available()) tempThreshold = Serial.parseFloat();
  else if (BT.available()) tempThreshold = Serial.parseFloat();

  sendToAll("Temperature threshold set to: " + String(tempThreshold));
  delay(1000);

  sendToAll("Thresholds set. Starting sensor monitoring...");
}

void loop() {
  int pirState = digitalRead(pirPin);
  int flameState = digitalRead(flamePin);
  int gasLevel = analogRead(gasPin);
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();

  // Flame detection
  if (flameState == LOW) {
    sendToAll("Flame detected!");
    digitalWrite(buzzerPin, HIGH);
  } else {
    sendToAll("No flame detected.");
  }

  // Gas detection
  bool gasDanger = (gasLevel > gasThreshold);
  if (gasDanger) {
    sendToAll("Gas level high!");
    for (int i = 0; i < 3; i++) {
      digitalWrite(buzzerPin, HIGH);
      delay(200);
      digitalWrite(buzzerPin, LOW);
      delay(200);
    }
  }

  // Temperature detection
  bool tempDanger = (temperature > tempThreshold);
  if (tempDanger) {
    sendToAll("Temperature high!");
    for (int i = 0; i < 3; i++) {
      digitalWrite(buzzerPin, HIGH);
      delay(200);
      digitalWrite(buzzerPin, LOW);
      delay(200);
    }
  }

  // Motion detection with LED and messages
  if (pirState == HIGH) {
    digitalWrite(ledPin, HIGH);
    sendToAll("Motion detected: LED ON");
  } else {
    digitalWrite(ledPin, LOW);
    sendToAll("No motion: LED OFF");
  }

  // Turn off buzzer if everything is safe
  if (!gasDanger && !tempDanger && flameState == HIGH) {
    digitalWrite(buzzerPin, LOW);
  }

  // Temperature and humidity reading
  if (!isnan(temperature) && !isnan(humidity)) {
    String reading = "Temp: " + String(temperature) + " °C, Hum: " + String(humidity) + " %";
    sendToAll(reading);
  }

  delay(2000);
}

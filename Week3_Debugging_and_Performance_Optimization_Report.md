
#include <Arduino.h>
#include <Wire.h>
#include <SPI.h>
#include <LiquidCrystal_I2C.h>
#include <MFRC522.h>
#include <BluetoothSerial.h>
#include "HX711.h"

// LCD Configuration
LiquidCrystal_I2C lcd(0x27, 16, 2);

// RFID Configuration
#define SS_PIN 21
#define RST_PIN 22
MFRC522 rfid(SS_PIN, RST_PIN);

// Bluetooth
BluetoothSerial SerialBT;

// Load Cell
#define DT 4
#define SCK 5
HX711 scale;

// Buzzer
#define BUZZER 13

// Motor Driver
#define IN1 15
#define IN2 33
#define IN3 27
#define IN4 32

int totalBill = 0;

unsigned long lastCommandTime = 0;
unsigned long lastRFIDTime = 0;

const unsigned long motorTimeout = 2000;
const unsigned long scanDelay = 1500;

String lastUID = "";

// Stop Motors
void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

// Forward Movement
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

// Backward Movement
void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

// LCD Update
void updateLCD() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Smart Trolley");

  lcd.setCursor(0, 1);
  lcd.print("Bill: Rs.");
  lcd.print(totalBill);
}

// RFID Debugging
void checkRFID() {
  if (!rfid.PICC_IsNewCardPresent()) return;
  if (!rfid.PICC_ReadCardSerial()) return;

  String uid = "";

  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) uid += "0";
    uid += String(rfid.uid.uidByte[i], HEX);
  }

  uid.toUpperCase();

  if (uid != lastUID || millis() - lastRFIDTime >= scanDelay) {
    totalBill += 50;

    lastUID = uid;
    lastRFIDTime = millis();

    Serial.println("RFID Product Detected");
    Serial.print("UID: ");
    Serial.println(uid);

    Serial.print("Updated Bill: ");
    Serial.println(totalBill);

    tone(BUZZER, 1000, 150);
    updateLCD();
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}

// Bluetooth Control
void checkBluetooth() {
  if (SerialBT.available()) {
    char command = SerialBT.read();

    if (command == 'F') {
      moveForward();
      lastCommandTime = millis();
      Serial.println("Moving Forward");
    }
    else if (command == 'B') {
      moveBackward();
      lastCommandTime = millis();
      Serial.println("Moving Backward");
    }
    else if (command == 'S') {
      stopMotors();
      Serial.println("Motor Stopped");
    }
  }
}

// Motor Safety Optimization
void motorSafety() {
  if (millis() - lastCommandTime >= motorTimeout) {
    stopMotors();
  }
}

// Load Cell Debugging
void checkWeight() {
  if (scale.is_ready()) {
    float weight = scale.get_units(1);

    Serial.print("Weight: ");
    Serial.println(weight);
  }
}

// Setup
void setup() {
  Serial.begin(115200);

  pinMode(BUZZER, OUTPUT);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  stopMotors();

  Wire.begin(25, 26);

  lcd.init();
  lcd.backlight();

  SPI.begin();
  rfid.PCD_Init();

  SerialBT.begin("Smart_Trolley");

  scale.begin(DT, SCK);

  lcd.setCursor(0, 0);
  lcd.print("System Starting");

  Serial.println("Debugging Started");
  Serial.println("ESP32 Initialization Complete");

  updateLCD();
}

// Main Loop
void loop() {
  checkRFID();
  checkBluetooth();
  motorSafety();
  checkWeight();
}

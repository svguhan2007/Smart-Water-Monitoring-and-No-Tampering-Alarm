# Smart-Water-Monitoring-and-No-Tampering-Alarm
An Arduino-based security system that detects unauthorized movement near a water tank using a PIR sensor. It monitors water level using an ultrasonic sensor and measures pH and turbidity sensor outputs. A buzzer alerts users to detected motion, while an LCD displays real-time monitoring information.

#include <LiquidCrystal.h>

// LCD: RS, E, D4, D5, D6, D7
LiquidCrystal lcd(8, 9, 10, 11, 12, 13);

// Pin connections
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;
const int PIR_PIN = 4;
const int BUZZER_PIN = 7;
const int PH_PIN = A0;
const int TURBIDITY_PIN = A1;

// Tank dimensions
const float TANK_HEIGHT_CM = 25.0;

// pH calibration values - calibrate for your own module
// Replace these after testing with pH buffer solutions.
const float PH7_VOLTAGE = 2.50;
const float PH4_VOLTAGE = 3.00;

// Alarm settings
const unsigned long MOTION_CONFIRM_MS = 1500;
const unsigned long LCD_CHANGE_MS = 3000;

unsigned long motionStart = 0;
unsigned long lastLCDChange = 0;
bool alarmActive = false;
bool secondScreen = false;

float readVoltage(int pin) {
  long total = 0;

  for (int i = 0; i < 10; i++) {
    total += analogRead(pin);
    delay(5);
  }

  float average = total / 10.0;
  return average * (5.0 / 1023.0);
}

float readPH() {
  float voltage = readVoltage(PH_PIN);

  // Two-point calibration approximation
  float slope = (7.0 - 4.0) /
                (PH7_VOLTAGE - PH4_VOLTAGE);

  return 7.0 + slope * (voltage - PH7_VOLTAGE);
}

float readTurbidityVoltage() {
  return readVoltage(TURBIDITY_PIN);
}

float readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(3);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  unsigned long duration =
      pulseIn(ECHO_PIN, HIGH, 30000UL);

  if (duration == 0) return -1;

  return duration * 0.0343 / 2.0;
}

int readWaterLevel(float distance) {
  if (distance < 0) return -1;

  float level =
      ((TANK_HEIGHT_CM - distance) / TANK_HEIGHT_CM) * 100.0;

  return constrain((int)(level + 0.5), 0, 100);
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(9600);
  lcd.begin(16, 2);

  lcd.setCursor(0, 0);
  lcd.print("WATER MONITOR");
  lcd.setCursor(0, 1);
  lcd.print("Starting...");
  delay(2000);
  lcd.clear();
}

void loop() {
  float ph = readPH();
  float turbidityV = readTurbidityVoltage();
  float distance = readDistanceCM();
  int level = readWaterLevel(distance);

  // PIR motion confirmation
  bool motion = digitalRead(PIR_PIN) == HIGH;

  if (motion) {
    if (motionStart == 0) {
      motionStart = millis();
    }

    if (millis() - motionStart >= MOTION_CONFIRM_MS) {
      alarmActive = true;
    }
  } else {
    motionStart = 0;
    alarmActive = false;
  }

  // Buzzer sounds while confirmed motion is detected
  digitalWrite(BUZZER_PIN, alarmActive ? HIGH : LOW);

  // Alternate LCD screens
  if (millis() - lastLCDChange >= LCD_CHANGE_MS) {
    lastLCDChange = millis();
    secondScreen = !secondScreen;
    lcd.clear();
  }

  if (secondScreen) {
    lcd.setCursor(0, 0);
    lcd.print("pH:");
    lcd.print(ph, 1);
    lcd.print(" TV:");
    lcd.print(turbidityV, 1);

    lcd.setCursor(0, 1);
    lcd.print("LEVEL:");

    if (level >= 0) {
      lcd.print(level);
      lcd.print("%");
    } else {
      lcd.print("Sensor Error");
    }
  } else {
    lcd.setCursor(0, 0);

    if (alarmActive) {
      lcd.print("TAMPER ALERT!");
    } else {
      lcd.print("No Motion");
    }

    lcd.setCursor(0, 1);
    lcd.print("Tank Monitoring");
  }

  // Serial Monitor readings
  Serial.print("pH: ");
  Serial.print(ph, 2);
  Serial.print(" | Turbidity V: ");
  Serial.print(turbidityV, 2);
  Serial.print(" | Level: ");
  Serial.print(level);
  Serial.print("% | Motion: ");
  Serial.println(motion ? "YES" : "NO");

  delay(100);
}

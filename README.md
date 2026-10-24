/*
  ==========================================================
       ESP32 RADIATION & SPACE-WEATHER MONITOR
       Integrated Firmware - BMP280 Version
  ==========================================================

  HARDWARE:
    ESP32
    BMP280 pressure and temperature sensor
    AS358 photodiode amplifier circuit
    AZ393 comparator
    Red, Green and Blue LEDs
    Active buzzer module with BC547B driver

  PIN CONFIGURATION:
    BMP280 SDA       -> GPIO 21
    BMP280 SCL       -> GPIO 22
    Comparator OUT   -> GPIO 27
    Red LED          -> GPIO 26
    Green LED        -> GPIO 14
    Blue LED         -> GPIO 12
    Buzzer driver    -> GPIO 25

  BMP280 I2C ADDRESS: 0x76
  SERIAL BAUD RATE:   115200

  IMPORTANT:
  This experimental photodiode circuit is not a validated
  ionising-radiation detector and cannot measure radiation
  dose. Comparator alerts indicate an electrical threshold.
*/

// ==================== LIBRARIES ====================

#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BMP280.h>

// ==================== PIN DEFINITIONS ====================

constexpr uint8_t SDA_PIN = 21;
constexpr uint8_t SCL_PIN = 22;

constexpr uint8_t COMPARATOR_PIN = 27;

constexpr uint8_t RED_LED_PIN   = 26;
constexpr uint8_t GREEN_LED_PIN = 14;
constexpr uint8_t BLUE_LED_PIN  = 12;

constexpr uint8_t BUZZER_PIN = 25;

// ==================== CONFIGURATION ====================

constexpr uint8_t BMP_ADDRESS = 0x76;

// AZ393 open-collector output with pull-up.
// If alert behaviour is reversed, verify the comparator
// wiring and threshold before changing this setting.
constexpr uint8_t ALERT_ACTIVE_LEVEL = LOW;

constexpr unsigned long SENSOR_INTERVAL_MS = 2000;
constexpr unsigned long LED_INTERVAL_MS = 500;

// ==================== OBJECTS ====================

Adafruit_BMP280 bmp;

// ==================== STATE VARIABLES ====================

bool bmpAvailable = false;
bool comparatorAlert = false;
bool ledPhase = false;

unsigned long lastSensorRead = 0;
unsigned long lastLedChange = 0;

// ==================== LED FUNCTIONS ====================

void allLedsOff() {
  digitalWrite(RED_LED_PIN, LOW);
  digitalWrite(GREEN_LED_PIN, LOW);
  digitalWrite(BLUE_LED_PIN, LOW);
}

void normalIndicators() {
  digitalWrite(RED_LED_PIN, LOW);

  digitalWrite(BLUE_LED_PIN, ledPhase ? HIGH : LOW);
  digitalWrite(GREEN_LED_PIN, ledPhase ? LOW : HIGH);
}

void alertIndicators() {
  digitalWrite(RED_LED_PIN, HIGH);
  digitalWrite(GREEN_LED_PIN, LOW);
  digitalWrite(BLUE_LED_PIN, LOW);
}

// ==================== STARTUP TEST ====================

void testIndicators() {
  allLedsOff();

  // Test red LED
  digitalWrite(RED_LED_PIN, HIGH);
  delay(300);
  digitalWrite(RED_LED_PIN, LOW);

  // Test green LED
  digitalWrite(GREEN_LED_PIN, HIGH);
  delay(300);
  digitalWrite(GREEN_LED_PIN, LOW);

  // Test blue LED
  digitalWrite(BLUE_LED_PIN, HIGH);
  delay(300);
  digitalWrite(BLUE_LED_PIN, LOW);

  // Brief buzzer test
  digitalWrite(BUZZER_PIN, HIGH);
  delay(150);
  digitalWrite(BUZZER_PIN, LOW);

  allLedsOff();
}

// ==================== BMP280 INITIALIZATION ====================

bool initializeBMP280() {
  if (bmp.begin(BMP_ADDRESS)) {
    bmp.setSampling(
      Adafruit_BMP280::MODE_NORMAL,
      Adafruit_BMP280::SAMPLING_X2,
      Adafruit_BMP280::SAMPLING_X16,
      Adafruit_BMP280::FILTER_X16,
      Adafruit_BMP280::STANDBY_MS_500
    );

    return true;
  }

  return false;
}

// ==================== SENSOR READING ====================

void readBMP280() {
  float temperature = bmp.readTemperature();
  float pressure = bmp.readPressure() / 100.0F;

  if (isnan(temperature) || isnan(pressure) ||
      pressure <= 0) {
    // Prefix with # so it is not treated as a data series.
    Serial.println("# ERROR: Invalid BMP280 reading");
    return;
  }

  /*
    Keep one data line per sample for Serial Plotter.
    Temperature is in degrees Celsius.
    Pressure is in hPa.
    Comparator is 0 = normal, 1 = alert.
  */

  Serial.print("Temperature_C:");
  Serial.print(temperature, 2);

  Serial.print("\tPressure_hPa:");
  Serial.print(pressure, 2);

  Serial.print("\tComparator:");
  Serial.println(comparatorAlert ? 1 : 0);
}

// ==================== SETUP ====================

void setup() {
  Serial.begin(115200);
  delay(500);

  pinMode(COMPARATOR_PIN, INPUT_PULLUP);

  pinMode(RED_LED_PIN, OUTPUT);
  pinMode(GREEN_LED_PIN, OUTPUT);
  pinMode(BLUE_LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  allLedsOff();
  digitalWrite(BUZZER_PIN, LOW);

  // Check LEDs and buzzer before starting sensor monitoring.
  testIndicators();

  // Initialize I2C bus.
  Wire.begin(SDA_PIN, SCL_PIN);

  Serial.println();
  Serial.println("# ESP32 Space-Weather Monitor starting");
  Serial.println("# Checking BMP280 at address 0x76");

  bmpAvailable = initializeBMP280();

  if (bmpAvailable) {
    Serial.println("# BMP280 initialized successfully");
  } else {
    Serial.println("# ERROR: BMP280 not detected at 0x76");
    Serial.println("# Check sensor type and wiring");
  }

  Serial.println("# Startup complete");
}

// ==================== MAIN LOOP ====================

void loop() {
  const unsigned long now = millis();

  // 1. Read comparator state.
  comparatorAlert =
      (digitalRead(COMPARATOR_PIN) == ALERT_ACTIVE_LEVEL);

  // 2. Update LEDs and buzzer.
  if (comparatorAlert) {
    alertIndicators();
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    digitalWrite(BUZZER_PIN, LOW);

    // Alternate green and blue during normal monitoring.
    if (now - lastLedChange >= LED_INTERVAL_MS) {
      lastLedChange = now;
      ledPhase = !ledPhase;
    }

    normalIndicators();
  }

  // 3. Read BMP280 every two seconds.
  if (bmpAvailable &&
      now - lastSensorRead >= SENSOR_INTERVAL_MS) {
    lastSensorRead = now;
    readBMP280();
  }
}
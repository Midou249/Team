#include <Wire.h>
#include "MAX30100_PulseOximeter.h"
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <SoftwareSerial.h>
#include <TinyGPS++.h>
#include <MPU6050.h>

#define SCREEN_WIDTH 128 // OLED display width, in pixels
#define SCREEN_HEIGHT 64 // OLED display height, in pixels
#define OLED_RESET     -1 // Reset pin # (or -1 if sharing Arduino reset pin)
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

#define ONE_WIRE_BUS 2
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

PulseOximeter pox;

SoftwareSerial BTserial(10, 11); // RX | TX
TinyGPSPlus gps;
MPU6050 mpu;
float previous_distance = 0;
float current_distance = 0;
float current_speed = 0;

void onBeatDetected()
{
  Serial.println("Heartbeat!");
}

void setup()
{
  Serial.begin(9600);
  BTserial.begin(9600);

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }

  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(1);
  display.setCursor(0,0);
  display.println("Heart Rate: ");
  display.println("Temperature: ");
  display.println("Distance: ");
  display.println("Speed: ");
  display.display();

  sensors.begin();

  if (!pox.begin()) {
    Serial.println("MAX30100 not found!");
    while (1);
  }

  pox.setOnBeatDetectedCallback(onBeatDetected);

  if (!gps.begin(9600)) {
    Serial.println("GPS not found!");
    while (1);
  }

  if (!mpu.begin(MPU6050_SCALE_2000DPS, MPU6050_RANGE_2G)) {
    Serial.println("MPU6050 not found!");
    while (1);
  }
  mpu.setThreshold(3);
}

void loop()
{
  pox.update();

  if (millis() - lastTempUpdate >= 1000) {
    sensors.requestTemperatures();
    float temperatureC = sensors.getTempCByIndex(0);
    float temperatureF = DallasTemperature::toFahrenheit(temperatureC);
    Serial.print("Temperature: ");
    Serial.print(temperatureF);
    Serial.println(" F");
    display.setCursor(70,0);
    display.println(pox.getHeartRate());
    display.setCursor(70,10);
    display.println(temperatureF);
    display.display();
    lastTempUpdate = millis();

    // Send temperature data to Bluetooth
    BTserial.print("T:");
    BTserial.println(temperatureF);
  }

  if (pox.getHeartRate() > 0) {
    Serial.print("Heart Rate: ");
    Serial.print(pox.getHeartRate());
    Serial.println(" bpm");
    display.setCursor(70,0);
    display.println(pox.getHeartRate

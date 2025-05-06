#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "DHTesp.h"

#define DHT_PIN 15
#define LED_MERAH 2
#define LED_HIJAU 27
#define BUZZER_PIN 14  // Pin untuk buzzer


const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* webhookUrl = "https://devispringga.sman20garut.com/iot/webhook.php";

DHTesp dht;
LiquidCrystal_I2C lcd(0x27, 16, 2); // I2C LCD address 0x27
bool sudahKirim = false;

void setup() {
  Serial.begin(115200);
  dht.setup(DHT_PIN, DHTesp::DHT22);

  lcd.init();
  lcd.backlight();

  pinMode(LED_MERAH, OUTPUT);
  pinMode(LED_HIJAU, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT); // Inisialisasi pin buzzer

  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println(" Connected!");
}

void loop() {
  TempAndHumidity data = dht.getTempAndHumidity();
  float suhu = data.temperature;
  float hum = data.humidity;

  if (isnan(suhu)) {
    Serial.println("Gagal membaca suhu!");
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Sensor error");
    return;
  }

  lcd.setCursor(0, 0);
  lcd.print("Suhu: ");
  lcd.print(suhu, 1);
  lcd.print(" C   ");
  lcd.setCursor(0, 1);
  lcd.print("Hum : ");
  lcd.print(hum, 1);
  lcd.print(" %   ");

  if (suhu > 8) {
    digitalWrite(LED_MERAH, HIGH);
    digitalWrite(LED_HIJAU, LOW);

    // Nyalakan buzzer
    digitalWrite(BUZZER_PIN, HIGH);
    delay(500);  // Buzzer menyala selama 500ms
    digitalWrite(BUZZER_PIN, LOW);
    delay(500);  // Buzzer mati selama 500ms

    if (!sudahKirim) {
      Serial.println("Mengirim data ke webhook...");
      HTTPClient http;
      String url = String(webhookUrl) + "?suhu=" + String(suhu);
      http.begin(url);
      int response = http.GET();

      if (response > 0) {
        Serial.println("✅ Webhook sukses: " + String(response));
      } else {
        Serial.println("❌ Gagal kirim webhook: " + http.errorToString(response));
      }

      http.end();
      sudahKirim = true;
    }
  } else {
    digitalWrite(LED_MERAH, LOW);
    digitalWrite(LED_HIJAU, HIGH);
    sudahKirim = false;
  }

  delay(10000);
}

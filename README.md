#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <SoftwareSerial.h>
#include <TinyGPS++.h>

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 32 

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// GPS Connections
static const int RXPin = 4, TXPin = 3;
static const uint32_t GPSBaud = 9600;

TinyGPSPlus gps;
SoftwareSerial ss(RXPin, TXPin);

unsigned long lastScreenSwitch = 0;
int screenMode = 0; // 0: Lat, 1: Lng, 2: Status, 3: Time, 4: Date

void setup() {
  Serial.begin(115200); 
  ss.begin(GPSBaud);

  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { 
    Serial.println(F("SSD1306 allocation failed"));
    for(;;);
  }
  
  display.clearDisplay();
  display.setTextSize(2); 
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 8);
  display.println("STARTING");
  display.display();
  delay(2000);
}

void loop() {
  // Free loop bina delay ke taaki GPS data aur seconds lagatar update hon
  while (ss.available() > 0) {
    gps.encode(ss.read());
  }

  // Har 10 seconds mein screen cycle hogi
  if (millis() - lastScreenSwitch >= 10000) {
    lastScreenSwitch = millis();
    screenMode = (screenMode + 1) % 5; // Total 5 states (Lat aur Lng alag alag)
  }

  if (gps.location.isValid()) {
    displayGPSData();
  } else {
    display.clearDisplay();
    display.setTextSize(2);
    display.setCursor(20, 8);
    display.println("NO LOCK");
    display.display();
    
    Serial.print("Waiting for Lock... Sats: ");
    Serial.println(gps.satellites.value());
  }
}

void displayGPSData() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  if (screenMode == 0) {
    // ---- 1. LATITUDE (FULL BIG SIZE & NO CUTTING) ----
    display.setTextSize(2); 
    display.setCursor(0, 8);
    display.print("LT:"); 
    display.print(gps.location.lat(), 4); // 4 decimal tak rakha hai taaki Size 2 mein screen se bahar na jaye

    // Serial Monitor par full value print hogi
    Serial.print("LIVE Latitude: "); Serial.println(gps.location.lat(), 6);

  } else if (screenMode == 1) {
    // ---- 2. LONGITUDE (FULL BIG SIZE & NO CUTTING) ----
    display.setTextSize(2); 
    display.setCursor(0, 8);
    display.print("LN:"); 
    display.print(gps.location.lng(), 4); // 4 decimal tak taaki fit aaye

    Serial.print("LIVE Longitude: "); Serial.println(gps.location.lng(), 6);

  } else if (screenMode == 2) {
    // ---- 3. SPEED & SATELLITES ----
    display.setTextSize(2); 
    display.setCursor(0, 0);
    display.print(gps.speed.kmph(), 0); display.println(" km/h");
    display.print("SATS: "); display.println(gps.satellites.value());

    Serial.print("LIVE Status -> Speed: "); Serial.print(gps.speed.kmph());
    Serial.print(" km/h | Sats: "); Serial.println(gps.satellites.value());

  } else if (screenMode == 3) {
    // ---- 4. TIME WITH FAST LIVE SECONDS (12HR AM/PM) ----
    int hour = gps.time.hour() + 5;
    int minute = gps.time.minute() + 30;
    int second = gps.time.second();

    if (minute >= 60) { minute -= 60; hour += 1; }
    if (hour >= 24) { hour -= 24; }

    String ampm = "AM";
    if (hour >= 12) {
      ampm = "PM";
      if (hour > 12) hour -= 12;
    }
    if (hour == 0) hour = 12;

    // Is design mein seconds live tick karenge aur text screen par perfectly center hoga
    display.setTextSize(2); 
    display.setCursor(0, 0);
    if (hour < 10) display.print("0");
    display.print(hour); display.print(":");
    if (minute < 10) display.print("0");
    display.print(minute); display.print(":");
    if (second < 10) display.print("0");
    display.print(second);
    
    display.setCursor(45, 18);
    display.setTextSize(1.5); // AM/PM thoda niche center mein
    display.print(ampm);

    Serial.print("LIVE Time: "); Serial.print(hour); Serial.print(":"); Serial.print(minute); Serial.print(":"); Serial.print(second); Serial.println(" " + ampm);

  } else if (screenMode == 4) {
    // ---- 5. DATE & MONTH & YEAR (POORI EK HI LINE MEIN) ----
    int day = gps.date.day();
    int month = gps.date.month();
    int year = gps.date.year();

    // 128x32 screen par DD/MM/YYYY ko ek line mein Size 2 mein rakhne ka tarika:
    display.setTextSize(2); 
    display.setCursor(0, 8); // Bilkul y-axis ke center mein
    
    if (day < 10) display.print("0");
    display.print(day); display.print("/");
    if (month < 10) display.print("0");
    display.print(month); display.print("/");
    
    // Year ko chota karne ke liye standard last 2 digits ya fir font tight space use kiya hai
    display.print(year - 2000); // 2026 banega sirf "26" jo ek hi line mein solid bada dikhega

    Serial.print("LIVE Date: "); Serial.print(day); Serial.print("/"); Serial.print(month); Serial.print("/"); Serial.println(year);
  }

  display.display();
}

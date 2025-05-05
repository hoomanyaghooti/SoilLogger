# ESP32 Soil & Temperature Logger

## 1. معرفی پروژه
این پروژه با استفاده از برد **ESP32-DevKitV1** و سنسورهای خازنی رطوبت خاک (DFRobot SEN0193) و دمای **DS18B20** پیاده‌سازی شده تا:
- داده‌ها را روی نمایشگر **OLED 0.96″** با رابط I2C نمایش دهد.
- داده‌های اندازه‌گیری‌شده را با پروتکل **HTTP/HTTPS** و **JWT** هر ساعت به سرور مرکزی ارسال کند.
- مصرف انرژی را با **Deep-Sleep** و **Wake-up on EXT0** به حداقل برساند.

## 2. محتویات پوشه
- `esp32_soil_temp_http.ino` : کد Arduino  
- `README.md`           : مستندات پروژه  
- `docs/`               : شامل شماتیک سخت‌افزار (PDF)

## 3. سخت‌افزار
| قطعه                       | مدل/نوع                  | دلیل انتخاب                              |
|----------------------------|---------------------------|-------------------------------------------|
| میکروکنترلر               | ESP32-DevKitV1            | مصرف انرژی پایین، Wi-Fi داخلی           |
| سنسور رطوبت خاک            | DFRobot SEN0193 (خازنی)   | پایداری بالا، دقت مناسب، مقاوم در برابر خوردگی |
| سنسور دما                  | DS18B20                   | دقت ±0.5°C، آب‌بندی برای نصب در خاک      |
| نمایشگر OLED              | SSD1306 0.96″ I2C         | مصرف کم، پشتیبانی کتابخانه‌ای قوی        |
| باتری                      | 18650 Li-ion              | دسترسی آسان، ظرفیت کافی                  |
| ماژول شارژ و محافظ باتری  | TP4056 + Protection PCB   | ساده و ارزان                              |

## 4. نرم‌افزار و کتابخانه‌ها
- **Arduino IDE**  
- کتابخانه‌های مورد استفاده:
  ```
  WiFi.h
  WiFiClientSecure.h
  HTTPClient.h
  ArduinoJson.h
  Adafruit_SSD1306.h
  OneWire.h
  DallasTemperature.h
  ```

## 5. پیکربندی کاربر (User Configuration)
در بالای فایل `esp32_soil_temp_http.ino` مقادیر زیر را تنظیم کنید:
```cpp
#define WIFI_SSID             "نام_شبکه"
#define WIFI_PASSWORD         "رمز_شبکه"
#define DEVICE_ID             "esp32-001"
#define SLEEP_INTERVAL_SEC    3600        // فاصله ارسال (ثانیه)
#define TIME_SYNC_MODE        2           // 0=NTP, 1=Server, 2=Hybrid
```

## 6. نصب و اجرای اولیه
1. **Arduino IDE** را اجرا کرده و برد **ESP32 Dev Module** و پورت USB را انتخاب کنید.  
2. فایل `esp32_soil_temp_http.ino` را باز و مطمئن شوید مقادیر پیکربندی را تنظیم کرده‌اید.  
3. روی **Upload** کلیک کنید تا کد به برد منتقل شود.  
4. **Serial Monitor** را با **115200 baud** باز کنید.  
5. دکمه روی برد را فشار دهید تا نمایشگر **OLED** روشن و مقادیر نمایش داده شوند.  
6. پس از **۶۰ ثانیه** نمایشگر خاموش می‌شود و دستگاه وارد **Deep-Sleep** می‌شود.  
7. هر ساعت یک بار (یا بر اساس مقدار `SLEEP_INTERVAL_SEC`) داده به سرور ارسال و پیام تأیید در سریال چاپ می‌شود.

## 7. شرح عملکرد کد
چرخه کاری دستگاه:
1. **Wake-up** بر اساس Timer یا Button (EXT0).  
2. **اتصال به Wi-Fi**.  
3. **همگام‌سازی زمان** (NTP/Server/Hybrid).  
4. **خواندن سنسورها** (`readSensors()`).  
5. **نمایش کوتاه** داده‌ها روی **OLED** (`displayReadings()`).  
6. **ارسال داده** با HTTP POST و JWT (`sendDataWithRetry()`).  
7. **Deep-Sleep** تا زمان بعدی.

## 8. جزئیات ارتباط با سرور
- **احراز هویت**:
  ```
  POST https://api.fanap-infra.com/v1/auth/login
  Body: { "username": "...", "password": "..." }
  ```
- **ارسال داده**:
  ```
  POST https://api.fanap-infra.com/v1/devices/{DEVICE_ID}/data
  Headers:
    Content-Type: application/json
    Authorization: Bearer <JWT_TOKEN>
  Body:
  {
    "timestamp": "2025-05-05T14:30:00Z",
    "soil_moisture": 45.2,
    "temperature": 22.8
  }
  ```

## 9. Retry/Backoff و اعتبارسنجی داده
- **حداکثر تلاش**: `MAX_RETRIES = 3` با تأخیر تصاعدی  
- **محدوده معتبر**:
  - رطوبت: 0–100%  
  - دما: -40–85°C  
- چنانچه خطای `5xx` یا `4xx` رخ دهد، تلاش مجدد صورت می‌گیرد.

## 10. حالت‌های همگام‌سازی زمان
- `TIME_SYNC_MODE = 0`: فقط **NTP**  
- `TIME_SYNC_MODE = 1`: صرفاً **زمان سرور**  
- `TIME_SYNC_MODE = 2`: **ترکیبی** (اول NTP، سپس سرور)

## 11. نسخه‌گذاری با Git (Git Workflow ساده)
- **راه‌اندازی مخزن**:
  ```bash
  git init
  git add .
  git commit -m "Initial project setup"
  ```
- **ساخت شاخه برای ویژگی جدید**:
  ```bash
  git checkout -b feature/button-wake
  ```
- **ثبت تغییرات**:
  ```bash
  git add .
  git commit -m "feat(button): add 60s OLED wake on button press"
  ```
- **بازگشت به main و ادغام**:
  ```bash
  git checkout main
  git merge feature/button-wake
  ```

## 12. تست و اشکال‌زدایی
- **Button Wake**: مطمئن شوید با فشردن دکمه OLED روشن شود.  
- **ارسال HTTP**: در **Serial Monitor** پیام **”Data sent successfully”** را بررسی کنید.  
- **حالت‌های زمان**: حالت‌های **NTP-only**, **Server-only** و **Hybrid** را تست کنید.

## 13. شماتیک سخت‌افزار
شماتیک و دیاگرام جریان در پوشه `docs/` موجود است.

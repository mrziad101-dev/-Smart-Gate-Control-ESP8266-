# -Smart-Gate-Control-ESP8266-
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <Servo.h>

// ===== بيانات الواي فاي =====
const char* ssid     = "AH_Family";
const char* password = "iwjfr;@ke96^z_3d";

// ===== تعريف البنات =====
#define TRIG_PIN D2
#define ECHO_PIN D1
#define SERVO_PIN D5

// ===== متغيرات عامة =====
ESP8266WebServer server(80);
Servo gateServo;

int openAngle  = 150;    // زاوية فتح السيرفو (درجة)
int closeAngle = 0;      // زاوية الغلق
int detectDist = 10;     // مسافة الفتح (سم)
int margin     = 2;      // مسافة إضافية للقفل (Hysteresis)
unsigned long holdTime = 1000; // يبقى مفتوح ثانية بعد اختفاء الجسم

bool gateOpen = false;
unsigned long lastDetected = 0;
int lastServoPos = -1;    // لحفظ آخر زاوية أرسلت للسيرفو

// ===== قراءة المسافة (مع فلترة) =====
long readDistance() {
  const int samples = 5;     // عدد العينات لمتوسط القراءة
  long readings[samples];
  for (int i = 0; i < samples; i++) {
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);

    long duration = pulseIn(ECHO_PIN, HIGH, 20000); // مهلة 20ms
    readings[i] = (duration == 0) ? 999 : duration * 0.034 / 2;
    delay(5);
  }
  long total = 0;
  for (int i = 0; i < samples; i++) total += readings[i];
  return total / samples;
}

// ===== أمر آمن للسيرفو (يكتب بس لو فيه تغيير) =====
void safeServoWrite(int pos) {
  if (pos != lastServoPos) {
    gateServo.write(pos);
    lastServoPos = pos;
  }
}

// ===== صفحة التحكم =====
void handleRoot() {
  long dist = readDistance();
  String html =
  "<!DOCTYPE html><html><head><meta charset='UTF-8'>"
  "<meta name='viewport' content='width=device-width, initial-scale=1'/>"
  "<style>"
  "body{font-family:Arial;background:#1e3c72;color:#fff;text-align:center;margin:0;padding:20px;}"
  ".box{background:#fff;color:#333;padding:20px;border-radius:15px;max-width:350px;margin:auto;box-shadow:0 0 15px #0004;}"
  "input[type=range]{width:90%;}"
  "button{background:#1e90ff;color:#fff;border:none;padding:10px 20px;border-radius:8px;font-size:16px;cursor:pointer;}"
  "</style></head><body>"
  "<div class='box'>"
  "<h2>🚧 Smart Gate Control 🚧</h2>"
  "<p><b>Current Distance:</b> " + String(dist) + " cm</p>"
  "<form action='/set' method='get'>"
  "<p>Open Angle: <span id='aVal'>" + String(openAngle) + "</span>°<br>"
  "<input type='range' name='angle' min='20' max='180' value='" + String(openAngle) + "' "
  "oninput='aVal.innerText=this.value'></p>"
  "<p>Detect Distance: <span id='dVal'>" + String(detectDist) + "</span> cm<br>"
  "<input type='range' name='dist' min='2' max='50' value='" + String(detectDist) + "' "
  "oninput='dVal.innerText=this.value'></p>"
  "<button type='submit'>Apply</button>"
  "</form></div></body></html>";
  server.send(200, "text/html", html);
}

// ===== تحديث الإعدادات =====
void handleSet() {
  if (server.hasArg("angle")) openAngle  = server.arg("angle").toInt();
  if (server.hasArg("dist"))  detectDist = server.arg("dist").toInt();
  server.sendHeader("Location", "/");
  server.send(303); // يعيد توجيه للصفحة الرئيسية
}

// ===== الإعداد =====
void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  gateServo.attach(SERVO_PIN);
  safeServoWrite(closeAngle);

  WiFi.begin(ssid, password);
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println("\nConnected! IP: " + WiFi.localIP().toString());

  server.on("/", handleRoot);
  server.on("/set", handleSet);
  server.begin();
}

// ===== الحلقة الرئيسية =====
void loop() {
  server.handleClient();
  long dist = readDistance();

  if (!gateOpen && dist <= detectDist) {
    // افتح لو لسه مقفول
    safeServoWrite(openAngle);
    gateOpen = true;
    lastDetected = millis();
  }
  else if (gateOpen) {
    // لو مفتوح بالفعل
    if (dist <= detectDist) {
      lastDetected = millis(); // لسه في جسم قدام
    }
    else if (dist > detectDist + margin && millis() - lastDetected > holdTime) {
      // يقفل بعد المهلة
      safeServoWrite(closeAngle);
      gateOpen = false;
    }
  }
}

# Scan-system-tower-ESP32S3-
#include <WiFi.h>
#include <HTTPClient.h>
#include "Audio.h"

const char* WIFI_SSID = "ABCDEF";
const char* WIFI_PASS = "123456789";
const char* WEBAPP    = "https://??????????????????????????????????????????????????///exec?action=latest";

#define I2S_BCLK 15
#define I2S_LRC  16
#define I2S_DOUT  7

Audio audio;
String lastTime = "";

String jsonField(const String& body, const char* key) {
  String pat = String("\"") + key + "\":";
  int i = body.indexOf(pat);
  if (i < 0) return "";
  i += pat.length();
  while (i < body.length() && (body[i] == ' ')) i++;
  if (body[i] == '"') {
    int j = body.indexOf('"', i + 1);
    if (j < 0) return "";
    return body.substring(i + 1, j);
  }
  int j = i;
  while (j < body.length() && body[j] != ',' && body[j] != '}') j++;
  return body.substring(i, j);
}

void setup() {
  Serial.begin(115200);
  audio.setPinout(I2S_BCLK, I2S_LRC, I2S_DOUT);
  audio.setVolume(21);

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(400);
    Serial.print(".");
  }
  Serial.println(" OK");
}

void loop() {
  audio.loop();

  static unsigned long t = 0;
  if (millis() - t < 2000) return;
  t = millis();
  if (audio.isRunning()) return;
  if (WiFi.status() != WL_CONNECTED) return;

  HTTPClient http;
  http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS);
  http.begin(WEBAPP);
  int code = http.GET();
  if (code != 200) {
    Serial.println(code);
    http.end();
    return;
  }
  String body = http.getString();
  http.end();
  Serial.println(body);

  String status = jsonField(body, "status");
  String text   = jsonField(body, "text");
  String time   = jsonField(body, "time");

  if (status == "none" || text == "" || time == "" || time == lastTime) return;
  lastTime = time;

  Serial.println(status + " " + text);
  audio.connecttospeech(text.c_str(), "vi");
}

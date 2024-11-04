# C-
![image](https://github.com/user-attachments/assets/eafeb574-f1fc-4692-891c-ac2e1a779bdc)
#include <WiFi.h>
#include <PubSubClient.h>
#include <Arduino.h>
#include <ArduinoJson.h>

const int adcPin = 34;
const int sampleCount = 100;  // 每次采?的次?
const int averageCount = 10;  // ?算平均值的次?

const char ssid[] = "Lab203";
const char pwd[] = "203203203";


const char clientID[] = "masite001";
const char *mqttServer = "192.168.0.4";
const int mqttPort = 1883;
const char *mqttUser = "masite";
const char *mqttPsw = "12345";
const char *mqttopic = "esp32";

String msgStr = "";
char json_output[100];
StaticJsonDocument<200> json_doc;
DeserializationError json_error;

WiFiClient espClient;
PubSubClient client(espClient);

void callback(char *topic, byte *payload, unsigned int length) {
  Serial.print("Topic: ");
  Serial.print(topic);
  Serial.print("  ,Info: ");

  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
  Serial.println("-------------");
}

void setup() {
  // put your setup code here, to run once:
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, pwd);

  Serial.print("WiFi connecting");

  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }

  Serial.println("");
  Serial.print("IP位址:");
  Serial.print(WiFi.localIP());  //讀取IP位址
  Serial.print("  ,WiFi RSSI:");
  Serial.println(WiFi.RSSI());  //讀取WiFi強度
  Serial.println(ssid);

  client.setServer(mqttServer, mqttPort);
  Serial.print("MQTT connecting...");
  client.setCallback(callback);

  while (!client.connected()) {
    Serial.print(".");

    if (client.connect("ESP32Masite", mqttUser, mqttPsw)) {
      Serial.println("Connection successful!");
    } else {
      Serial.println("Connection fail!");
      Serial.println(client.state());
      delay(2000);
    }
  }
  
  
  client.subscribe(mqttopic);
  Serial.println("Waiting info from MQTT server...");
  //client.publish(mqttopic, "Pub HelloWorld From ESP32");
  analogReadResolution(12);
}

void loop() {
  // put your main code here, to run repeatedly:
  
  
  uint32_t sum = 0;
  
  // 多次采?以?少噪?
  for (int i = 0; i < averageCount; i++) {
    sum += averageADC(sampleCount);
  }
  
  uint32_t averageValue = sum / averageCount;
  //String currentTime = getCurrentTime();
  String currentTime = "";
  json_doc["datatime"] = currentTime;
  json_doc["esp32"] = averageValue;
 
  serializeJson(json_doc, json_output);
  client.loop();
  client.publish(mqttopic, json_output);
  Serial.println(json_output);
  Serial.println(averageValue);
  delay(100);
}

uint32_t averageADC(int count) {
  uint32_t sum = 0;
  
  for (int i = 0; i < count; i++) {
    sum += analogRead(adcPin);
    delay(0.1);
  }
  
  return sum / count;
}

String getCurrentTime(){
 struct tm timeinfo;
 
 if(!getLocalTime(&timeinfo)){
  Serial.println("Failed to obtain time");
  return "";
 }
 char curTime[25];
 strftime(curTime, sizeof(curTime), "%Y %m %d %H:%M:%S", &timeinfo);
 String buffer( curTime );
 return buffer;
}

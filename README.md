# Trabajo-final---Aplicacion-para-Mantenimiento-de-Generadores-
El proyecto presentado es una aplicación digital para realizar actividades de mantenimiento en un generador de una central hidroeléctrica. 
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>
// Pines
#define DHTPIN 15
#define DHTTYPE DHT22
#define LED_ROJO 26
#define LED_VERDE 25
#define VENTILADOR 14
#define BUZZER 27
// Sensor DHT
DHT dht(DHTPIN, DHTTYPE);

// WiFi
const char* ssid = "Wokwi-GUEST";
const char* password = "";

// MQTT
const char* mqtt_server = "test.mosquitto.org";

WiFiClient espClient;
PubSubClient client(espClient);

const char* topic_paro = "ja/paro";
const char* topic_temp = "ja/temp";
const char* topic_hum = "ja/hum";
//const char* topic_led1 = "l1/temp";
//const char* topic_led2 = "l2/hum";

bool system_active = true;
unsigned long lastPublishTime = 0;
const long publishInterval = 2000;

// --- Conexión WiFi ---
void setup_wifi() {
  Serial.println("Conectando al WiFi...");
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(250);
    Serial.print(".");
  }

  Serial.println("\nWiFi conectado");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());
}

// --- Callback MQTT ---
void callback(char* topic, byte* payload, unsigned int length) {
  String msg;
  for (int i = 0; i < length; i++) {
    msg += (char)payload[i];
  }

  Serial.print("Mensaje recibido en ");
  Serial.print(topic);
  Serial.print(": ");
  Serial.println(msg);

  if (String(topic) == topic_paro) {
    if (msg == "OFF") {
      system_active = false;
      digitalWrite(VENTILADOR, LOW);
      digitalWrite(BUZZER, LOW);
      digitalWrite(LED_ROJO, LOW);
      digitalWrite(LED_VERDE, LOW);
      Serial.println("Sistema detenido");
    } 
    else if (msg == "ON") {
      system_active = true;
      Serial.println("Sistema reactivado");
    }
  }
}

// --- Reconectar MQTT ---
void reconnect() {
  while (!client.connected()) {
    Serial.print("Conectando a MQTT... ");

    String clientId = "ESP32Client-";
    clientId += String(random(0xffff), HEX);  // cliente único

    if (client.connect(clientId.c_str())) {
      Serial.println("Conectado");
      client.subscribe(topic_paro);
    } else {
      Serial.print("Fallo, rc=");
      Serial.print(client.state());
      Serial.println(" Reintentando en 3 segundos...");
      delay(3000);
    }
  }
}

// --- Setup ---
void setup() {
  Serial.begin(115200);

  pinMode(LED_ROJO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  pinMode(VENTILADOR, OUTPUT);
  pinMode(BUZZER, OUTPUT);

  dht.begin();

  setup_wifi();

  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

// --- Loop principal ---
void loop() {
  // Mantener conexión WiFi
  if (WiFi.status() != WL_CONNECTED) {
    setup_wifi();
  }

  // Mantener conexión MQTT
  if (!client.connected()) {
    reconnect();
  }

  client.loop();  // **Súper importante para que Node-RED responda rápido**

  unsigned long now = millis();

  // Publicar cada 2s
  if (now - lastPublishTime >= publishInterval) {
    lastPublishTime = now;

    float temp = dht.readTemperature();
    float hum = dht.readHumidity();

    if (!isnan(temp) &&  !isnan(hum)) {
      client.publish(topic_temp, String(temp).c_str());
      client.publish(topic_hum, String(hum).c_str());
      //client.publish(topic_led1, String(LED_ROJO).c_str());
     //client.publish(topic_led2, String(LED_VERDE).c_str());

      Serial.print("Temp: ");
      Serial.print(temp);
      Serial.print(" °C | Hum: ");
      Serial.print(hum);
      Serial.println(" %");
    }

    // Lógica del sistema
    if (system_active) {
      digitalWrite(LED_ROJO, temp > 30);
      digitalWrite(VENTILADOR, temp > 30);
      digitalWrite(BUZZER, temp > 30);
      digitalWrite(LED_VERDE, hum < 40);
    }
  }
}
////////////////////////////////////////////////////////////////////////////////////////////////////






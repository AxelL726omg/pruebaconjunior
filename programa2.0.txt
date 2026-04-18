#include <WiFi.h>
#include <WiFiManager.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <UniversalTelegramBot.h>
#include <DHT.h>

// ------------------------------
// Configuración de Telegram
// ------------------------------
#define BOT_TOKEN "TU_BOT_TOKEN"
const char* CHAT_ID = "TU_CHAT_ID";

WiFiClientSecure telegramClient;
UniversalTelegramBot bot(BOT_TOKEN, telegramClient);

// ------------------------------
// Configuración de Supabase
// ------------------------------
const char* supabaseUrl  = "https://xjmqzdnhactuschmmaaa.supabase.co";
const char* supabaseKey  = "TU_SUPABASE_ANON_KEY";
const char* supabaseTable = "tu_tabla";   // ejemplo: alertas, sensores, etc.

WiFiClientSecure supabaseClient;
HTTPClient http;

// ------------------------------
// Pines y sensores
// ------------------------------
#define PIN_LED 25
#define PIN_BUZZER 26
#define PIN_GAS 34
#define PIN_DHT 4

#define DHTTYPE DHT22
DHT dht(PIN_DHT, DHTTYPE);

// ------------------------------
// Umbrales
// ------------------------------
#define UMBRAL_GAS 300
#define UMBRAL_TEMP 5
#define UMBRAL_HUM 70

// ------------------------------
// Variables de tiempo
// ------------------------------
unsigned long tiempoSensado = 0;
unsigned long tiempoEnvio = 0;

const unsigned long intervaloSensado = 2000;
const unsigned long intervaloEnvio = 60000;

// ------------------------------
// Variables de sensores
// ------------------------------
int gas = 0;
float temperatura = 0;
float humedad = 0;

bool alertaActiva = false;

// ------------------------------
// Enviar datos a Supabase
// ------------------------------
bool enviarASupabase(int gas, float temperatura, float humedad, bool alerta) {
  if (WiFi.status() != WL_CONNECTED) return false;

  String url = String(supabaseUrl) + "/rest/v1/" + supabaseTable;

  http.begin(supabaseClient, url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("apikey", supabaseKey);
  http.addHeader("Authorization", String("Bearer ") + supabaseKey);
  http.addHeader("Prefer", "return=minimal");

  String json = "{";
  json += "\"gas\":" + String(gas) + ",";
  json += "\"temperatura\":" + String(temperatura, 1) + ",";
  json += "\"humedad\":" + String(humedad, 1) + ",";
  json += "\"alerta\":" + String(alerta ? "true" : "false");
  json += "}";

  int httpResponseCode = http.POST(json);

  Serial.print("Supabase HTTP: ");
  Serial.println(httpResponseCode);

  if (httpResponseCode > 0) {
    Serial.println(http.getString());
  }

  http.end();

  return (httpResponseCode >= 200 && httpResponseCode < 300);
}

void setup() {
  Serial.begin(115200);

  pinMode(PIN_LED, OUTPUT);
  pinMode(PIN_BUZZER, OUTPUT);
  digitalWrite(PIN_LED, LOW);
  digitalWrite(PIN_BUZZER, LOW);

  dht.begin();

  WiFiManager wifiManager;
  wifiManager.startConfigPortal("ESP32-Config");

  Serial.println("\n✅ Conectado a WiFi");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // Para HTTPS
  telegramClient.setInsecure();
  supabaseClient.setInsecure();
}

void loop() {
  unsigned long ahora = millis();

  // --------- Lectura de sensores cada 2s ---------
  if (ahora - tiempoSensado >= intervaloSensado) {
    tiempoSensado = ahora;

    gas = analogRead(PIN_GAS);
    temperatura = dht.readTemperature();
    humedad = dht.readHumidity();

    if (isnan(temperatura) || isnan(humedad)) {
      Serial.println("❌ Error leyendo el DHT22");
      return;
    }

    Serial.print("Gases: ");
    Serial.print(gas);
    Serial.print(" | Temp: ");
    Serial.print(temperatura);
    Serial.print("°C | Humedad: ");
    Serial.print(humedad);
    Serial.println("%");

    alertaActiva = (gas > UMBRAL_GAS || temperatura < UMBRAL_TEMP || humedad > UMBRAL_HUM);

    if (alertaActiva) {
      digitalWrite(PIN_LED, HIGH);
      digitalWrite(PIN_BUZZER, HIGH);
    } else {
      digitalWrite(PIN_LED, LOW);
      digitalWrite(PIN_BUZZER, LOW);
    }

    // Guardar cada lectura en Supabase
    enviarASupabase(gas, temperatura, humedad, alertaActiva);
  }

  // --------- Envío de alerta a Telegram cada 60s ---------
  if (alertaActiva && (ahora - tiempoEnvio >= intervaloEnvio)) {
    tiempoEnvio = ahora;

    String mensaje = "⚠️ ALERTA DE ALIMENTOS ⚠️\n";
    mensaje += "Gases: " + String(gas) + "\n";
    mensaje += "Temp: " + String(temperatura) + "°C\n";
    mensaje += "Humedad: " + String(humedad) + "%\n";
    mensaje += "¡Revisa los alimentos inmediatamente!";

    if (bot.sendMessage(String(CHAT_ID), mensaje, "")) {
      Serial.println("✅ Mensaje enviado a Telegram.");
    } else {
      Serial.println("❌ Error al enviar el mensaje.");
    }
  }
}

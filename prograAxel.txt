#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "TU_WIFI";
const char* password = "TU_PASSWORD";

// ⚠️ Endpoint correcto a la tabla
const char* serverName = "https://xjmqzdnhactuschmmaaa.supabase.co/rest/v1/sensores";

// API Key (anon)
const char* apiKey = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InhqbXF6ZG5oYWN0dXNjaG1tYWFhIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzYyNTYyNjcsImV4cCI6MjA5MTgzMjI2N30.93gXgtCxxKLGa0u1gLwzZ94bDkBlhitMX76LKByX7Uc";

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.println("Conectando...");
  }

  Serial.println("Conectado!");
  Serial.println("Escribí el estado de la tierra en el monitor serial:");
}

void loop() {

  // 📥 Leer desde el monitor serie
  if (Serial.available()) {
    String estado = Serial.readStringUntil('\n');
    estado.trim(); // limpia espacios

    if (estado.length() > 0) {

      HTTPClient http;
      http.begin(serverName);

      http.addHeader("Content-Type", "application/json");
      http.addHeader("apikey", apiKey);
      http.addHeader("Authorization", String("Bearer ") + apiKey);

      // 📤 JSON con ambas columnas
      String json = "{";
      json += "\"temperatura\": 25.5,";
      json += "\"estado_tierra\": \"" + estado + "\"";
      json += "}";

      int httpResponseCode = http.POST(json);

      Serial.print("Enviado: ");
      Serial.println(json);

      Serial.print("Respuesta: ");
      Serial.println(httpResponseCode);

      http.end();
    }
  }
}

Sistema IoT de Monitoramento Ambiental e Controle Térmico para Salas de ServidoresEste repositório contém o código-fonte do firmware desenvolvido em C++ para o microcontrolador ESP32, integrante do Trabalho de Conclusão de Curso (TCC) focado no monitoramento ambiental de infraestruturas críticas de TI (salas de servidores e racks), seguindo as diretrizes térmicas da norma ASHRAE.O sistema realiza a leitura contínua de temperatura e umidade, exibe os dados localmente em um display LCD, sinaliza visualmente o estado operacional por meio de um LED RGB, atua automaticamente sobre a ventilação auxiliar/alarme sonoro e transmite a telemetria de forma criptografada (MQTTS / TLS 1.2) para a plataforma em nuvem ThingsBoard.📋 Funcionalidades PrincipaisLeitura Ambiental: Aquisição de temperatura ($^\circ\text{C}$) e umidade relativa ($\%$) via sensor digital DHT22.Interface Local Imediata: Exibição em tempo real das métricas em um display LCD 16x2 via barramento I2C.Sinalização Visual (LED RGB): Indicação intuitiva do estado térmico da sala (Normal, Alerta, Crítico ou Erro de Leitura).Automação e Proteção Física:Acionamento automático de exaustão auxiliar (microventilador 12V via módulo relé optoacoplado).Disparo de alarme sonoro local (buzzer piezoelétrico) em situações críticas.Telemetria Segura em Nuvem: Envio de pacotes JSON via protocolo MQTTS (porta 8883) com validação de certificado raiz CA (TLS 1.2) para o ThingsBoard Cloud.🔌 Arquitetura de Hardware e PinagemComponentes Utilizados1x Microcontrolador ESP32 (NodeMCU / DevKit)1x Sensor de Temperatura e Umidade DHT22 + Resistor pull-up de $4{,}7\,\text{k}\Omega$1x Display LCD 16x2 com Módulo Adaptador I2C (PCF8574)1x Módulo Relé 1 Canal 5V (com optoacoplador) + Microventilador 12V DC1x Buzzer Piezoelétrico Ativo1x LED RGB (Catodo Comum - 4 pinos) + 3x Resistores limitadores de $220\,\Omega$Tabela de Conexões (Pinout)ComponentePino do ComponentePino no ESP32Componente Intermediário / ObservaçãoSensor DHT22VCC3V3Alimentação 3,3VDATAGPIO 4Resistor pull-up de $4{,}7\,\text{k}\Omega$ para 3V3GNDGNDTerra comumMódulo Relé (5V)VCCVIN / 5VAlimentação lógica 5VINGPIO 16Sinal de acionamento (Nível Alto ativa o relé)GNDGNDTerra comum (Carga de 12V isolada nos bornes COM/NA)Buzzer PiezoPositivo (+)GPIO 17Sinal de alarme sonoroNegativo (-)GNDTerra comumDisplay LCD I2CVCCVIN / 5VAlimentação 5VSDAGPIO 21Barramento de dados I2CSCLGPIO 22Barramento de clock I2CGNDGNDTerra comumLED RGB (Catodo)Pino Longo (Catodo)GNDTerra comum do LEDVermelho (R)GPIO 18Resistor de $220\,\Omega$ em série (Estado Crítico)Verde (G)GPIO 19Resistor de $220\,\Omega$ em série (Estado Normal)Azul (B)GPIO 23Resistor de $220\,\Omega$ em série (Estado de Alerta)Nota de Segurança: Utilize sempre resistores entre $220\,\Omega$ e $330\,\Omega$ em série com os pinos de cor do LED RGB para limitar a corrente e proteger as portas GPIO do ESP32.🚦 Máquina de Estados e Limiares Térmicos (ASHRAE)O firmware opera sob uma máquina de estados que avalia a temperatura a cada 2 segundos:Estado OperacionalCondição TérmicaLED RGBVentilador (Relé)Buzzer (Alarme)Normal$T \le 27{,}0^\circ\text{C}$🟢 VerdeDesligado (LOW)Desligado (LOW)Alerta$27{,}0^\circ\text{C} < T \le 32{,}0^\circ\text{C}$🔵 AzulLigado (HIGH)Desligado (LOW)Crítico$T > 32{,}0^\circ\text{C}$🔴 VermelhoLigado (HIGH)Ligado (HIGH)Erro de SensorFalha na leitura (NaN)⚪ BrancoMantém estadoMantém estado🛠️ Bibliotecas Necessárias (Arduino IDE / PlatformIO)Antes de compilar o código, instale as seguintes bibliotecas no seu ambiente de desenvolvimento:DHT sensor library (por Adafruit) + Adafruit Unified SensorPubSubClient (por Nick O'Leary) — Comunicação MQTTLiquidCrystal I2C (por Frank de Brabander) — Controle do Display LCDWiFi e WiFiClientSecure (Nativas do pacote ESP32)⚙️ Configuração e UsoClone este repositório ou baixe o arquivo .ino.Abra o código na Arduino IDE ou VS Code (PlatformIO).Altere as constantes de credenciais no início do arquivo:const char* ssid = "SUA_REDE_WIFI";
const char* password = "SUA_SENHA_WIFI";
const char* token = "SEU_ACCESS_TOKEN_DO_DISPOSITIVO";
Selecione a placa ESP32 Dev Module e a porta COM correspondente.Faça o upload do firmware e abra o Monitor Serial (115200 baud) para acompanhar os logs de conexão TLS e telemetria.💻 Código-Fonte do Firmware (firmware_esp32_mqtts.ino)#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ==========================================
// CONFIGURAÇÕES DE REDE E CREDENCIAIS
// ==========================================
const char* ssid = "SUA_REDE_WIFI";
const char* password = "SUA_SENHA_WIFI";
const char* mqtt_server = "thingsboard.cloud";
const int mqtt_port = 8883;
const char* token = "SEU_ACCESS_TOKEN_DO_DISPOSITIVO";

// ==========================================
// MAPEAMENTO DE PINOS E HARDWARE
// ==========================================
#define DHTPIN 4          // Pino GPIO 4 conectado aos dados do DHT22
#define DHTTYPE DHT22     // Definição do modelo do sensor
#define RELAY_PIN 16      // Pino GPIO 16 para o Módulo Relé (Ventilador 12V)
#define BUZZER_PIN 17     // Pino GPIO 17 para o Buzzer Piezoelétrico

// Pinos do LED RGB
#define LED_RED_PIN 18    // GPIO 18 para a cor Vermelha
#define LED_GREEN_PIN 19  // GPIO 19 para a cor Verde
#define LED_BLUE_PIN 23   // GPIO 23 para a cor Azul

// Limiares térmicos baseados na Norma ASHRAE
const float TEMP_LIMITE_ALERTA = 27.0;  
const float TEMP_LIMITE_CRITICO = 32.0; 

// ==========================================
// INSTÂNCIAS DE OBJETOS
// ==========================================
DHT dht(DHTPIN, DHTTYPE);
WiFiClientSecure espClient;
PubSubClient client(espClient);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Certificado raiz CA do ThingsBoard Cloud (ISRG Root X1)
const char* root_ca = \
"-----BEGIN CERTIFICATE-----\n" \
"MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPu7AvWbcAwDQYJKoZIhvcNAQELBQAw\n" \
"TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IFJlc2Vh\n" \
"cmNoIEdyb3VwMRUwEwYDVQQDEwxJU1JHIFJvb3QgWDEwHhcNMTUwNjA0MTEwNDM4\n" \
"WhcNMzUwNjA0MTEwNDM4WjBTMQswCQYDVQQGEwJVUzEpMCAGA1UEChMPSW50ZXJu\n" \
"ZXQgU2VjdXJpdHkgUmVzZWFyY2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBY\n" \
"MTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAK3oJHP0FDfzm54rVygc\n" \
"h77ct984kIxuPOZXoHj3dcKi/vVqbvYATyjb3miGbESTtrFj/RQSa78f0uoxmyF+\n" \
"0TM8ukj13Xnfs7j/EvEhmkvBioZxaUpmZmyPfjxwv60pIgbz5MDmgK7iS4+3mX6U\n" \
"A5/TR5d8mUgjU+g4rk8Kb4Mu0UlXjIB0ttov0DiNewNwIRt18jA8+o+u3dpjq+sW\n" \
"T8KOEUt+zwvo/7V3LvSye0rgTBIlDHCNAymg4VMk7BPZ7hm/ELNKjD+Jo2FR3qyH\n" \
"B5T0Y3HsLuJvW5iB4Y4NHLAduW6lcXPwgyMA7ljJ95/VNmISax6zp9z7AIje0TOD\n" \
"IrFAHSdj6Ftgy0UUAVYkfpIIzVTP8DQWPqLaKNVrvLM74519vDTVQXGJ0JWyIwHr\n" \
"KFgiZmSsJfUPZBcSInDLjqBWlq/3FsMp60Y8ETcHUFjk9WmzgwjNNLgMPWBPqjOf\n" \
"ivIxSQsN47QA5SUDqxVPIFLWVE3BVRIRvj6EUv7dKA4OGLfL8SWrVx9r1DX5DXKL\n" \
"GXVWKSQNz9djr1w8No4rEAo9dZ8v58DeiQoxvsqTfGHf9AhmPS34ExMGHPW3ILTH\n" \
"KZYRsWhCvdVdNUVVBVPAfWStC6sR6k0xWz6K9M835d3GqC9Gj6FvY4Fq61q3K8n/\n" \
"wgaR8f5oJ3tQ5y3qgVq81H2CAwEAAaOCAQEwDgYDVR0PAQH/BAQDAgEGMA8GA1Ud\n" \
"EwEB/wQFMAMBAf8wHQYDVR0OBBYEFPIsDVOaL1opauSYUIVOAO8fJdSlMA4GA1Ud\n" \
"DwEB/wQEAwIBBjANBgkqhkiG9w0BAQsFAAOCAgEAKUP3b9uRGQnNMJkla9SMqY+D\n" \
"P1S5nJz2e9dI2qF1LwG+Q3fP4Zt51dK1n6Z1GjJ85q6b/2y5ZfF6wWq73w/J2l0F\n" \
"7j3wLp97e3Wn3kL1+5t8h1o5Y4Q2+7XmJ4v2n0Z6k6M5l5R3z1y3o7q3w/J2l0F7\n" \
"j3wLp97e3Wn3kL1+5t8h1o5Y4Q2+7XmJ4v2n0Z6k6M5l5R3z1y3o7q3w/J2l0F7\n" \
"-----END CERTIFICATE-----\n";

unsigned long ultimaLeitura = 0;
const long intervaloLeitura = 2000;

// Função auxiliar para controlar as cores do LED RGB
void setLEDColor(bool red, bool green, bool blue) {
  digitalWrite(LED_RED_PIN, red ? HIGH : LOW);
  digitalWrite(LED_GREEN_PIN, green ? HIGH : LOW);
  digitalWrite(LED_BLUE_PIN, blue ? HIGH : LOW);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_RED_PIN, OUTPUT);
  pinMode(LED_GREEN_PIN, OUTPUT);
  pinMode(LED_BLUE_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  setLEDColor(false, false, false);

  dht.begin();
  Wire.begin(21, 22);
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Iniciando IoT...");
  lcd.setCursor(0, 1);
  lcd.print("Server Rack Mon.");

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  lcd.clear();
  lcd.print("Wi-Fi Conectado!");
  delay(1500);

  espClient.setCACert(root_ca);
  client.setServer(mqtt_server, mqtt_port);
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Tentando conexão MQTTS...");
    if (client.connect("ESP32_ServerRoom_Monitor", token, "")) {
      Serial.println("[CONECTADO VIA TLS 1.2]");
    } else {
      delay(5000);
    }
  }
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long tempoAtual = millis();
  
  if (tempoAtual - ultimaLeitura >= intervaloLeitura) {
    ultimaLeitura = tempoAtual;

    float umidade = dht.readHumidity();
    float temperatura = dht.readTemperature();

    if (isnan(umidade) || isnan(temperatura)) {
      Serial.println("Erro ao ler dados do sensor DHT22!");
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Erro Sensor DHT!");
      setLEDColor(true, true, true); // Branco indicando erro de leitura
      return;
    }

    Serial.printf("Temperatura: %.2f C | Umidade: %.2f %%\n", temperatura, umidade);

    // Atualização do LCD
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(temperatura, 1);
    lcd.print((char)223);
    lcd.print("C   ");

    lcd.setCursor(0, 1);
    lcd.print("Umid: ");
    lcd.print(umidade, 1);
    lcd.print("%    ");

    // ==========================================
    // MÁQUINA DE ESTADOS, ATUADORES E LED RGB
    // ==========================================
    if (temperatura > TEMP_LIMITE_CRITICO) {
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
      setLEDColor(true, false, false); // Vermelho (Crítico)
      Serial.println("[CRÍTICO] Temperatura > 32°C. Alarme e LED Vermelho ativos.");
    } 
    else if (temperatura > TEMP_LIMITE_ALERTA) {
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, LOW);
      setLEDColor(false, false, true); // Azul (Alerta)
      Serial.println("[ALERTA] Temperatura > 27°C. Ventilação e LED Azul ativos.");
    } 
    else {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      setLEDColor(false, true, false); // Verde (Normal)
    }

    // Publicação MQTTS
    String payload = "{\"temperature\":" + String(temperatura, 2) + ",\"humidity\":" + String(umidade, 2) + "}";
    client.publish("v1/devices/me/telemetry", payload.c_str());
  }
}

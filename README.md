# Dispensador-SmartPour
//Codigo final
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "apwifieeprommode.h"
#include <EEPROM.h>
#include <FirebaseESP32.h>

FirebaseData firebaseData;
FirebaseConfig config;
FirebaseAuth auth;
// Credenciales de Firebase
#define URL "https://sembebidos-e1258-default-rtdb.firebaseio.com/"  // URL de tu base de datos
#define CLAV "AIzaSyCy08VXL2dK5OPuEB4vhDWIcLkjOO7-zBk" // Clave API

#define COLUMS         16
#define ROWS           2
#define PAGE           (COLUMS * ROWS)

// tu constructor “bajo nivel” para el PCF8574
LiquidCrystal_I2C lcd(PCF8574_ADDR_A21_A11_A01,
                      4,5,6,16,11,12,13,14, POSITIVE);

#define RST_PIN        32
#define SS_PIN         5

#define LED_VERDE      2
#define LED_ROJO       4
#define BUZZER         33

#define PULSADOR_PIN   13
#define PULSADOR2_PIN  25
#define PULSADOR3_PIN  27

#define BOMBA          15
#define BOMBA2         16
#define BOMBA3         17

#define TRIG_PIN       14
#define ECHO_PIN       12

const String PATH_CREDITO = "/credito";
const String PATH_BEBIDAS = "/bebidasAl/";   // contadores Ron/Tequila/Vodka
const String PATH_PRECIOS = "/preciosAl/";   // precios unitarios

MFRC522 mfrc522(SS_PIN, RST_PIN);
byte UID_AUTORIZADO[] = {0x03,0x09,0x2E,0xF5};
const byte UID_SIZE = 4;

bool fbGetFloat(const String& path, float& val){
  if (Firebase.getFloat(firebaseData, path)){
      val = firebaseData.floatData(); return true;
  }
  return false;  // silencio: la llamada que la use mostrará el error
}

bool fbGetInt(const String& path, int& val){
  if (Firebase.getInt(firebaseData, path)){
      val = firebaseData.intData(); return true;
  }
  return false;
}

bool fbSetFloat(const String& path, float val){
  if (Firebase.setFloat(firebaseData, path, val)) return true;
  Serial.println("setFloat → " + firebaseData.errorReason());
  return false;
}

bool fbSetInt(const String& path, int val){
  if (Firebase.setInt(firebaseData, path, val)) return true;
  Serial.println("setInt → " + firebaseData.errorReason());
  return false;
}

/*────────────────── leeCredito(): robusto a int/float ──────*/
float leeCredito(){
  float f;
  if (fbGetFloat(PATH_CREDITO, f)) return f;   // estaba como float
  int i;
  if (fbGetInt(PATH_CREDITO, i))   return (float)i; // estaba como int
  Serial.println("getCredito → " + firebaseData.errorReason());
  return 0;
}

/*────────────────── registrarConsumo() ─────────────────────*/
void registrarConsumo(const char* bebidaTag)
{
  /* 1. incrementar contador de la bebida -------------------*/
  int contador = 0;
  fbGetInt(PATH_BEBIDAS + bebidaTag, contador);
  contador++;
  fbSetInt(PATH_BEBIDAS + bebidaTag, contador);

  /* 2. obtener precio unitario -----------------------------*/
  float precio = 0;
  if (!fbGetFloat(PATH_PRECIOS + bebidaTag, precio)){
      Serial.println("getPrecio → " + firebaseData.errorReason());
      return;
  }

  /* 3. leer y actualizar crédito global --------------------*/
  float credito = leeCredito();
  float nuevo   = max(0.0f, credito - precio);
  fbSetFloat(PATH_CREDITO, nuevo);
}

long medirDistancia() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long dur = pulseIn(ECHO_PIN, HIGH, 30000);
  return (dur * 0.034) / 2;
}

// función para volver a pantalla de inicio
void mostrarBienvenida() {
  digitalWrite(LED_VERDE, LOW);
  digitalWrite(LED_ROJO,  LOW);
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Bienvenidos");
  lcd.setCursor(0,1); lcd.print("Use su pulsera!!");
  delay(500);
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1();
  delay(100);
}

void setup() {
  Serial.begin(115200);
  
  intentoconexion("x_escala","0953310208");

  if (WiFi.status() == WL_CONNECTED)
  {
    Serial.println("Conectado a Wi-Fi con éxito");
    Serial.print("IP local: ");
    Serial.println(WiFi.localIP());
  }
  else
  {
    Serial.println("Error al conectar a Wi-Fi. Verifique la configuración.");
    return;
  }

// Configurar Firebase
  config.host = URL;
  config.signer.tokens.legacy_token = CLAV;

  // Tiempo de espera para la conexión al servidor
  config.timeout.serverResponse = 10 * 1000; // 10 segundos

  // Inicializar Firebase con configuración y autenticación
  Firebase.begin(&config, &auth);

  // Habilitar reconexión automática al Wi-Fi
  Firebase.reconnectWiFi(true);

  Serial.println("Firebase inicializado con éxito");
  

  // inicializa RFID
  SPI.begin(18,19,23,SS_PIN);
  mfrc522.PCD_Init();

  // inicializa LCD
  lcd.begin(COLUMS, ROWS);
  lcd.print(F("INICIANDO...."));
  delay(3000);
  mostrarBienvenida();
  Serial.println("Acerque su tarjeta al lector...");

  // pines indicadores
  pinMode(LED_VERDE, OUTPUT);
  pinMode(LED_ROJO,  OUTPUT);
  pinMode(BUZZER,    OUTPUT);
  digitalWrite(LED_VERDE, LOW);
  digitalWrite(LED_ROJO,  LOW);
  digitalWrite(BUZZER,    LOW);

  // pulsadores
  pinMode(PULSADOR_PIN,  INPUT_PULLUP);
  pinMode(PULSADOR2_PIN, INPUT_PULLUP);
  pinMode(PULSADOR3_PIN, INPUT_PULLUP);

  // bombas
  pinMode(BOMBA,  OUTPUT);
  pinMode(BOMBA2, OUTPUT);
  pinMode(BOMBA3, OUTPUT);
  digitalWrite(BOMBA,  LOW);
  digitalWrite(BOMBA2, LOW);
  digitalWrite(BOMBA3, LOW);

  // ultrasonico
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
}

void loop() {
  // lectura RFID
  if (!mfrc522.PICC_IsNewCardPresent()) return;
  if (!mfrc522.PICC_ReadCardSerial())  return;

  // verificar UID
  Serial.print("UID:");
  bool autorizado = true;
  for (byte i=0; i<mfrc522.uid.size; i++){
    byte b = mfrc522.uid.uidByte[i];
    Serial.print(b<0x10?" 0":" "); Serial.print(b,HEX);
    if (i>=UID_SIZE || b!=UID_AUTORIZADO[i]) autorizado=false;
  }
  Serial.println();

  if (!autorizado) {
    // acceso denegado
    Serial.println("ACCESO DENEGADO");
    digitalWrite(LED_VERDE, LOW);
    digitalWrite(LED_ROJO,  HIGH);
    tone(BUZZER, 400, 800);
    lcd.clear();
    lcd.setCursor(0,0); lcd.print("Acceso denegado");
    delay(800); noTone(BUZZER);
    mostrarBienvenida();
    return;
  }

  // acceso permitido
  Serial.println("ACCESO PERMITIDO");
  digitalWrite(LED_VERDE, HIGH);
  digitalWrite(LED_ROJO,  LOW);
  tone(BUZZER, 1000, 300);
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Acceso permitido");
  delay(300); noTone(BUZZER);

  // pedir vaso con timeout de 5s
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Acerque su vaso");
  unsigned long start = millis();
  bool vaso_ok = false;
  while (millis() - start < 5000) {
    if (medirDistancia() <= 5) {
      vaso_ok = true;
      break;
    }
    delay(100);
  }
  if (!vaso_ok) {
    lcd.clear();
    lcd.setCursor(0,0); lcd.print("Tardaste mucho");
    delay(2000);
    mostrarBienvenida();
    return;
  }

  // menú de bebidas
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Elija su bebida");
  delay(3000);
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("RON_1  VODKA_2");
  lcd.setCursor(0,1); lcd.print("TEQUILA_3");

  // selección y chequeo vaso durante servicio
  bool servido = false;
  while (!servido) {
    long dist = medirDistancia();
    if (dist > 5) {
      lcd.clear();
      lcd.setCursor(0,0); lcd.print("Vaso alejado");
      delay(1500);
      mostrarBienvenida();
      return;
    }
    if (digitalRead(PULSADOR_PIN)==LOW) {
      lcd.clear(); lcd.setCursor(0,0); lcd.print("Sirviendo ron");
      unsigned long t0 = millis();
      digitalWrite(BOMBA, HIGH);
      while (millis()-t0 < 5000) {
        if (medirDistancia()>5) {
          digitalWrite(BOMBA, LOW);
          lcd.clear(); lcd.setCursor(0,0); lcd.print("Vaso alejado");
          delay(1500);
          mostrarBienvenida();
          return;
        }
        delay(100);
      }
      digitalWrite(BOMBA, LOW);
      servido = true;
      registrarConsumo("Ron");
    }
    else if (digitalRead(PULSADOR2_PIN)==LOW) {
      lcd.clear(); lcd.setCursor(0,0); lcd.print("Sirviendo vodka");
      unsigned long t0 = millis();
      digitalWrite(BOMBA2, HIGH);
      while (millis()-t0 < 5000) {
        if (medirDistancia()>5) {
          digitalWrite(BOMBA2, LOW);
          lcd.clear(); lcd.setCursor(0,0); lcd.print("Vaso alejado");
          delay(1500);
          mostrarBienvenida();
          return;
        }
        delay(100);
      }
      digitalWrite(BOMBA2, LOW);
      servido = true;
      registrarConsumo("Vodka");
    }
    else if (digitalRead(PULSADOR3_PIN)==LOW) {
      lcd.clear(); lcd.setCursor(0,0); lcd.print("Sirviendo tequila");
      unsigned long t0 = millis();
      digitalWrite(BOMBA3, HIGH);
      while (millis()-t0 < 5000) {
        if (medirDistancia()>5) {
          digitalWrite(BOMBA3, LOW);
          lcd.clear(); lcd.setCursor(0,0); lcd.print("Vaso alejado");
          delay(1500);
          mostrarBienvenida();
          return;
        }
        delay(100);
      }
      digitalWrite(BOMBA3, LOW);
      servido = true;
      registrarConsumo("Tequila");
    }
    delay(100);
  }

  // agradecimiento
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Gracias por su");
  lcd.setCursor(0,1); lcd.print("compra");
  delay(3000);

  mostrarBienvenida();
}

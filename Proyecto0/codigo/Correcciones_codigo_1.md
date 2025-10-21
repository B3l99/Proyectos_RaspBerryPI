## Codigo inicial

#include <Arduino.h>

    //Lista de pines 
    const C_rojo = 2;
    const C_amarillo = 3;
    const C_verde = 4;
    const P_rojo = 5;
    const P_verde = 6;
    const button_PIN = 10;

    //constantes de tiempo que no tengo muy claro como van
    // ---- Tiempos (ms) ----
    const unsigned long DEBOUNCE_MS     = 50;
    //esto es para el boton para que no haya doble pulsacion dejamos un 
    //pequeño intervalo para que no haya probles ni pulsaciones fantasma
    const unsigned long MIN_INTERVAL_MS = 3000; // entre ciclos
    //esto es un delay para que no haya varias acciones simultaneas como
    //por ejemplo pasar el semaforo de verde a ambar directamente
    const unsigned long YELLOW_MS       = 2000;
    //como el ambar es transitorio le metemos un temporizador
    const unsigned long ALL_RED_MS      = 500;
    //temporizzador para el rojo
    const unsigned long WALK_MS         = 5000;
    //temporizador par el verde de los peatones 
    const unsigned long BLINK_TOTAL_MS  = 2000;
    //medida de seguridad todos en rojo para que no haya accidentes 
    const uint8_t       BLINK_STEPS     = 6;    // nº de destellos del verde peatón

    //empezamos con los estados de los leds
    //Basicamente pensamos como deberian estar las luxes cuando puede pasar el coche
    void coches_on(){
    digitalWrite(C_verde, HIGH);
    digitalWrite(C_rojo, LOW);
    digitalWrite(C_amarillo, LOW);
    digitalWrite(P_rojo, HIGH);
    digitalWrite(P_verde, LOW);
    }

    //Lo mismo pero para los peatones 
    void peatones_on(){
      digitalWrite(C_verde, LOW);
      digitalWrite(C_rojo, HIGH);
      digitalWrite(C_amarillo, LOW);
      digitalWrite(P_rojo, LOW);
      digitalWrite(P_verde, HIGH);
    }

    //otro pero para el de seguridad todo en rojo
    // este la verdad que yo no lo habria puesto

    void seguridad(){
    digitalWrite(C_verde, LOW);
    digitalWrite(C_rojo, HIGH);
    digitalWrite(C_amarillo, LOW);
    digitalWrite(P_rojo, HIGH);
    digitalWrite(P_verde, LOW);
    }

    //Este no lo tengo muy claro la verdad 
    void enterState(State s) {
    state = s;                 // 1) Guardamos en qué estado estamos ahora
    stateStart = millis();     // 2) Anotamos la hora de entrada (para medir tiempos)

    switch (s) {               // 3) Acciones específicas según el nuevo estado
    case IDLE:
      coches_on();             // Coches en verde, peatón en rojo
      Serial1.println("[Estado] IDLE: coches en VERDE");
      break;

    case YELLOW:
      // Amarillo para coches, peatón en rojo
      digitalWrite(C_verde, LOW);
      digitalWrite(C_rojo, LOW);
      digitalWrite(C_amarillo, HIGH);
      digitalWrite(P_rojo, LOW);
      digitalWrite(P_verde, LOW);
      Serial1.println("[Estado] YELLOW: coches en AMARILLO");
      break;

    case seguridad:
      seguridad();             // Todo en rojo un instante (colchón de seguridad)
      Serial1.println("[Estado] ALL_RED: todo en ROJO (seguridad)");
      break;

    case WALK:
      peatones_on();      // Peatones en verde, coches en rojo
      Serial1.println("[Estado] WALK: peatones en VERDE");
      break;

    case BLINK:
      // Preparar el parpadeo del verde peatón
      blinkInterval = BLINK_TOTAL_MS / BLINK_STEPS; // cada cuánto alterna
      blinkCount = 0;                                // cuántos toggles llevamos
      digitalWrite(P_verde, HIGH);                 // empezamos encendido
      lastBlinkTog = millis();                       // hora del último toggle
      Serial1.println("[Estado] BLINK: parpadeo verde peatón");
      break;
      }
    }


    void setup() {
      // Serial de depuración (UART en Pico)
      Serial1.begin(115200);
      Serial1.println("Hello, Raspberry Pi Pico!");

    // Configuración de pines
    pinMode(C_rojo,    OUTPUT);
    pinMode(C_amarillo, OUTPUT);
    pinMode(C_verde,  OUTPUT);
    pinMode(P_rojo,    OUTPUT);
    pinMode(P_verde,  OUTPUT);

    pinMode(BUTTON_PIN, INPUT_PULLDOWN); // botón a 3V3

    // Estado inicial
    coches_ono();
    state = IDLE;
    stateStart = millis();
    lastCycleEnd = millis();
    }

    void loop() {
    unsigned long now = millis();

    // ---- Antirrebote del botón + detección de flanco ----
    bool reading = digitalRead(BUTTON_PIN);
    if (reading != lastReading) {
    lastDebounceTime = now;
    }
    if ((now - lastDebounceTime) > DEBOUNCE_MS) {
    if (reading != buttonState) {
      buttonState = reading;
    }
    }
    bool pressedEdge = (buttonState == HIGH && lastButtonState == LOW);
    lastButtonState = buttonState;
    lastReading = reading;
  
    // ---- Lógica de la máquina de estados ----
    switch (state) {
    case IDLE:
      if (pressedEdge && (now - lastCycleEnd) > MIN_INTERVAL_MS) {
        enterState(YELLOW);
      }
      break;

    case YELLOW:
      if (now - stateStart >= YELLOW_MS) {
        enterState(ALL_RED);
      }
      break;

    case ALL_RED:
      if (now - stateStart >= ALL_RED_MS) {
        enterState(WALK);
      }
      break;

    case WALK:
      if (now - stateStart >= WALK_MS) {
        enterState(BLINK);
      }
      break;

    case BLINK:
      if (now - lastBlinkTog >= blinkInterval) {
        // Toggle verde peatón
        digitalWrite(P_verde, !digitalRead(PED_GREEN));
        lastBlinkTog = now;
        blinkCount++;
        if (blinkCount >= BLINK_STEPS) {
          enterState(IDLE);
          lastCycleEnd = now;
        }
      }
      break;
    }
    
    // Pequeño respiro al simulador
    delay(1);
    }


## Codigo con las correcciones 

#include <Arduino.h>

    // ---- Estados del semáforo ----
    enum State {
    IDLE,     // Coches en verde
    YELLOW,   // Coches en amarillo
    ALL_RED,  // Todo en rojo (seguridad)
    WALK,     // Peatones en verde
    BLINK     // Parpadeo verde peatón
    };

    // ---- Lista de pines ----
    const int C_rojo = 2;
    const int C_amarillo = 3;
    const int C_verde = 4;
    const int P_rojo = 5;
    const int P_verde = 6;
    const int BUTTON_PIN = 10;  // Cambiado a mayúsculas para consistencia

    // ---- Constantes de tiempo (ms) ----
    const unsigned long DEBOUNCE_MS     = 50;    // Anti-rebote del botón
    const unsigned long MIN_INTERVAL_MS = 3000;  // Tiempo mínimo entre ciclos
    const unsigned long YELLOW_MS       = 2000;  // Duración amarillo
    const unsigned long ALL_RED_MS      = 500;   // Duración todo en rojo
    const unsigned long WALK_MS         = 5000;  // Duración verde peatones
    const unsigned long BLINK_TOTAL_MS  = 2000;  // Duración total del parpadeo
    const uint8_t       BLINK_STEPS     = 6;     // Número de destellos

    // ---- Variables globales ----
    State state;                    // Estado actual
    unsigned long stateStart;       // Tiempo de inicio del estado actual
    unsigned long lastCycleEnd;     // Fin del último ciclo completo
    unsigned long lastDebounceTime; // Para anti-rebote
    unsigned long lastBlinkTog;     // Último toggle en BLINK
    unsigned long blinkInterval;    // Intervalo de parpadeo
    uint8_t blinkCount;             // Contador de parpadeos

    // Variables del botón
    bool buttonState = LOW;
    bool lastButtonState = LOW;
    bool lastReading = LOW;

    // ---- Funciones de control de LEDs ----

    // Semáforo para coches (verde coches, rojo peatones)
    void coches_on(){
    digitalWrite(C_verde, HIGH);
    digitalWrite(C_rojo, LOW);
    digitalWrite(C_amarillo, LOW);
    digitalWrite(P_rojo, HIGH);
    digitalWrite(P_verde, LOW);
    }

    // Semáforo para peatones (rojo coches, verde peatones)
    void peatones_on(){
    digitalWrite(C_verde, LOW);
    digitalWrite(C_rojo, HIGH);
    digitalWrite(C_amarillo, LOW);
    digitalWrite(P_rojo, LOW);
    digitalWrite(P_verde, HIGH);
    }

    // Todo en rojo (seguridad)
    void todo_rojo(){
    digitalWrite(C_verde, LOW);
    digitalWrite(C_rojo, HIGH);
    digitalWrite(C_amarillo, LOW);
    digitalWrite(P_rojo, HIGH);
    digitalWrite(P_verde, LOW);
    }

    // ---- Función para cambiar de estado ----
    void enterState(State s) {
    state = s;                 // Guardamos el nuevo estado
    stateStart = millis();     // Anotamos la hora de entrada

    switch (s) {
    case IDLE:
      coches_on();  // Coches en verde, peatón en rojo
      Serial.println("[Estado] IDLE: coches en VERDE");
      break;

    case YELLOW:
      // Amarillo para coches, peatón en rojo
      digitalWrite(C_verde, LOW);
      digitalWrite(C_rojo, LOW);
      digitalWrite(C_amarillo, HIGH);
      digitalWrite(P_rojo, HIGH);
      digitalWrite(P_verde, LOW);
      Serial.println("[Estado] YELLOW: coches en AMARILLO");
      break;

    case ALL_RED:
      todo_rojo();  // Todo en rojo (seguridad)
      Serial.println("[Estado] ALL_RED: todo en ROJO (seguridad)");
      break;

    case WALK:
      peatones_on();  // Peatones en verde, coches en rojo
      Serial.println("[Estado] WALK: peatones en VERDE");
      break;

    case BLINK:
      // Preparar el parpadeo del verde peatón
      blinkInterval = BLINK_TOTAL_MS / BLINK_STEPS;
      blinkCount = 0;
      digitalWrite(P_verde, HIGH);  // Empezamos encendido
      lastBlinkTog = millis();
      Serial.println("[Estado] BLINK: parpadeo verde peatón");
      break;
    }
    }

    // ---- Setup ----
    void setup() {
    // Serial de depuración
    Serial.begin(115200);
    while(!Serial) { ; }  // Esperar a que el puerto serial esté listo
    Serial.println("¡Semáforo iniciado!");

    // Configuración de pines
    pinMode(C_rojo, OUTPUT);
    pinMode(C_amarillo, OUTPUT);
    pinMode(C_verde, OUTPUT);
    pinMode(P_rojo, OUTPUT);
    pinMode(P_verde, OUTPUT);
    pinMode(BUTTON_PIN, INPUT_PULLDOWN);  // Botón con pull-down interno

    // Estado inicial
    coches_on();  // Corregido el typo
    state = IDLE;
    stateStart = millis();
    lastCycleEnd = millis();
    lastDebounceTime = 0;
    }

    // ---- Loop principal ----
    void loop() {
      unsigned long now = millis();

    // ---- Anti-rebote del botón y detección de flanco ----
      bool reading = digitalRead(BUTTON_PIN);
  
    if (reading != lastReading) {
    lastDebounceTime = now;
    }
  
    if ((now - lastDebounceTime) > DEBOUNCE_MS) {
    if (reading != buttonState) {
      buttonState = reading;
    }
    }
  
    bool pressedEdge = (buttonState == HIGH && lastButtonState == LOW);
    lastButtonState = buttonState;
    lastReading = reading;

    // ---- Máquina de estados ----
    switch (state) {
    case IDLE:
      // Si se presiona el botón y ha pasado el tiempo mínimo
      if (pressedEdge && (now - lastCycleEnd) > MIN_INTERVAL_MS) {
        Serial.println("Botón presionado - iniciando ciclo peatonal");
        enterState(YELLOW);
      }
      break;

    case YELLOW:
      // Después de 2 segundos en amarillo
      if (now - stateStart >= YELLOW_MS) {
        enterState(ALL_RED);
      }
      break;

    case ALL_RED:
      // Después de 0.5 segundos todo en rojo
      if (now - stateStart >= ALL_RED_MS) {
        enterState(WALK);
      }
      break;

    case WALK:
      // Después de 5 segundos de verde peatón
      if (now - stateStart >= WALK_MS) {
        enterState(BLINK);
      }
      break;

    case BLINK:
      // Parpadeo del verde peatón
      if (now - lastBlinkTog >= blinkInterval) {
        // Toggle del LED verde peatón
        digitalWrite(P_verde, !digitalRead(P_verde));  // Corregido
        lastBlinkTog = now;
        blinkCount++;
        
        // Si completamos los parpadeos, volver a IDLE
        if (blinkCount >= BLINK_STEPS) {
          enterState(IDLE

# Correciones y explicaciones 

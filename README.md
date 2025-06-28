# **Introdução**

O projeto apresentado consiste na implementação de um **pêndulo invertido**, desenvolvido como parte das atividades da disciplina **[[Modelagem e Controle de Sistemas II]]**. A estrutura base foi adaptada a partir de uma impressora comum, aproveitando seu motor embutido de **12V** para a movimentação do trilho. Além da impressora, foram utilizados os seguintes componentes:

| Componente                  | Descrição                                     | Datasheet                                                                                                                                                                                                           |
| --------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ESP32                       | Microcontrolador                              | [📄](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf)                                                                                                                            |
| Sensor de distância VL53L0X | Sensor de distância (posição do carrinho)     | [📄](https://www.alldatasheet.com/view.jsp?Searchword=Vl53l0x%20Datasheet&gad_source=1&gad_campaignid=1432848463&gclid=CjwKCAjw3MXBBhAzEiwA0vLXQTzKcui1WLHXg-_vCA1itTCsSSLOXApv7Bhh_TEmkd0yjqiV-MBufRoCYBwQAvD_BwE) |
| Sensor inercial MPU-6050    | Giroscópio + Acelerômetro (ângulo do pêndulo) | [📄](https://www.alldatasheet.com/view.jsp?Searchword=Mpu-6050%20datasheet&gad_source=1&gad_campaignid=163458844&gclid=CjwKCAjw3MXBBhAzEiwA0vLXQTH4CT-uLhW6-a2hkWFem5TBKgU2mwys2hFuboTLkVvxGFpHKglb2RoCXcMQAvD_BwE) |
| Driver de motor L298N       | Driver para motor DC                          | [📄](https://www.alldatasheet.com/datasheet-pdf/pdf/22440/STMICROELECTRONICS/L298N.html)                                                                                                                            |
| Fonte de alimentação 12V    | Alimentação do motor                          |                                                                                                                                                                                                                     |

A combinação desses componentes permite a estabilização do pêndulo invertido por meio de estratégias de controle em tempo real, explorando conceitos teóricos aplicados na disciplina que serão apresentados a seguir.
# **Objetivos do Projeto**

O principal objetivo deste projeto é **estabilizar um pêndulo invertido dentro de uma região linearizada**, mantendo sua oscilação dentro de um limite de **±10°** em relação ao ponto de equilíbrio vertical. Para isso, serão utilizadas **técnicas de identificação de sistemas**, especificamente modelos **ARX** (Auto Regressive with eXogenous input) e **ARMAX** (Auto Regressive Moving Average with eXogenous input), para:

1. **Identificar dinamicamente o sistema** a partir de dados experimentais, obtendo um modelo matemático que represente adequadamente o comportamento do pêndulo.
2. **Projetar uma estratégia de controle** (como **PID**, **realimentação de estados** ou **controle preditivo**) com base no modelo estimado, garantindo estabilidade dentro da faixa linear.
3. **Validar experimentalmente** o desempenho do controlador, analisando:
	- Tempo de estabilização.
	- Robustez a perturbações externas.
	- Limitações do modelo linearizado (considerando a restrição de ±10°).
4. **Comparar a eficiência** dos modelos **ARX** e **ARMAX** na representação do sistema.

# **Código**
### Sensores de distância
``` cpp
#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_VL53L0X.h>

// Objetos para os sensores
Adafruit_VL53L0X lox1 = Adafruit_VL53L0X();
Adafruit_VL53L0X lox2 = Adafruit_VL53L0X();

// Pinos XSHUT (reset) para cada sensor
#define LOX1_XSHUT_PIN 26
#define LOX2_XSHUT_PIN 27

void setup() {
  Serial.begin(115200);
  
  // Aguarda a serial (opcional para placas com USB nativo)
  while (!Serial) {
    delay(10);
  }

  Serial.println("\nIniciando dois sensores VL53L0X");

  // Configura pinos XSHUT
  pinMode(LOX1_XSHUT_PIN, OUTPUT);
  pinMode(LOX2_XSHUT_PIN, OUTPUT);
  
  // Inicialmente desliga ambos os sensores
  digitalWrite(LOX1_XSHUT_PIN, LOW);
  digitalWrite(LOX2_XSHUT_PIN, LOW);
  delay(10);

  // Ativa e configura o Sensor 1
  digitalWrite(LOX1_XSHUT_PIN, HIGH);
  delay(10);
  
  if (!lox1.begin(0x30)) {  // Endereço I2C 0x30
    Serial.println(F("Falha no Sensor 1 (VL53L0X)"));
    while (1);
  }
  Serial.println(F("Sensor 1 OK"));

  // Ativa e configura o Sensor 2
  digitalWrite(LOX2_XSHUT_PIN, HIGH);
  delay(10);
  
  if (!lox2.begin(0x31)) {  // Endereço I2C 0x31
    Serial.println(F("Falha no Sensor 2 (VL53L0X)"));
    while (1);
  }
  Serial.println(F("Sensor 2 OK"));

  // Inicia medição contínua
  lox1.startRangeContinuous();
  lox2.startRangeContinuous();

  Serial.println("\nPronto para medir...");
  Serial.println("------------------------");
}

void loop() {
  VL53L0X_RangingMeasurementData_t medida1;
  VL53L0X_RangingMeasurementData_t medida2;

  // Lê Sensor 1
  lox1.rangingTest(&medida1, false); // 'false' desativa debug no serial
  
  Serial.print("Sensor 1: ");
  if (medida1.RangeStatus != 4) {  // Status 4 = fora de alcance
    Serial.print(medida1.RangeMilliMeter);
    Serial.print("mm");
  } else {
    Serial.print("fora de alcance");
  }

  // Lê Sensor 2
  lox2.rangingTest(&medida2, false);
  
  Serial.print("  |  Sensor 2: ");
  if (medida2.RangeStatus != 4) {
    Serial.print(medida2.RangeMilliMeter);
    Serial.print("mm");
  } else {
    Serial.print("fora de alcance");
  }

  Serial.println(); // Nova linha
  delay(100); // Intervalo entre leituras
}
```
### Ponte H
``` cpp
#include <Arduino.h>

// ===== CONFIGURAÇÃO DE PINOS (AJUSTE CONFORME SUA MONTAGEM) =====
const int RPWM = 23;   // Pino PWM para rotação no sentido HORÁRIO (Direita)
const int LPWM = 18;   // Pino PWM para rotação no sentido ANTI-HORÁRIO (Esquerda)
const int R_EN = 19;   // Enable do lado direito (fixo em HIGH)
const int L_EN = 4;    // Enable do lado esquerdo (substitui o GPIO21)

// ===== CONFIGURAÇÃO DO PWM =====
const uint32_t freqPWM = 20000;    // Frequência de 20kHz (ideal para BTS7960)
const uint8_t resolution = 8;      // Resolução de 8 bits (0-255)
const int pwmMax = 255;            // Valor máximo do PWM

// ===== VARIÁVEIS DE CONTROLE =====
int potencia = 0;                  // Valor atual do PWM (0-255)
const int incremento = 5;          // Passo de incremento do PWM
bool direcaoFrente = true;         // Sentido do motor (true = frente, false = trás)
unsigned long ultimaAtualizacao = 0;
unsigned long ultimaMudancaDirecao = 0;
const unsigned long intervaloAtualizacao = 50;   // Tempo entre incrementos (ms)
const unsigned long intervaloDirecao = 1000;     // Tempo para mudar de direção (ms)

// ===== FUNÇÃO PARA CONFIGURAR PINOS E PWM =====
void setup() {
  Serial.begin(115200);
  
  // Configura os pinos de enable (sempre HIGH para habilitar o BTS7960)
  pinMode(R_EN, OUTPUT);
  pinMode(L_EN, OUTPUT);
  digitalWrite(R_EN, HIGH);
  digitalWrite(L_EN, HIGH);
  
  // Configuração dos canais PWM (RPWM e LPWM)
  ledcSetup(0, freqPWM, resolution);  // Canal 0 para RPWM
  ledcSetup(1, freqPWM, resolution);  // Canal 1 para LPWM
  ledcAttachPin(RPWM, 0);             // Associa RPWM ao canal 0
  ledcAttachPin(LPWM, 1);             // Associa LPWM ao canal 1
  
  Serial.println("Sistema iniciado: Controle de motor com BTS7960");
}

// ===== LOOP PRINCIPAL =====
void loop() {
  unsigned long agora = millis();
  
  // ---- CONTROLE DE VELOCIDADE (PWM GRADUAL) ----
  if (agora - ultimaAtualizacao >= intervaloAtualizacao) {
    ultimaAtualizacao = agora;
    
    // Incrementa a potência até o máximo
    if (potencia < pwmMax) {
      potencia += incremento;
      if (potencia > pwmMax) potencia = pwmMax;
    }
    
    // Aplica o PWM conforme a direção
    if (direcaoFrente) {
      ledcWrite(0, potencia);  // Ativa RPWM (sentido horário)
      ledcWrite(1, 0);         // Desativa LPWM
    } else {
      ledcWrite(0, 0);         // Desativa RPWM
      ledcWrite(1, potencia);  // Ativa LPWM (sentido anti-horário)
    }
    
    // Exibe informações no Serial Monitor
    Serial.print("Direção: ");
    Serial.print(direcaoFrente ? "Frente (RPWM)" : "Trás (LPWM)");
    Serial.print(" | Potência: ");
    Serial.print(map(potencia, 0, pwmMax, 0, 100));
    Serial.println("%");
  }
  
  // ---- CONTROLE DE MUDANÇA DE DIREÇÃO ----
  if (agora - ultimaMudancaDirecao >= intervaloDirecao) {
    ultimaMudancaDirecao = agora;
    direcaoFrente = !direcaoFrente;  // Inverte o sentido
    
    // Reinicia a potência para zero ao mudar de direção
    potencia = 0;
    Serial.println("--- Mudança de direção ---");
  }
}
```
### Encoder
``` cpp
// Pinos do encoder
#define ENCODER_A 34
#define ENCODER_B 35

// Variáveis globais voláteis
volatile int contador = 0;
volatile int direcao = 0; // 0 = parado, 1 = horário, -1 = anti-horário
volatile unsigned long ultimaInterrupcao = 0;
const unsigned long debounceTime = 200; // tempo anti-rebote em microssegundos

// Variáveis para monitoramento
unsigned long ultimaLeitura = 0;
float velocidade = 0;

// Função de interrupção melhorada
void IRAM_ATTR leEncoder() {
  unsigned long tempoAtual = micros();
  
  // Filtro de debounce
  if (tempoAtual - ultimaInterrupcao < debounceTime) return;
  ultimaInterrupcao = tempoAtual;

  static int ultimoEstadoA = LOW;
  int estadoAtualA = digitalRead(ENCODER_A);
  
  if (estadoAtualA != ultimoEstadoA) {
    int estadoB = digitalRead(ENCODER_B);
    
    if (estadoB != estadoAtualA) {
      contador++;
      direcao = 1;
    } else {
      contador--;
      direcao = -1;
    }
    ultimoEstadoA = estadoAtualA;
  }
}

void setup() {
  pinMode(ENCODER_A, INPUT_PULLUP);
  pinMode(ENCODER_B, INPUT_PULLUP);
  
  attachInterrupt(digitalPinToInterrupt(ENCODER_A), leEncoder, CHANGE);
  
  Serial.begin(115200);
  Serial.println("Detector de Pequenos Movimentos do Encoder");
}

void loop() {
  // Atualiza a cada 100ms
  if (millis() - ultimaLeitura >= 100) {
    noInterrupts();
    int contadorAtual = contador;
    int dirAtual = direcao;
    interrupts();
    
    Serial.print("Pulsos: ");
    Serial.print(contadorAtual);
    Serial.print(" | Direção: ");
    Serial.println(dirAtual == 1 ? "Horário" : (dirAtual == -1 ? "Anti-Horário" : "Parado"));
    
    ultimaLeitura = millis();
  }
  
  delay(10);
}
```
**Conexões**

| Encoder PNP (NPN) | ESP32                                   |
| ----------------- | --------------------------------------- |
| Vermelho (Vcc +)  | 5V (ou fonte externa 5-24V)             |
| Preto (V0/GND)    | GND                                     |
| Verde (Fase A)    | GPIO34 (e pull-down 10kΩ se necessário) |
| Branco (Fase B)   | GPIO35 (e pull-down 10kΩ se necessário) |

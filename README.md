# 🩺 Smart Ergonomics & Fatigue Monitor (IoT)

![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-brightgreen) ![IoT](https://img.shields.io/badge/IoT-ESP32-blue) ![Protocol](https://img.shields.io/badge/Protocol-MQTT-orange)

## 👥 Autores do Grupo
* **Bryan Willian** (RM551305)
* **Gabriel Freitas** (RM550187)
* **Felipe Terra** (RM99405)

---

## 📺 Demonstração do Projeto
Assista ao vídeo explicativo com a demonstração de funcionamento:

[![Vídeo do Projeto](https://img.youtube.com/vi/VBWhibMaJrA/0.jpg)](https://youtu.be/VBWhibMaJrA)

🔗 **Link direto:** [https://youtu.be/VBWhibMaJrA](https://youtu.be/VBWhibMaJrA)

---

## 🌍 Contexto e Problema
O futuro do trabalho já começou, trazendo consigo a flexibilidade do *Home Office* e dos ambientes híbridos. No entanto, essa liberdade trouxe desafios invisíveis para a saúde ocupacional:

* **Má Postura:** A falta de mobiliário adequado e supervisão leva a problemas crônicos de coluna e LER/DORT.
* **Fadiga e Burnout:** A ausência de limites claros entre trabalho e descanso faz com que colaboradores passem horas ininterruptas frente às telas.
* **Gestão Remota:** Gestores e RHs perderam a capacidade de monitorar o bem-estar físico de suas equipes à distância.

Este projeto visa resolver esses problemas utilizando a **Internet das Coisas (IoT)** para criar um ambiente de trabalho digitalmente assistido, seguro e mais humano.

## 💡 A Solução Proposta
Desenvolvemos uma **Estação de Trabalho Inteligente** baseada em ESP32. O dispositivo atua como um "guardião ativo" da saúde do colaborador, monitorando a distância (postura) e o tempo de trabalho (fadiga).

Diferente de soluções passivas, nosso sistema:
1.  **Alerta Imediatamente:** Avisa o usuário sobre má postura via *feedback* visual (LED RGB) e sonoro (Buzzer).
2.  **Impõe Pausas:** Trava o sistema após um período de trabalho excessivo, prevenindo a exaustão mental.
3.  **Conecta à Nuvem:** Utiliza o protocolo **MQTT** para permitir monitoramento e intervenção remota, simulando a gestão de saúde 4.0.

---

## 🛠️ Hardware e Componentes
O projeto foi simulado no ambiente **Wokwi** utilizando os seguintes componentes:

* **Microcontrolador:** ESP32 DevKit V1
* **Sensor:** HC-SR04 (Sensor Ultrassônico de Distância)
* **Atuador Visual:** LED RGB (Cátodo Comum)
* **Atuador Sonoro:** Buzzer Piezoelétrico
* **Conectividade:** Wi-Fi (Simulado) e Protocolo MQTT

### 📸 Diagrama do Circuito
![Diagrama do Circuito](image_ffe382.png)

---

## 📡 Arquitetura de Comunicação (MQTT)
A comunicação é o diferencial deste projeto, permitindo a integração entre o ambiente físico do trabalhador e a gestão digital corporativa.

* **Broker Utilizado:** `broker.hivemq.com` (TCP Porta 1883)
* **ID do Cliente:** `esp32-final-video-123`

### Tópicos e Endpoints

| Tópico | Tipo | Função | Descrição |
| :--- | :--- | :--- | :--- |
| `futuro_trabalho/ergonomia/status` | **PUBLISH** (Saída) | Telemetria | Envia o estado atual do sistema: `OK`, `ALERTA_POSTURA` ou `PAUSA_NECESSARIA`. Permite criar dashboards de monitoramento. |
| `futuro_trabalho/ergonomia/distancia` | **PUBLISH** (Saída) | Dados | Envia a distância exata medida em cm. Útil para histórico ergonômico. |
| `futuro_trabalho/ergonomia/pausa_cmd` | **SUBSCRIBE** (Entrada) | Comando | Recebe ordens externas. O payload `REINICIAR_TIMER` destrava o sistema após a pausa obrigatória. |

---

## 🚀 Instruções de Uso e Simulação

### 1. Acesso ao Projeto
Acesse a simulação funcional através do link abaixo:
🔗 **[https://wokwi.com/projects/448171544208091137](https://wokwi.com/projects/448171544208091137)**

### 2. Dependências
Para rodar este código (no Wokwi ou na IDE Arduino), é necessário instalar a seguinte biblioteca:
* `PubSubClient` (por Nick O'Leary) - Responsável pela comunicação MQTT.

### 3. Como Testar (Passo a Passo)

1.  **Início:** Dê o "Play" na simulação. O sistema conectará ao Wi-Fi e ao Broker MQTT.
2.  **Teste de Postura:** Clique no sensor HC-SR04.
    * Mantenha a distância entre **35cm e 55cm** → **LED Verde** (Postura Correta).
    * Mova para menos de 35cm → **LED Amarelo** + Bip (Alerta de Postura).
3.  **Teste de Fadiga (Pausa Obrigatória):**
    * Aguarde o tempo limite (configurado para 15 segundos para fins de demonstração no vídeo).
    * O sistema entrará em modo de bloqueio: **LED Vermelho Piscando** + Alarme Sonoro.
4.  **Desbloqueio Remoto (Simulação de Gestão):**
    * O sistema não destrava sozinho, simulando a necessidade de validação da pausa pelo RH ou Gestor.
    * Use um cliente MQTT Web (como o HiveMQ Web Client).
    * Publique a mensagem `REINICIAR_TIMER` no tópico `futuro_trabalho/ergonomia/pausa_cmd`.
    * O sistema receberá o comando, validará a pausa e o LED voltará a ficar **Verde**.

---

## 💻 Código Fonte (`sketch.ino`)

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

// ==========================================
// 1. CONFIGURAÇÕES E PINOS
// ==========================================
const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_client_id = "esp32-final-video-123"; 

const int trigPin = 18;
const int echoPin = 19;
const int ledR = 25;
const int ledG = 26;
const int ledB = 27;
const int buzzerPin = 13;

#define DISTANCIA_MIN 35 
#define DISTANCIA_MAX 55 

// ATENÇÃO: TEMPO CURTO PARA O VÍDEO (15 SEGUNDOS)
const unsigned long LIMITE_TRABALHO = 15000; 

unsigned long tempoInicioTrabalho = 0;
long distanciaAtual = 0;

// TÓPICOS MQTT
const char* TOPICO_STATUS = "futuro_trabalho/ergonomia/status";
const char* TOPICO_DISTANCIA = "futuro_trabalho/ergonomia/distancia";
const char* TOPICO_COMANDO = "futuro_trabalho/ergonomia/pausa_cmd";

WiFiClient espClient;
PubSubClient client(espClient);

// ==========================================
// 2. FUNÇÕES AUXILIARES
// ==========================================

void setCor(bool r, bool g, bool b) {
  // Configurado para CÁTODO COMUM (HIGH liga o LED)
  digitalWrite(ledR, r ? HIGH : LOW);
  digitalWrite(ledG, g ? HIGH : LOW);
  digitalWrite(ledB, b ? HIGH : LOW);
}

long lerDistancia() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duration = pulseIn(echoPin, HIGH);
  return duration * 0.034 / 2;
}

void setup_wifi() {
  Serial.print("Conectando Wifi...");
  WiFi.begin(ssid, password);
  int tentativas = 0;
  while (WiFi.status() != WL_CONNECTED && tentativas < 20) {
    delay(500);
    Serial.print(".");
    tentativas++;
  }
  Serial.println(WiFi.status() == WL_CONNECTED ? " OK!" : " Falha (Modo Offline)");
}

// CALLBACK MQTT (Recebe o comando remoto)
void callback(char* topic, byte* payload, unsigned int length) {
  String msg = "";
  for (int i = 0; i < length; i++) { msg += (char)payload[i]; }
  
  Serial.print("Chegou mensagem: "); Serial.println(msg); 

  if (String(topic) == TOPICO_COMANDO && msg == "REINICIAR_TIMER") {
    tempoInicioTrabalho = millis(); // ZERA O RELÓGIO
    noTone(buzzerPin);
    setCor(0,1,0); // VOLTA VERDE
    client.publish(TOPICO_STATUS, "TRABALHO_RETOMADO");
    Serial.println(">>> SISTEMA DESTRAVADO COM SUCESSO <<<");
  }
}

void reconnect() {
  if (WiFi.status() != WL_CONNECTED) return;
  if (!client.connected()) {
    if (client.connect(mqtt_client_id)) {
      client.subscribe(TOPICO_COMANDO); 
      Serial.println("MQTT Conectado e Assinado!");
    }
  }
}

// ==========================================
// 3. LOOP PRINCIPAL
// ==========================================

void setup() {
  Serial.begin(115200);
  pinMode(trigPin, OUTPUT); pinMode(echoPin, INPUT);
  pinMode(ledR, OUTPUT); pinMode(ledG, OUTPUT); pinMode(ledB, OUTPUT);
  pinMode(buzzerPin, OUTPUT);

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback); 
  tempoInicioTrabalho = millis();
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
     if (!client.connected()) reconnect();
     client.loop(); 
  }

  // Verifica se estourou o tempo (PAUSA OBRIGATÓRIA)
  if (millis() - tempoInicioTrabalho >= LIMITE_TRABALHO) {
    
    // Pisca LED e Buzzer sem usar delay (não trava o MQTT)
    if ((millis() / 500) % 2 == 0) { 
      setCor(1, 0, 0); // Vermelho
      tone(buzzerPin, 1000); 
    } else {
      setCor(0, 0, 0); // Apagado
      noTone(buzzerPin);
    }
    
    static unsigned long timerMsg = 0;
    if (millis() - timerMsg > 1000) {
      client.publish(TOPICO_STATUS, "PAUSA_NECESSARIA");
      timerMsg = millis();
    }
    return; 
  }

  // Monitoramento Normal de Postura
  static unsigned long delaySensor = 0;
  if (millis() - delaySensor < 500) return; 
  delaySensor = millis();

  distanciaAtual = lerDistancia();
  
  if (distanciaAtual < DISTANCIA_MIN || distanciaAtual > DISTANCIA_MAX) {
    setCor(1, 1, 0); // Amarelo
    client.publish(TOPICO_STATUS, "ALERTA_POSTURA");
    tone(buzzerPin, 500, 100); 
  } else {
    setCor(0, 1, 0); // Verde
    client.publish(TOPICO_STATUS, "OK");
    noTone(buzzerPin);
  }
}

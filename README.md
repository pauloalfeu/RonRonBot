# RonRonBot - Monitor de Bebedouro para Gatos

Sistema de monitoramento inteligente de bebedouro para pets utilizando ESP32 com integração ao Telegram.

**Autores:** Ana Caroline Pedrosa & Paulo Alfeu

---

## Organizacao do Projeto

- Sobre o Projeto
- Hardware Necessario
- Bibliotecas
- Configuracao Inicial
- Estrutura do Codigo
- Funcoes Principais
- Comandos do Telegram
- Maquina de Estados
- Troubleshooting

---

## Sobre o Projeto

O RonRonBot monitora automaticamente o bebedouro do seu gato, enviando notificacoes via Telegram sobre:

- **Nivel de agua** - Detecta quando esta baixo atraves de 3 sensores capacitivos
- **Temperatura** - Monitora e alerta quando a agua esta muito quente
- **Visitas do gato** - Detecta presenca atraves de tag RFID (requer configuracao)
- **Relatorios periodicos** - Envia atualizacoes automaticas a cada hora

---

## Hardware Necessario

### Componentes

| Componente | Quantidade | Funcao |
|------------|------------|--------|
| ESP32 | 1 | Microcontrolador principal |
| Sensor DS18B20 | 1 | Medicao de temperatura da agua |
| Sensores Capacitivos | 3 | Detecao de nivel de agua (alto, medio, baixo) |
| Modulo RFID MFRC522 | 1 | Identificacao do gato (opcional) |
| Tag RFID | 1 | Para coleira do gato (opcional) |
| LED | 1 | Indicacao visual de status |

### Pinagem

```
ESP32          Componente
GPIO 4    -->  Sensor DS18B20 (temperatura)
GPIO 13   -->  LED de status
GPIO 5    -->  RFID RST (opcional)
GPIO 15   -->  RFID SS/CS (opcional)
GPIO 6    -->  Sensor capacitivo 1 (superior)
GPIO 7    -->  Sensor capacitivo 2 (medio)
GPIO 8    -->  Sensor capacitivo 3 (inferior)
```

**Nota sobre RFID:** O codigo inclui suporte para RFID mas requer configuracao do UID da tag. O sistema funciona normalmente sem RFID, apenas nao contara visitas automaticamente.

---

## Bibliotecas

Instale as seguintes bibliotecas na Arduino IDE:

```cpp
#include <WiFi.h>                    // Conexao WiFi
#include <WiFiClientSecure.h>        // Comunicacao segura HTTPS
#include <UniversalTelegramBot.h>    // API do Telegram
#include <OneWire.h>                 // Protocolo OneWire
#include <DallasTemperature.h>       // Sensor DS18B20
#include <SPI.h>                     // Comunicacao SPI (para RFID)
#include <MFRC522.h>                 // Leitor RFID (opcional)
```

---

## Configuracao Inicial

### 1. Configurar WiFi

Edite no codigo:

```cpp
#define WIFI_SSID      "SuaRedeWiFi"
#define WIFI_PASSWORD  "SuaSenha"
```

### 2. Configurar Bot do Telegram

**Criando o Bot:**

1. Abra o Telegram e procure por `@BotFather`
2. Envie `/newbot`
3. Escolha um nome e username
4. Copie o **token** fornecido

**No codigo:**

```cpp
#define BOT_TOKEN      "SEU_TOKEN_AQUI"
const long ID_CHAT = SEU_ID_AQUI;
```

**Como descobrir seu ID:**
- Envie mensagem para `@userinfobot`
- Ele retornara seu ID numerico

### 3. Configurar Tag RFID (Opcional)

Se desejar usar a deteccao automatica de visitas:

1. Execute o codigo
2. Aproxime a tag do leitor
3. Veja o UID no Serial Monitor
4. Configure no codigo:

```cpp
const byte UID_GATO[] = {0xDE, 0xAD, 0xBE, 0xEF}; // Seu UID aqui
```

---

## Estrutura do Codigo

### Variaveis Globais Principais

```cpp
float temperatura_atual           // Temperatura lida pelo sensor
float temperatura_filtrada        // Media movel das ultimas 5 leituras
int visitas_hoje                  // Contador de visitas do gato
```

### Constantes Configuraveis

```cpp
VALOR_MAX_TEMPERATURA = 24.0           // Temperatura maxima (C)
INTERVALO_RELATORIO_MS = 3600000       // Relatorio a cada 1 hora
COOLDOWN_ALERTA_MS = 900000            // Espera 15 min entre alertas
TEMPO_MINIMO_PERMANENCIA_MS = 5000     // 5 seg para confirmar visita
```

---

## Funcoes Principais

### Funcoes de Leitura de Sensores

#### `float ler_sensor_temperatura()`
Le a temperatura do sensor DS18B20.

```cpp
float temp = ler_sensor_temperatura();
// Retorna temperatura em Celsius
```

#### `int ler_nivel_agua()`
Le os 3 sensores capacitivos e retorna quantos detectaram agua.

**Retorno:**
- `3` = Bebedouro cheio (100%)
- `2` = Nivel medio (66%)
- `1` = Nivel baixo (33%)
- `0` = Vazio

```cpp
int nivel = ler_nivel_agua();
```

#### `float calcular_temperatura_filtrada(float nova_leitura)`
Aplica filtro de media movel nas ultimas 5 leituras para remover ruido.

```cpp
float temp_bruta = ler_sensor_temperatura();
float temp_suave = calcular_temperatura_filtrada(temp_bruta);
```

### Funcoes de Formatacao

#### `String obter_status_nivel(int contagem)`
Converte a contagem de sensores em texto descritivo.

```cpp
// Entrada: 3 -> Saida: "Alto (100%)"
// Entrada: 2 -> Saida: "Medio (66%)"
// Entrada: 1 -> Saida: "Baixo (33%)"
// Entrada: 0 -> Saida: "Vazio (0%)"
```

### Funcoes RFID (Opcional)

#### `bool verificar_tag_rfid()`
Verifica se ha uma tag RFID presente e se e a tag cadastrada.

**Retorna:** `true` se detectou a tag do gato

**Nota:** Requer configuracao do `UID_GATO[]` para funcionar.

### Funcoes de Inicializacao

#### `void init_sesores_cap()`
Configura os pinos dos sensores capacitivos como entrada com pull-up interno.

#### `void init_rfid()`
Inicializa o modulo RFID MFRC522 via barramento SPI.

#### `void init_ds18b20()`
Configura o sensor de temperatura DS18B20 com resolucao de 12 bits.

#### `void init_wifi()`
Conecta a rede WiFi configurada e prepara conexao segura com Telegram.

---

## Comandos do Telegram

### `/status` - Status Completo

Exibe todas as informacoes do bebedouro.

**Resposta:**
```
Status do Bebedouro

Nivel da agua: Medio (66%)
Temperatura: 23.5 C
Visitas hoje: 2
```

### `/nivel` - Nivel de Agua

Mostra apenas o nivel atual da agua.

**Resposta:**
```
Nivel atual da agua: Medio (66%)
```

### `/temperatura` - Temperatura

Mostra a temperatura atual da agua.

**Resposta:**
```
Temperatura atual: 23.5 C
```

### `/visitas` - Historico de Visitas

Mostra o contador de visitas do dia.

**Resposta:**
```
Visitas hoje: 2
```

**Nota:** Requer tag RFID configurada para contagem automatica. Sem RFID, o contador permanece em 0.

### `/alertas` - Limites de Seguranca

Mostra a configuracao atual dos limites de alerta.

**Resposta:**
```
Configuracao de alertas:
Temperatura maxima: 24.0 C
Alerta quando nivel <= 33%
```

### `/help` ou `/start` - Lista de Comandos

Mostra todos os comandos disponiveis.

**Resposta:**
```
Ola, [Nome]! Eu sou o Monitor de Bebedouro.

Comandos disponiveis:
/status - Visao geral completa
/nivel - Nivel da agua
/temperatura - Temperatura
/visitas - Historico de idas
/alertas - Limites de seguranca
```

---

## Notificacoes Automaticas

### Relatorio Periodico

**Frequencia:** A cada 1 hora (configuravel via `INTERVALO_RELATORIO_MS`)

**Exemplo:**
```
Relatorio Periodico:
Nivel da agua: Medio (66%)
Temperatura: 23.5 C
Visitas hoje: 2
```

### Alerta de Nivel Baixo

**Quando:** Nivel <= 33% (1 sensor ou menos detectando agua)

**Exemplo:**
```
ALERTA!
Nivel da agua baixo: Baixo (33%). Adicione mais agua
```

**Cooldown:** 15 minutos entre alertas do mesmo tipo

### Alerta de Temperatura Alta

**Quando:** Temperatura > 24°C (configuravel via `VALOR_MAX_TEMPERATURA`)

**Exemplo:**
```
ALERTA!
Temperatura da agua elevada: 25.2 C.
```

---

## Maquina de Estados

O sistema funciona como uma maquina de estados finitos (FSM).

### Diagrama de Estados

```
INICIAL -> OCIOSO <-> GATO_DETECTADO -> MEDICAO -> ENVIO_DADOS -> OCIOSO
                                            |
                                            v
                                        ALERTA -> OCIOSO
```

### Descricao dos Estados

#### ESTADO_INICIAL
**Executado:** Uma unica vez ao ligar

**Acoes:**
- Inicializa sensores capacitivos
- Inicializa RFID
- Inicializa sensor de temperatura
- Conecta ao WiFi
- Envia mensagem "Sistema Conectado" ao Telegram

**LED:** Ligado durante inicializacao

**Proximo estado:** OCIOSO

---

#### ESTADO_OCIOSO
**Executado:** Maior parte do tempo (aguardando eventos)

**Acoes:**
- Verifica se ha tag RFID proxima
- Conta tempo para proximo relatorio periodico

**Transicoes:**
- Se detectar tag RFID -> GATO_DETECTADO
- Se passou 1 hora -> MEDICAO

**LED:** Desligado

---

#### ESTADO_GATO_DETECTADO
**Executado:** Quando tag RFID e detectada

**Acoes:**
- Aguarda 5 segundos (TEMPO_MINIMO_PERMANENCIA_MS)
- Se tag continuar presente, confirma visita
- Incrementa contador `visitas_hoje`

**LED:** Piscando (1 segundo on/off)

**Proximo estado:** MEDICAO

---

#### ESTADO_MEDICAO
**Executado:** Apos visita confirmada ou periodicamente

**Acoes:**
1. Le nivel de agua
2. Le temperatura
3. Aplica filtro de media movel
4. Verifica condicoes de alerta

**Transicoes:**
- Se nivel <= 33% OU temperatura > 24°C -> ALERTA
- Se passou 1 hora desde ultimo relatorio -> ENVIO_DADOS
- Caso contrario -> OCIOSO

**LED:** Ligado durante medicao

---

#### ESTADO_ENVIO_DADOS
**Executado:** A cada 1 hora

**Acoes:**
- Compila relatorio completo
- Envia via Telegram
- Reseta timer de relatorio

**LED:** Desligado

**Proximo estado:** OCIOSO

---

#### ESTADO_ALERTA
**Executado:** Quando detecta problema

**Acoes:**
- Verifica cooldown (15 min desde ultimo alerta)
- Se passou cooldown:
  - Monta mensagem de alerta
  - Envia via Telegram
  - Atualiza timestamp do ultimo alerta

**Condicoes de alerta:**
- Nivel de agua <= 33%
- Temperatura > 24°C

**LED:** Piscando rapido (0.5 segundo on/off)

**Proximo estado:** OCIOSO

---

## Fluxo de Execucao

### Loop Principal

```cpp
void loop() {
    // 1. Verifica mensagens do Telegram (a cada 1 segundo)
    if (WiFi conectado && passou 1 segundo) {
        processar_mensagens();
    }
    
    // 2. Executa maquina de estados
    fsm();
}
```

### Processamento de Comandos

1. Bot verifica novas mensagens
2. Filtra apenas mensagens do chat autorizado (ID_CHAT)
3. Identifica o comando
4. Le sensores se necessario
5. Formata resposta
6. Envia via Telegram

---

## Customizacoes

### Alterar Intervalo de Relatorios

```cpp
// Para 30 minutos
const unsigned long INTERVALO_RELATORIO_MS = 1800000;

// Para 2 horas
const unsigned long INTERVALO_RELATORIO_MS = 7200000;
```

### Alterar Temperatura Maxima

```cpp
// Para 26°C
const float VALOR_MAX_TEMPERATURA = 26.0;
```

### Alterar Cooldown de Alertas

```cpp
// Para 30 minutos
const unsigned long COOLDOWN_ALERTA_MS = 1800000;

// Para 5 minutos
const unsigned long COOLDOWN_ALERTA_MS = 300000;
```

### Alterar Tempo de Confirmacao de Visita

```cpp
// Para 3 segundos
const unsigned long TEMPO_MINIMO_PERMANENCIA_MS = 3000;

// Para 10 segundos
const unsigned long TEMPO_MINIMO_PERMANENCIA_MS = 10000;
```

---

## Troubleshooting

### Bot nao responde

**Verificar:**
1. Token do bot esta correto?
2. ID do chat esta correto?
3. WiFi conectado? (Serial Monitor mostra "WiFi connected")
4. Certificado SSL configurado?

**Solucao temporaria:**
```cpp
// Em init_wifi(), adicione antes de setCACert:
secured_client.setInsecure(); // APENAS PARA TESTES
```

### Sensor de temperatura retorna -127°C

**Causa:** Sensor nao encontrado ou mal conectado

**Verificar:**
- Conexao do pino de dados (GPIO 4)
- Resistor pull-up de 4.7kΩ entre dados e VCC
- Alimentacao 3.3V ou 5V
- Sensor esta funcionando

### RFID nao detecta tag

**Verificar:**
- Conexoes SPI corretas (MOSI, MISO, SCK)
- Pinos SS (GPIO 15) e RST (GPIO 5)
- Alimentacao do modulo (3.3V)
- UID configurado no codigo

**Para descobrir UID da tag:**
Aproxime a tag e veja o Serial Monitor durante inicializacao.

### Sensores capacitivos sempre retornam 0

**Verificar:**
- Pinos configurados como INPUT_PULLUP
- Sensores em modo digital (nao analogico)
- Distancia entre sensor e agua (max ~5mm atraves do plastico)
- Sensores estao funcionando (teste individual)

### Erro de compilacao no init_wifi()

**Causa:** Variaveis de configuracao de tempo nao definidas

O codigo original tem uma chamada a `configTime()` mas nao define as variaveis necessarias. Remova ou comente a linha:

```cpp
// configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
```

---

## Monitoramento via Serial

Abra o Serial Monitor (115200 baud) para ver:

```
Estado: INICIALIZANDO
Modulo RFID MFRC522 inicializado.
1 sensores.
Endereco sensor: 28FF...
Connecting to Wifi SSID boanoite
.....
WiFi connected. IP address: 192.168.1.100

Visita confirmada.
Alerta em Cooldown.
```

---

## Conceitos Implementados

### 1. Maquina de Estados Finitos (FSM)
Organizacao clara do comportamento do sistema em estados bem definidos.

### 2. Filtro de Media Movel
Suaviza leituras do sensor de temperatura removendo ruido e oscilacoes.

### 3. Debouncing Temporal
Aguarda tempo minimo (5s) antes de confirmar presenca do gato via RFID.

### 4. Cooldown de Alertas
Evita spam de notificacoes aguardando 15 minutos entre alertas do mesmo tipo.

### 5. API REST via Telegram
Comunicacao com bot do Telegram usando requests HTTPS.

---

## Proximas Melhorias

- Adicionar sensor ultrassonico para medicao precisa de volume
- Implementar graficos de consumo no Telegram
- Suporte para multiplos gatos (varias tags RFID)
- Modo noturno (desabilitar alertas em horarios especificos)
- Integracao com Google Sheets para analise historica
- Bomba automatica para reabastecer agua
- Display LCD para visualizacao local

---

## Licenca

Projeto educacional de codigo aberto.

**Duvidas?** Entre em contato com os autores.
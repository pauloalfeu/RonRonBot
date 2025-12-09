# 🐱 RonRonBot - Monitor de Bebedouro para Gatos

Sistema de monitoramento inteligente de bebedouro para pets utilizando ESP32 com integração ao Telegram.

## 📋 Índice

- [Sobre o Projeto](#sobre-o-projeto)
- [Hardware Necessário](#hardware-necessário)
- [Bibliotecas](#bibliotecas)
- [Configuração Inicial](#configuração-inicial)
- [Estrutura do Código](#estrutura-do-código)
- [Funções Principais](#funções-principais)
- [Comandos do Telegram](#comandos-do-telegram)
- [Máquina de Estados](#máquina-de-estados)

---

## 🎯 Sobre o Projeto

O RonRonBot é um sistema automatizado que monitora o bebedouro do seu gato em tempo real, enviando notificações via Telegram sobre:

- **Nível de água** - Detecta quando está baixo
- **Temperatura** - Alerta quando está muito quente
- **Visitas do gato** - Conta quantas vezes seu pet bebeu água
- **Relatórios periódicos** - Envia atualizações a cada hora

---

## 🔧 Hardware Necessário

### Componentes Principais

| Componente | Quantidade | Função |
|------------|------------|--------|
| ESP32 | 1 | Microcontrolador principal |
| Sensor DS18B20 | 1 | Medição de temperatura da água |
| Sensores Capacitivos | 3 | Detecção de nível de água |
| Módulo RFID MFRC522 | 1 | Identificação do gato via tag |
| Tag RFID | 1 | Coleira do gato |
| LED | 1 | Indicação visual de status |

### Pinagem

```
ESP32          Componente
GPIO 4    -->  Sensor DS18B20 (temperatura)
GPIO 13   -->  LED de status
GPIO 5    -->  RFID RST
GPIO 15   -->  RFID SS (CS)
GPIO 6    -->  Sensor capacitivo 1 (superior)
GPIO 7    -->  Sensor capacitivo 2 (médio)
GPIO 8    -->  Sensor capacitivo 3 (inferior)
```

---

## 📚 Bibliotecas

Instale as seguintes bibliotecas na Arduino IDE:

```cpp
#include <WiFi.h>                    // Conexão WiFi
#include <WiFiClientSecure.h>        // Comunicação segura
#include <UniversalTelegramBot.h>    // API do Telegram
#include <OneWire.h>                 // Protocolo OneWire
#include <DallasTemperature.h>       // Sensor DS18B20
#include <SPI.h>                     // Comunicação SPI
#include <MFRC522.h>                 // Leitor RFID
#include <time.h>                    // Funções de tempo
```

---

## ⚙️ Configuração Inicial

### 1. Configurar WiFi

Edite as credenciais da sua rede:

```cpp
#define WIFI_SSID      "SuaRedeWiFi"
#define WIFI_PASSWORD  "SuaSenha"
```

### 2. Configurar Bot do Telegram

#### Criando o Bot:

1. Abra o Telegram e procure por `@BotFather`
2. Envie o comando `/newbot`
3. Escolha um nome e username para seu bot
4. Copie o **token** fornecido

#### No código:

```cpp
#define BOT_TOKEN      "SEU_TOKEN_AQUI"
const long ID_CHAT = SEU_ID_AQUI;
```

**Como descobrir seu ID do chat:**
- Envie uma mensagem para `@userinfobot`
- Ele retornará seu ID

### 3. Descobrir UID da Tag RFID

Execute o código e aproxime a tag do leitor. No Serial Monitor aparecerá:

```
UID: DE AD BE EF
```

Insira no código:

```cpp
const byte UID_GATO[] = {0xDE, 0xAD, 0xBE, 0xEF};
```

---

## 🏗️ Estrutura do Código

### Variáveis Globais Principais

```cpp
float temperatura_atual           // Temperatura lida pelo sensor
int visitas_hoje                  // Contador de visitas do dia
int visitas_ontem                 // Visitas do dia anterior
float temperatura_filtrada        // Média das últimas 5 leituras
```

### Constantes Configuráveis

```cpp
VALOR_MAX_TEMPERATURA = 24.0      // Temperatura máxima permitida (°C)
INTERVALO_RELATORIO_MS = 3600000  // Intervalo entre relatórios (1 hora)
COOLDOWN_ALERTA_MS = 900000       // Tempo mínimo entre alertas (15 min)
TEMPO_MINIMO_PERMANENCIA_MS = 5000 // Tempo para confirmar visita (5 seg)
```

---

## 🔧 Funções Principais

### 📡 Funções de Tempo

#### `String getTime()`
Retorna a data e hora completa formatada.

```cpp
// Exemplo de retorno: "09/12/2025 14:32:15"
String timestamp = getTime();
```

#### `String getTimeShort()`
Retorna apenas a hora no formato HH:MM.

```cpp
// Exemplo de retorno: "14:32"
String hora = getTimeShort();
```

#### `int getDiaAtual()`
Retorna o dia atual do mês (1-31).

```cpp
int dia = getDiaAtual(); // Retorna 9 se for dia 09
```

#### `void verificar_reset_diario()`
Verifica se mudou o dia e reseta os contadores.

**Como funciona:**
- Compara o dia atual com o dia anterior
- Se mudou: salva visitas de hoje em "ontem" e zera "hoje"
- Executada automaticamente no estado OCIOSO

---

### 🌡️ Funções de Sensores

#### `float ler_sensor_temperatura()`
Faz a leitura do sensor DS18B20.

```cpp
float temp = ler_sensor_temperatura();
// Retorna temperatura em Celsius
```

#### `int ler_nivel_agua()`
Lê os 3 sensores capacitivos e conta quantos detectaram água.

**Retorno:**
- `3` = Bebedouro cheio (100%)
- `2` = Nível médio (66%)
- `1` = Nível baixo (33%)
- `0` = Vazio

```cpp
int nivel = ler_nivel_agua();
if(nivel <= 1) {
    // Alerta de água baixa
}
```

#### `float calcular_temperatura_filtrada(float nova_leitura)`
Aplica filtro de média móvel nas últimas 5 leituras.

**Por que usar média móvel?**
- Remove ruídos e oscilações momentâneas
- Torna a leitura mais estável
- Evita alertas por variações rápidas

```cpp
float temp_bruta = ler_sensor_temperatura();
float temp_suave = calcular_temperatura_filtrada(temp_bruta);
```

---

### 📊 Funções de Formatação

#### `String obter_status_nivel(int contagem)`
Converte a contagem de sensores em texto descritivo.

```cpp
// Entrada: 3 → Saída: "Alto (100%)"
// Entrada: 2 → Saída: "Medio (66%)"
// Entrada: 1 → Saída: "Baixo (33%)"
// Entrada: 0 → Saída: "Vazio (0%)"
```

#### `int obter_percentual_nivel(int contagem)`
Converte contagem em percentual numérico.

```cpp
int percentual = obter_percentual_nivel(2); // Retorna 66
```

---

### 🔔 Funções de Alertas

#### `void adicionar_alerta(String tipo, String detalhe)`
Registra um alerta no histórico com timestamp.

**Parâmetros:**
- `tipo`: Categoria do alerta ("Agua Baixa", "Temperatura Alta")
- `detalhe`: Informação específica ("33%", "25.5 C")

**Funcionamento:**
- Armazena os últimos 10 alertas
- Quando chega no 11º, sobrescreve o mais antigo (buffer circular)

```cpp
adicionar_alerta("Agua Baixa", "33%");
// Registra: "Agua Baixa: 33%" com timestamp
```

---

### 🏷️ Funções RFID

#### `bool comparar_uid(byte *buffer1, const byte *buffer2, int size)`
Compara dois UIDs byte por byte.

```cpp
// Retorna true se forem idênticos
bool igual = comparar_uid(uid_lido, UID_GATO, 4);
```

#### `bool verificar_tag_rfid()`
Verifica se há uma tag presente e se é a do gato.

**Fluxo:**
1. Verifica se há algum cartão próximo
2. Tenta ler os dados do cartão
3. Compara com o UID cadastrado
4. Retorna `true` se for o gato

```cpp
if(verificar_tag_rfid()) {
    // O gato está perto do bebedouro!
}
```

---

### 🚀 Funções de Inicialização

#### `void init_sesores_cap()`
Configura os pinos dos sensores capacitivos.

**O que faz:**
- Define LED como saída
- Define sensores capacitivos como entrada com pull-up interno
- Acende o LED durante inicialização

#### `void init_rfid()`
Inicializa o módulo RFID MFRC522.

**Passo a passo:**
1. Inicia barramento SPI
2. Inicializa o leitor RFID
3. Confirma no Serial Monitor

#### `void init_ds18b20()`
Configura o sensor de temperatura DS18B20.

**Configurações:**
- Resolução: 12 bits (precisão de 0.0625°C)
- Modo: Sem espera (leitura assíncrona)
- Mostra endereço do sensor encontrado

#### `void init_wifi()`
Conecta à rede WiFi e sincroniza horário.

**Processo:**
1. Conecta ao WiFi configurado
2. Aguarda conexão (mostra pontos no Serial)
3. Configura certificado SSL para Telegram
4. Sincroniza com servidor NTP (hora de Brasília)
5. Exibe IP e hora atual

---

## 💬 Comandos do Telegram

### `/status` - Status Completo

Exibe todas as informações do bebedouro de uma vez.

**Exemplo de resposta:**
```
Status do Bebedouro

Nivel da agua: 66%
Temperatura: 23.5 C
Visitas hoje: 4

Atualizado: 14:32
```

**Quando usar:**
- Para verificação rápida completa
- Ao acordar/chegar em casa
- Antes de sair

---

### `/nivel` - Nível de Água

Mostra informações detalhadas sobre o nível.

**Exemplo de resposta:**
```
Nivel da Agua

Atual: 66% (Medio (66%))

[======----]

Nivel adequado.
```

**Elementos:**
- Percentual numérico
- Status descritivo
- Barra visual de 10 caracteres
- Avaliação do nível

**Quando usar:**
- Antes de sair de casa por muito tempo
- Para saber se precisa adicionar água

---

### `/temperatura` - Temperatura da Água

Verifica se a água está em temperatura adequada.

**Exemplo de resposta:**
```
Temperatura da Agua

Atual: 23.5 C
Limite max: 24.0 C

Temperatura ideal.
```

**Status possíveis:**
- "Temperatura acima do limite!" (> 24°C)
- "Temperatura proxima do limite." (22-24°C)
- "Temperatura ideal." (< 22°C)

**Quando usar:**
- Em dias muito quentes
- Se o bebedouro está exposto ao sol

---

### `/visitas` - Histórico de Visitas

Mostra quantas vezes o gato bebeu água.

**Exemplo de resposta:**
```
Historico de Visitas

Hoje: 4 visitas
Ontem: 6 visitas

Hidratacao normal!
```

**Análises automáticas:**
- 0 visitas: "Seu gato ainda nao bebeu agua hoje."
- 1-2 visitas: "Poucas visitas hoje."
- 3-5 visitas: "Hidratacao normal!"
- 6+ visitas: "Muito ativo hoje!"

**Quando usar:**
- Para monitorar hábitos do pet
- Detectar mudanças de comportamento
- Verificar hidratação adequada

---

### `/alertas` - Histórico de Alertas

Mostra os últimos alertas enviados pelo sistema.

**Exemplo de resposta (com alertas):**
```
Historico de Alertas

Ultimos alertas:

- Agua Baixa: 33%
  09/12/2025 12:30:15

- Temperatura Alta: 25.2 C
  09/12/2025 10:15:42

Total de alertas: 8
```

**Exemplo (sem alertas):**
```
Historico de Alertas

Nenhum alerta registrado!

Seu bebedouro esta funcionando perfeitamente.
```

**Informações exibidas:**
- Últimos 5 alertas
- Data e hora de cada ocorrência
- Total de alertas desde a inicialização

**Quando usar:**
- Para revisar problemas passados
- Identificar padrões (sempre alerta no mesmo horário?)
- Verificar se sistema está funcionando

---

### `/help` ou `/start` - Lista de Comandos

Mostra todos os comandos disponíveis.

**Exemplo de resposta:**
```
Ola, João! Eu sou o Monitor de Bebedouro.

Comandos disponiveis:

/status - Visao geral completa
/nivel - Nivel da agua
/temperatura - Temperatura
/visitas - Historico de idas
/alertas - Historico de alertas
```

**Quando usar:**
- Primeira interação com o bot
- Esqueceu algum comando

---

## 🤖 Notificações Automáticas

### Relatório Periódico

**Quando acontece:** A cada 1 hora (3600000 ms)

**Exemplo:**
```
Relatorio Periodico:
Nivel da agua: 66%
Temperatura: 23.5 C
Visitas hoje: 4
Horario: 14:00
```

**Configuração:** Altere `INTERVALO_RELATORIO_MS` para ajustar

---

### Alerta de Nível Baixo

**Quando acontece:** Quando 1 ou menos sensores detectam água (≤ 33%)

**Exemplo:**
```
ALERTA!

Nivel da agua baixo: 33%. Adicione mais agua
14:32
```

**Cooldown:** 15 minutos entre alertas do mesmo tipo

---

### Alerta de Temperatura Alta

**Quando acontece:** Temperatura ultrapassa 24°C

**Exemplo:**
```
ALERTA!

Temperatura da agua elevada: 25.2 C.
14:32
```

**Dica:** Ajuste o limite em `VALOR_MAX_TEMPERATURA`

---

## 🔄 Máquina de Estados

O sistema funciona como uma máquina de estados finitos. Cada estado executa uma função específica.

### Diagrama de Estados

```
INICIAL → OCIOSO ⟷ GATO_DETECTADO → MEDICAO → ENVIO_DADOS
                                          ↓
                                      ALERTA → OCIOSO
```

### Estados Detalhados

#### 1️⃣ ESTADO_INICIAL
**Executado:** Uma única vez ao ligar

**O que faz:**
- Inicializa todos os sensores
- Conecta ao WiFi
- Sincroniza horário
- Envia mensagem de boas-vindas ao Telegram
- Transita para OCIOSO

**LED:** Ligado durante inicialização

---

#### 2️⃣ ESTADO_OCIOSO
**Executado:** Maior parte do tempo

**O que faz:**
- Verifica reset diário (meia-noite)
- Aguarda detecção de tag RFID
- Conta tempo para próximo relatório

**Transições:**
- Se detectar tag → GATO_DETECTADO
- Se passou 1 hora → MEDICAO

**LED:** Desligado

---

#### 3️⃣ ESTADO_GATO_DETECTADO
**Executado:** Quando tag RFID é detectada

**O que faz:**
- Aguarda 5 segundos para confirmar presença
- Se confirmado: incrementa contador de visitas
- Transita para MEDICAO

**Por que 5 segundos?**
- Evita contagens falsas
- Garante que o gato realmente está bebendo

**LED:** Piscando (1 segundo ligado, 1 segundo desligado)

---

#### 4️⃣ ESTADO_MEDICAO
**Executado:** Após visita confirmada ou periodicamente

**O que faz:**
1. Lê nível de água
2. Lê temperatura
3. Aplica filtro de média móvel
4. Verifica condições de alerta

**Transições:**
- Se alerta necessário → ALERTA
- Se passou 1 hora → ENVIO_DADOS
- Caso contrário → OCIOSO

**LED:** Ligado durante medição

---

#### 5️⃣ ESTADO_ENVIO_DADOS
**Executado:** A cada 1 hora

**O que faz:**
- Compila relatório completo
- Envia mensagem ao Telegram
- Reseta timer de relatório
- Transita para OCIOSO

**Formato do relatório:**
```
Relatorio Periodico:
Nivel da agua: 66%
Temperatura: 23.5 C
Visitas hoje: 4
Horario: 14:00
```

**LED:** Desligado

---

#### 6️⃣ ESTADO_ALERTA
**Executado:** Quando detecta problema

**O que faz:**
1. Verifica cooldown (15 min desde último alerta)
2. Monta mensagem específica do problema
3. Registra no histórico de alertas
4. Envia ao Telegram
5. Retorna para OCIOSO

**Condições de alerta:**
- Nível ≤ 33% (1 sensor ou menos)
- Temperatura > 24°C

**LED:** Piscando rápido (0.5s ligado, 0.5s desligado)

---

## 🔍 Fluxo de Execução

### Loop Principal

```cpp
void loop() {
    // 1. Verifica mensagens do Telegram (a cada 1 segundo)
    if (WiFi conectado && passou 1 segundo) {
        processar_mensagens_telegram();
    }
    
    // 2. Executa estado atual da máquina
    fsm(); // Finite State Machine
}
```

### Processamento de Comandos

1. Bot recebe mensagem
2. Verifica se é do chat autorizado (ID_CHAT)
3. Identifica comando
4. Executa ação correspondente
5. Envia resposta formatada

---

## 📝 Customizações Comuns

### Alterar Intervalo de Relatórios

```cpp
// Para 30 minutos (1800000 ms)
const unsigned long INTERVALO_RELATORIO_MS = 1800000;

// Para 2 horas (7200000 ms)
const unsigned long INTERVALO_RELATORIO_MS = 7200000;
```

### Alterar Temperatura Máxima

```cpp
// Para 26°C
const float VALOR_MAX_TEMPERATURA = 26.0;
```

### Alterar Cooldown de Alertas

```cpp
// Para 30 minutos (1800000 ms)
const unsigned long COOLDOWN_ALERTA_MS = 1800000;

// Para 5 minutos (300000 ms) - mais alertas
const unsigned long COOLDOWN_ALERTA_MS = 300000;
```

### Alterar Tempo de Confirmação de Visita

```cpp
// Para 3 segundos (3000 ms)
const unsigned long TEMPO_MINIMO_PERMANENCIA_MS = 3000;

// Para 10 segundos (10000 ms) - mais rigoroso
const unsigned long TEMPO_MINIMO_PERMANENCIA_MS = 10000;
```

---

## 🐛 Troubleshooting

### Bot não responde

**Verificar:**
1. Token do bot está correto?
2. ID do chat está correto?
3. WiFi está conectado? (veja Serial Monitor)
4. Certificado SSL está configurado?

**Solução temporária:**
```cpp
// Na função init_wifi(), adicione antes de setCACert:
secured_client.setInsecure(); // APENAS PARA TESTES!
```

### Sensor de temperatura retorna -127°C

**Causa:** Sensor não encontrado ou mal conectado

**Verificar:**
- Conexão do pino de dados (GPIO 4)
- Resistor pull-up de 4.7kΩ entre dados e VCC
- Alimentação 3.3V ou 5V

### RFID não detecta tag

**Verificar:**
- Conexões SPI (MOSI, MISO, SCK, SS, RST)
- Alimentação do módulo (3.3V)
- UID da tag está correto no código

**Descobrir UID:**
```cpp
// Adicione no loop temporariamente:
if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    Serial.print("UID: ");
    for (byte i = 0; i < 4; i++) {
        Serial.print(mfrc522.uid.uidByte[i], HEX);
        Serial.print(" ");
    }
    Serial.println();
}
```

### Sensores capacitivos sempre retornam 0

**Verificar:**
- Pinos configurados como INPUT_PULLUP
- Sensores estão com jumper em modo digital (não analógico)
- Distância entre sensor e água (máx ~5mm através do plástico)

---

## 📊 Monitoramento via Serial

Durante execução, o Serial Monitor (115200 baud) mostra:

```
=== Iniciando Sistema ===
Modulo RFID MFRC522 inicializado.
1 sensores.
Endereco sensor: 28FF...
Connecting to Wifi SSID boanoite
.....
WiFi connected. IP address: 192.168.1.100
Hora atual: 09/12/2025 14:32:15

Estado: INICIALIZANDO
Visita confirmada.
Alerta em Cooldown.
Reset diario realizado.
```

---

## 🎓 Conceitos Aprendidos

### 1. Máquina de Estados Finitos (FSM)
Organização de código complexo em estados bem definidos, facilitando manutenção.

### 2. Filtro de Média Móvel
Técnica para suavizar leituras de sensores e remover ruídos.

### 3. Buffer Circular
Estrutura de dados eficiente para armazenar histórico limitado (alertas).

### 4. Debouncing Temporal
Aguardar tempo mínimo antes de confirmar evento (visita do gato).

### 5. API RESTful
Comunicação com serviços web (Telegram) via HTTP/HTTPS.

---

## 👥 Autores

**Ana Caroline Pedrosa & Paulo Alfeu**

---

## 📄 Licença

Projeto educacional de código aberto.

---

## 🚀 Próximas Melhorias

- [ ] Adicionar sensor ultrassônico para medir volume exato
- [ ] Implementar gráfico de consumo no Telegram
- [ ] Suporte para múltiplos gatos (várias tags)
- [ ] Modo noturno (desabilitar alertas)
- [ ] Integração com Google Sheets para análise de dados
- [ ] Bomba automática para reabastecer água

---

**Dúvidas?** Entre em contato ou abra uma issue no repositório!
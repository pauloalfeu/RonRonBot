/*
  Ana Caroline Pedrosa & Paulo Alfeu
  Monitoramento de Agua para Gato - Versao em Portugues
*/

// Bibliotecas
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <SPI.h>
#include <MFRC522.h>

//Conexao com o bot
#define WIFI_SSID      "boanoite"
#define WIFI_PASSWORD  "12345678"
#define BOT_TOKEN      "8598587149:AAEZWik67FU_ZsX3SMyIhlHG_fZVMslK_2A"
const long ID_CHAT = 1098236355; 

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);
const unsigned long INTERVALO_VERIFICACAO_MS = 1000;
unsigned long ultima_verificacao_bot;

// Pinagem dos sensores
#define PINO_DS18B20    4    // Sensor de temperatura
#define PINO_LED_STATUS 13   // LED para indicacao de estado
#define PINO_RFID_RST   5    // Pino de Reset do RFID
#define PINO_RFID_SS    15   // Pino Slave Select do RFID (CS)

// Sensores de para o nivel de agua - Capacitivos
const int TAMANHO_SENSORES = 3;
const int PINOS_SENSORES[TAMANHO_SENSORES] = {6, 7, 8}; 
int leituras_sensores[TAMANHO_SENSORES];

// Sensor de temperatura
OneWire barramento(PINO_DS18B20);
DallasTemperature sensores_temperatura(&barramento);
DeviceAddress endereco_sensor1;

// Tag - para detectar se o gato esta perto
MFRC522 mfrc522(PINO_RFID_SS, PINO_RFID_RST);
const byte UID_GATO[] = {}; //verificar o ID da tag
const int TAMANHO_UID = 4;

// Limites - para verificar as variaveis
const float VALOR_MAX_TEMPERATURA = 24.0; 
const unsigned long INTERVALO_RELATORIO_MS = 3600000; // 1 hora
const unsigned long COOLDOWN_ALERTA_MS = 900000; // 15 minutos

// variaveis de controle
float temperatura_atual = 0.0;
unsigned long tempo_ultima_leitura = 0;
unsigned long tempo_ultimo_alerta = 0;
int visitas_hoje = 0;

// Variaveis para controlar o detected do gato
bool gato_confirmado = false; 
unsigned long tempo_ultima_deteccao = 0;
const unsigned long TEMPO_MINIMO_PERMANENCIA_MS = 5000; 

// media movel da temperatura
const int TAMANHO_JANELA_FILTRO = 5; 
float historico_temperatura[TAMANHO_JANELA_FILTRO] = {0.0};
int indice_filtro_temp = 0;
float temperatura_filtrada = 0.0;


// Estados do bebedouro para gatos
enum EstadoBebedouro{
  ESTADO_INICIAL = 0,
  ESTADO_OCIOSO = 1,
  ESTADO_GATO_DETECTADO = 2,
  ESTADO_MEDICAO = 3,
  ESTADO_ENVIO_DADOS = 4,
  ESTADO_ALERTA = 5,
};

EstadoBebedouro estado_atual = ESTADO_INICIAL;

// Funcoes auxiliares para o beberouro
float calcular_temperatura_filtrada(float nova_leitura) {
    historico_temperatura[indice_filtro_temp] = nova_leitura;
    indice_filtro_temp = (indice_filtro_temp + 1) % TAMANHO_JANELA_FILTRO;
    
    float soma = 0.0;
    for (int i = 0; i < TAMANHO_JANELA_FILTRO; i++) {
        soma += historico_temperatura[i];
    }
    temperatura_filtrada = soma / TAMANHO_JANELA_FILTRO;
    return temperatura_filtrada;
}

int ler_nivel_agua() {
    int nivel = 0; 
    for(int i = 0; i < TAMANHO_SENSORES; i++){
        leituras_sensores[i] = digitalRead(PINOS_SENSORES[i]); 
        if(leituras_sensores[i] == LOW) { 
            nivel++;
        }
    }
    return nivel; 
}

float ler_sensor_temperatura() {
    sensores_temperatura.requestTemperatures();
    delay(100); 
    temperatura_atual = sensores_temperatura.getTempCByIndex(0);
    return temperatura_atual;
}

String obter_status_nivel(int contagem) {
    if (contagem == 3){
      return "Alto (100%)";
    } 
    if (contagem == 2) {
      return "Medio (66%)";
    }
    if (contagem == 1) {
      return "Baixo (33%)";
      }

    return "Vazio (0%)"; 
}

// Funcoes RFID
bool comparar_uid(byte *buffer1, const byte *buffer2, int size) {
    for (int i = 0; i < size; i++) {
        if (buffer1[i] != buffer2[i]) {
            return false;
        }
    }
    return true;
}

bool verificar_tag_rfid() {
    if (!mfrc522.PICC_IsNewCardPresent()) {
        return false;
    }
    if (!mfrc522.PICC_ReadCardSerial()) {
        return false;
    }

    if (comparar_uid(mfrc522.uid.uidByte, UID_GATO, TAMANHO_UID)) {
        mfrc522.PICC_HaltA();
        return true;
    }
    
    mfrc522.PICC_HaltA();
    return false;
}
//Funcoes para inicializacao dos sessores e do wifi

void init_sesores_cap(){
  pinMode(PINO_LED_STATUS, OUTPUT);
  digitalWrite(PINO_LED_STATUS, HIGH);

  for(int i = 0; i < TAMANHO_SENSORES; i++){
      pinMode(PINOS_SENSORES[i], INPUT_PULLUP);
  }
}

void init_rfid(){
  SPI.begin();       
  mfrc522.PCD_Init(); 
  Serial.println("Modulo RFID MFRC522 inicializado.");
}

void init_ds18b20() {
  sensores_temperatura.begin();

  Serial.print(sensores_temperatura.getDeviceCount(), DEC);
  Serial.println(" sensores.");
  if (!sensores_temperatura.getAddress(endereco_sensor1, 0))
    Serial.println("Sensores nao encontrados !");
  // Mostra o endereco do sensor encontrado no barramento
  Serial.print("Endereco sensor: ");

  sensores_temperatura.getAddress(endereco_sensor1, 0);
  const int resolution = 12;
  sensores_temperatura.setResolution(endereco_sensor1, resolution);
  sensores_temperatura.setWaitForConversion(false);
}

void init_wifi() {
  Serial.print("Connecting to Wifi SSID ");
  Serial.print(WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  secured_client.setCACert(TELEGRAM_CERTIFICATE_ROOT);  // Add root certificate for api.telegram.org
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.print("\nWiFi connected. IP address: ");
  Serial.println(WiFi.localIP());
  Serial.println();
}


//Funcoes referentes aos estados do bebedouro, cada funcao e responsavel por executar uma acao e passar para o priximo estado
void estado_inicial() {
    Serial.println("Estado: INICIALIZANDO");
    
    init_sesores_cap();
    init_rfid();
    init_ds18b20();
    init_wifi();
    bot.sendMessage(ID_CHAT, "Sistema de Monitoramento de RonRonWater Conectado!", "");
    
    digitalWrite(PINO_LED_STATUS, LOW);
    tempo_ultima_leitura = millis(); //passa um tempo ate para o proximo estado inciar
    estado_atual = ESTADO_OCIOSO;
}

void estado_ocioso() {
    digitalWrite(PINO_LED_STATUS, LOW);

    //fica verificando se ha sinal da tag do gato
    if (verificar_tag_rfid()) {
        tempo_ultima_deteccao = millis();
        estado_atual = ESTADO_GATO_DETECTADO;
        return;
    }

    // Fica no aguardo e quando da o tempo envia o relatorio 
    if (millis() - tempo_ultima_leitura >= INTERVALO_RELATORIO_MS) {
        estado_atual = ESTADO_MEDICAO;
        return;
    }
}

void estado_gato_detectado() {
    digitalWrite(PINO_LED_STATUS, (millis() % 1000 < 500) ? HIGH : LOW);

    if (millis() - tempo_ultima_deteccao >= TEMPO_MINIMO_PERMANENCIA_MS) {
        if (!gato_confirmado) {
            visitas_hoje++;
            Serial.println("Visita confirmada.");
            gato_confirmado = true; 
        }
        
        estado_atual = ESTADO_MEDICAO;
        gato_confirmado = false;
        return;
    }
}

void estado_medicao() {
    digitalWrite(PINO_LED_STATUS, HIGH);

    // Verifica o nivel de agua
    int contagem_molhados = ler_nivel_agua();
    
    // Le e filtra a temperatura
    float temp_bruta = ler_sensor_temperatura();
    float temp_filtrada = calcular_temperatura_filtrada(temp_bruta);
    
    // verificar se o ultomo sensor esta em alerta
    if (contagem_molhados <= 1 || temp_filtrada > VALOR_MAX_TEMPERATURA) {
        estado_atual = ESTADO_ALERTA;
        return;
    }

    if (millis() - tempo_ultima_leitura >= INTERVALO_RELATORIO_MS) {
        estado_atual = ESTADO_ENVIO_DADOS;
    } else {
        estado_atual = ESTADO_OCIOSO; 
    }
}

void estado_envio_dados() {
    digitalWrite(PINO_LED_STATUS, LOW);

    int contagem_molhados = ler_nivel_agua(); 
    String status_nivel = obter_status_nivel(contagem_molhados);

    String mensagem = "*Relatorio Periodico:*\n";
    mensagem += "Nivel da agua: " + status_nivel + "\n";
    mensagem += "Temperatura: " + String(temperatura_filtrada, 1) + " C\n";
    mensagem += "Visitas hoje: " + String(visitas_hoje);
    
    bot.sendMessage(ID_CHAT, mensagem, "Markdown");
    
    tempo_ultima_leitura = millis();
    estado_atual = ESTADO_OCIOSO;
}

void estado_alerta() {
    digitalWrite(PINO_LED_STATUS, (millis() % 500 < 250) ? HIGH : LOW);

    if (millis() - tempo_ultimo_alerta >= COOLDOWN_ALERTA_MS) {
        String mensagem_alerta = "*ALERTA!* ";
        
        int contagem_molhados = ler_nivel_agua();
        float temp_filtrada = temperatura_filtrada;

        if (contagem_molhados <= 1) { 
            mensagem_alerta += "\nNivel da agua baixo: " + obter_status_nivel(contagem_molhados) + ". Adicione mais agua";
        }
        if (temp_filtrada > VALOR_MAX_TEMPERATURA) {
            mensagem_alerta += "\nTemperatura da agua elevada: " + String(temp_filtrada, 1) + " C.";
        }
        
        bot.sendMessage(ID_CHAT, mensagem_alerta, "Markdown");
        tempo_ultimo_alerta = millis(); 
    } else {
        Serial.println("Alerta em Cooldown.");
    }
    
    estado_atual = ESTADO_OCIOSO;
}

void fsm(){
  switch (estado_atual) {
    case ESTADO_INICIAL: 
      estado_inicial(); 
    break;
    case ESTADO_OCIOSO: 
      estado_ocioso(); 
    break;
    case ESTADO_GATO_DETECTADO: 
      estado_gato_detectado(); 
    break;
    case ESTADO_MEDICAO: 
      estado_medicao(); 
    break;
    case ESTADO_ENVIO_DADOS: 
      estado_envio_dados(); 
    break;
    case ESTADO_ALERTA: 
      estado_alerta(); 
    break;
  }
}

void setup() {
  Serial.begin(115200); 
}

void loop(){
  // --- TELEGRAM ---
  if (WiFi.status() == WL_CONNECTED && millis() - ultima_verificacao_bot > INTERVALO_VERIFICACAO_MS){
    int numNovasMensagens = bot.getUpdates(bot.last_message_received + 1);

    while (numNovasMensagens)
    {
      tratar_novas_mensagens(numNovasMensagens);
      numNovasMensagens = bot.getUpdates(bot.last_message_received + 1);
    }
    ultima_verificacao_bot = millis();
  }
  
  fsm();
}


void tratar_novas_mensagens(int numNovasMensagens)
{
    for (int i = 0; i < numNovasMensagens; i++)
    {
        long id_chat = bot.messages[i].chat_id;
        String texto = bot.messages[i].text;
        String nome_usuario = bot.messages[i].from_name;

        if (id_chat != ID_CHAT) continue; 

        String resposta = "";
        int contagem_nivel = ler_nivel_agua();
        String status_nivel = obter_status_nivel(contagem_nivel);

        if (texto == "/status") {
            ler_sensor_temperatura(); 
            
            resposta = "*Status do Bebedouro*\n\n";
            resposta += "Nivel da agua: " + status_nivel + "\n";
            resposta += "Temperatura: " + String(temperatura_filtrada, 1) + " C\n";
            resposta += "Visitas hoje: " + String(visitas_hoje);
            
            bot.sendMessage(id_chat, resposta, "Markdown");
            
        } 
        else if (texto == "/nivel") {
            resposta = "Nivel atual da agua: " + status_nivel;
            bot.sendMessage(id_chat, resposta, "");

        } 
        
        else if (texto == "/temperatura") {
            ler_sensor_temperatura(); 
            
            resposta = "Temperatura atual: " + String(temperatura_filtrada, 1) + " C";
            
            bot.sendMessage(id_chat, resposta, "");

        } 
      
        else if (texto == "/visitas") {
            resposta = "Visitas hoje: " + String(visitas_hoje);
            
            bot.sendMessage(id_chat, resposta, "");
        } 
       
        else if (texto == "/alertas") {
            resposta = "Configuracao de alertas:\n";
            resposta += "Temperatura maxima: " + String(VALOR_MAX_TEMPERATURA, 1) + " C\n";
            resposta += "Alerta quando nivel <= 33%";
            
            bot.sendMessage(id_chat, resposta, "");
        }
        // Comando /start ou desconhecido
        else if (texto == "/start" || texto == "/help") {
            resposta = "Ola, " + nome_usuario + "! Eu sou o Monitor de Bebedouro.\n\n";
            resposta += "Comandos disponiveis:\n";
            resposta += "/status - Visao geral completa\n";
            resposta += "/nivel - Nivel da agua\n";
            resposta += "/temperatura - Temperatura\n";
            resposta += "/visitas - Historico de idas\n";
            resposta += "/alertas - Limites de seguranca";
            bot.sendMessage(id_chat, resposta, "");
        }
    }
}
🐱 Monitoramento de Água para Gato (RonRonWater)
================================================

Olá! Este é o projeto desenvolvido por **Ana Caroline Pedrosa** e **Paulo Alfeu**.

Criamos um sistema para monitorar o bebedouro do nosso gato, garantindo que a água esteja fresca e disponível. O código utiliza um microcontrolador com conexão Wi-Fi para ler sensores e nos manter informados através de um **Bot no Telegram**.

⚙️ Componentes e Pinagem (Hardware)
-----------------------------------

Baseado nas definições do nosso código, o sistema está configurado para utilizar os seguintes pinos e sensores:

*   **Sensor de Temperatura (DS18B20):** Conectado no pino **4**.
    
*   **Sensores de Nível de Água (Capacitivos):** Usamos 3 sensores conectados nos pinos **6, 7 e 8**.
    
    *   O código entende que o nível subiu quando o sensor lê LOW.
        
*   **Sensor RFID (MFRC522):** Para identificar a presença do gato.
    
    *   Pino Reset (RST): **5**
        
    *   Pino Slave Select (SS/CS): **15**
        
*   **LED de Status:** No pino **13** (pisca ou acende dependendo do estado do sistema).
    

🧠 Como o Código Funciona (Lógica Interna)
------------------------------------------

Organizamos o comportamento do bebedouro utilizando uma **Máquina de Estados Finitos (FSM)**. O sistema alterna entre 6 modos diferentes:

1.  **ESTADO\_INICIAL:**
    
    *   Inicia os sensores (Nível, RFID, Temperatura) e conecta ao Wi-Fi (rede definida como _"boanoite"_).
        
    *   Envia a mensagem "Sistema de Monitoramento de RonRonWater Conectado!" para o Telegram.
        
2.  **ESTADO\_OCIOSO:**
    
    *   O sistema fica aguardando. Se o sensor RFID ler uma tag, ele vai para o estado de detecção.
        
    *   Se passar 1 hora (INTERVALO\_RELATORIO\_MS), ele vai para o estado de medição para enviar um relatório periódico.
        
3.  **ESTADO\_GATO\_DETECTADO:**
    
    *   Verifica se o gato permanece no bebedouro por pelo menos **5 segundos** (TEMPO\_MINIMO\_PERMANENCIA\_MS).
        
    *   Se confirmado, incrementamos o contador de visitas\_hoje.
        
4.  **ESTADO\_MEDICAO:**
    
    *   Lê quantos sensores de nível estão ativos.
        
    *   Lê a temperatura e aplica um **filtro de média móvel** (média das últimas 5 leituras) para estabilizar o valor.
        
    *   Decide se vai para o estado de **ALERTA** (se houver problema) ou **ENVIO\_DADOS** (se for hora do relatório).
        
5.  **ESTADO\_ALERTA:**
    
    *   Ativado se: Nível de água for muito baixo (1 sensor ou menos) OU Temperatura for maior que **24.0°C**.
        
    *   Envia mensagem urgente no Telegram.
        
    *   Possui um "Cooldown" de 15 minutos (COOLDOWN\_ALERTA\_MS) para não enviar alertas repetidos.
        
6.  **ESTADO\_ENVIO\_DADOS:**
    
    *   Envia o relatório padrão e zera o cronômetro para o próximo ciclo.
        

📱 Comandos do Telegram
-----------------------

No código, configuramos a função tratar\_novas\_mensagens para responder aos seguintes comandos enviados pelo chat:

**ComandoResposta do Sistema**/start ou /helpApresenta o bot e lista todos os comandos disponíveis./status**Relatório Completo:** Lê a temperatura na hora e retorna o Nível (texto), Temperatura (°C) e contagem de Visitas do dia./nivelInforma apenas o status da água: "Alto (100%)", "Médio (66%)", "Baixo (33%)" ou "Vazio (0%)"./temperaturaLê o sensor e retorna a temperatura atual filtrada em graus Celsius./visitasRetorna o número armazenado na variável visitas\_hoje./alertasMostra as configurações de segurança atuais: Temp. Máxima (24°C), Nível Mínimo (33%), Intervalo de Relatórios (60 min) e Cooldown (15 min).

📝 Configurações Necessárias
----------------------------

Para o código funcionar, existem algumas variáveis globais definidas no início que precisam de atenção (atualmente estão com valores de exemplo):

*   **Wi-Fi:**
    
    *   SSID: "boanoite"
        
    *   Senha: "12345678"
        
*   **Telegram:**
    
    *   Token do Bot: "not today satan" (Isso precisa ser alterado para um token real).
        
    *   ID do Chat: 123456789 (Deve ser o ID do usuário que receberá as mensagens).
        
*   **RFID:**
    
    *   UID\_GATO: A lista está vazia {}. É necessário colocar os bytes da tag do seu gato aqui para o reconhecimento funcionar.
        

_Projeto desenvolvido por Ana Caroline Pedrosa & Paulo Alfeu._
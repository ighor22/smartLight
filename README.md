# SmartLight
Projeto acadêmico que visa reduzir o consumo de energia elétrica nas casas. O usuário é alertado via telegram quando esquecer a luz acesa em um cômodo e consegue controlar a lâmpada remotamente via protocolo MQTT, podendo apagá-la ou acendê-la.

## Hardware 

Lista de componentes uitilizados para a construção do projeto:
*	1 Protoboard – 830 pontos
* 1 NodeMCU(ESP8266)
*	1 PIR
*	1 Resistor 10K
*	1 LDR 5mm
*	1 Módulo Rele (5V – 1 módulo)
*	1 Fonte P4 8V
*	1 Fonte ajustável para protoboard
*	3 Resistores 220R
*	3 LEDs
*	Jumpers Macho/Macho
*	Jumpers Macho/Fêmea

## Software
 Podemos ter uma ideia melhor de seu funcionamento ao analisar o fluxograma do aplicativo:

![Fluxograma do SmartLight](https://github.com/ighor22/smartLight/blob/master/Fluxograma%20SmartLight.png)
A seguir, um fluxograma especificamente para a parte da comunicação realizada pelo MQTT: 

![Fluxograma do MQTT](https://github.com/ighor22/smartLight/blob/master/Fluxograma%20MQTT.png)

O código a seguir foi programado em C++ utilizando a ArduinoIDE

```c++
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <NTPClient.h>
#include <WiFiUdp.h>

//Configuração NTP
const long utcOffsetInSeconds = -10800; //Fuso horario (-3*60*60)
char daysOfTheWeek[7][12] = {"Domingo", "Segunda", "Terça", "Quarta", "Quinta", "Sexta", "Sabado"};
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", utcOffsetInSeconds);

//Configuração Bot Telegram
#define BOTtoken "*******************************"
WiFiClientSecure clientBot;
UniversalTelegramBot bot(BOTtoken, clientBot);

//Usuario e senha do WiFi
const char *ssid = "*********";
const char *password = "*******";

// Dados do dispositivo no Watson Iot
const String ORG = "*****";
const String DEVICE_TYPE = "******";
const char* DEVICE_ID = "******";
#define DEVICE_TOKEN "********"

//Definições dos pinos utilizados 
int led1 = 5; //pino D1
int PIR = 15;  //pino D8
int led2 = 4; //pino D2
int led3 = 12; //D6
int rele = 14; //D5

//Variaveis globais
int contador = 0;
boolean msgEnviada = false;

//Variaveis para customização 
int horarioAtivacao = 19; //Hora que o sistema comecará a notificar o usuário caso a luz esteja acesa
int tempoSemPresenca = 900; // Em segundos a quantidade de tempo sem detecção de movimento no ambiente 

//Configurações MQTT
const String CLIENT_ID = "d:" + ORG + ":" + DEVICE_TYPE + ":" + DEVICE_ID;
const String MQTT_SERVER = ORG + ".messaging.internetofthings.ibmcloud.com";
#define topicoLampada1 "iot-2/cmd/lampada1/fmt/json" //Define o topico que será inscrito
WiFiClient wifiClient;
PubSubClient client(MQTT_SERVER.c_str(), 1883, wifiClient);

void setup() {
   Serial.begin(9600);   
   pinMode(led1, OUTPUT);
   pinMode(PIR, INPUT); 
   pinMode(led2, OUTPUT);
   pinMode(led3, OUTPUT);
   pinMode(rele, OUTPUT);

  clientBot.setInsecure();  //Ajuste para funcionamento do bot do telegram
  
  //Configuração do wifi
   int timeout = 0;
   WiFi.begin(ssid, password);
  Serial.print("Conectando NodeMCU ao ");
  Serial.print(ssid);  
  Serial.print(' '); 
  while(WiFi.status() != WL_CONNECTED){
     
      delay(1000);
      timeout++;
             
      Serial.print('.');
      if(timeout == 60){
        timeout = 0;
        Serial.print('\n');
        Serial.println("Falha ao conectar-se!");
        Serial.println("Tentando Novamente");
        Serial.print('\n');
        Serial.print("Conectando NodeMCU ao ");
        Serial.print(ssid);
        Serial.print(' '); 
       }
    }
 
    Serial.print('\n');
    Serial.println("Conectado ao WiFi!");
    Serial.print("Meu IP:\t");
    Serial.println(WiFi.localIP());
    //Substituir ***** pelo id do usuario no telegram
    bot.sendMessage("***********", "Iniciando sistema!", ""); //Inicio do Bot do Telegram
    timeClient.begin(); //Inicio do NTP
    connectMQTTServer(); //Conexão ao servidor MQTT
    
}

void loop() {
  
  //Atualização e print da hora atual
  timeClient.update();
  Serial.print(daysOfTheWeek[timeClient.getDay()]);
  Serial.print(", ");
  Serial.print(timeClient.getHours());
  Serial.print(":");
  Serial.print(timeClient.getMinutes());
  Serial.print(":");
  Serial.println(timeClient.getSeconds());

  client.loop(); //loop para verificar se houveram mudanças nos topicos do MQTT

 //Leitura e conversão dos dados captados pelo fotorresistor
 int LDR = analogRead(A0);  
 float voltage = LDR * (5.0 / 1023.0);   // Converte valor analogico (0 - 1023) para voltagem (0 - 5V)
  Serial.println(voltage);   

  //Caso esteja escuro o ambiente, reinicia o contador
  if (voltage <=1){
    digitalWrite(led1, HIGH); //Acende led para debug
    contador= 0;
    msgEnviada = false; //Define variavel global para falso para que, caso detecte a luz acesa, mande a mensagem para o usuario
  } else{
      digitalWrite(led1, LOW); //Apaga led para debug
  }

  //Realiza leitura dos dados do PIR, verificando se há movimentação no ambiente
  long state = digitalRead(PIR);
  //Caso tenha movimentação, reinicia o contador
    if(state == HIGH) {
      digitalWrite (led2, HIGH); //Acende led para debug
      Serial.println("Movimento Detectado!");
      delay(1000);
      contador = 0; 
      msgEnviada = false; //Define variavel global para falso para que, caso detecte a luz acesa, mande a mensagem para o usuario
    }
    //Caso não tenha movimentação, acrescenta 1 segundo no contador
    else {
      digitalWrite (led2, LOW);
      Serial.println("Nenhum Movimento Detectado!");
      //Delay de 1 segundo seguido do incremento do contador, acrescentando 1 segundo
      delay(1000);
      }
      contador+=1;
      Serial.println(contador);
      
}


//Verifica tempo sem movimento, nivel de claridade e horario atual. 
if(contador >= tempoSemPresenca && voltage >1 && timeClient.getHours() >= horarioAtivacao){
  digitalWrite(led3, HIGH); //Acende led para debug
  //Caso não tenha enviado a mensagem, envia
  if(msgEnviada == false){
    Serial.println("Enviando mensagem...");
    //Substituir **** por id do usuario telegram
    bot.sendMessage("*******", "Você esqueceu a luz acesa! Acesse https://iknodered.mybluemix.net/ui/ para apagá-la.", "");
    msgEnviada = true; //Define variavel global para true para não mandar mensagem novmente a cada iteração
  }
}
else{
  digitalWrite(led3, LOW); //Apaga led para debug
}

 //Função responsavel por reiniciar a contagem
void reiniciaContagem(){
  delay(1000);
  contador = 0;
  msgEnviada = false; //Define variavel global para falso para que, caso detecte a luz acesa, mande a mensagem para o usuario
}

//Função responsável pela conexão ao servidor MQTT
void connectMQTTServer() {
  Serial.println("Connectando ao servidor MQTT...");
  if (client.connect(CLIENT_ID.c_str(), "use-token-auth", DEVICE_TOKEN)) { //Caso consiga se conectar com esses dados
    Serial.println("Conectado ao Broker MQTT");
    client.setCallback(callback); //Define callback
    client.subscribe(topicoLampada1);  //Se inscreve no tópico
  } else {
    //Printa o tipo do erro
     Serial.print("erro = ");
     Serial.println(client.state());
     connectMQTTServer(); //chama a função novamente para outra tentativa
  }
}

//Função callback para o topico do MQTT (ativa sempre que houver alteração/mensagem)
void callback(char* topic, unsigned char* payload, unsigned int length) {
  //Printa o topico que teve alteração
    Serial.print("topic ");
    Serial.println(topic);
    
    //Realiza a desserialização do JSON recebido 
    StaticJsonBuffer<30> jsonBuffer;
    JsonObject& root = jsonBuffer.parseObject(payload);

    //Caso dê erro na desserialização indica que houve um erro
    if(!root.success())
    {
        Serial.println("Json Parse Error");
        return;
    }

  //Pega o atributo "value" do json
    int value = root["value"];
    //Verifica se a string topic é a mesma da string topicoLampada1
    if (strcmp(topic, topicoLampada1) == 0)
    {
        digitalWrite(rele, value); //altera o pino do rele com o valor recebido pelo topico
        reiniciaContagem();
    }
    
}
```


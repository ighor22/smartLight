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

## Funcionamento
 O sistema funciona da seguinte maneira: O sensor PIR juntamente com o LDR verificam constantemente o nível de claridade no ambiente e se há algum tipo de presença no local. Após 15 minutos sem detectar presença nesse ambiente, o sistema envia uma mensagem via Telegram para o usuario, informando que o mesmo esqueceu a luz acesa, enviando junto um link que permite ao usuário controlar a lâmpada a distância. Esse controle da lâmpada é feito via protocolo MQTT, e utilizando o broker IBM Cloud. Para que ele não fique notificando o usuário de dia (já que vai estar iluminado naturalmente), o sistema permite que você escolha o horario em que ele deve começar a notificá-lo. Por padrão, é a partir das 19 horas.


## Software
 Podemos ter uma ideia melhor de seu funcionamento ao analisar o fluxograma do aplicativo:

![Fluxograma do SmartLight](https://github.com/ighor22/smartLight/blob/master/Fluxograma%20SmartLight.png)
A seguir, um fluxograma especificamente para a parte da comunicação realizada pelo MQTT: 

![Fluxograma do MQTT](https://github.com/ighor22/smartLight/blob/master/Fluxograma%20MQTT.png)
O código a seguir foi programado em C utilizando a ArduinoIDE

Iniciamos a codiificação com a inclusão de todas as bibliotecas utilizadas, para conexão com o wifi, o funcionamento do MQTT, para o uso do BOT do Telegram e configuração do horário.
```c
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
```
Em seguida, configuramos o NTP: Em uma variável foi armazenado o fuso horário do Brasil, em segundos, e em um array foram armazenados os dias da semana e, com o uso de funções disponíveis na biblioteca importada, configuramos o NTP
```c
//Configuração NTP
const long utcOffsetInSeconds = -10800; //Fuso horario (-3*60*60)
char daysOfTheWeek[7][12] = {"Domingo", "Segunda", "Terça", "Quarta", "Quinta", "Sexta", "Sabado"};
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", utcOffsetInSeconds);
```

Depois configuramos o BOT do Telegram, onde recebemos um Token, que será armazenado em uma variável e configuramos o usuário e senha do wifi, que será utilizado para conectar o Sistema.
```c
//Configuração Bot Telegram
#define BOTtoken "*******************************"
WiFiClientSecure clientBot;
UniversalTelegramBot bot(BOTtoken, clientBot);

//Usuario e senha do WiFi
const char *ssid = "*********";
const char *password = "*******";
```
Configuramos o dispositivo para utilizer o Broker da IBM – Watson IOT. Ao cadastrar o dispotivo na IBM Cloud, recebemos um nome para a organização, um tipo de dispositivo, um ID e um Token que são armazenados em variáveis.
```c
// Dados do dispositivo no Watson Iot
const String ORG = "*****";
const String DEVICE_TYPE = "******";
const char* DEVICE_ID = "******";
#define DEVICE_TOKEN "********"
```
Definimos em variáveis os pinos utilizados para a conexão dos componentes (LED’s, PIR e o modulo Rele) e variáveis globais (um Contador e um flag).
```c
//Definições dos pinos utilizados 
int led1 = 5; //pino D1
int PIR = 15;  //pino D8
int led2 = 4; //pino D2
int led3 = 12; //D6
int rele = 14; //D5

//Variaveis globais
int contador = 0;
boolean msgEnviada = false;
```
Na declaração das variáveis para customização, declaramos as variáveis que armazenam os valores de horário de ativação do Sistema e o tempo que o Sistema deve considerar sem presença humana no ambiente.
```c
//Variaveis para customização 
int horarioAtivacao = 19; //Hora que o sistema comecará a notificar o usuário caso a luz esteja acesa
int tempoSemPresenca = 900; // Em segundos a quantidade de tempo sem detecção de movimento no ambiente 
```
Fizemos a configuração do MQTT, onde o link de conexão recebe os valores armazenados nas variáveis declaradas na configuração do dispositivo no Broker. E declaramos o tópico da lâmpada.
```c
//Configurações MQTT
const String CLIENT_ID = "d:" + ORG + ":" + DEVICE_TYPE + ":" + DEVICE_ID;
const String MQTT_SERVER = ORG + ".messaging.internetofthings.ibmcloud.com";
#define topicoLampada1 "iot-2/cmd/lampada1/fmt/json" //Define o topico que será inscrito
WiFiClient wifiClient;
PubSubClient client(MQTT_SERVER.c_str(), 1883, wifiClient);
```
No setup(), iniciamos o Serial, e declaramos os pinos como Input ou Output. Incluimos o comando para o funcionamento do Bot e a configuração do wifi, onde, a partir de um loop, são feitas as tentativas de conexão até que se obtenha sucesso.
```c
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
 ```
Depois de ter o wifi conectado, o Sistema irá enviar uma mensagem para o usuário que é passado na função, avisando que iniciou o Sistema. Iniciamos, então, o NTP e a conexão ao MQTT. 
```c
    
    //Substituir ***** pelo id do usuario no telegram
    bot.sendMessage("***********", "Iniciando sistema!", ""); //Inicio do Bot do Telegram
    timeClient.begin(); //Inicio do NTP
    connectMQTTServer(); //Conexão ao servidor MQTT
    
}
```
Iniciando o loop(), pegamos do NTP a hora e printamos na tela. Chamamos o loop do Client, onde serão executados os métodos descritos anteriormente.
Em seguida são lidos os valores recebidos do sensor LDR e convertidos em voltagem. Se o ambiente estiver escuro, o Contador será reiniciado e a mensagem enviada será definida para false.
```c
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
  ```
  Na leitura no PIR, se for detectado movimento, o Contador será zerado e não enviará mensagem ao usuário. Caso não detecte movimento o Contador será incrementado até atingir o tempo determinado para enviar a mensagem ao usuário.
  ```c

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
```
Logo após, teremos uma verificação se está sem detectar presença no ambiente, se tem detecção de luz e se já passa do horário determinado no Sistema. Caso as três condições sejam positivas e o Sistema ainda não tenha enviado uma mensagem ao usuário, ele envia e mostra que a enviou.
```c
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
```
Essa função reinicia a contagem, ela conta 1 segundo, zera o Contador e o flag, declarado no inicio do Código, recebe false.
```c
 //Função responsavel por reiniciar a contagem
void reiniciaContagem(){
  delay(1000);
  contador = 0;
  msgEnviada = false; //Define variavel global para falso para que, caso detecte a luz acesa, mande a mensagem para o usuario
}
```
Para a conexão com o MQTT, o Sistema utilizará a URL e os dados declarados no início do Código. Caso tenha sucesso na conexão, ele irá “setar” um callback, uma função que será ativada todas as vezes que houver mudança no estado do tópico inserido, e será inscrito no tópico da lâmpada. Caso não seja possível realizar a conexão ao MQTT, o Sistema irá mostrar o erro e tentar novamente.
```c
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
```
A função callback recebe, entre os parâmetros, o tópico da lâmpada. Ele irá printar qual foi o tópico onde teve alteração, quando a função é chamada.
O Sistema irá receber um JSON e irá desserializá-lo. Se tiver sucesso na desserealização, ele acessará o atributo “value“ do JSON que poderá ser 0 ou 1. O valor dessa variável será escrita na porta do Módulo Rele.
Nesta parte do Código, fizemos uma condição pensando na inclusão de mais lâmpadas no Sistema como uma proposta para o futuro. Essa condição é responsável por verificar se o tópico passado como parâmetro é o tópico da lâmpada sobre a qual se deseja realizar alguma ação. Caso o tópico seja o mesmo, o valor da variável “value” será escrito na porta do Módulo Rele.
```c
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


# esp32web-cam
## Variables globales
#### esp32Client
Cliente con conexión wifi y después un cliente MQTT.
*Tipo: client*
```c++
WiFiClient esp32Client;
PubSubClient mqttClient(esp32Client);
```

#### miDireccionIP
Dirección ip de la cámara.
*Tipo: IPAdress*
```c++
IPAddress miDireccionIP;
```

#### count
Bandera para el ciclo manual.
*Tipo: long*
```c++
unsigned long count = 0;
```

#### server
Link del mqtt.
*Tipo char*
```c++
char *server = "broker.emqx.io";
```

#### port
Puerto de conexión.
*Tipo: int*
```c++
int port = 1883;
```

#### estado
Bandera que indica la etapa de conexión.
*Tipo: int*
```c++
int estado = 0;
```

#### Max_buffer_size
Tamaño del buffer de entrada.
*Tipo: int*
```c++
const int MAX_BUFFER_SIZE = 64;
```

#### ssid
Nombre de la red wifi.
*Tipo: char*
```c++
char ssid[60];
```

#### password
Contraseña de la red wifi.
*Tipo: char*
```c++
char password[60];
```

#### IP
Dirección que tendrá la cámara.
*Tipo: char*
```c++
char IP[] = "xxx.xxx.xxx.xxx";
```

#### buffer
Almacena los datos recibidos.
*Tipo: char*
```c++
char buffer[MAX_BUFFER_SIZE];
```

#### bufferIndex
Indica el estado actual del bufffer.
*Tipo: int*
```c++
int bufferIndex = 0;
```

## Funciones
#### stream_handler
Revisa que la transmisión se mantenga estable.
###### Parámetros:
*httpd_req_t*
###### Devuelve:
*boolean* la respuesta si la cámara esta encendida
```c++
static esp_err_t stream_handler(httpd_req_t *req){
```

#### startCameraServer
Inicia el stream.
###### Devuelve:
*void*
```c++
void startCameraServer()
```

#### setup
Comienza la configuración del programa.
###### Devuelve:
*void*
```c++
void setup()
```

#### camaraencendida
Comienza con la configuración de la cámara, conecta la red wifi y el servidor mqtt.
###### Devuelve:
*void*
```c++
void camaraencendida()
```

#### reconnect
En caso de que no este conectado al mqtt lo reconecta
###### Devuelve:
*void*
```c++
void reconnect()
```

#### loop
Genera el ciclo de envio de ip y conexion MQTT.
###### Devuelve:
*void*
```c++
void loop()
```

## Módulos
#### esp_camera.h
Librería encargada de configurar el esp32-cam.
```c++
#include "esp_camera.h"
```

#### WiFi.h
Librería que conecta el esp32 a wifi.
```c++
#include <WiFi.h>
```

#### esp_timer.h
Librería que permite usar el timer del esp32.
```c++
#include "esp_timer.h"
```

#### img_converters.h
Librería para manipular imágenes.
```c++
#include "img_converters.h"
```

#### Arduino.h
Librería para usar varios aditamentos del lenguaje de arduino.
```c++
#include "Arduino.h"
```

#### fb_gfx.h
Librería que sirve para manipular las imágenes.
```c++
#include "fb_gfx.h"
```

#### soc/soc.h
Librería para usar temporizadores internos.
```c++
#include "soc/soc.h"
```

#### soc/rtc_cntl_reg.h
Librería para también relojes internos.
```c++
#include "soc/rtc_cntl_reg.h"
```

#### esp_http_server.h
Librería para crear puntos de acceso con el esp32.
```c++
#include "esp_http_server.h"
```

#### PubSubClient.h
Librería para hacer uso de la comunicación MQTT.
```c++
#include <PubSubClient.h>
```

## Código completo
```c++
#include "esp_camera.h"

#include <WiFi.h>

#include "esp_timer.h"

#include "img_converters.h"

#include "Arduino.h"

#include "fb_gfx.h"

#include "soc/soc.h"

#include "soc/rtc_cntl_reg.h"  

#include "esp_http_server.h"

#include <PubSubClient.h>

  

WiFiClient esp32Client;

PubSubClient mqttClient(esp32Client);

IPAddress miDireccionIP;

  

unsigned long count = 0;

  

char *server = "broker.emqx.io";

int port = 1883;

int estado = 0;

  

const int MAX_BUFFER_SIZE = 64; // Tamaño máximo del búfer de entrada

char ssid[60];

char password[60];

char IP[] = "xxx.xxx.xxx.xxx";

  

char buffer[MAX_BUFFER_SIZE]; // Búfer para almacenar los datos recibidos

int bufferIndex = 0; // Índice actual en el búfer

  

#define PART_BOUNDARY "123456789000000000000987654321"

  

#define CAMERA_MODEL_AI_THINKER

  

#elif defined(CAMERA_MODEL_AI_THINKER)

  #define PWDN_GPIO_NUM     32

  #define RESET_GPIO_NUM    -1

  #define XCLK_GPIO_NUM      0

  #define SIOD_GPIO_NUM     26

  #define SIOC_GPIO_NUM     27

  #define Y9_GPIO_NUM       35

  #define Y8_GPIO_NUM       34

  #define Y7_GPIO_NUM       39

  #define Y6_GPIO_NUM       36

  #define Y5_GPIO_NUM       21

  #define Y4_GPIO_NUM       19

  #define Y3_GPIO_NUM       18

  #define Y2_GPIO_NUM        5

  #define VSYNC_GPIO_NUM    25

  #define HREF_GPIO_NUM     23

  #define PCLK_GPIO_NUM     22

#else

  #error "Camera model not selected"

#endif

  

static const char* _STREAM_CONTENT_TYPE = "multipart/x-mixed-replace;boundary=" PART_BOUNDARY;

static const char* _STREAM_BOUNDARY = "\r\n--" PART_BOUNDARY "\r\n";

static const char* _STREAM_PART = "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

  

httpd_handle_t stream_httpd = NULL;

  

static esp_err_t stream_handler(httpd_req_t *req){

  camera_fb_t * fb = NULL;

  esp_err_t res = ESP_OK;

  size_t _jpg_buf_len = 0;

  uint8_t * _jpg_buf = NULL;

  char * part_buf[64];

  

  res = httpd_resp_set_type(req, _STREAM_CONTENT_TYPE);

  if(res != ESP_OK){

    return res;

  }

  

  while(true){

    fb = esp_camera_fb_get();

    if (!fb) {

      Serial.println("Camera capture failed");

      res = ESP_FAIL;

    } else {

      if(fb->width > 400){

        if(fb->format != PIXFORMAT_JPEG){

          bool jpeg_converted = frame2jpg(fb, 80, &_jpg_buf, &_jpg_buf_len);

          esp_camera_fb_return(fb);

          fb = NULL;

          if(!jpeg_converted){

            Serial.println("JPEG compression failed");

            res = ESP_FAIL;

          }

        } else {

          _jpg_buf_len = fb->len;

          _jpg_buf = fb->buf;

        }

      }

    }

    if(res == ESP_OK){

      size_t hlen = snprintf((char *)part_buf, 64, _STREAM_PART, _jpg_buf_len);

      res = httpd_resp_send_chunk(req, (const char *)part_buf, hlen);

    }

    if(res == ESP_OK){

      res = httpd_resp_send_chunk(req, (const char *)_jpg_buf, _jpg_buf_len);

    }

    if(res == ESP_OK){

      res = httpd_resp_send_chunk(req, _STREAM_BOUNDARY, strlen(_STREAM_BOUNDARY));

    }

    if(fb){

      esp_camera_fb_return(fb);

      fb = NULL;

      _jpg_buf = NULL;

    } else if(_jpg_buf){

      free(_jpg_buf);

      _jpg_buf = NULL;

    }

    if(res != ESP_OK){

      break;

    }

  }

  return res;

}

  

void startCameraServer(){

  httpd_config_t config = HTTPD_DEFAULT_CONFIG();

  config.server_port = 80;

  

  httpd_uri_t index_uri = {

    .uri       = "/",

    .method    = HTTP_GET,

    .handler   = stream_handler,

    .user_ctx  = NULL

  };

  if (httpd_start(&stream_httpd, &config) == ESP_OK) {

    httpd_register_uri_handler(stream_httpd, &index_uri);

  }

}

  

void setup() {

  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);

  Serial.begin(115200);

  Serial.setDebugOutput(false);

}

  

void camaraencendida(){

  camera_config_t config;

  config.ledc_channel = LEDC_CHANNEL_0;

  config.ledc_timer = LEDC_TIMER_0;

  config.pin_d0 = Y2_GPIO_NUM;

  config.pin_d1 = Y3_GPIO_NUM;

  config.pin_d2 = Y4_GPIO_NUM;

  config.pin_d3 = Y5_GPIO_NUM;

  config.pin_d4 = Y6_GPIO_NUM;

  config.pin_d5 = Y7_GPIO_NUM;

  config.pin_d6 = Y8_GPIO_NUM;

  config.pin_d7 = Y9_GPIO_NUM;

  config.pin_xclk = XCLK_GPIO_NUM;

  config.pin_pclk = PCLK_GPIO_NUM;

  config.pin_vsync = VSYNC_GPIO_NUM;

  config.pin_href = HREF_GPIO_NUM;

  config.pin_sscb_sda = SIOD_GPIO_NUM;

  config.pin_sscb_scl = SIOC_GPIO_NUM;

  config.pin_pwdn = PWDN_GPIO_NUM;

  config.pin_reset = RESET_GPIO_NUM;

  config.xclk_freq_hz = 20000000;

  config.pixel_format = PIXFORMAT_JPEG;

  config.frame_size = FRAMESIZE_VGA;  // Cambia el tamaño del fotograma a SVGA

  config.jpeg_quality = 20;

  config.fb_count = 1;

  // Camera init

  esp_err_t err = esp_camera_init(&config);

  if (err != ESP_OK) {

    Serial.printf("Camera init failed with error 0x%x", err);

    return;

  }

  // Wi-Fi connection

  Serial.print(ssid);

  Serial.print(password);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {

    delay(500);

    Serial.print(".");

  }

  Serial.println("");

  Serial.println("WiFi connected");

  Serial.print("Camera Stream Ready! Go to: http://");

  Serial.print(WiFi.localIP());

  mqttClient.setServer(server, port);

  miDireccionIP = WiFi.localIP();

  miDireccionIP.toString().toCharArray(IP, 16);

  // Start streaming web server

  startCameraServer();

  estado = 3;

}

  

void reconnect() {

  while (!mqttClient.connected()) {

    Serial.print("Intentando conectarse MQTT...");

  

    if (mqttClient.connect("arduinoClient")) {

      Serial.println("Conectado");

    } else {

      Serial.print("Fallo, rc=");

      Serial.print(mqttClient.state());

      Serial.println(" intentar de nuevo en 5 segundos");

      // Wait 5 seconds before retrying

      delay(5000);

    }

  }

}

  

void loop() {

  while (Serial.available() > 0) {

    char incomingChar = Serial.read(); // Lee el próximo carácter disponible

  

    // Verifica si el carácter es un retorno de carro o nueva línea

    if (incomingChar == '\r' || incomingChar == '\n') {

      // Fin de la línea, procesa el mensaje y reinicia el búfer

      buffer[bufferIndex] = '\0'; // Agrega el terminador nulo al final del texto

  

      // Comprueba si el mensaje comienza con "con" (para el primer mensaje)

      if (strncmp(buffer, "con", 3) == 0) {

        mensaje1 = String(buffer + 3); // Almacena el mensaje sin los primeros 3 caracteres

        mensaje1.toCharArray(password, sizeof(password));

        Serial.println(password);

        //mensaje1.toCharArray(ssid, sizeof(ssid)); // Almacena el mensaje sin los primeros 3 caracteres

        //Serial.println(ssid);

        if (estado == 1){

          estado = 2;

        }

      }

      // Comprueba si el mensaje comienza con "wif" (para el segundo mensaje)

      else if (strncmp(buffer, "wif", 3) == 0) {

        mensaje2 = String(buffer + 3); // Almacena el mensaje sin los primeros 3 caracteres

        mensaje2.toCharArray(ssid, sizeof(ssid));

        Serial.println(ssid);

        //Serial.println(password);

        if (estado == 0){

          estado = 1;

        }

      }

  

      // Reinicia el índice del búfer

      bufferIndex = 0;

    } else {

      // Agrega el carácter al búfer si no es un retorno de carro o nueva línea

      buffer[bufferIndex] = incomingChar;

      bufferIndex = (bufferIndex + 1) % MAX_BUFFER_SIZE; // Evita desbordamiento del búfer

    }

  }

  if (estado == 2){

    camaraencendida();

  }

  

  if (count == 6000){

    if (!mqttClient.connected()) {

      reconnect();

    }

    mqttClient.loop();

    mqttClient.publish("stream", IP);

    Serial.print("http://");

    Serial.print(IP);

    count = 0;

  }

  

  delay(1);

}
```

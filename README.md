# esp32web-cam
## Variables globales
##### Conexión
```c++
WiFiClient esp32Client;

PubSubClient mqttClient(esp32Client);

IPAddress miDireccionIP;
```

##### Cuenta manual
```c++
unsigned long count = 0;
```

##### MQTT
```c++
char *server = "broker.emqx.io";

int port = 1883;

int estado = 0;
```


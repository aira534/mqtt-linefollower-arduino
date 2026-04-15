# Jeepetones
## Ainara Yu Elio San Martín y Rui Bartolomé Segura
Robot siguelineas con comunicación IoT a través de MQTT.

## 1. Organización del código en la Arduino
El arduino se encarga de seguir la linea, comprobar la distancia a objetos y de generar los mensajes que luego se enviarán por comunicación en serie a la placa ESP32 para que esta los envíe al servidor mqtt. <br>
Para esto disponemos de tres tareas controladas por RTOS de arduino, estas son:

### Tarea Ultrasonido
Tarea con nivel 2 de prioridad y un ratio de 200 ms.<br>
Se encarga de medir la distancia hasta un objeto y detectar cuando el robot se acerca más de lo permitido a él. La distancia incluye un pequeño margen para compensar posibles retrasos en la lectura, la prioridad de otras tareas, el tiempo de frenado del robot y la inercia causada por su velocidad antes de que se ejecute la orden de detener los motores.<br><br>
Además del frenado básico, se incluye un margen de seguridad adicional. Este margen permite que, cuando el robot se acerque al obstáculo pero aún no haya alcanzado el límite establecido, reduzca la velocidad. De esta manera, el frenado completo se realiza de manera más eficiente y segura.
```c
// Variables de la distancia
#define DIST_STOP 10 // With a bit of gap because the intertia of movement
#define DIST_SLOW 15 // Distance where the robot is going to star slowing the speed

// Dentro de la tarea
// Filter wrong measures (0) and check if obstacle is close enough to slow the vel
if (dist <= (DIST_SLOW + DIST_STOP) && dist > 0.1 ) {
  MAXIMUM_SPEED = 100;
  BASE_SPEED = 75;
}
```

### Tarea Infrarojos y PD
Tarea con nivel de prioridad 3 y un ratio de 50 ms.<br>
Tarea dedicada a la detección de los infrarrojos y a la regulación de la velocidad por un PD. Está tarea distingue dos casos, el primero si ningún sensor detecta línea y el otro si alguno de los sensores detecta.
Si ningún sensor detecta línea el robot girará hacia el lado del último sensor que haya detectado la línea. 
```c
if (left <= 600 && middle <= 600 && right <= 600) {
  /* Find line protocol, if last seen in left sensor, turn left and viceversa */
  if (prev_right > 600) {
    analogWrite(PIN_Motor_PWML, vel_turn);
    analogWrite(PIN_Motor_PWMR, 0);

  } else if (prev_left > 600) {
    analogWrite(PIN_Motor_PWML, 0);
    analogWrite(PIN_Motor_PWMR, vel_turn);
  }
}
```
Si alguno de los sensores lee que hay línea se calculará la velocidad de los mototres con un PD.
```c
// PD logic
error = right - left;

P = error;
D = error - lastError;

float pid_output = (Kp * P) + (Kd * D);

lastError = error; // Save last error to compare after

// Calculate velocities
int speedLeft = BASE_SPEED + pid_output;
int speedRight = BASE_SPEED - pid_output;
```
### Tarea que envía los PINGs
Tarea con nivel de prioridad 1 y un ratio de 4000ms.<br>
Enviará los pings requeridos cada 4 segundos.

## 2. Organización del código en la ESP32
Tiene como función recibir la información que le mande la placa Arduino a través del puerto serie y reenviarla al broker a usando MQTT.<br>

### Setup
Al tener dos puertos series el que se destinó a la comunicación fue el número 2.
```c
void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);
```
El resto del setup es la conexión wifi y MQTT una vez ha conseguido enlazarse con todo envía un mensaje por el puerto serie al Arduino avisando de que ya está todo listo para empezar.
```c
  Serial.println("MQTT connected");
  Serial2.print("r"); // Advise the arduino everything is ready
} // fin setup
```

### Parseo de mensajes
Usamos como estándar para todos los mensajes un comienzo con "{" y una terminación "}". Todos los mensajes van abreviados en 2 letras y de ser necesaria más información va acto seguido después de las 2 letras.
```c
// Ejemplo de cómo se parsean los mensajes del ping
else if (rcv_msg[1] == 'p' && rcv_msg[2] == 'm') { //ping_message
  int time = rcv_msg.substring(3).toInt();
  ping_message(time);

}
```
```c
// Ejemplo de cómo se envió desde el Arduino
long ping_time = get_time_diff(start_time, millis());
String ping_message = "{pm" + String(ping_time) + "}";
Serial.print(ping_message); // Send ping message every 4s with low priority
```
Tabla con las abreviaturas y su función:

| Código | Descripción de la función |
|--------|--------------------------|
| ll     | lost_line()              |
| vl     | visible_line(value)      |
| el     | end_lap(value)           |
| od     | obst_det(dist)           |
| ls     | line_search()            |
| ss     | stop_search()            |
| lf     | line_found()             |
| pm     | ping_message(time)       |
| sl     | start_lap()              |

Para enviar el mensaje se convertía a formato json.
```c
void lost_line() {
  StaticJsonDocument<200> doc;

  doc["team_name"] = team_name;
  doc["id"] = id;
  doc["action"] = "LINE_LOST";

  mqttClient.beginMessage(topic);
  serializeJson(doc, mqttClient);
  mqttClient.endMessage();
}
```

## 3. Vídeo
[![Ver video](https://drive.google.com/file/d/1xAjtnyFRYlQitorrwMDDlR-E5NOdGq3v/view?usp=sharing)](https://drive.google.com/file/d/1uTETSnpztl95fZv3-GyCv4zzPdv2T2iK/view?usp=sharing)



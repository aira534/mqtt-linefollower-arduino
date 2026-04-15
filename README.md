# mqtt-linefollower-arduino
## Ainara Yu Elio San Martín and Rui Bartolomé Segura
Line-following robot with IoT communication through MQTT.

## Video
[![Watch video](https://drive.google.com/file/d/1xAjtnyFRYlQitorrwMDDlR-E5NOdGq3v/view?usp=sharing)](https://drive.google.com/file/d/1uTETSnpztl95fZv3-GyCv4zzPdv2T2iK/view?usp=sharing)

## 1. Code organization on the Arduino
The Arduino is in charge of following the line, checking the distance to objects and generating the messages that will then be sent through serial communication to the ESP32 board so that it sends them to the MQTT server. <br>
For this we have three tasks controlled by Arduino's RTOS, these are:

### Ultrasound Task
Task with priority level 2 and a rate of 200 ms.<br>
It is in charge of measuring the distance to an object and detecting when the robot gets closer than allowed to it. The distance includes a small margin to compensate for possible delays in reading, the priority of other tasks, the braking time of the robot and the inertia caused by its speed before the order to stop the motors is executed.<br><br>
In addition to basic braking, an additional safety margin is included. This margin allows that, when the robot approaches the obstacle but has not yet reached the established limit, it reduces the speed. This way, the complete braking is carried out in a more efficient and safe manner.
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

### Infrared and PD Task
Task with priority level 3 and a rate of 50 ms.<br>
Task dedicated to the detection of the infrareds and to the speed regulation by a PD. This task distinguishes two cases, the first if no sensor detects line and the other if any of the sensors detects.
If no sensor detects line the robot will turn towards the side of the last sensor that detected the line.
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
If any of the sensors reads that there is line, the speed of the motors will be calculated with a PD.
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

### Task that sends the PINGs
Task with priority level 1 and a rate of 4000ms.<br>
It will send the required pings every 4 seconds.

## 2. Code organization on the ESP32
Its function is to receive the information that the Arduino board sends through the serial port and forward it to the broker using MQTT.<br>

### Setup
Having two serial ports, the one designated for communication was number 2.
```c
void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);
```
The rest of the setup is the wifi and MQTT connection, once it has managed to link with everything it sends a message through the serial port to the Arduino notifying that everything is ready to start.
```c
  Serial.println("MQTT connected");
  Serial2.print("r"); // Advise the arduino everything is ready
} // fin setup
```

### Message parsing
We use as standard for all messages a beginning with "{" and an ending "}". All messages are abbreviated in 2 letters and if more information is necessary it goes right after the 2 letters.
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
Table with the abbreviations and their function:

| Code | Description of the function |
|------|------------------------------|
| ll   | lost_line()                  |
| vl   | visible_line(value)          |
| el   | end_lap(value)               |
| od   | obst_det(dist)               |
| ls   | line_search()                |
| ss   | stop_search()                |
| lf   | line_found()                 |
| pm   | ping_message(time)           |
| sl   | start_lap()                  |

To send the message it was converted to json format.
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

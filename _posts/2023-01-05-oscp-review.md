---
title: OSCP Review
author: xdann1
date: 2023-01-05
categories: [Review, Certificación]
tags: [Certificación, Pentesting, OSCP]
pin: true
image:
  path: ../../assets/img/commons/oscp/oscp-icon.png
  alt: Banner OSCP
---

El pasado día 5 de enero Offensive Security me mandó el mail que confirma que oficialmente he aprobado el OSCP (Offensive Security Certified Professional), por lo que quería compartir mi experiencia en esta nueva certificación, analizándola de manera crítica.

## Curso PEN-200

Los contenidos del curso con la base más elemental que puedas imaginar (empiezan como si no hubieras tocado la terminal en la vida), sin embargo, esto no quiere decir que la certificación vaya a ser un paseo. Cuando me preguntan sobre los contenidos me gusta decir esta frase: "La certificación puede llegar a ser de 'entry level' pero te aseguro que no vas a salir de esta teniendo el nivel que tenías cuando entraste".

Estos son módulos que contiene el curso:

1. Penetration Testing with Kali Linux: Genral Course Information
2. Getting Comfortable with Kali Linux
3. Command Line Fun
4. Practical Tools
5. Bash Scripting
6. Passive Information Gathering
7. Active Information Gathering
8. Vulnerability Scanning
9. Web Application Attacks
10. Introduction to Buffer Overflows
11. Windows Buffer Overflows
12. Linux Buffer Overflows
13. Client-Side Attacks
14. Locating Public Exploits
15. Fixing Exploits
16. File Transfers
17. Antivirus Evasion
18. Privilege Escalation
19. Password Attacks
20. Port Redirection and Tunneling
21. Active Directory Attacks
22. The Metasploit Framework
23. PowerShell Empire
24. Assembling the Pieces: Penetration Test Breakdown
25. Trying Harder: The Labs

Cada módulo del curso viene explicado de dos formas, mediante vídeos o mediante diapositivas. La verdad, los módulos buenos se cuentan con los dedos de las manos. Los que podría resaltar son el 11, 12, 18 y 20. Exceptuando los citados anteriormente, ningún módulo profundiza demasiado y tampoco son muy complejos.

<h3 data-toc-skip>¿Es suficiente el curso para poder aprobar?</h3> 

La verdad es que no, mis sensaciones es que ellos te dan lo mínimo de cada tema (hay alguna excepción) para que puedas orientarte y ya luego tú investigas por tu parte. Por ejemplo, en el apartado "Web Application Attacks" te enseñan 4 vulnerabilidades y cómo explotarlas, pero esto no significa que no te puedas topar con otras en los labs o en el propio examen, de hecho eso es lo que me paso a mi.

El curso deja bastante que desear, cualquier otra certificación tiene más y mejor contenido. Con esto no quiero decir que el contenido en general sea de lo peor. Si comparas el directorio activo (AD) del curso con otros te darás cuenta de que no tiene nada ver. Pero, ¿por qué pasa esto? Pues la explicación es bastante sencilla, al tener un curso de, por ejemplo, 1000 páginas en el que tienes que dar 10 temas se te quedarán 100 páginas por tema de modo que no profundizas mucho en cada uno. Por el contrario, en otro curso en el que den 500 páginas pero que solo se centre en 2 temas se quedará en 250 páginas por tema, pudiendo profundizar muchisimo más en cada cosa. Aunque el contenido en general de este último curso sea la mitad de largo por cada tema será el doble de largo.

Entonces, ¿el temario del curso del OSCP es malo? En general no, lo que le pasa es que tiene el contenido distribuido en bastantes temas. Te recomiendo que si quieres profundizar en un aspecto muy concreto (active directory, exploiting, web...) vayas a otra certificación, ya que esta si bien es cierto que da todo esto, lo hace de forma superficial.

<h3 data-toc-skip>Laboratorios</h3> 

Una de las partes fuertes de la certificación son los laboratorios, una serie de máquina (73 aproximadamente) que te preparan para el examen. Junto con las máquinas de [Proving Grounds](https://www.offensive-security.com/labs/individual/) son los mejores laboratorios que he probado.  Te preparan no solo de forma de aprender nuevas técnicas, sino que también te ayudan a forjar una buena metodología.

Puedo decir con seguridad que son los laboratorios más realistas y que mejor te preparan que he probado, todos los conceptos que se ven en el curso los vas a poder probar en los laboratorios sin excepción alguna. Aun así, tiene algunos fallos gordos, por ejemplo, las máquinas son compartidas de tal forma que los usuarios pueden llegar a molestar a otros quitando cosas de alguna máquina. Algo parecido me pasó a mí, estaba haciendo una máquina en la que en un momento te tienes que conectar por RDP y cada cierto tiempo me echaba ya que había otro usuario entrando por esa misma sesión, algo bastante molesto la verdad.

Aparte de los propios laboratorios que te proporcionan en el OSCP tienes otras muchas formas de preparar el examen. Yo usé el siguiente [excel](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#) para ver que máquinas parecidas al OSCP tenía cada plataforma. Especialmente recomiendo las máquina del "Proving Grounds Practice" ya que son máquinas creadas por Offensive Security, incluso hay algunas máquinas que antes estaban en examenes.

## Experiencia Previa 

Antes de empezar el curso tenía aproximadamente 50 máquinas hechas en [Hack The Box](https://www.hackthebox.com/) y otras cuantas en otras plataformas. En cuanto a certificaciones, solo tenía el eJPTv1. Como veis, no tenía mucha experiencia previa, ni siquiera tengo experiencia laboral en el sector así que no os preocupéis demasiado por esto.

## Examen OSCP

Este es uno de los puntos fuertes de la certificación junto con los labs. El examen se divide en 2 partes, la explotación y el reporte, durando cada una un día. En el día de explotación tendrás que intentar entrar en los diferentes equipos y escalar privilegios, en el día de reporte como su propio nombre indica tendrás que realizar el reporte de lo que hiciste el día anterior.

Para aprobar el examen deberás tener 70 puntos o más hasta un máximo de 100 puntos, se dividen de la siguiente forma:

- 60 puntos: Máquinas Independientes.
  - 3 máquinas siendo cada una de 20 puntos (10 puntos entrada + 10 puntos de escalada).
- 40 puntos: Set de AD (Active Directory).
  - 2 clientes y un DC (Domain Controller), si no consiges comprometer todos los equipos no consiges los 40 puntos.

Además, aparte de los puntos que puedas realizar en el examen puedes conseguir hasta 10 puntos extra, los podrás conseguir si haces lo siguiente (existe una forma más de poder conseguir estos puntos, pero no la explicaré, ya que a partir del 31/01/2023 se dejará de poder usar):

- 30 máquinas hechas en el laboratorio
- 80% de los ejercicios hechos por cada categoría del curso.

El examen es "Proctored", es decir, que hay una persona que te está vigilando en todo lo que haces para impedir que puedas hacer trampa. Además, antes de poder empezar a realizar el examen tienes que hacer un procedimiento muy riguroso en el que te pedirán que muestres la habitación, el DNI, que alejes el móvil...

Personalmente, yo tardé aproximadamente unas 8 horas en conseguir los puntos suficientes para aprobar, teniendo ya comprometido todo el set de AD, una máquina independiente y los 10 puntos adicionales. Luego de otras 4 horas conseguí comprometer otra máquina, en total conseguí 2 máquinas independientes (40 puntos), set de AD (40 puntos) y los 10 puntos adicionales, quedandome en unos 90 puntos en total.

A continuación llega una de las preguntas que más he escuchado, ¿con qué nivel se asemejan las máquinas del examen en comparación con otras plataformas? Lo primero que quiero decir es que en cada caso de examen será distinto, por lo que puede que a otro se salga un examen más fácil o más difícil que el mío. Teniendo claro esto, lo compararé con las máquinas de 2 plataformas, [Hack The Box](https://www.hackthebox.com/) y [Proving Grounds](https://www.offensive-security.com/labs/individual/) ya que son las que más he utilizado. Diría que en comparación con Hack The Box le pondría un nivel entre "Medium" y "Hard", en comparación con Proving Grounds le asignaría el nivel "Hard".

## Recomendaciones

- Crea tu propia cheatsheet, no es lo mismo copiar una de otra persona que hacer una propia. Haceros apuntes de todo, desde la enumeración de cada servicio (SSH, web, SMB...) hasta conceptos (privilege escalation, port forwarding, active directory...). 
- Crea una buena metodología, siento ser pesado, pero es la verdad. Todo se basa en la metodología, es lo que va a definir si eres capaz o no de encontrar la vulnerabilidad. 
- Utiliza recursos externos, recomiendo encarecidamente que inviertas en conocimiento paralelamente al OSCP, por ejemplo, [Privilege Escalation Tib3rius](https://courses.tib3rius.com/), [Hack The Box](https://www.hackthebox.com/), [Proving Grounds](https://www.offensive-security.com/labs/individual/), [PortSwigger](https://portswigger.net/) y otros más.
- Try Harder!

## Conclusiones

Si bien es cierto que el OSCP no es la certificación por excelencia en cuanto a contenido respecta, es una de las mejores por no decir la mejor en cuanto a prestigio y reconocimiento. Es por esto que realmente la recomiendo si quieres empezar tu carrera en la ciberseguridad, si quieres mejorar tu puesto de trabajo, si quieres tener una certificación que realmente avale tus conocimientos o si simplemente quieres ponerte un reto.

![Certificación OSCP]({{ 'assets/img/commons/oscp/oscp_cert.png' | relative_url }}){: .center-image }
_Certificación OSCP_

---
title: eJPT/PTS Review
author: xdann1
date: 2022-04-20
categories: [Review, Certificación]
tags: [Certificación, Pentesting, eJPT]
---

## Introducción

Recientemente conseguí la cerfificación eJPT (eLearnSecurity Junior Penetration Tester) de eLearnSecurity, por lo que quería compartir mi experiencia personal sobre esta certificación. 

## Curso PTS

PTS (Penetration Testing Student) es el curso oficial para la certificación eJPT, es un curso totalmente gratuito que te introduce en el mundo de las pruebas de penetración, además te puede servir para preparar el examen del eJPT. El curso esta dividido en 3 secciones principales, estas secciones a su vez se componen de módulos. Cada modulo puede tener varios tipos de contenidos, diapositivas, videos y laboratorios.

A continuación dejo descritos los tres módulos:
- Penetration Testing Prerequisites: este primer módulo nos introduce fundamentos de redes y aplicaciones web, considero este módulo muy importante si no se tiene mucho conocimiento.
- Penetration Testing: Preliminary Skills & Programming: este módulo ofrece los fundamentos sobre C++, Python y Bash. Este módulo te puede ser muy útil si no estabas familiarizado con algunos de los lenguajes mencionados.
- Penetration Testing Basics: este último módulo cubre los principales ataques, herramientas y técnicas utilizadas durante una prueba de penetración.

En la útlima versión del curso introdujeron una nueva sección en la que hay varios laboratorios, 3 de estos laboratorios son black boxs en los que se pueden practicar los conocimientos adquiridos en el curso, personalmente me parecieron más dificiles que el propio examen. Te recomiendo que hagas todos los labs del curso, de esta forma podr  s asentar los conocimiento adquiridos anteriormente.

Aún así el curso tiene algunos puntos negativos desde mi punto de vista, el primero de los problemas que le encuentro al curso es que algunos conceptos se enseñan con software obsoleto o con herramientas automáticas, la última pega que le tengo al curso es que a la hora de comenzar con un laboratorio no se te proporciona un archivo con el que puedas conectarte a su red, sino que tienes que acceder a una máquina desde tu propio navegador, a mi personalmente esta última opción se me hace bastante incomoda.

## Experiencia Previa 

Antes de empezar el examen contaba con 30 máquinas hechas en [Hack The Box](https://www.hackthebox.com/) y otras cuantas de otras plataformas. No creo que sea necesario tener una gran cantidad de máquinas hechas, ya que el examen no es exactamente lo que se podría esperar de un CTF. Donde realmente está el quid de la cuestión para superar el examen es en tener definida una buena metodología  de pentesting con la que te puedas sentir comodo y ser eficaz.

## Examen eJPT

Actualmente, el examen para conseguir la certificación eJPT cuesta 200$, al cambio a euros son 184,94€.

Una vez comprado el ticket del examen tendrás 180 días para activar el examen. Cuando comienzes el examen dispondras de 3 días para terminarlo, aunque normalmente se termina mucho antes, entre 6 a 10 horas.

Al comenzar el examen se te entregarán unos archivos, entre ellos hay una carta de compromiso en la que se nos explica que el examen consiste en una prueba de penetración de tipo black box a una empresa, en la que tendrás que comprometer varios equipos.

El examen consiste en responder 20 preguntas, estas preguntas son de tipo de test, siendo algunas de respuesta multiple y otras de respuesta única, la solución de cada pregunta la irás encontrando a medida de que vulneres distintos equipos . Para aprobar el examen tendrás que responder 15 preguntas bien, la nota del examen se te entregará automaticamente luego de entregarlo.

Por decirlo de alguna forma, el examen está dividido en dos partes, la anterior al pivoting y la posterior. En algún momento acabarás llegando a una parte en la que si no sabes pivotar o agregar rutas no podrás responder el resto de las preguntas. Echale un vistazo al enrutamiento y pivoting antes de empezar el examen :)

## Recomendaciones

- Crea una buena metodolgía.
- Lee todas las preguntas detenidamente. 
- Recuerda que el reconocimiento es la base de todo.
- Toma nota de todo lo que encuentres.
- Intenta no utilizar herramientas automáticas como Metasploit, aprenderas más si haces las cosas manualmente.
- Recuerda que el examen no trata de encontrar flags.

## Conclusiones

Bajo mi experiencia puedo decir que recomendaría está certificación como la primera en caso de que este en un nivel junior o que no tengas aún otra certificación. Me ha parecido una muy buena primera experiencia con el tema de las certificaciones. 

![Certificación eJPT]({{ 'assets/img/commons/ejpt/ejpt_cert.png' | relative_url }}){: .center-image }
_Certificación eJPT_

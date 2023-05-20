---
title: CRTO Review
author: xdann1
date: 2023-04-27
categories: [Review, Certificación]
tags: [Certificación, Pentesting, CRTO]
image:
  path: ../../assets/img/commons/crto/crto-icon.png
  alt: Banner CRTO
---

El pasado día 11 de Abril Zero-Point Security me mandó el mail que confirma que oficialmente he aprobado el CRTO (Certified Red Team Operator), por lo que quería compartir mi experiencia en esta nueva certificación.

## Curso

El curso se divide en varios temas, cada uno de estos temas está dividido en varios artículos, la mayoría de estos artículos vienen explicados en texto, en todo el curso te vas a encontrar con solo 14 vídeos.

El curso te enseña en que se basa un C2 (Command and Control), sus características y como funcionan, en este caso, en el curso se usa Cobalt Strike, uno de los C2 más conocidos del mundo. Además de enseñarte como usar Cobalt Strike, te enseñan conceptos y skills necesarias para introducirte en el mundo del Red Team.

Los contenidos del curso son buenos en general, sin embargo, le faltan cosas. Me explico, los contenidos abarcan una gran cantidad de temas pero no se para mucho a explicar el porqué de lo que se explota. Me da la sensación que el curso solo te explica como explotar las cosas con Cobalt Strike y no se para explicar en profundidad.

Realmente esto no es malo en sí, simplemente puede ser que el curso esté orientado a personas que ya sepan desenvolverse en un Directorio Activo con soltura y que simplemente quieran experimentar con Cobalt Strike. Así que la conclusión que saco del curso es que el material que hay es bueno pero le falta añadir contexto en muchas cosas (o no).

Aquí podéis ver los módulos que contiene el curso (bajad un poco):

[Syllabus CRTO](https://training.zeropointsecurity.co.uk/courses/red-team-ops)

### ¿Es suficiente el curso para poder aprobar?

Aunque le falte mucho contenido que hable del porqué de las cosas, considero que el curso tiene todo y más de lo que necesitas para aprobar el examen. Eso sí, aunque el curso tenga el contenido necesario recomiendo encarecidamente que te mires por tu cuenta por qué se dan ciertas situaciones, ya que te va a ayudar mucho a ir mucho más rápido y tener una visión general más clara. 

Algo muy interesante del curso y que me gustó mucho es que a lo largo del curso te van dando consejos de OPSEC y advirtiendo de malas prácticas que deberías evitar. Aunque en el examen y en los laboratorios no tenga mucha importancia seguir los consejos que dan creo que le da un muy buen punto a la certificación, ya que uno de los puntos primordiales del Red Team es tratar de pasar desapercibido.

### Laboratorio

El laboratorio a mi me gustó bastante, vas a poder practicar todo los contenidos del curso y además se puede resolver de varias maneras, este concetpo es muy bueno ya que así no solo te limitas a copiar lo que se hace en el curso sino que también podrás hacer el famoso "Think outside of the box". 

Este es el link a los laboratorios: [Laboratorios](https://dashboard.snaplabs.io/events/p) 

Es un poco raro como lo tienen montado, el laboratorio no lo tiene hosteado la empresa que proporciona la certificación (Zero-Point Security) sino que lo tiene otra empresa (Snap Labs), para acceder al laboratorio tendrás que pagar una suscripción por meses (la suscripción solo la puedes pagar si ya has comprado la certificacición). Esta suscripción no te da acceso las horas que quieras durante un mes, sino que según cuanto dinero pagues tendrás más horas de acceso al laboratorio, tenéis los precios aquí: [Precios Laboratorios CRTO](https://training.zeropointsecurity.co.uk/pages/red-team-ops-lab)

La conexión al laboratorio se realiza mediante "Apache Guacamole" y no tendrás conexión a Internet, por lo que no podrás usar herramientas personalizadas, solo podrás usar las herramientas que te proporcionan.

En cuanto a problemas relacionados con el laboratorio, puedo decir que se ha portado bastante bien. El único problema que tenido ha sido que de forma casual no podía poner algunos caracteres como el "\\", quitando esto, no he experimentado más problemas en el laboratorio.

Además de lo que puede ser considerado como el propio laboratorio (entorno de equipos a explotar), hay unas aplicaciones que podemos usar si deseamos, una de ellas, Kibana es utilizada en el curso de forma opcional para ver los rastros que dejamos al momento de realizar ciertas acciones. Es lo mismo que he dicho antes con los consejos OPSEC, no hace falta mirarlo pero le da mucho valor al laboratorio para saber que trazas estás dejando, algo que considero importante en el Red Team.

## Experiencia Previa 

Antes de empezar el curso disponía de varias certificaciones, la más importante el OSCP, por otro lado, no dispongo de experiencia laboral ni de Pentester ni de Red Team Operator. Como veis, no tengo mucha experiencia previa en Red Team, así que no os preocupéis demasiado por esto.

## Examen

El examen consiste en una prueba práctica de 48 horas que puedes repartir en 4 días en el que debes poner a prueba los conocimientos adquiridos en el curso. Es un examen que no es vigilado (protored) y en el cual no es necesario la elaboración de un reporte. El examen tiene un enfoque en el cual hemos sido contratados para tratar de emular el comportamiento del "APT99" en el cliente.

Como dije en un apartado anterior, con el curso te da de sobra para aprobar, de hecho, hay bastantes cosas que no vas a usar en el examen.

Al igual que en el curso, la conexión será mediante "Apache Guacamole" a una máquina en la dispondrás de todas las herramientas que posteriormente usarás, además de esto, no tendrá salida a Internet. El objetivo del examen es llegar a Administrador en 6 máquinas de 8 que hay en el examen  (75%). El enfoque del examen es un "Assumed Breach", es decir, se presupone que ya ha habido una brecha de seguridad en la organización.

Hablando de los problemas relacionados con el examen no puedo decir lo mismo que dije del laboratorio, en este caso, todo lo que podía salir mal me salió mal. En mi caso, la experiencia del examen fue horrible, se me caían todo el rato los beacons, tenía que ejecutar 10 veces el mismo comando para que me funcionara, había flags que no estaban donde tenían que estar...

Teniendo todos estos problemas solamente conseguí los requisitos mínimos (6 flags), decidí que en cuanto consiguiera los requisitos mínimos dejaría el examen por lo molesto que estaban siendo los problemas.

## Pros vs Contras

Tras tener una visión general de la certificación voy a pasar a mostrar los pros y contras de la certificación:

### Pros

Los principales "pros" que le veo a la certificación son el dinero que cuesta y lo única que es. En el mercado, el CRTO es de la únicas certificaciones que te enseñan a usar un C2. Por otro lado, el precio de la certificación son unos "£365.00", bastante barata si lo comparamos con otras certificaciones y además este curso es de acceso de por vida, por lo que pagas una vez y si lo vuelven a actualizar podrás acceder a estos nuevos contenidos.

### Contras

Aunque al final no usé el soporte del CRTO durante el examen, he visto gente quejandose del soporte porque a veces tardan mucho en responder, esta imagen habla por si sola:

![Resumen del soporte]({{ 'assets/img/commons/crto/soporte.png' | relative_url }}){: .center-image }
_Resumen del soporte_

Otra contra (no puedo comprobarla) es que en España no es muy conocida (o esa es mi sensación), he estado preguntando a gente y no muchos la conocen y en las ofertas no se suele nombrar mucho esta certificación. Por otro lado, tengo algún amigo que ha entrado de Red Team Operator teniendo esta certificación, así que esto puede variar entre empresas y paises.

## Conclusiones

Es una certificación que vale su costo sin duda, es una certificación barata y tiene un contenido único y extenso. Tiene algunas contras pero considero que los pros tienen mucho mayor peso. Por esto recomiendo esta certificación si quieres aprender a usar un C2 bastante conocido y a la vez aprender ataques y fallos en Directorio Activo.

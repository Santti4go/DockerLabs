Máquina: [Upload](https://dockerlabs.es/#/#Upload), from DockerLabs

## 1) La ip víctima es 172.17.0.2
## 2) Arrancamos como siempre haciendo una enumeración de puertos abiertos
```
nmap -p- --open -sS -vvv --min-rate 5000 -n -Pn 172.17.0.2
```
El único puerto abierto es el 80. 
## 3) Procedemos a correr scripts de reconocimiento sobre dicho puerto para conocer un poco más
```
nmap -p80 -sC -vvv 172.17.0.2
```
Observamos que la pagina del puerto 80 soporta los siguientes métodos: GET/POST/HEAD/OPTIONS.
Y además menciona que hay una página con el título "Upload here your file" lo cual nos da una pauta de que podremos subir un archivo. ¿Podremos lograr RCE (remote code executio) por medio de un archivo?
Sería ideal para entablar una reverse shell.
## 4) Hacemos fuzzing para encontrar endpoints y saber dónde podemos subir un archivo
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -x php,txt
```
Aparecen 2 resultados: \
| Endpoint | Status code|
|    :--    |    :--: |
|/upload.php | 200|
|/uploads    | 301|
|/.php       | 403|

## 5) Veamos qué hay en la página 172.17.0.2/upload.php
Efectivamente hay un botón que permite subir cualquier tipo de archivo, APARENTEMENTE.

## 6) Inspeccionamos qué ocurre al subir el archivo con BurpSuite
Luego de jugar un rato con BurpSuite y subir varios archivos no se observa ningún comportamiento "extraño" ni que permita intuir cómo ejecutar el archivo que acabamos de subir.

En el paso (4) encontramos que había otro endpoint (/uploads) quizá allí se almacenan los archivos y los podamos ejecutar.

Efectivamente encontramos ahí los archivos pero no podemos ejecutarlos, se descargan, al menos los que subimos sin extensión o con extensión .sh. ¿Y si probamos subiendo alguno .php?

## 7) Creamos un payload PHP con el objetivo de poder ejecutarlo y entablar una reverse shell. Para ello usamos revshells.com/
Dentro de revshells.com cargamos la IP de nuestra máquina atacante y el puerto por el que queremos escuchar.
De payload seleccionamos el modo 'PHP PentestMonkey' y lo copiamos a un archivo con extensión .php y lo subimos a /upload.php

## 8) Abrimos el puerto para escuchar del lado de la máquina atacante con netcat
```
nc -n -vv -l -p 8080
```
`-n: solo IPs, evita resolución DNS`\
`-vv: doble verbosidad`\
`-l: modo escucha`\
`-p: puerto local`\

## 9) Buscamos ejecutar el archivo PHP que subimos con el payload para la reverse shell

Si vamos al endpointe `/uploads` ya vimos que se listaban todos los archivos. Si clickeamos a un archivo con extensión .php se va a ejecutar y recibimos, en la terminal con netcat corriendo, una conexión.

El usuario es `www-data`, ahora ya es cuestión de escalar privilegios.

## 10) Necesitamos escalar privilegios
Lo principal es ver si tenemos algún permiso especial para ejecutar comandos, para ello usamos\
```
sudo -ll
```
`-ll: lista los privilegios del usuario actual. Doble para más detalles`

Sorprendentemente el usuario actual tiene permisos para ejecutar `/usr/bin/env` como root. Entonces simplemente podemos aprovecharnos de ese permiso para abrir una terminal.

## 11) Escalando privilegios

```
sudo /usr/bin/env bash
```
Así de fácil logramos que se abra una terminal donde el usuario ahora es `root`.


## Resumen del ataque

Este método funcionó por lo siguiente:
* logramos un RCE via la subida de un archivo
* no se requirió contraseña del usuario actual para ejecutar `sudo`
* el usuario actual tiene acceso a `/usr/bin/env` como root

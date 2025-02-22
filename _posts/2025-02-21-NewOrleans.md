---
title: "Microcorruption CTF: Nivel 1, New Orleans"
date: 2025-02-21 17:00:02 +0100
categories: [Binary Exploitation, CTF]
tags: [microcorruption, ctf, reverse engineering, writeup]     # TAG names should always be lowercase
description: Introducción a la CTF microcorruption y solución del primer nivel.
---
[Microcorruption](https://microcorruption.com/) es una CTF en la que debemos desbloquear unos candados Bluetooth introduciendo inputs especialmente diseñados para trucar estos candados. Para completar esta tarea contamos con un debugger, con el que podremos ejecutar el código paso a paso, revisar la memoria, establecer breakpoints... Tras pasar el tutorial nos enfrentamos al primer nivel, New Orleans. 

## Candados LockIT Pro

Antes de lanzarnos a encontrar fallos en el código debemos familiarizarnos con el dispositivo. Para ello contamos con el manual del candado y el de la MCU (microcontroller unit) de estos candados, el MSP430. Este chip de Texas Instruments es un MCU con arquitectura RISC y 16 registros de 16 bits, que ha sido modificado para poder utilizar interrupciones de software con las que controlar los módulos de seguridad, la cerradura, imprimir caracteres o leerlos...


El set de instrucciones no es muy complejo y tenemos definidas las principales en el manual del candado, adicionalmente podemos consultar el manual del chip para ver más ejemplos y algunas funciones más avanzadas.


## El debugger

Al empezar el nivel contamos con un debugger en el que podemos consultar el código del programa, la memoria, los registros, la instrucción actual y una terminal desde la que controlamos el flujo del programa y otras opciones. Ejecutando el comando `help` obtenemos una lista de los comandos disponibles:

```
> help


Valid commands:

  Help - show this message

  Solve - solve the level on the real lock

  Reset - reset the state of the debugger

  (C)ontinue - run until next breakpoint

  (S)tep [count] - step [count] instructions

  step Over / (N)ext - step until out or pc is next instruction

  step Out / (F)inish - step until the function returns

  (B)reak [expr] - set a breakpoint at address

  (U)nbreak [expr] - remove a breakpoint

  (R)ead [expr] [c] - read [c] bytes starting at [expr]

  track [reg] - track the given register in memory

  untrack [reg] - removes the tracking of the given register

  (L)et [reg]/[addr] = [expr] - write to register or memory

  Breakpoints - show a list of breakpoints

  Insncount - count number of CPU cycles executed

  Manual - show the manual for this page


Scripting commands:

  #define name [commands] - alias "name" to run [commands].

  command;command - run first command, then second comamnd.


List of types:

  [reg] := 'r' followed by a number 0-15

  [addr] := base-16 integer or label name (e.g., 'main')

  [expr] := [reg] or [addr] or

            [expr]+[expr] or [expr]-[expr]

```


## Resolver el nivel

Despues de entender el dispositivo y conocer el debugger, es momento de buscar un fallo en el código. Lo primero que debemos hacer es entender la lógica del programa


### Entendiendo el programa

Lo primero que vamos a hacer es poner un breakpoint en la función main con el comando `break main`, despues ejecutamos el programa con el comando `c`.

![/assets/img/neworleans/main.png]


Podemos ver que el main llama a varias funciones, para imprimir strings y pedirle la contraseña al usuario. Hay dos funciones que llaman la atención por sus nombres, `create_password` y `check_password`. Empezemos por `create_password`, ponemos un breakpoint en la función y seguimos ejecutando el programa.

![/assets/img/neworleans/create_password]


La primera instrucción es `mov`, esta instrucción carga el valor 0x2400 al registro r15. Las siguientes instrucciones `mov.b`, mueven un único byte a la dirección que hay en r15 más N bytes (0xN(r15)). Según este código la contraseña se almacena en 0x2400 y es siempre la misma, pero vamos a comprobarlo viendo la función `check_password` ponemos un break en la función y continuamos la ejecución. El programa parara para pedirnos la contraseña, ponemos cualquier cosa y ejecutamos el comando `c` otra vez.


Si nos fijamos en la dirección 0x439c podemos ver que nuestra string esta almacenada allí. Siguiendo con la función `check_password`:

![/assets/img/neworleans/check_password.png]


Podemos ver que el programa almacena la dirección del primer caracter de nuestra string que esta en r15 en r13 con la instrucción `mov` y que despues le añade el valor de r14 que al principio es 0. Despues compara `cmp.b  @r13, 0x2400(r14)` el valor que se encuentra en la dirección almacenada en r13 con lo que hay en la dirección de r14 mas 0x2400 bytes. Si ambos valores son iguales la flag ZERO se pone en el registro `sr` (las flags activas se pueden ver poniendo el cursor encima de los valores del registro, Z es la flag ZERO y la C es de carry), despues viene un salto condicional `jnz $+0xc <check_password+0x16>` que se ejecuta si la flag Z no esta activa, este salto termina la función. La función en caso de que los dos primeros valores sean iguales aumenta el valor de r14 y lo compara con 0x8, esto permite que se cree un bucle que compara los 8 caracteres de la contraseña con los 8 primeros caracteres de la string (el octavo es el null byte 0x00), en c esta función se podría escribir como algo asi:

```c

char password[8] = {'p', 'a', 's', 's', 'w', 'o', 'r', 'd'}; // Este array representa cada caracter de la contraseña que se encuentran en 0x2400, 0x2401...

char user_input[9] = {'t', 'e', 's', 't', '_', 'p', 'a', 's', 's'}; // Este array son los caracteres que se encuentran en 0x439c, 0x439d...


// En esta función la i sería el valor en r14, el if sería el primer `cmp.b` y `jnz` y el segundo `cmp` y `jnz` el bucle for

void check_password() {

  for (int i = 0; i < 7; i++) {

    if (password[i] != [user_input]) {

      return;

    }

  }

}

```

Para ver paso a paso la ejecución de la función podemos usar el comando `s` y asi ver como va cambiando el valor de los registros. El programa termina escribiendo en el registro el byte 0x1 que luego servira para decidir si ejecutar la función `unlock_door`.


### Extrayendo la contraseña de la memoria

Ahora que entendemos como funciona el programa podemos recuperar la contraseña directamente de la memoria. Quitamos los breakpoints con el comando `unbreak` o clicando sobre ellos y dejamos el breakpoint de la función `create_password`. Ahora podemos volver a ejecutar el programa con los comandos `reset` y `c`. Ahora que nos encontramos en la función `create_password` podemos hacer varias cosas, copiar los bytes uno a uno o copiar los caracteres directamente de la memoria.


Para copiar las caracteres de la memoria debemos ejecutar la función entera con el comando `f`, si vamos a la dirección 0x2400 podremos ver la contraseña y copiarla.

![/assets/img/neworleans/password.png]

O podemos usar el siguiente comando `read 0x2400 7`. Ahora que ya tenemos la contraseña podemos introducirla y comprobar que funciona, despues podemos resolver el nivel con el comando `solve`.


## Conclusión

Este nivel permite aprender los básicos del lenguaje ensamblador de este chip y como funciona un programa básico, ademas, de familiarizarse con el debugger. En los proximos niveles, se iran complicando las cosas y pasaremos a explotar vulnerabilidades más complejas en el código. 

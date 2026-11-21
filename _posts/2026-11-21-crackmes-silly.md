---
title: "crackmes.one: Silly (Easy)"
date: 2026-11-21 12:00:00 -0500
categories: [Reversing]
tags: [crackmes, ctf, gdb, pwndbg, x86, easy]
description: "Writeup del crackme Silly: comparación de cadenas con repz cmpsb en un ELF de 32 bits."
---


# 1. Identificar el binario 


```sh
file main                                                                            ✔ 
main: ELF 32-bit LSB executable, Intel i386, version 1 (SYSV), statically linked, not stripped
```

- **ELF 32-bit**: Ejecutable de Linux de 32 bits.
- **LSB** : Little Endian (orden de bytes).
- **Intel i386** : Arquitectura x86 de 32 bits.
- **Version 1 (SYVS)** : ELF version 1, ABI System V.
- **Statically Linked** : Las librerias van incluidas en el binario y no depende de '.so' externos.
- **Not stripped** : Conserva los simbolos  (nombres de funciones), lo que facilita el analisis.

> [!warning] Interesante  
> - Tomar encuenta esto para reverse
> 
> - Un ejecutable puede estar:
> - Stripped: en este se han eliminado determiandos simbolos y nombres de funciones por la cual dificultad  el analisis
> - Not stripped: este conserva su informacion de simbolos que puede facilitar el analisis
> 


![[Pasted image 20261121171729.png]]


---

# 2. Ejecutarlo con un entrada cualquiera


```sh
./main                                                                               ✔ 
33
WRONG
```

- Al ingresar una entrada incorrecta, el programa imprime 'WRONG.'

![[Pasted image 20261121171949.png]]


# 3. Analisis (pwndbg)


## 3.1 Funciones del binario

```sh
info functions

Non-debugging symbols:
0x08049000  _start
0x08049032  trole
0x08049066  congratulation
0x08049088  error
0x080490aa  exit
```


- Al no estar stripped, los nombres ya sugerieren el flujo, 'Congratulation' , 'Exito' y '
  'error' , 'Fallo'.


## 3.2 Desensamblado de '_start'

```sh
disassemble _start
Dump of assembler code for function _start:
   0x08049000 <+0>:	call   0x8049032 <trole>
   0x08049005 <+5>:	mov    eax,0x3
   0x0804900a <+10>:	mov    ebx,0x0
   0x0804900f <+15>:	mov    ecx,0x804a02a
   0x08049014 <+20>:	mov    edx,0x11
   0x08049019 <+25>:	int    0x80
   0x0804901b <+27>:	lea    esi,ds:0x804a02a
   0x08049021 <+33>:	lea    edi,ds:0x804a018
   0x08049027 <+39>:	mov    ecx,0x11
   0x0804902c <+44>:	repz cmps BYTE PTR ds:[esi],BYTE PTR es:[edi]
   0x0804902e <+46>:	jne    0x8049088 <error>
   0x08049030 <+48>:	jmp    0x8049066 <congratulation>
```



| Instrucciones              | Que hacen                                                                                                                             |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| call trole                 | TODO: dIsassemble trole  **Probablemente imprime el prompt**                                                                          |
| mov eax, 3  ... int 0x80   | Syscall Linux x86 read(0, 0x804a02a, 17), lee hasta 0x11 **17 bytes** desde stdin hacia 0x804a02a                                     |
| lea esi / lea edi          | Carga las direcciones de origen *ESI* y de destino *EDI*  para la comparacion, de buffer de entrada vs la cadena esperada como "lmao" |
| mov ecx, 0x11 + repz cmpsb | Compara byte a byte hasta 0x11 *17* veces mientras *ZF = 1* y *ECX != 0*  ([ESI] frente a [EDI], avanzando punteros segun DF)         |
| jne error                  | Si algun byte difiere (ZF = 0), salta a error (normalmene imprime WRONG)                                                              |
| jmp congratulation         | Si todos los bytes coinciden, salta incondicionalmente a [congratulation]   (mensaje de extio).                                       |

## 3.3 Breakpoint en la comparacion

- Pues el breakpoint justo antes de *repz cmpsb*, porque en ese punto  *ESI* y *EDI* contienen los dos lados de la comparacion.


```sh
pwndbg> b *0x0804902c
Breakpoint 1 at 0x804902c
pwndbg> r
aaaaaaa

[ REGISTERS / show-flags off / show-compact-regs off ]
 EAX  8
 EBX  0
 ECX  0x11
 EDX  0x11
 EDI  0x804a018 (lmao) ◂— arpl word ptr [eax + 0x69], bp /* 0x63696863; 'chicken baguette\n' */
 ESI  0x804a02a (string) ◂— popal  /* 0x61616161; 'aaaaaaa\n' */    
```



| ESI | Apunta a mi entrada (aaaaaaa)\n                                                |
| --- | ------------------------------------------------------------------------------ |
| EDI | Apunta a la cadena esperada: *chicken baguette\n*.                             |
| EAX | =     8  es el retorno de *read* , bytes leidos (7 caracteres + \n)            |
| ECX | =  0x17 es la cantidad de bytes a comparar, *16 caracteres + el \n del enter*. |

- La cadena esperada esta en texto plano en *.data*, sin ofuscar.

![[Pasted image 20261121175735.png]]


# 4. Solucion

```sh
chicken baguette
```

```sh
./main
chicken baguette
you did the thing
```



- Happy Reverse :)



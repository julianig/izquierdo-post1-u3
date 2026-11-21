En la salida del comando D, la columna izquierda indica la dirección exacta de memoria en formato segmento:desplazamiento, avanzando 16 bytes (0x10) por cada fila. La columna central presenta el contenido de esos 16 bytes en formato hexadecimal, separados en dos bloques de ocho bytes mediante un guion medio. Finalmente, la columna derecha muestra la traducción de cada byte a su carácter ASCII correspondiente, representando con un punto (.) aquellos valores que caen fuera del rango imprimible (0x20–0x7E), como ocurre con 0xAB, 0xCD y 0xEF 



Decisión Técnica 1 — Verificación No Destructiva de una Escritura en Memoria (Paso 12):



Para confirmar que la escritura puntual realizada en DS:0300 tuvo el efecto esperado sin alterar el estado del programa, el comando correcto es D (Dump). Invocar E 300 sin la lista explícita de bytes activa el modo interactivo de DEBUG, el cual muestra el contenido actual de cada dirección y queda a la espera de pulsaciones de teclado, introduciendo un alto riesgo de sobrescribir bytes accidentalmente si el usuario presiona una tecla distinta de espacio o retorno de carro. Por su parte, aunque F (Fill) también opera sobre un rango completo de direcciones, es una instrucción estrictamente de escritura que sobreescribiría toda la región con un patrón nuevo, destruyendo los datos previos. En contraste con R, E y F —que permiten modificar registros o celdas de memoria—, D es el único comando puramente de solo lectura, propiedad indispensable para auditar el estado de la memoria de forma segura e intacta.



Decisión Técnica 2 — Direccionamiento Inmediato vs. Directo a Memoria (Paso 14):



Aunque MOV AX,0005 (B8 05 00) y MOV AX,\[0300] (A1 00 03) ocupan exactamente 3 bytes cada una, el direccionamiento directo a memoria (MOV AX,\[0300]) requiere un ciclo adicional de acceso al bus de memoria durante su ejecución. Esto ocurre porque en el modo inmediato el dato (0x0005) viaja embebido dentro del propio flujo de bytes de la instrucción ya captada, mientras que en el modo directo la instrucción solo contiene una dirección efectiva (0x0300) que la CPU debe resolver para realizar una segunda lectura en memoria y extraer el dato de 16 bits allí almacenado. Se prefiere el direccionamiento directo cuando se requiere leer valores dinámicos que pueden cambiar en tiempo de ejecución —como los bytes escritos previamente con E— en lugar de constantes fijas conocidas al ensamblar. Finalmente, el comando U permite verificar de manera estática qué codificación (B8 frente a A1) generó cada comando A sin necesidad de ejecutar ninguna instrucción.





(Checkpoint 1 - Parte 2)



Instrucción	AX	BX	CX	IP siguiente	ZF	CF	SF

MOV AX, 000A	000A	0000	0000	0103	0 (NZ)	0 (NC)	0 (PL)

MOV BX, 0005	000A	0005	0000	0106	0 (NZ)	0 (NC)	0 (PL)

MOV CX, 0003	000A	0005	0003	0109	0 (NZ)	0 (NC)	0 (PL)

ADD AX, BX	000F	0005	0003	010B	0 (NZ)	0 (NC)	0 (PL)

ADD AX, CX	0012	0005	0003	010D	0 (NZ)	0 (NC)	0 (PL)



(Checkpoint 2 y Paso 8)



Al inspeccionar la región CS:0100 mediante el comando D, se verificó que el programa completo ocupa exactamente 13 bytes en memoria (B8 00 00 B9 04 00 05 02 00 E2 FB CD 20). Las instrucciones de carga y suma inmediata (MOV AX,0000, MOV CX,0004 y ADD AX,0002) ocupan 3 bytes cada una al incluir su opcode de 1 byte seguido de un operando de 16 bits en formato little-endian, mientras que LOOP 0106 (E2 FB) e INT 20 (CD 20) ocupan 2 bytes cada una. En particular, el byte 0xFB de LOOP corresponde al desplazamiento relativo con signo de -5 en complemento a dos (0x0106 - 0x010B = -5), lo que permite al procesador retroceder exactamente hasta el inicio del cuerpo del bucle.

| Iteración | Instrucción | AX después | CX después | IP siguiente | ¿LOOP salta? |

| :--- | :--- | :---: | :---: | :---: | :---: |

| Inicialización 1 | `MOV AX, 0000` | `0000` | — | `0103` | No aplica |

| Inicialización 2 | `MOV CX, 0004` | `0000` | `0004` | `0106` | No aplica |

| Iteración 1 (cuerpo) | `ADD AX, 0002` | `0002` | `0004` | `0109` | No aplica |

| Iteración 1 (control) | `LOOP 0106` | `0002` | `0003` | `0106` | Sí (`CX ≠ 0`) |

| Iteración 2 (cuerpo) | `ADD AX, 0002` | `0004` | `0003` | `0109` | No aplica |

| Iteración 2 (control) | `LOOP 0106` | `0004` | `0002` | `0106` | Sí (`CX ≠ 0`) |

| Iteración 3 (cuerpo) | `ADD AX, 0002` | `0006` | `0002` | `0109` | No aplica |

| Iteración 3 (control) | `LOOP 0106` | `0006` | `0001` | `0106` | Sí (`CX ≠ 0`) |

| Iteración 4 (cuerpo) | `ADD AX, 0002` | `0008` | `0001` | `0109` | No aplica |

| Iteración 4 (control) | `LOOP 0106` | `0008` | `0000` | `010B` | No (`CX = 0`) |

| Terminación | `INT 20` | `0008` | `0000` | — | Finaliza programa |



\### Tabla de Traza — Bucle Equivalente con `DEC` y `JNZ`



| Iteración | Instrucción | AX después | CX después | IP siguiente | ¿JNZ salta? |

| :--- | :--- | :---: | :---: | :---: | :---: |

| Inicialización 1 | `MOV AX, 0000` | `0000` | `0000` | `0203` | No aplica |

| Inicialización 2 | `MOV CX, 0004` | `0000` | `0004` | `0206` | No aplica |

| Iteración 1 (cuerpo) | `ADD AX, 0002` | `0002` | `0004` | `0209` | No aplica |

| Iteración 1 (decremento) | `DEC CX` | `0002` | `0003` | `020A` | No aplica |

| Iteración 1 (salto) | `JNZ 0206` | `0002` | `0003` | `0206` | Sí (`ZF = 0`) |

| Iteración 2 (cuerpo) | `ADD AX, 0002` | `0004` | `0003` | `0209` | No aplica |

| Iteración 2 (decremento) | `DEC CX` | `0004` | `0002` | `020A` | No aplica |

| Iteración 2 (salto) | `JNZ 0206` | `0004` | `0002` | `0206` | Sí (`ZF = 0`) |

| Iteración 3 (cuerpo) | `ADD AX, 0002` | `0006` | `0002` | `0209` | No aplica |

| Iteración 3 (decremento) | `DEC CX` | `0006` | `0001` | `020A` | No aplica |

| Iteración 3 (salto) | `JNZ 0206` | `0006` | `0001` | `0206` | Sí (`ZF = 0`) |

| Iteración 4 (cuerpo) | `ADD AX, 0002` | `0008` | `0001` | `0209` | No aplica |

| Iteración 4 (decremento) | `DEC CX` | `0008` | `0000` | `020A` | No aplica |

| Iteración 4 (salto) | `JNZ 0206` | `0008` | `0000` | `020C` | No (`ZF = 1`) |

| Terminación | `INT 20` | `0008` | `0000` | — | Finaliza programa |



\### Comparación de Instrucciones Ejecutadas (`LOOP` vs. `DEC/JNZ`)



Ambos programas producen exactamente el mismo resultado final (`AX = 0008`), pero presentan distinto costo de ejecución: la versión con `LOOP` requirió \*\*11 instrucciones ejecutadas\*\* (2 de inicialización, 4 iteraciones de 2 instrucciones y 1 de terminación), mientras que la versión equivalente con `DEC` y `JNZ` requirió \*\*15 instrucciones ejecutadas\*\* (2 de inicialización, 4 iteraciones de 3 instrucciones y 1 de terminación), es decir, 4 instrucciones adicionales correspondientes al decremento explícito en cada vuelta.



\### Decisión Técnica 3 — Selección de Mecanismo de Control de Bucle (`LOOP` vs. `DEC/JNZ`)



Para un bucle contador simple donde el registro `CX` no se requiere dentro del cuerpo de iteración, el estudiante recomienda emplear la instrucción `LOOP` (`E2 FB`, 2 bytes) en lugar de la combinación `DEC CX` seguida de `JNZ` (`49 75 FA`, 3 bytes). Esta elección se justifica tanto por el ahorro de memoria de código (1 byte menos) como por la eficiencia de ejecución, ya que el procesador realiza una sola extracción (\*fetch\*) y decodificación de instrucción por iteración en lugar de dos. Sin embargo, el estudiante preferiría `DEC` y `JNZ` si el cuerpo del bucle necesitara utilizar `CX` para operaciones aritméticas o si la condición de salida dependiera de una comparación previa con `CMP` evaluando otras banderas. Finalmente, mediante el comando `U`, el estudiante verifica estáticamente los bytes ensamblados de ambas versiones sin ejecutar ninguna instrucción.



\### Decisión Técnica 4 — Comando de Verificación para Bucles de Muchas Iteraciones (`T` vs. `G`)



Al verificar el resultado final de un bucle con 100 iteraciones (`CX = 0064`), ejecutar `G 20C` resulta mucho más práctico que invocar `T` cientos de veces, pues el depurador ejecuta el bloque a velocidad completa hasta alcanzar el punto de interrupción en `020C`, aunque a cambio se pierde la visibilidad de los estados intermedios de los registros y banderas en cada vuelta. Por el contrario, en los Pasos 4, 7 y 11 del laboratorio fue estrictamente necesario utilizar `T` debido a que la construcción de las tablas de traza exigía observar la evolución exacta instrucción por instrucción. Finalmente, tras detenerse con `G 20C`, el estudiante utilizaría el comando `R AX` (o `R`) para confirmar únicamente el valor final almacenado en el acumulador.



\### Conclusiones



Mediante el desarrollo de este laboratorio en DOSBox, el estudiante comprobó el funcionamiento interno de la arquitectura x86 en modo real, verificando cómo el procesador interpreta indistintamente código y datos en memoria y almacena palabras de 16 bits en formato \*little-endian\*. Asimismo, el uso combinado de los comandos `R`, `D`, `F`, `E`, `U`, `A`, `T` y `G` permitió contrastar la inspección no destructiva de memoria frente a la modificación puntual, así como evaluar el impacto en tamaño de código y ciclos de ejecución entre estructuras de control como `LOOP` y `DEC/JNZ`.


---

### Evidencias Visuales del Laboratorio

A continuación se relacionan las siete capturas de pantalla almacenadas en el directorio `capturas/` que respaldan cada checkpoint de la práctica:

1. **Parte 1 — Checkpoint 1:** `capturas/CP1_registros.png` (Inspección y modificación de registros con `R`).
2. **Parte 1 — Checkpoint 2:** `capturas/CP2_volcado_memoria.png` (Volcado y llenado de memoria con `D` y `F`).
3. **Parte 1 — Checkpoint 3:** `capturas/CP3_ensamblado_desensamblado.png` (Ensamblado y desensamblado con `A` y `U`).
4. **Parte 1 — Checkpoint 4:** `capturas/CP4_memoria_direccionamiento.png` (Escritura puntual con `E` y direccionamiento directo).
5. **Parte 2 — Checkpoint 1:** `capturas/CP1_traza_suma.png` (Traza instrucción por instrucción del programa de suma con `T`).
6. **Parte 2 — Checkpoint 2:** `capturas/CP2_traza_loop.png` (Traza del bucle con `LOOP` y análisis de código máquina).
7. **Parte 2 — Checkpoint 3:** `capturas/CP3_traza_dec_jnz.png` (Traza del bucle equivalente `DEC`/`JNZ` y ejecución rápida con `G`).

---

### Especificaciones del Entorno de Ejecución

El laboratorio fue desarrollado en el emulador **DOSBox** utilizando la herramienta **DEBUG.EXE** (modo real de 16 bits, arquitectura Intel 8086/x86) bajo la estructura de directorios `LAB3POST1` y `LAB3POST2`, manteniendo el control de versiones mediante **Git** y **GitHub**.

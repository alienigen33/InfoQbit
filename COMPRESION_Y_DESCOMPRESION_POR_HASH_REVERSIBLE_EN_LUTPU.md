markdown
# COMPRESIÓN Y DESCOMPRESIÓN POR HASH REVERSIBLE EN LUTPU

**Documento técnico oficial de PROCESADORXOR v5.0 (LUTPU)**

**Autores:** Carlos Javier Avila & Bibiana Mariel Lopez

**Licencia:** MIT License — Copyright (c) 2025-2026

---

## 1. CONCEPTO FUNDAMENTAL

En LUTPU, **comprimir y descomprimir son exactamente la misma operación ejecutada en direcciones opuestas**. No se utiliza un algoritmo de compresión convencional, sino un **circuito de puertas lógicas reversibles** que reduce cualquier volumen de datos a un hash único de 32 bytes, y que después permite recuperar los datos originales simplemente invirtiendo el orden de aplicación de las puertas.

### Principio

- **Comprimir:** Pasar los datos originales por un circuito de puertas reversibles (XOR, Toffoli, Fredkin, etc.) hasta obtener una representación fija de 32 bytes (el hash).
- **Descomprimir:** Tomar ese hash y aplicar el mismo circuito **en sentido inverso** (última puerta primero, primera puerta al final). Como cada puerta es reversible, se reconstruyen los datos originales sin pérdida.
- **Propiedad clave:** No se almacenan los datos, sino la **secuencia de transformación** (el circuito). El hash solo es la semilla comprimida. La descompresión es la ejecución del programa inverso.

---

## 2. COMPRESIÓN: DE DATOS A HASH

La compresión en LUTPU consiste en **ejecutar un programa determinista hecho exclusivamente con puertas lógicas reversibles**. No hay análisis de patrones ni diccionarios. Simplemente se aplican transformaciones bit a bit que conservan toda la información.
DATOS ORIGINALES (cualquier tamaño)
│
▼
┌─────────────────────────────────────────┐
│ Circuito reversible (secuencia fija) │
│ Puertas: XOR, CNOT, Toffoli, Fredkin, │
│ ROTR, BSWAP, ADD, MIX, etc. │
└─────────────────────────────────────────┘
│
▼
HASH (32 bytes fijos)

text

- **Tamaño de entrada:** Arbitrario (bytes, kilobytes, gigabytes…).
- **Tamaño de salida:** Siempre 32 bytes.
- **Determinismo:** Los mismos datos de entrada siempre producen el mismo hash.
- **Sin colisiones:** Por ser reversible, dos entradas distintas nunca generan el mismo hash.

---

## 3. DESCOMPRESIÓN: DE HASH A DATOS ORIGINALES

La descompresión es **el mismo circuito ejecutado en orden inverso**. No hay un algoritmo de descompresión separado; solo se invierte la secuencia de puertas.
HASH (32 bytes)
│
▼
┌─────────────────────────────────────────┐
│ Circuito inverso (puertas en orden │
│ contrario al de compresión) │
└─────────────────────────────────────────┘
│
▼
DATOS ORIGINALES (reconstrucción exacta)

text

- **Sin pérdida:** El resultado es idéntico al original.
- **Tiempo de ejecución:** Depende de la profundidad del circuito, pero siempre es determinista.
- **Sin RAM adicional:** Los datos no se expanden durante el proceso; se van reconstruyendo paso a paso.

---

## 4. PUERTAS LÓGICAS REVERSIBLES EMPLEADAS

Las 16 puertas lógicas de la ALU LUTPU son el único elemento del circuito. Cada una realiza una operación sobre bloques de 64 bits y es estrictamente reversible.

| # | Puerta | Reversible porque... |
|:---|:---|:---|
| 0 | XOR | Es su propia inversa |
| 1 | OR (con shift) | Se invierte con la puerta AND correspondiente |
| 2 | AND (con shift) | Se invierte con OR |
| 3 | NOT | Es su propia inversa |
| 4 | ROTR | ROTL la revierte |
| 5 | BSWAP | Es su propia inversa |
| 6 | ADD | SUB la revierte |
| 7 | MIX | Se invierte con XOR2 |
| 8 | XOR2 | Se invierte con MIX |
| 9 | OR2 | Se invierte con AND2 |
| 10 | AND2 | Se invierte con OR2 |
| 11 | NOT2 | Es su propia inversa |
| 12 | ROTL | ROTR la revierte |
| 13 | SUB | ADD la revierte |
| 14 | MUL | Se invierte con tabla precalculada (o XNOR) |
| 15 | XNOR | Es su propia inversa |

El circuito de compresión es una secuencia predefinida de estas puertas. El circuito de descompresión es exactamente la misma secuencia, pero con cada puerta reemplazada por su inversa y ejecutada en orden contrario.

---

## 5. POR QUÉ FUNCIONA PARA TODOS LOS TAMAÑOS

1. **Procesamiento por bloques:** Los datos de entrada se dividen en fragmentos de 64 bits, y el circuito opera sobre cada bloque.
2. **Encadenamiento reversible:** La salida de un bloque se realimenta con el siguiente mediante XOR o Toffoli, permitiendo que el circuito "pliegue" progresivamente la información hasta un único vector de 256 bits.
3. **Profundidad adaptable:** Para un archivo de gigabytes, el circuito simplemente aplica más vueltas, pero la lógica subyacente sigue siendo la misma.
4. **Sin pérdida de información:** Cada etapa es reversible, así que todos los bits originales sobreviven en el hash y se pueden recuperar.

---

## 6. RELACIÓN CON EL SISTEMA DE DIRECCIONES LUTPU

La compresión descrita aquí es independiente del concepto de que "la dirección ES el dato". Sin embargo, ambos mecanismos pueden convivir:

- **Compresión reversible:** Se usa para almacenar, transmitir o archivar cualquier contenido llevándolo a 32 bytes.
- **Direccionamiento directo:** Cuando se necesita acceder a un elemento individual (por ejemplo, un peso de red neuronal), se genera la dirección correspondiente desde el hash expandido. La dirección **ya es** el peso.

En otras palabras, el hash comprimido sirve como semilla para el espacio de direcciones, y el circuito reversible es el puente entre ambos mundos.

---

## 7. EJEMPLO CONCEPTUAL

**Objetivo:** Comprimir y recuperar la frase "Hola Mundo".

1. **Compresión:**
   - Datos: 10 bytes.
   - Circuito: XOR → Toffoli → Fredkin → ADD → MIX → BSWAP → (resultado 32 bytes).
   - Hash: `0xA3F2B1C9...` (32 bytes).

2. **Descompresión:**
   - Se toma el hash.
   - Se aplica el circuito inverso: BSWAP → MIX → SUB → Fredkin inverso → Toffoli inverso → XOR.
   - Resultado: "Hola Mundo".

---

## 8. COMPARACIÓN CON MÉTODOS TRADICIONALES

| Característica | Compresión tradicional (ZIP, gzip...) | Compresión LUTPU |
|:---|:---|:---|
| Técnica | Análisis de patrones, diccionarios | Circuito reversible de puertas lógicas |
| Tamaño resultante | Variable | Fijo: 32 bytes |
| Descompresión | Algoritmo separado | Mismo circuito en orden inverso |
| Pérdida | Sin pérdida / Con pérdida | Siempre sin pérdida |
| Dependencia | Del algoritmo y los diccionarios | Solo depende del circuito reversible |
| Almacenamiento del modelo | Se necesita guardar el algoritmo | Solo se necesita guardar la secuencia de puertas (unos pocos bytes) |

---

## 9. LICENCIA MIT
MIT License

Copyright (c) 2025-2026 Carlos Javier Avila & Bibiana Mariel Lopez

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

text

---

## 10. VERSIÓN EN ESPAÑOL DE LA LICENCIA
Licencia MIT

Copyright (c) 2025-2026 Carlos Javier Ávila y Bibiana Mariel López

Se concede permiso por la presente, sin cargo alguno, a cualquier persona que
obtenga una copia de este software y sus archivos de documentación asociados
(el "Software"), para tratar el Software sin restricción alguna, incluyendo sin
limitación los derechos de usar, copiar, modificar, fusionar, publicar,
distribuir, sublicenciar y/o vender copias del Software, y permitir que las
personas a quienes se les proporcione el Software lo hagan, sujeto a las
siguientes condiciones:

El aviso de derechos de autor anterior y este aviso de permiso se incluirán en
todas las copias o porciones sustanciales del Software.

EL SOFTWARE SE PROPORCIONA "TAL CUAL", SIN GARANTÍA DE NINGÚN TIPO, EXPRESA O
IMPLÍCITA, INCLUYENDO, ENTRE OTRAS, LAS GARANTÍAS DE COMERCIABILIDAD,
IDONEIDAD PARA UN PROPÓSITO PARTICULAR Y NO INFRACCIÓN. EN NINGÚN CASO LOS
AUTORES O TITULARES DE LOS DERECHOS DE AUTOR SERÁN RESPONSABLES DE CUALQUIER
RECLAMACIÓN, DAÑO U OTRA RESPONSABILIDAD, YA SEA EN UNA ACCIÓN CONTRACTUAL,
EXTRACONTRACTUAL O DE OTRO MODO, QUE SURJA DE, FUERA DE O EN CONEXIÓN CON EL
SOFTWARE O EL USO U OTROS TRATOS EN EL SOFTWARE.

text

---

**Fin del documento.**

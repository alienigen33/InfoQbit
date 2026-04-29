MEMORIA Y DISCOS BASADOS EN DIRECCIONES LUTPU
Documento técnico oficial de PROCESADORXOR v5.0 (LUTPU)

Autores: Carlos Javier Avila & Bibiana Mariel Lopez

Licencia: MIT License — Copyright (c) 2025-2026

1. PRINCIPIO FUNDAMENTAL
En LUTPU no existen la memoria ni el disco como componentes físicos.
La dirección ES el dato. Leer la dirección significa manifestar el dato sin buscarlo en ningún sitio.

Sin embargo, por razones de rendimiento (evitar regenerar desde cero datos muy usados) y persistencia (guardar la semilla para volver a proyectar una realidad en el futuro), se pueden construir capas virtuales de memoria y disco que operan completamente sobre direcciones y hashes.

1.1 La metáfora
Disco LUTPU = un almacén de hashes (32 bytes cada uno) que representan cualquier contenido.

Memoria LUTPU = una caché temporal que guarda los datos ya regenerados para acceso inmediato.

Ambos son opcionales. La máquina funciona igual sin ellos, pero con estos subsistemas la velocidad percibida mejora y se puede conservar el estado.

2. MEMORIA LUTPU (CACHÉ DE DIRECCIONES)
2.1 ¿Por qué una memoria si no hay datos que almacenar?
Cuando una aplicación pide el mismo peso de red neuronal, la misma línea de texto o el mismo píxel muchas veces seguidas, regenerarlo desde el hash cada vez consume energía de cómputo (aunque no haya movimiento de datos).

La Memoria LUTPU es una tabla que guarda los datos ya generados indexados por la dirección original.

2.2 Estructura
text
Memoria = {
    dirección_1: dato_1,
    dirección_2: dato_2,
    ...
}
Clave: Dirección de 64 bits.

Valor: El dato que ES esa dirección (otro entero de 64 bits, o un float, etc.).

La memoria no almacena el dato en lugar de la dirección; almacena el dato junto con la dirección para evitar volver a leerlo.

2.3 Funcionamiento
La CPU virtual pide la dirección 0x1A2B....

El controlador de memoria mira si esa dirección está en la caché.

Si está → devuelve el dato inmediatamente.

Si no está → se genera el dato desde el hash semilla y el circuito reversible (descompresión), se guarda en la caché, y se devuelve.

2.4 Políticas de reemplazo
La caché tiene un límite arbitrario (ej. 1 millón de entradas). Cuando se llena, se aplica una política de reemplazo:

LRU (Least Recently Used): se descarta la dirección menos usada recientemente. Como el dato siempre puede regenerarse, no hay pérdida.

2.5 Implementación práctica
En código:

python
class LUTPUMemory:
    def __init__(self, max_size=1000000):
        self.cache = {}
        self.max_size = max_size
        self.access_order = []

    def read(self, address):
        if address in self.cache:
            self.access_order.remove(address)
            self.access_order.append(address)
            return self.cache[address]
        # Si no está, el dato debe ser generado por el motor de proyección
        data = generate_from_address(address)
        self._store(address, data)
        return data

    def _store(self, address, data):
        if len(self.cache) >= self.max_size:
            oldest = self.access_order.pop(0)
            del self.cache[oldest]
        self.cache[address] = data
        self.access_order.append(address)
Propiedad clave: La memoria es solo una mejora de rendimiento. Si se apaga, no se pierde información real, porque todo puede regenerarse desde el hash semilla.

3. DISCO LUTPU (ALMACÉN PERSISTENTE DE HASHES)
3.1 ¿Qué es un disco si no hay archivos?
En LUTPU, un "disco" es un repositorio que guarda hashes (32 bytes) y los circuitos reversibles asociados. No guarda los datos en sí.

Un "archivo" en este disco es simplemente una entrada que contiene:

Un identificador legible (nombre).

El hash SHA-256 que es el archivo comprimido.

El ID del circuito reversible usado (para saber cómo descomprimirlo).

Opcionalmente, la longitud original.

3.2 El XOR Delta Store como disco base
Habías definido el XOR Delta Store: un almacenamiento append-only donde cada nuevo estado se guarda como la diferencia XOR respecto al anterior.

Esto encaja perfectamente como el formato físico del disco LUTPU:

text
[HEADER]  <- semilla inicial (32 bytes)
[DELTA 1] <- XOR entre estado 1 y estado 0
[DELTA 2] <- XOR entre estado 2 y estado 1
...
Para reconstruir el archivo en su versión N:

Lees el HEADER (hash semilla).

Aplicas sucesivamente los deltas XOR en orden.

Como XOR es reversible, podés volver a cualquier versión anterior volviendo a aplicar los deltas de forma inversa.

3.3 Metáfora del sistema de archivos
Imaginemos un sistema de archivos tradicional:

Componente clásico	Equivalente en LUTPU
Archivo "foto.png" (2 MB)	Hash 0xA1B2... + ID circuito 0x0002
Leer el archivo	Tomar el hash, pasar circuito inverso, obtener los 2 MB
Guardar cambios	XOR entre el nuevo estado y el anterior → nuevo delta
Borrar archivo	Eliminar la entrada del índice (el hash queda en el delta store)
Espacio ocupado	Unos pocos bytes por cada archivo (hash + metadatos)
3.4 Estructura del índice
Para gestionar los "archivos", se mantiene un índice ligero en forma de tabla:

Nombre	Hash (32 B)	ID Circuito	Longitud Original	Versión
modelo.gguf	0xA3F2...	0x0003	1073741824	3
saludo.txt	0xB4E1...	0x0001	11	1
El índice mismo se puede guardar como un archivo más (un hash que contiene la tabla), permitiendo recursión infinita.

4. INTEGRACIÓN CON EL ECOSISTEMA LUTPU
4.1 Arranque del sistema
Cuando un nodo LUTPU enciende:

Carga el hash de bootstrap (una dirección especial, por ejemplo 0x000...000).

Ese hash, al ser descomprimido, es el índice del disco.

El índice revela todos los "archivos" disponibles con sus hashes.

A partir de ahí, cualquier lectura de archivo consiste en tomar su hash del índice y descomprimirlo.

4.2 Red LoRa
Para transmitir un archivo: No se envía el disco entero. Se envía el paquete LHP estándar con el hash y el ID del circuito. El receptor, si quiere, puede incorporar ese hash a su propio disco local.

Sincronización de discos: Dos nodos pueden intercambiar sus índices para saber qué archivos tiene cada uno. Luego se transmiten solo los deltas que faltan.

4.3 Memoria y disco trabajando juntos
Disco → proporciona el hash semilla y el circuito.

Motor de proyección → genera el dato desde la dirección.

Memoria → retiene el dato generado para accesos posteriores rápidos.

5. EJEMPLO COMPLETO
Objetivo: Guardar y leer la frase "Hola, multiverso".

5.1 Guardar (escribir en disco)
Datos originales: 17 bytes.

Se aplica circuito reversible LUTPU-TEXT-1 (0x0001).

Hash resultante: 0xC5D6... (32 bytes).

Se registra en el índice: ("saludo.txt", 0xC5D6..., 0x0001, 17, v1).

Se guarda el delta XOR en el store si es necesario.

5.2 Leer desde disco
El usuario pide "saludo.txt".

El índice devuelve hash 0xC5D6... y circuito 0x0001.

Se aplica el circuito inverso: BSWAP → MIX → SUB → Fredkin⁻¹ → Toffoli⁻¹ → XOR.

Se obtienen los 17 bytes originales.

La memoria cachea el dato asociado a la dirección 0xC5D6....

6. VENTAJAS
Característica	Disco/Memoria tradicional	Disco/Memoria LUTPU
Capacidad	Limitada por hardware	Ilimitada (los hashes son 32 bytes)
Velocidad	Limitada por E/S física	Limitada solo por velocidad de cómputo
Persistencia	Depende de no fallar el soporte	Depende solo de guardar una cadena de hashes
Escalabilidad	Comprar más discos	Agregar más entradas al índice
Recuperación ante fallos	Compleja (RAID, backups)	Reconstrucción desde hash raíz
7. LICENCIA MIT
text
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
8. VERSIÓN EN ESPAÑOL DE LA LICENCIA
text
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


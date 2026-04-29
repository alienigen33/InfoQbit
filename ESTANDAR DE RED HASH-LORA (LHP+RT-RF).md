ESTÁNDAR DE RED HASH-LORA (LHP + RT-RF)
Protocolo de Transmisión de Realidad en Tiempo Real
Documento técnico oficial de PROCESADORXOR v5.0 (LUTPU)

Autores: Carlos Javier Avila & Bibiana Mariel Lopez

Licencia: MIT License — Copyright (c) 2025-2026

1. PROPÓSITO
Este documento define el protocolo estándar de comunicación para la transmisión de hashes a través de radio LoRa en la red LUTPU, incluyendo el modo de tiempo real.

El objetivo es garantizar que cualquier nodo que reciba una señal de radio pueda:

Operar en dos modos: Transferencia (archivos completos) y Tiempo Real (streaming).

Identificar que se trata de un paquete LUTPU válido.

Extraer el hash de 32 bytes que representa el contenido comprimido.

Retransmitir paquetes en tiempo real con latencia mínima mediante flooding inteligente.

2. MODOS DE OPERACIÓN
El protocolo LHP define dos modos principales de transmisión. El campo Tipo de Contenido en el paquete determina cuál se usa.

2.1 MODO TRANSFERENCIA (Tipo 0x01 a 0x06)
Uso: Transmisión de archivos completos (modelos de IA, documentos, imágenes, audio, código).

Hash: Representa el archivo entero comprimido reversiblemente.

Entrega: Store-and-forward. El nodo recibe, descomprime y almacena/usa el archivo completo.

Latencia: No crítica. Puede tardar varios segundos o minutos.

2.2 MODO TIEMPO REAL — RT-RF (Tipo 0xF0 y 0xF1)
Uso: Streaming de realidad sensorial en vivo (videollamadas, audio, telemetría).

Hash: Representa un "frame" o instante del contenido. Se generan múltiples hashes por segundo.

Entrega: Flooding inteligente con TTL. Cada nodo retransmite el paquete UNA SOLA VEZ.

Latencia: Crítica. Cada salto añade solo ~8 ms.

3. ESTRUCTURA DEL PAQUETE ESTÁNDAR
3.1 Paquete de Transferencia (46 bytes fijos)
Offset	Campo	Tamaño	Descripción
0	Preámbulo	1 byte	0xAA (fijo).
1	Versión	1 byte	0x01.
2	ID del Circuito	2 bytes	Secuencia de puertas reversibles usada.
4	Tipo de Contenido	1 byte	0x01-0x06.
5	Longitud Original	4 bytes	Tamaño del archivo original en bytes.
9	Hash SHA-256	32 bytes	Hash comprimido del archivo completo.
41	Sal / Futuro	4 bytes	Reservado (0x00000000).
45	Suma de Verificación	1 byte	XOR de todos los bytes anteriores.
Transmisión por LoRa (10 kbps): 46 bytes × 8 bits / 10,000 bps = 36.8 ms.

3.2 Paquete de Tiempo Real RT-RF (22 bytes fijos + opcionales)
Este paquete está optimizado para la mínima latencia posible.

Offset	Campo	Tamaño	Descripción
0	Preámbulo	1 byte	0xBB (fijo, distinto de transferencia).
1	Tipo 0xF0	1 byte	0xF0 = Frame de realidad, 0xF1 = Audio.
2	TTL	1 byte	Tiempo de vida (3-5 inicial).
3	Saltos	1 byte	Contador de cuántos nodos han retransmitido (empieza en 0).
4	Timestamp	2 bytes	Milisegundos (0-65535).
6	Hash SHA-256	16 bytes	Hash truncado a 128 bits (suficiente para un frame).
22	Suma de Verificación	1 byte	XOR de los bytes 0-21.
Total: 23 bytes.

Transmisión por LoRa (SF5, 500 kHz): 23 bytes × 8 bits / 50,000 bps ≈ 3.7 ms por paquete.

4. FUNCIONAMIENTO DEL MODO TIEMPO REAL (RT-RF)
El modo RT-RF se basa en flooding inteligente con TTL.

4.1 Reglas de flooding
Un nodo origen transmite un paquete RT-RF con TTL inicial (3-5) y Saltos=0.

Todos los nodos que escuchan:

Procesan el hash inmediatamente (proyectan el frame/audio).

Si TTL > 1, retransmiten el paquete con TTL-1 y Saltos+1.

Si ya recibieron ese mismo timestamp antes (hash+timestamp duplicado), ignoran.

El paquete se propaga por la red en árbol, cubriendo hasta 5 saltos.

4.2 Latencia por salto
Configuración LoRa	Latencia por paquete (23 bytes)
SF7, 500 kHz	~8 ms
SF6, 500 kHz	~4.6 ms
SF5, 500 kHz	~3.7 ms
Ejemplo con 5 saltos (SF5):

text
[Nodo A] → (3.7ms) → [Nodo B] → (3.7ms) → [Nodo C] → (3.7ms) → [Nodo D] → (3.7ms) → [Nodo E]
Latencia total: 18.5 ms para cubrir 5 saltos (~200 km en campo abierto).
4.3 Ancho de banda para tiempo real
Aplicación	Frames por segundo	Bytes/segundo	Latencia máxima
Voz (full duplex)	50	1,150 B/s (9.2 kbps)	< 50 ms
Video + Voz	30	690 B/s (5.5 kbps)	< 50 ms
Telemetría	100	2,300 B/s (18.4 kbps)	< 100 ms
LoRa a 50 kbps soporta todos estos escenarios sin problemas.

5. IDENTIFICADOR DEL CIRCUITO REVERSIBLE (ID del Circuito)
Este campo es común a ambos modos y define la secuencia de puertas a aplicar.

5.1 Circuitos Predefinidos (Obligatorios)
ID	Nombre	Secuencia de Puertas	Uso Típico
0x0001	LUTPU-TEXT-1	XOR → Toffoli → Fredkin → ADD → MIX → BSWAP	Texto y datos pequeños
0x0002	LUTPU-MEDIA-1	XOR → Toffoli → Fredkin → ROTR → MIX → ADD → BSWAP → XOR2	Imágenes, audio
0x0003	LUTPU-LLM-1	XOR → Toffoli → Fredkin → ADD → ROTL → MIX → BSWAP → AND2 → OR2	Modelos de IA
0x0004	LUTPU-RT-1	XOR → Toffoli → Fredkin → MIX	Tiempo real (mínima latencia)
5.2 Circuitos Personalizados
IDs mayores a 0x00FF deben publicarse en el registro público de extensiones de la red LUTPU.

6. FLUJO DE OPERACIÓN COMPLETO
6.1 Transferencia de archivo (Modo Transferencia)
Emisor comprime archivo con circuito reversible → obtiene hash de 32 bytes.

Construye paquete LHP de 46 bytes.

Transmite por LoRa.

Receptor descomprime el hash con el circuito inverso → obtiene archivo original.

6.2 Llamada en tiempo real (Modo RT-RF)
Emisor genera frame de realidad (ej. audio de 20 ms).

Comprime con circuito 0x0004 (LUTPU-RT-1) → hash de 16 bytes.

Construye paquete RT-RF de 23 bytes con TTL=4, timestamp actual.

Transmite por LoRa (3.7 ms).

Cada nodo que escucha:

Proyecta el frame inmediatamente (sin esperar).

Retransmite el paquete (si TTL lo permite).

La red se inunda en tiempo real.

7. COMPARACIÓN DE MODOS
Característica	Modo Transferencia	Modo RT-RF (Tiempo Real)
Tamaño paquete	46 bytes	23 bytes
Hash	32 bytes (SHA-256)	16 bytes (truncado)
Latencia por paquete	~37 ms	~3.7 ms
Entrega	Punto a punto o store-and-forward	Flooding con TTL
Retransmisión	Bajo demanda	Automática por cada nodo
Uso	Archivos, modelos, documentos	Voz, video, telemetría, sincronización
8. LICENCIA MIT
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
9. VERSIÓN EN ESPAÑOL DE LA LICENCIA
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
Fin del documento.

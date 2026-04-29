ESTÁNDAR DE PUENTE LORA-INTERNET (LIP - LUTPU Internet Protocol)
Documento técnico oficial de PROCESADORXOR v5.0 (LUTPU)

Autores: Carlos Javier Avila & Bibiana Mariel Lopez

Licencia: MIT License — Copyright (c) 2025-2026

1. PROPÓSITO Y ALCANCE
1.1 ¿Qué es el LIP?
El Protocolo de Puente LUTPU-Internet (LIP) define cómo un nodo de la red LUTPU-LoRa actúa como punto de salida y entrada entre la malla de radio descentralizada y la Internet global basada en IP.

Este puente es asimétrico por diseño: aprovecha la estructura de la red LUTPU para ser extremadamente eficiente en la dirección LoRa → Internet y viceversa.

1.2 Funciones del Puente
Puerta de Enlace de Subida (Uplink): Recibir paquetes LHP (de transferencia o tiempo real) de la malla LoRa, regenerar los datos originales mediante el circuito reversible, y publicarlos en Internet a través de APIs estándar.

Puerta de Enlace de Bajada (Downlink): Recibir solicitudes o datos de Internet, comprimirlos en un hash LHP, y transmitirlos a los nodos correspondientes dentro de la malla LoRa.

Nodo de Caché Universal: Almacenar los hashes y los datos más solicitados. Una vez que el puente ha visto un contenido, servirlo de nuevo a cualquier nodo de la malla no cuesta ancho de banda adicional.

2. ARQUITECTURA DE LA CONEXIÓN
El Puente LIP es físicamente un nodo con dos interfaces de red: una de radio (el módulo LoRa SX1262) y una de red IP (Ethernet, WiFi, o 4G).

text
                    ┌───────────────────────┐
                    │        INTERNET       │
                    │     (Cloud, APIs)     │
                    └───────────┬───────────┘
                                │ (Ethernet/WiFi/4G)
                    ┌───────────┴───────────┐
                    │    PUENTE LIP         │
                    │ ┌───────────────────┐ │
                    │ │  Servidor LUTPU   │ │
                    │ │  (Descompresor)   │ │
                    │ └───────────────────┘ │
                    │ ┌───────────────────┐ │
                    │ │  Proxy/Caché LHP  │ │
                    │ └───────────────────┘ │
                    └───────────┬───────────┘
                                │ (USB/SPI)
                    ┌───────────┴───────────┐
                    │     Módulo LoRa      │
                    │      (SX1262)        │
                    └───────────┬───────────┘
                                │
                    ~~~~~~ AIRE (915 MHz) ~~~~~~
                                │
                    ┌───────────┴───────────┐
                    │   MALLA LUTPU-LoRa    │
                    │  (Nodos hoja, RT-RF) │
                    └───────────────────────┘
El Puente actúa como un "traductor universal" entre el mundo de los hashes (la malla LoRa) y el mundo de los datos (Internet). Los usuarios estándar de Internet no necesitan saber que están interactuando con una red LoRa.

3. PROTOCOLO DE OPERACIÓN
El puente maneja dos tipos de tráfico de manera distinta, respetando el diseño del estándar LHP + RT-RF.

3.1 Modo Transferencia (Archivos, Modelos, Datos)
Este modo se usa para publicar contenido desde un nodo a Internet o viceversa.

Subida de un Nodo a la Web:

Un nodo de la malla envía un paquete LHP de transferencia (46 bytes) al puente.

El puente recibe el paquete, lee el ID del Circuito y descomprime el archivo completo usando las puertas reversibles.

El puente publica el archivo (ej. una lectura de sensor o un modelo de IA) en un servidor MQTT en la nube o en una API REST, utilizando JSON estándar sobre TCP/IP.

Bajada de la Web a un Nodo:

Un servicio en Internet envía un archivo o comando al puente (ej. a través de una API REST).

El puente comprime el archivo con el circuito reversible estándar para obtener el hash de 32 bytes.

Construye un paquete LHP de transferencia y lo transmite al nodo de destino a través de LoRa, usando la malla para el enrutamiento.

3.2 Modo Tiempo Real (RT-RF)
El puente actúa como un nodo más de la malla para el tráfico RT-RF, pero es el único nodo raíz con TTL inicial para las transmisiones que se originan en Internet.

De Internet a la Malla: Si un usuario en Internet está enviando su voz, el puente recibe los paquetes de audio, los convierte en hashes RT-RF y los inyecta en la malla LoRa con un TTL de 3.

De la Malla a Internet: El puente recibe los paquetes RT-RF desde la malla, regenera el audio original (gracias al circuito reversible) y lo envía a la aplicación de Internet, logrando una latencia total de apenas unas pocas decenas de milisegundos.

4. COMPONENTES DE SOFTWARE DEL PUENTE
Para operar, el Puente LIP ejecuta una pila de software con tres capas principales:

4.1 Agregador de Paquetes LoRa (basado en ChirpStack)
Aunque no usamos la red LoRaWAN, el puente necesita un software robusto para interactuar con el hardware SX1262. Esta capa se encarga de la comunicación de bajo nivel con el módulo de radio, recibiendo y transmitiendo paquetes LHP en crudo. ChirpStack Gateway Bridge es una excelente base para esto, ya que abstrae la complejidad del hardware.

4.2 Servidor de Red LUTPU (LNS)
Es el cerebro del puente. En lugar de ser un simple reenviador ("packet forwarder"), el LNS inspecciona cada paquete:

Identifica si es un paquete LHP o RT-RF.

Aplica el circuito reversible inverso correspondiente.

Toma decisiones sobre el enrutamiento (¿es para Internet o para otro nodo de la malla?).

4.3 Puente de Aplicación (MQTT/P2P)
Es la interfaz con el mundo IP. Para una máxima compatibilidad y sencillez, se usa un bróker MQTT local (como Mosquitto) que convierte los mensajes de la red LUTPU en un flujo de datos estándar. Cualquier aplicación (Node-RED para dashboards, Telegram Bots, bases de datos, asistentes de IA) puede suscribirse a estos tópicos MQTT para interactuar con la malla.

5. SEGURIDAD Y PRIVACIDAD END-TO-END
El Puente LIP está diseñado para ser un componente de máxima confianza (como el router de casa) o de confianza cero (punto de acceso público), y el protocolo se adapta a ambos.

Cifrado Inherente: El puente solo ve el hash y el identificador de circuito. Si el nodo emisor y el receptor final han acordado un circuito reversible personalizado (no estándar), el puente no puede descifrar el contenido. Solo lo reenvía. Esto proporciona un cifrado de extremo a extremo sin sobrecoste.

Modo Proxy Transparente: Si el puente debe optimizar el tráfico, puede actuar como proxy. Inspecciona el contenido para cachearlo y servirlo localmente a otros nodos sin consumir ancho de banda de Internet. Piensa en ello como tener una copia local de Wikipedia, actualizable por goteo, a la que todos los nodos pueden acceder al instante.

6. IMPLEMENTACIÓN DE REFERENCIA
Para mantener el espíritu de bajo costo y la máxima accesibilidad, la implementación de referencia del Puente LIP se hace sobre Raspberry Pi:

6.1 Configuración de Hardware
Procesador: Raspberry Pi 4 (o similar).

Radio LoRa: Módulo USB a LoRa SX1262 industrial, conectado directamente al puerto USB.

Antena: Antena LoRa de alta ganancia (ej. 8 dBi) para exterior.

Red IP: Conectado al router doméstico por cable Ethernet (para máxima fiabilidad).

6.2 Pila de Software
Sistema Operativo: Raspberry Pi OS (Linux).

Framework de Radio: ChirpStack Gateway Bridge (para la gestión del hardware LoRa).

Bróker MQTT: Mosquitto.

Servidor LUTPU: Implementación propia en Python o C++ de duck_brain que maneja los circuitos reversibles.

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

# Protocolo DHCP - Seguridad

## 1. Fundamentos y Funcionamiento del Protocolo DHCP

### El Proceso DORA (¿Cómo pedimos una IP?)
El protocolo DHCP sirve para asignar direcciones IP de forma automática a los equipos que se conectan a una red. Para conseguir una IP, el ordenador cliente y el servidor hablan mediante 4 pasos sencillos conocidos como **DORA**:

1. **Discover (Descubrimiento):** El ordenador acaba de conectarse y no sabe en qué red está ni quién es el servidor. Envía un mensaje "a grito pelado" (Broadcast) a toda la red diciendo: *"¡Hola! ¿Hay algún servidor DHCP por aquí? Necesito una IP"*.
2. **Offer (Oferta):** El servidor DHCP escucha el grito y responde: *"Hola, tengo libre la IP 192.168.1.50, ¿te va bien?"*.
3. **Request (Petición):** El cliente responde a toda la red: *"¡Perfecto! Me quedo con la IP 192.168.1.50 que me ha ofrecido este servidor"*.
4. **Ack (Confirmación):** El servidor le contesta: *"Anotado. Tienes esa IP reservada durante un tiempo determinado. ¡Ya puedes navegar!"*.

#### El peligro de "hablar a gritos" (Broadcast)
Al enviar los mensajes abiertos a toda la red, **cualquier equipo conectado al mismo cable o red Wi-Fi puede ver lo que pasa**. Un atacante puede aprovechar esto para:
* Escuchar a escondidas y saber qué IPs hay en la red.
* Hacerse pasar por un servidor DHCP falso (Rogue DHCP) y darle a las víctimas una IP con datos engañosos para robarles la información.

---

### ¿Qué hay dentro del mensaje DHCP? (Capa 2 vs Capa 7)
Un mensaje DHCP lleva información en distintas "capas":

* **Dirección MAC de la tarjeta de red (Capa 2):** Es como el DNI físico del chip de red. Viene grabado de fábrica en la tarjeta de red.
* **Campo `chaddr` (Capa 7):** Es una casilla dentro del texto del propio mensaje DHCP donde el cliente escribe su MAC para identificarse ante el servidor.

#### ¿Por qué el servidor DHCP mira la casilla `chaddr` y no la MAC física?
Si la red tiene varias plantas o zonas separadas por un router, el router reenvía la petición DHCP reemplazando la MAC física exterior por la suya propia. Si el servidor DHCP mirase solo la MAC exterior, parecería que todos los ordenadores de la otra planta son la misma máquina. Por eso, el servidor lee el campo `chaddr` escrito en el mensaje.

#### La vulnerabilidad (La trampa):
Como el servidor se fía de lo que pone en la casilla interna `chaddr`, un atacante puede mantener su MAC de red real y enviar miles de mensajes cambiando solo la casilla `chaddr`. El servidor pensará que son miles de ordenadores distintos pidiendo IP.

---

### El problema: DHCP no pide contraseña (Sin Autenticación)
El estándar original de DHCP (RFC 2131) **no tiene ninguna contraseña ni forma de comprobar quién es quién**. Se diseñó pensando en que todas las personas conectadas a una red local eran de confianza.

* **¿Por qué no se usa el intento de mejora (RFC 3118)?**  
  Se propuso un sistema con contraseñas o certificados (RFC 3118), pero **casi nadie lo usa** porque rompería la ventaja principal de DHCP: enchufar un cable y tener internet sin configurar nada (*Plug and Play*). Gestionar claves o certificados en cada móvil, impresora o portátil antes de conectarlos a la red sería un caos imposible de mantener.

---

## 2. Ataques: Agotamiento de IPs (DHCP Starvation)

### ¿Cómo funciona el ataque?
El ataque de **Starvation** (Agotamiento) consiste en dejar al servidor DHCP sin direcciones IP libres para que los usuarios reales no puedan conectarse a internet.

```text
[Atacante]                                  [Servidor DHCP]
   |                                               |
   |-- "Soy MAC 1, dame IP" (DHCPDISCOVER) ------->| -- Asigna IP 1
   |-- "Soy MAC 2, dame IP" (DHCPDISCOVER) ------->| -- Asigna IP 2
   |-- "Soy MAC 3, dame IP" (DHCPDISCOVER) ------->| -- Asigna IP 3
   |                   ...                         |
   |-- "Soy MAC 250, dame IP" (DHCPDISCOVER) ----->| -- ¡Pool agotado! (Sin IPs)
```

#### Tipos de ataque:
1. **Ataque Simple:** El atacante usa su MAC real de red en el sobre (Capa 2) y solo inventa la MAC del texto interno `chaddr` (Capa 7).
2. **Ataque Complejo:** El atacante falsifica tanto la MAC del sobre como la del texto interno. Es como si enviara cada carta con un remitente y un DNI inventados distintos.

**¿Por qué el ataque complejo es peor?**  
Porque engaña por completo a los aparatos de la red (switches). El switch cree que se están conectando miles de cables u ordenadores reales al mismo tiempo, lo que puede colapsar el equipo de red por completo.

---

### Herramientas y Tiempo de Alquiler (*Lease Time*)
* **Herramientas (Yersinia y Scapy):** Son programas que permiten crear scripts en Python para enviar miles de peticiones DHCP falsas en cuestión de un segundo. Hacerlo manualmente llevaría días; con un programa automático lleva un par de segundos.
* **El tiempo de concesión (*Lease Time*):** Es el tiempo que el servidor te "presta" la IP.
  * **Si el tiempo es de 7 días:** Aunque el atacante apague su ordenador, las IPs falsas quedarán bloqueadas durante toda una semana. La red estará rota durante 7 días.
  * **Si el tiempo es de 5 minutos:** A los 5 minutos de parar el ataque, las IPs falsas caducan y el servidor recupera la normalidad automáticamente.

---

### ¿Qué le pasa a la víctima? (La dirección APIPA)
Cuando el servidor se queda sin IPs, si un usuario intenta conectarse a la red, su ordenador no recibe respuesta.

* **¿Qué hace el sistema operativo?**  
  Tanto Windows como Linux se asignan a sí mismos una IP especial que empieza por **`169.254.X.X`**. Esto se llama dirección **APIPA**.
* **¿Qué puedo hacer con una IP APIPA?**  
  **No tendrás acceso a Internet** ni a la puerta de enlace de la empresa. Solo podrás hablar con otros ordenadores de la misma sala que se hayan quedado en tu misma situación.

---

## 3. Defensas y Protección (Hardening)

### Solución en la infraestructura (En los Switches)

#### 1. DHCP Snooping (Inspección DHCP)
Es una función de los switches para controlar qué entra y sale por cada puerto:
* **Puerto de Confianza (Trusted):** Es el puerto donde está enchufado el servidor DHCP de verdad. Se le permite enviar ofertas de IP.
* **Puerto No Confiable (Untrusted):** Es el puerto donde se conectan los usuarios. **Si alguien intenta responder como servidor DHCP desde aquí, el switch bloquea el mensaje al instante.**

Además, el switch guarda una lista (*Binding Database*) anotando qué IP le corresponde a qué puerto y a qué MAC.

#### 2. Port Security (Seguridad de Puerto)
Permite decirle al switch: *"En este enchufe de pared solo se permite un ordenador a la vez"*. Si el atacante intenta enviar peticiones simulando miles de MACs distintas desde el mismo puerto, el switch se da cuenta, frena el ataque y apaga el puerto automáticamente.

---

### Solución dentro de un Servidor Linux

Como en nuestro laboratorio de VirtualBox no tenemos un switch físico, protegetemos directamente el propio servidor Linux (`isc-dhcp-server`):

#### 1. Módulo `hashlimit` de `iptables`
Sirve para limitar la velocidad a la que aceptamos peticiones. Le dice al cortafuegos de Linux: *"Si recibes más de 3 peticiones DHCP por segundo desde un mismo sitio, tira las demás a la basura"*. Esto detiene los scripts automáticos de ataque.

Ejemplo de cómo se configura:
```bash
# Permitir un máximo de 3 peticiones DHCP por segundo
iptables -A INPUT -p udp --dport 67 -m hashlimit \
  --hashlimit-name limite_dhcp \
  --hashlimit-mode srcip \
  --hashlimit-upto 3/sec --hashlimit-burst 5 \
  -j ACCEPT

# Bloquear todo lo que supere ese ritmo
iptables -A INPUT -p udp --dport 67 -j DROP
```

#### 2. `ebtables` (Comprobador de MACs)
Mientras que `iptables` mira direcciones IP, `ebtables` trabaja revisando las tramas físicas. Permite comprobar que la MAC de la carta de la red sea **idéntica** a la MAC que va escrita en el texto de la petición. Si no coinciden, la destruye antes de que afecte al servidor.

---

### Buenas prácticas en la configuración

* **Ajustar el tiempo de alquiler (*Lease Time*):**
  * En una oficina con los mismos ordenadores siempre: se pueden poner tiempos más largos (ej. 12 horas).
  * En una cafetería o red Wi-Fi pública: conviene poner tiempos cortos (ej. 1 o 2 horas) para recuperar rápido las IPs que dejan libres las personas que se van.
* **Detección de conflictos (`ping check`):**
  Antes de dar una IP a un usuario nuevo, el servidor lanza un "ping" rápido a esa IP. Si alguien responde, el servidor comprende que la IP está ocupada y entrega otra, evitando que dos ordenadores tengan la misma IP por error.

---

## 4. Entorno Virtual y Consideraciones Legales

### El Modo Promiscuo en VirtualBox
Por defecto, una tarjeta de red solo escucha las cartas que van dirigidas exactamente a su dirección. El **modo promiscuo** es una opción que obliga a la tarjeta de red a escuchar **todo el tráfico que circula por el cable**, aunque no sea para ella.

#### ¿Por qué es necesario activarlo en nuestras pruebas?
Si el atacante usa una herramienta como Scapy para pedir IPs con MACs falsas, el servidor responderá a esas MACs falsas. **Si Kali Linux no tiene activo el "Modo Promiscuo: Permitir todo" en VirtualBox, la tarjeta de red tirará las respuestas a la basura porque no van dirigidas a su MAC real**, y la prueba de ataque no funcionará.

---

### Aspectos Éticos y Legales
Realizar ataques de denegación de servicio (DoS) es una acción peligrosa que puede dejar a una empresa sin servicio.

* **¿Cuándo es legal hacer estas pruebas?**  
  Únicamente en dos casos:
  1. En un **entorno de laboratorio propio o aislado** (como máquinas virtuales locales que no afectan a nadie).
  2. En una auditoría real si se cuenta con un **permiso escrito y firmado** por los responsables de la red.

* **Consecuencias en el Código Penal Español:**  
  Realizar un ataque de denegación de servicio sin autorización en una red real es un delito informático (Artículos 264 y 264 bis del Código Penal). Puede acarrear:
  * **Penas de prisión de 6 meses a 3 años** (pudiendo subir hasta los 5 años si se ataca a un hospital, organismo público o infraestructura crítica).
  * **Multas económicas muy elevadas** para pagar todos los daños causados a la empresa.

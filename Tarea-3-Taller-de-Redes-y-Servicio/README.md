# Intercepción, Fuzzing y modificacion de Tráfico HTTP con Scapy

Este proyecto permite capturar, modificar e inyectar tráfico HTTP entre contenedores Docker usando Scapy. Se simula un entorno cliente-servidor, y se analiza cómo reacciona un servidor ante tráfico malicioso o inesperado mediante técnicas de fuzzing.

---

## 1. Estructura del Repositorio
Tarea 3 /  
│  
├── README   
├── scapy/
│ └── Dockerfile
│
├── scripts/  
│ └── Interceptar Trafico  
│ └── Fuzzing
│ └── Modificaciones del Protocolo
│  
├── imágenes/  
│ ├── interceptar.png 
│ ├── fuzzing1.png 
│ └── fuzzing2.png
│  
├── Capturas Wireshark/  
│ ├── modificacion1.png  
│ └── modificacion2.png
│ └── modificacion3.png

---

##  2. Entorno de Trabajo

Se utilizaron 3 contenedores:

-   **cliente**: envía peticiones HTTP usando `wget`
    
-   **servidor**: ejecuta `nginx`
    
-   **sniffer**: contenedor que corre Scapy y ejecuta scripts
    
---
## 3. Instalar Scapy:
Creamos un contenedor con Python y Scapy.

#### 3.1 Crear un Dockerfile para Scapy

Crea una carpeta llamada `scapy/` (en la misma ubicacion que cliente y servidor)  y dentro de un Dockerfile coloca el siguiente archivo:

```Dockerfile
FROM python:3.10-slim

RUN pip install --no-cache-dir scapy

RUN apt update && apt install -y iputils-ping tcpdump net-tools

CMD ["python3"]
```
#### 3.2 Construye la imagen
```bash
sudo docker build -t scapy ..
```
#### 3.3 Corre el contenedor en la misma red
```bash
sudo docker run -it --rm --name sniffer --network red --cap-add=NET_ADMIN scapy
```
-   --network red : conecta el contenedor a la red de cliente y servidor
    
-   --cap-add=NET_ADMIN : otorga permisos para sniffing

##  4. Interceptar trafico con Scapy
Dentro del contenedor Scapy, ejecuta:
```bash
from scapy.all  import * 
pkts = sniff(filter="tcp port 80", count=10)
pkts.summary()
```
Esto permite observar el intercambio entre cliente y servidor HTTP
Ver captura: ` imágenes/interceptar.png`

## 5. Fuzzing de Tráfico HTTP

### 5.1. Fuzzing 1: Datos aleatorios
Se trata de enviar paquetes con contenido aleatorio o inválido para observar cómo reacciona el servidor.
```bash
from scapy.all import *
pkt1 = IP(dst="172.18.0.2")/TCP(dport=80, sport=RandShort(), flags="PA")/ Raw(load=RandString(size=100))
send(pkt1)

Print(">> Enviando Fuzzing 1")
```
Ver captura: `imágenes/fuzzing1.png`

----------

### 5.2. Fuzzing 2: Banderas TCP inusuales

Se construye un paquete con las banderas `FIN`, `PSH` y `URG`, que no son comunes juntas en HTTP.
```bash
from scapy.all import *
pkt2 = IP(dst="172.18.0.2")/TCP(dport=80, sport=RandShort(), flags="FPU")/ Raw(load="FUZZ!!!")
send(pkt2)

Print(">> Enviando Fuzzing 2")
```
Ver captura: `imágenes/fuzzing2.png`

----------

## 6. Modificaciones especificas del Protocolo

### 6.1 Modificacion 1
Esto crea y envía un paquete TCP con carga HTTP simulando un POST en vez de un GET. 
```bash
from scapy.all import *

ip = IP(dst="172.18.0.1") 
tcp = TCP(sport=RandShort(), dport=80, flags="PA", seq=1000, ack=100)
http = b"POST / HTTP/1.1\r\nHost: nginx\r\n\r\n"

pkt = ip / tcp / Raw(load=http)
send(pkt)
```
Para ver las modificaciones: `Capturas Wireshark/modificacion1.png`

### 6.1 Modificacion 2
Se altera el valor del campo Host en la cabecera HTTP, cambiándolo de el valor nginx a uno inexistente 
```bash
from scapy.all import *

ip = IP(dst="172.18.0.1")
tcp = TCP(sport=RandShort(), dport=80, flags="PA", seq=1001, ack=100)
http = b"GET / HTTP/1.1\r\nHost: hacker.local\r\n\r\n"

pkt = ip / tcp / Raw(load=http)
send(pkt)
```
Para ver las modificaciones: `Capturas Wireshark/modificacion2.png`

### 6.3 Modificacion 3
Se altera el contenido del encabezado User-Agenr de la solicitud HTTP, con un fragmento de código JavaScript ().
```bash
from scapy.all import *

ip = IP(dst="172.18.0.1")
tcp = TCP(sport=RandShort(), dport=80, flags="PA", seq=1002, ack=100)
http = b"GET / HTTP/1.1\r\nHost: nginx\r\nUser-Agent: <script>alert(1)</script>\r\n\r\n"

pkt = ip / tcp / Raw(load=http)
send(pkt)
```
Para ver las modificaciones: `Capturas Wireshark/modificacion1.png`


# 🛡️ HAProxy + Coraza SPOA WAF (OWASP CRS)

Plantilla de arquitectura de seguridad perimetral que implementa un Web Application Firewall (WAF) Out-of-Band utilizando **HAProxy** (Proxy Inverso, SSL/TLS, Rate Limiting) y **Coraza SPOA** (Agente WAF con OWASP CRS) que aplica las reglas del **OWASP Core Rule Set (CRS)**.


## 🎯 Arquitectura y Flujo de Tráfico


Cliente (HTTPS) -> **HAProxy** -> **Coraza SPOA (WAF)** -> **HAProxy** -> Backend 

---

## ⚙️ Configuración y Estructura
├── docker-compose.yml           # Orquestación de los 3 servicios
├── haproxy/
│   ├── haproxy.cfg              # Configuración de HAProxy
│   └── coraza.cfg               # Configuración del SPOE de Coraza
├── coraza-spoa/
│   ├── config.yml               # Configuración del agente Coraza
│   └── coraza-rules/            # Reglas del WAF (ej. OWASP CRS)
├── coraza-spoa-src/
    └── GIT REPO
    
> **Nota:** Se utiliza `network_mode: host` ->

### 1. Requisitos Previos

1.  **Clonar el Repositorio de Coraza SPOA:**
    ```bash
    git clone [https://github.com/corazawaf/coraza-spoa.git](https://github.com/corazawaf/coraza-spoa.git) coraza-spoa-src
    ```
2.  **Generación de SSL:** Colocar `certificado.pem` (CRT + KEY) en `./haproxy/ssl/`.

    # Generar clave privada y certificado (Autofirmado por 365 días)
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout ./haproxy/ssl/server.key \
        -out ./haproxy/ssl/server.crt \
        -subj "/C=AR/ST=BA/L=BA/O=MiEmpresa/OU=IT/CN=APPexample"

    # Concatenar a formato PEM
    cat ./haproxy/ssl/server.crt ./haproxy/ssl/server.key > ./haproxy/ssl/certificado.pem


3.  **OWASP CRS:** Las reglas deben estar en `./coraza-spoa/coraza-rules/`.


### 2. Modificación Crítica del Dockerfile

Para que el agente Coraza use su configuración de producción montada por volumen y no el archivo de ejemplo, debe modificar el `Dockerfile` oficial.

**Abrir:** `./coraza-spoa-src/example/Dockerfile`

**Modificar la línea `CMD` final:**

```dockerfile
# Antes: CMD ["/coraza-spoa", "--config", "/config.yaml"]

# Después (Usando el volumen montado):
CMD ["/coraza-spoa", "--config", "/etc/coraza/config.yml"]



Configurar El Backend (Spring Security)
Para que tu API reconozca la IP real del cliente (enviada por HAProxy en el encabezado X-Forwarded-For), añade esto a tu archivo de configuración (application.properties):
└──server.forward-headers-strategy=native
    └──BEAN req.getRemoteAddr();
    
    
Modo A: Red Aislada (Estándar y Recomendado) no HOST

docker-compose.yml      	Eliminar network_mode: host	Permite el aislamiento de la red. Mapear puertos  
HAProxy (haproxy.cfg)	    Comunicación por Nombre de Servicio	✅ Se llama a Coraza por coraza:9000 y al backend por backend:8080. (network_mode HOST -> ip host)


Modo B: network_mode: host (Desarrollo y Depuración)

HAProxy (haproxy.cfg)	    Comunicación por IP Local	✅ Se llama a Coraza por 127.0.0.1:9000 (o localhost:9000), ya que Coraza escucha en la interfaz de loopback del host.

🧭 Flujo Lógico en HAProxy


global      Configuración de la máquina y el proceso HAProxy.
        "Definición de logs, límites de conexión (maxconn)."
        
defaults    Configuración base que se aplica a todas las secciones.
        "Tiempos de espera (timeout connect, client, server) y modo de operación (mode http)."
        
frontend    "PUNTO DE ENTRADA (El ""Escuchador"")."
        "Define en qué puerto escucha HAProxy, maneja el SSL Offloading, y evalúa las reglas."
        
backend     "PUNTO DE SALIDA (El ""Destino"")."
        "Define la IP/puerto del servidor de destino (API, Coraza, Frontend), el método de balanceo y aplica headers de respuesta."

1. 👂 Sección frontend entrada_publica_httpsEsta es la sección más importante, ya que contiene toda la lógica de decisión y seguridad.
bind *:443 ssl crt...         Puerto de escucha y configuración SSL Offloading.
        Recibe tráfico en el puerto 443 y maneja el certificado.
        
http-request set-var...         CAPTURA DE DATOS (CORS).
        http-request set-var(txn.origin) hdr(Origin): Guarda el Origin para devolverlo dinámicamente.
        
acl [nombre] [condición]        ACL (Access Control List): Define una condición lógica.
        acl is_options method OPTIONS: Condición: ¿Es una petición OPTIONS?
        
use_backend [nombre] if [acl]   RUTEO CONDICIONAL: Envía el tráfico a un backend.
        use_backend cors_options if is_options: Si es OPTIONS, ve al backend de preflight.
        
filter spoe...                  WAF: Envía metadatos de la petición a Coraza.Intercepta la petición para que Coraza la evalúe.

http-request deny...            BLOQUEO WAF: Detiene la petición si Coraza lo ordena.
        http-request deny... if { var(txn.coraza.action) -m str deny }.
        

2. 🎯 Secciones backend [nombre]
server [nombre] [ip:puerto] 	Define el servidor de destino real.	
        server web_backend backend:8080 check.

# Informe Técnico: Despliegue de Central Telefónica IP (PBX) con Asterisk en Docker Swarm

**Fecha:** 22 de Noviembre de 2025
**Autor:** Tadeo / Asistente Técnico (Copilot)
**Proyecto:** Virtualización de PBX Corporativa
**Versión del Documento:** 1.0

---

## 1. Resumen Ejecutivo

El presente documento detalla el procedimiento técnico llevado a cabo para el despliegue, configuración y puesta en marcha de una central telefónica IP basada en **Asterisk**, utilizando tecnologías de orquestación de contenedores **Docker Swarm**.

El objetivo principal fue establecer una infraestructura de telecomunicaciones ligera, portátil y escalable, capaz de gestionar extensiones SIP internas, asegurando la persistencia de datos y la correcta transmisión de voz sobre IP (VoIP) en un entorno de red local (LAN).

La implementación final ha resultado en un servicio estable (`pbx_asterisk`), con extensiones funcionales (`100`, `101`), plan de marcado interno operativo y corrección de problemas de NAT y señalización SDP.

---

## 2. Arquitectura de la Solución

### 2.1. Tecnologías Utilizadas
*   **Sistema Operativo Host:** Linux (Ubuntu/Debian based).
*   **Motor de Contenedores:** Docker Engine (Community Edition).
*   **Orquestador:** Docker Swarm (Modo Single-Node para este despliegue).
*   **Imagen Base:** `mlan/asterisk:mini` (Alpine Linux optimizado para Asterisk).
*   **Protocolo de Señalización:** SIP (vía canal PJSIP).
*   **Protocolo de Transporte de Medios:** RTP (Real-time Transport Protocol).

### 2.2. Diseño de Red y Puertos
Uno de los desafíos críticos en la virtualización de VoIP es la gestión de puertos RTP y la traducción de direcciones de red (NAT). Para mitigar la complejidad de usar redes `overlay` de Docker con tráfico de voz UDP, se optó por una estrategia híbrida:

1.  **Señalización SIP (Puerto 5060 UDP):** Expuesto en modo `host` para evitar capas de enrutamiento innecesarias que suelen romper la cabecera `Via` del protocolo SIP.
2.  **Audio RTP (Puertos 10000-10999 UDP):** Se definió un rango específico de 1000 puertos, mapeados directamente al host para garantizar que el audio fluya sin interrupciones (One-way audio issues).

### 2.3. Persistencia de Datos
Se implementaron dos estrategias de almacenamiento para garantizar que la PBX no pierda su estado ante reinicios:
*   **Bind Mount (`./config` -> `/etc/asterisk`):** Permite la edición de archivos de configuración (`pjsip.conf`, `extensions.conf`) directamente desde el sistema de archivos del host, facilitando la administración sin entrar al contenedor.
*   **Volumen Nombrado (`asterisk_data` -> `/var/lib/asterisk`):** Almacena bases de datos internas, correos de voz y registros de llamadas.

---

## 3. Implementación Paso a Paso

### 3.1. Preparación del Entorno
Se inicializó el clúster de Docker Swarm para habilitar las capacidades de despliegue de servicios y manejo de secretos/configs (aunque en esta fase se usaron archivos locales).

```bash
docker swarm init
```

Se creó la estructura de directorios de trabajo en `/home/tadeo/Documents/virtual`:
*   `/config`: Directorio para archivos `.conf`.
*   `docker-compose.yml`: Manifiesto de despliegue.

### 3.2. Definición del Stack (docker-compose.yml)
El archivo de orquestación fue diseñado meticulosamente para corregir errores de sintaxis comunes en Swarm respecto a rangos de puertos.

**Archivo:** `docker-compose.yml`
```yaml
version: '3.7'
services:
  asterisk:
    image: mlan/asterisk:mini
    container_name: pbx_asterisk_dev
    ports:
      # Puerto SIP crítico expuesto en modo host para visibilidad directa
      - target: 5060
        published: 5060
        protocol: udp
        mode: host
      # Rango RTP definido explícitamente con sintaxis compatible con Swarm
      - "10000-10999:10000-10999/udp"
    volumes:
      # Persistencia de la Configuración (Para Respaldo y Modificación Local)
      - type: bind
        source: /home/tadeo/Documents/virtual/config
        target: /etc/asterisk
      # Persistencia de Datos, Voicemail y Grabaciones
      - asterisk_data:/var/lib/asterisk
    restart: unless-stopped

    # === CONFIGURACIÓN PARA DOCKER SWARM ===
    deploy:
      replicas: 1
      placement:
        # Ejecutar solo en el nodo Manager, ya que la PBX no se escala fácilmente
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure

volumes:
  asterisk_data:

```

### 3.3. Configuración del Canal PJSIP
Se migró del antiguo canal `chan_sip` al moderno `chan_pjsip`. La configuración se divide en secciones: Transporte, Auth, AOR (Address of Record) y Endpoint.

**Problema Detectado:** Inicialmente, los clientes recibían un error `Unparsable SDP (Code 28)` o falta de audio.
**Causa:** Asterisk enviaba su IP interna de Docker (`172.18.x.x`) en los paquetes SDP, la cual es inalcanzable para los teléfonos físicos/softphones.
**Solución:** Se forzó la declaración de la IP externa en la sección `[transport-udp]`.

**Archivo Final:** `config/pjsip.conf`
```ini
[global]
type=global
user_agent=PBX Interno

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
allow_reload=yes
; Configuración CRÍTICA para NAT y Audio
external_media_address=192.168.1.9
external_signaling_address=192.168.1.9
local_net=172.16.0.0/12
local_net=192.168.0.0/16

; === EXTENSIÓN 100 ===
[100]
type=endpoint
context=internas
disallow=all
allow=ulaw
auth=auth100
aors=100
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes
direct_media=no

[auth100]
type=auth
auth_type=userpass
password=MiClaveSegura123
username=100

[100]
type=aor
max_contacts=1

; === EXTENSIÓN 101 ===
[101]
type=endpoint
context=internas
disallow=all
allow=ulaw
auth=auth101
aors=101
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes
direct_media=no

[auth101]
type=auth
auth_type=userpass
password=MiClaveSegura123
username=101

[101]
type=aor
max_contacts=1
```

### 3.4. Configuración del Plan de Marcado (Dialplan)
Se estableció un contexto básico `[internas]` que permite la comunicación entre extensiones de tres dígitos que comiencen con 1.

**Archivo:** `config/extensions.conf`
```ini
[internas]
exten => _1XX,1,NoOp(Llamada interna a la extension ${EXTEN})
same => n,Dial(PJSIP/${EXTEN},20)
same => n,Hangup()
```

---

## 4. Fase de Depuración y Limpieza (Troubleshooting)

Durante el despliegue, se encontraron anomalías que requirieron intervención técnica avanzada.

### 4.1. El Problema de los "Endpoints Fantasma"
**Síntoma:** Al ejecutar `pjsip show endpoints`, aparecían extensiones no deseadas como `john.doe`, `jane.doe` y `itsp:mydoe`, mezcladas con las extensiones legítimas `100` y `101`.
**Diagnóstico:** La imagen base `mlan/asterisk:mini` incluye archivos de configuración de demostración por defecto que no fueron sobrescritos completamente por el montaje del volumen. Específicamente, el archivo `pjsip_wizard.conf` incluía referencias a `pjsip_endpoint.conf`.
**Resolución:** Se procedió a la eliminación forzosa de los archivos conflictivos dentro del volumen montado:
```bash
rm config/pjsip_wizard.conf config/pjsip_endpoint.conf config/pjsip_transport.conf
```
Tras reiniciar el servicio, la lista de endpoints quedó limpia, mostrando únicamente la configuración personalizada.

### 4.2. Error de Autenticación 401
**Síntoma:** Los softphones Zoiper reportaban "Registration Failed (401)".
**Diagnóstico:** Discrepancia entre la contraseña configurada en el cliente (`1234`) y la definida en `pjsip.conf` (`MiClaveSegura123`).
**Resolución:** Se estandarizaron las credenciales en los clientes para coincidir con el servidor.

---

## 5. Procedimientos Operativos Estándar (SOP)

Para el mantenimiento continuo de la PBX, se definen los siguientes comandos de operación.

### 5.1. Despliegue y Actualización
Para aplicar cambios realizados en los archivos de configuración:
```bash
# Opción A: Recarga suave (sin cortar llamadas)
docker exec $(docker ps -q -f name=pbx_asterisk) asterisk -rx "core reload"

# Opción B: Reinicio forzoso del servicio (aplica cambios de red/puertos)
docker service update --force pbx_asterisk
```

### 5.2. Monitoreo en Tiempo Real
Para ver el registro de eventos, errores SIP y depuración:
```bash
docker service logs -f pbx_asterisk
```

### 5.3. Gestión de Llamadas vía CLI
Para verificar extensiones conectadas:
```bash
docker exec $(docker ps -q -f name=pbx_asterisk) asterisk -rx "pjsip show endpoints"
```

Para ver llamadas activas:
```bash
docker exec $(docker ps -q -f name=pbx_asterisk) asterisk -rx "core show channels"
```

Para forzar el corte de una llamada específica:
```bash
docker exec $(docker ps -q -f name=pbx_asterisk) asterisk -rx "channel request hangup <Nombre_Canal>"
```

---

## 6. Guía de Configuración de Cliente (Usuario Final)

Para conectar un nuevo dispositivo a la central, utilice los siguientes parámetros. Se recomienda el uso de **Zoiper** o **MicroSIP**.

| Parámetro | Valor | Notas |
| :--- | :--- | :--- |
| **Dominio / Host** | `192.168.1.9` | IP estática del servidor Linux |
| **Puerto** | `5060` | Puerto UDP estándar |
| **Transporte** | `UDP` | **Crucial:** No usar TCP ni TLS por ahora |
| **Usuario** | `100` o `101` | Según asignación |
| **Contraseña** | `MiClaveSegura123` | Case-sensitive (distingue mayúsculas) |
| **Auth ID** | (Igual al usuario) | Dejar en blanco si es opcional |

---

## 7. Conclusión y Próximos Pasos

La infraestructura actual es **funcional y estable**. Se ha logrado establecer una comunicación bidireccional con audio claro dentro de la red local. La limpieza de archivos de configuración predeterminados asegura que no existan brechas de seguridad ni configuraciones "basura".

**Recomendaciones futuras:**
1.  **Seguridad:** Implementar `fail2ban` para proteger el puerto 5060 de ataques de fuerza bruta si se planea exponer a Internet.
2.  **Buzón de Voz:** Configurar `voicemail.conf` para habilitar buzones para las extensiones 100 y 101.
3.  **TLS/SRTP:** Para entornos de producción real, se recomienda cifrar la señalización (TLS) y el audio (SRTP) modificando el transporte en `pjsip.conf`.

***
*Fin del Informe Técnico.*

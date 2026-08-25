# Canal SAP — guía completa

El Ekumetrics Agent, en modo **Sensor**, observa el tráfico hacia componentes SAP **sin acceso administrativo** al sistema. No entra al SID, no usa usuario RFC, no consulta Solution Manager / Cloud ALM y **no descifra** SNC, TLS ni HTTPS.

Esta guía cubre arquitectura, colocación, qué se ve, catálogo de métricas, YAML, POC solo-agente y límites. El resumen operativo de series y campos también está en [USO.md](USO.md). El sensor está sujeto a la prueba de 15 días o a la licencia Gradotech (sección Licencia de [USO.md](USO.md)).

Nombre comercial honesto: **observabilidad del canal SAP** / *SAP Network Experience*. No es “monitoreo S/4” ni APM ABAP.

---

## 1. Qué es y qué no es

SAP S/4HANA se describe en tres capas: presentación, aplicación ABAP y base de datos HANA. Los usuarios llegan a la capa de aplicación (Fiori, SAP GUI, integraciones). El tráfico ABAP–HANA en RISE / PCE queda **dentro del landscape administrado** y no atraviesa la red del cliente.

Por eso una sonda en la red del cliente ve el **borde remoto** (experiencia de red hacia SAP), no el interior del SID.

| Objetivo | ¿Es posible con este sensor? |
|----------|------------------------------|
| Monitorear conectividad de usuarios hacia SAP | Sí, si el SPAN ve ese camino |
| Medir latencia TCP, retransmisiones, RST, sesiones | Sí |
| Comparar calidad entre sedes (un sensor por sede) | Sí |
| Detectar caída o degradación del endpoint SAP | Sí (TCP/TLS del borde) |
| Identificar equipos/IP que usan SAP | Sí (journal; no como label Prometheus) |
| Identificar usuario corporativo | Solo correlacionando DHCP, AD, VPN o CMDB fuera del agente |
| Identificar usuario SAP | No de forma fiable |
| Identificar transacción (`VA01`, `ME21N`, …) | No de forma fiable |
| Medir CPU, memoria, locks o SQL de HANA | No |
| Ver tráfico ABAP–HANA interno en RISE | No |
| Determinar que “SAP es la causa” | No; solo localizar el síntoma hasta el borde remoto |

**Public Edition:** el acceso estándar es HTTPS al SAP Web Dispatcher. El sensor ve el borde cifrado (TCP/TLS, SNI si el ClientHello es visible). No ve la topología interna.

**Private Edition / RISE:** puede haber SAP GUI/DIAG, RFC y conectividad privada. El tráfico interno HANA sigue dentro del landscape administrado.

El código **no distingue edición**. La edición se decide en el descubrimiento (días 1–2 de la POC).

---

## 2. Arquitectura del agente

Un **solo binario** (`ekumetrics-agent`), un YAML (`/etc/ekumetrics-agent/agent.yaml`). Cada señal tiene interruptor. Lo que no se activa no arranca.

| Tipo visible | YAML `agent.mode` | Uso |
|--------------|-------------------|-----|
| Servidor | `site` | CPU/disco/red de esa máquina; opcional SNMP, syslog, NetFlow |
| Sensor | `sensor` | Tráfico de un puerto espejo. **Rol de este canal** |
| NOC | `central` | APIs de gestores (reservado) |
| Endpoint | `endpoint` | PC o portátil |

El canal SAP **no** va en el YAML de un agente de sede. Se instala un sensor aparte, junto al SPAN/TAP.

### POC solo-agente (sin plataforma)

```
Usuarios / Fiori / GUI / RFC              Landscape SAP
        │  32NN DIAG  33NN RFC  36NN MS           │
        │  39NN ENQ   3299 Router  ICM / HANA     │
        └──────────────── switch / VLAN ──────────┘
                              │
                         SPAN / TAP
                        (ida y vuelta)
                              │
                 ┌────────────┴────────────┐
                 │  VM Linux (sensor)      │
                 │  NIC1  gestión          │
                 │  NIC2  captura (eth1)   │
                 │  ekumetrics-agent       │
                 │  mode: sensor           │
                 │                         │
                 │  :9090/healthz          │
                 │  :9090/metrics          │
                 │  journal (JSON/sesión)  │
                 └─────────────────────────┘
                              │
                    Grafana del cliente
                    (scrape a :9090)
                    o curl / journalctl
```

`export.otlp` **vacío**: recolecta y no reenvía. No hace falta tenant, portal ni salida a Gradotech.

Si más adelante hay plataforma, el mismo proceso hace **push OTLP** al collector. En la POC descrita aquí no aplica.

El agente **no pinta paneles**. Expone Prometheus en `127.0.0.1:9090`. Sin alguien que scrapee, Grafana queda vacío.

### Qué hace el proceso

1. Abre la NIC de captura (`AF_PACKET`, `cap_net_raw` + `cap_net_admin`).
2. Conserva TCP hacia **puertos SAP conocidos**, o hacia IPs ya vistas como landscape por **otro puerto**.
3. Arma **sesiones** (intentos, establecimiento, bytes, paquetes, RTT, retransmisiones, RST, dup ACK, desorden, zero-window).
4. Etiqueta el canal **por puerto** (`sap-diag`, `sap-rfc`, …).
5. Si ve ClientHello/ServerHello, anota versión, SNI y cipher **sin descifrar** el resto.
6. Publica series en `:9090` y, si `logEvents: true`, una línea JSON por sesión.

Por defecto **no abre payload** DIAG/RFC. Eso exige `decode.authorized`.

---

## 3. Modalidad: no es sondeo

| Pregunta | Respuesta |
|----------|-----------|
| ¿Polling a SAP? | No |
| ¿Usuario RFC / CCMS / SolMan / HANA SQL? | No |
| ¿Agente dentro del SID? | No |
| ¿Cómo entra el dato? | El switch **copia** el tráfico al sensor |
| ¿Cada cuánto “pregunta”? | No pregunta. Las sesiones se cierran al FIN/RST, a 30 s si el SYN no responde, o a 5 min de idle |
| ¿Salida en la POC? | Lectura local de `:9090` (push OTLP solo si hay destino) |

Sin mirror **bidireccional** no hay datos útiles. El fallo típico es un SPAN de un solo sentido (bytes sent o received en cero, RTT ausente).

Si el espejo está en el lado WAN de una VPN, se verá ESP o UDP 4500 y las IP de los gateways. Si SAP GUI corre dentro de Citrix/VDI/RDS, la sonda debe ir **entre** los servidores VDI y SAP; desde la red del usuario solo se ve ICA/RDP.

---

## 4. Dónde instalar la sonda

- En el lado **LAN**, **antes del NAT**, para conservar la IP de origen.
- Antes de que el gateway encapsule en **IPsec**.
- En un punto con visibilidad de **ambos sentidos**.
- **Fuera** del camino de datos: no es proxy ni bridge.
- **Dos NIC**: captura (sin dirección IP, recomendado) y gestión.

Un TAP es preferible en producción. SPAN sirve para la POC; puede descartar si el puerto de destino se satura. El agente mide `ekms_sap_capture_drops_total` (descartes del socket, **toda** la NIC, no solo SAP).

La VM es Linux (Debian/Ubuntu/RHEL). No se instala software en los application servers.

---

## 5. Qué tráfico se reconoce

`sap_system` y `sap_instance` **salen del YAML**, no del paquete. El puerto identifica el componente (`NN` = instancia: `3200` = dispatcher 00).

| Puerto | Componente | Etiqueta | Significado |
|--------|------------|----------|-------------|
| 3200–3298 | Dispatcher / SAP GUI | `sap-diag` | Diálogo, pantallas, logon GUI |
| 3300–3399 | Gateway | `sap-rfc` | RFC, IDocs, PI/PO, jobs remotos |
| 3600–3699 | Message Server | `sap-ms` | Logon group, balanceo |
| 3900–3999 | Enqueue | `sap-enqueue` | Bloqueos (casi solo entre AS) |
| 3299 | SAProuter | `sap-router` | Entrada remota; puede ocultar el destino final |
| 8000–8099, 80 | ICM HTTP | `sap-http` | WebGUI, Fiori, SICF, OData en claro |
| 44300–44399, 443, 8443 | ICM HTTPS | `sap-https` | Lo mismo, cifrado |
| 3NN15 / 3NN13 / 3NN17 | HANA | `sap-hana` | SQL/MDX que **cruza** la red observada (no el texto SQL) |
| Otro puerto hacia un host SAP ya visto | — | `sap-unexpected` | Fuera de catálogo |

Puertos Java `50000`/`50001` **no** están en el filtro. HANA interno RISE **no** se ve.

`encrypted: true` en la sesión significa puerto HTTPS o handshake TLS visto. **No** significa “este GUI usa SNC” salvo que se active decode y se vea SNC.

---

## 6. Qué se observa (tres capas)

### Capa 1 — Canal (POC, por defecto)

Metadatos de sesión TCP. Sin payload. Suficiente para demostrar valor.

### Capa 2 — TLS visible (automática, sin decode)

Si el ClientHello o ServerHello van en claro (el caso típico de Fiori/HTTPS), se anotan versión, SNI y cipher. No hay MITM. El resto del stream sigue opaco.

### Capa 3 — Decode de protocolo (opcional, autorizado)

`decode.authorized: true`. No descifra. En claro puede clasificar NI, SNC, Router, HTTP. Los parsers DIAG, RFC, HANA, Message Server y Enqueue **nacen apagados**. Sigue sin haber usuario, tcode ni SQL.

### Capa 4 — Trabajo (fuera de la POC)

`sap.work` + JSONL (STAD/audit). Usuario **hasheado**, tcode. No sale del cable.

---

## 7. Catálogo de métricas (`:9090`)

Las series SAP viven en el registry del módulo (el handler de `:9090` las expone). Las IP de cliente **no** son labels (cardinalidad). El detalle por IP va al journal.

### Sesión y TCP

| Serie | Tipo | Significado | Nombre honesto |
|-------|------|-------------|----------------|
| `ekms_sap_sessions_total` | counter | Sesiones cerradas o expiradas (`protocol`, `encrypted`, `sap_system`) | Sesiones TCP, no logons |
| `ekms_sap_tcp_attempts_total` | counter | SYN cliente | Intentos TCP |
| `ekms_sap_tcp_established_total` | counter | SYN-ACK visto | Establecimientos |
| `ekms_sap_tcp_syn_timeout_total` | counter | SYN sin SYN-ACK (30 s) | SYN sin respuesta |
| `ekms_sap_tcp_rejected_total` | counter | RST remoto sin handshake | Rechazadas |
| `ekms_sap_tcp_setup_seconds` | histogram | SYN → SYN-ACK | **No** es tiempo de logon SAP |
| `ekms_sap_session_duration_seconds` | histogram | Duración TCP | **No** es duración de transacción |
| `ekms_sap_sessions_active` | gauge | Sesiones abiertas | Concurrentes TCP |
| `ekms_sap_session_bytes_total` | counter | Bytes `sent` / `received` | Volumen |
| `ekms_sap_session_packets_total` | counter | Segmentos SYN, datos, FIN, RST | No cuenta ACK vacíos aislados |
| `ekms_sap_session_retransmissions_total` | counter | Seq que no avanza | Retransmisiones aparentes |
| `ekms_sap_session_resets_total` | counter | RST (`origin`: `client` / `remote`) | Cortes |
| `ekms_sap_tcp_dupack_total` | counter | Mismo ACK repetido sin payload | Dup ACK |
| `ekms_sap_tcp_outoforder_total` | counter | Seq por encima de lo esperado | Fuera de orden |
| `ekms_sap_tcp_zerowin_total` | counter | Ventana 0 tras haber visto ventana > 0 | Receptor lleno / backpressure |

p95 de establecimiento:

```promql
histogram_quantile(0.95, sum(rate(ekms_sap_tcp_setup_seconds_bucket[5m])) by (le, protocol, sap_system))
```

Throughput aproximado:

```promql
rate(ekms_sap_session_bytes_total[1m])
```

### Captura (salud de la sonda)

| Serie | Significado |
|-------|-------------|
| `ekms_sap_capture_packets_total` | TCP aceptado (puerto SAP o host SAP por otro puerto) |
| `ekms_sap_capture_drops_total` | Descartes del socket `AF_PACKET` (toda la NIC) |

Si `capture_drops` crece, no confundir con pérdida de la red SAP: el SPAN puede estar saturado.

### TLS (handshake en claro)

| Serie | Significado |
|-------|-------------|
| `ekms_sap_tls_handshakes_total` | Handshake visto (`version`) |
| `ekms_sap_tls_handshake_seconds` | ClientHello → ServerHello |
| `ekms_sap_tls_sni_total` | SNI (se espera poca cardinalidad en SAP) |
| `ekms_sap_tls_cipher_total` | Cipher del ServerHello |

No hay descifrado. SNI/cipher ausentes = handshake no visto o TLS 1.3 con extensiones no parseadas.

### Endpoints

| Serie | Significado |
|-------|-------------|
| `ekms_sap_new_endpoints_total` | Primera vez que aparece un `IP:puerto` SAP (sin label de IP) |
| `ekms_sap_unexpected_port_packets_total` | TCP a un host SAP conocido por un puerto fuera de catálogo (`port`) |

Los hosts se siembran con `inventory` y se **aprenden** al ver un puerto SAP. Sin inventario, primero tiene que pasar tráfico a 32NN/33NN/….

### Identidad y licencia (siempre en `:9090`)

`ekms_agent_info`, `ekms_agent_identity`, `ekms_agent_module_enabled{module="sap"}`, `ekms_agent_license_valid`, `ekms_agent_license_until_timestamp_seconds`.

### Sondas activas (`modules.probes`, no se encienden solas)

| Serie | Tipo de sonda |
|-------|----------------|
| `ekms_probe_up` | icmp, tcp, https, dns |
| `ekms_probe_rtt_seconds` | En `https` es el handshake TLS |
| `ekms_probe_loss_ratio` | 1 si falló |
| `ekms_probe_tls_not_after_timestamp_seconds` | Fin del certificado (`https`) |

Son pruebas **autorizadas** al FQDN/IP que declare el cliente. No descifran el tráfico de usuarios. `https` usa `InsecureSkipVerify` solo para medir disponibilidad, no para validar confianza de negocio.

### Decode (`ekms_sap_protocol_*`)

Solo con `decode.authorized`. Conteos de mensajes, bytes, errores, malformadas, cifradas. No forman parte de la POC mínima.

---

## 8. Evento JSON (`logEvents: true`)

Una línea por sesión (stdout / journal). Campos:

| Campo | Significado |
|-------|-------------|
| `timestamp` | Cierre o expiración |
| `sensor` | Nombre del sensor |
| `protocol` | Canal por puerto |
| `src_ip` / `src_port` / `dst_ip` / `dst_port` | Extremos |
| `duration_ms` | Duración TCP |
| `bytes_sent` / `bytes_received` | Volumen |
| `packets_sent` / `packets_received` | Segmentos de sesión |
| `tcp_rtt_ms` | SYN → SYN-ACK |
| `retransmissions` / `dup_ack` / `out_of_order` / `zero_window` | Calidad |
| `resets` / `reset_origin` | `client` o `remote` |
| `outcome` | `established`, `syn_timeout`, `rejected`, `unknown` |
| `encrypted` | HTTPS o TLS hello |
| `tls_version` / `tls_sni` / `tls_cipher` / `tls_handshake_ms` | Handshake visible |
| `sap_system` / `sap_instance` | Catálogo YAML |

`outcome=unknown` = sesión vista a mitad (sin SYN), típico si el SPAN se activa tarde.

---

## 9. Cómo nombrar los indicadores

| No decir | Decir |
|----------|--------|
| Tiempo de logon SAP | `tcp_setup_seconds` / establecimiento TCP |
| Usuarios SAP | Clientes IP activos (journal) |
| Duración de transacción | Duración de conexión TCP |
| Falla interna de HANA | Indicio de degradación remota / del enlace |
| Usuario SAP | IP de origen (y correlacionar fuera) |

---

## 10. Seguridad y privacidad

- Filtro a puertos SAP y a IPs aprendidas/inventario. No hay BPF en kernel: se captura la NIC y se filtra en user space.
- Sin PCAP persistente por defecto. `pcapDump` en el YAML **aún no escribe** fichero (solo deja constancia en log si se activa).
- Sin `decode.authorized` no se abre payload DIAG/RFC.
- ClientHello se lee en memoria para SNI/versión. No se guarda el payload.
- Nada de MITM ni uso de claves TLS/SNC.
- Pruebas `https`/`dns` solo si el cliente las declara.
- Un SAP GUI **sin SNC** puede llevar credenciales en claro: por eso el decode nace apagado.
- Retención: el journal del sistema; las métricas agregadas viven en quien scrapee `:9090`.

---

## 11. Configuración

```yaml
agent:
  mode: sensor
  environment: poc
  site: poc-cliente
  agentId: sap-sensor-01
  # export.otlp vacío en la POC

modules:
  sap:
    enabled: true
    channel: true
    sensor: sap-sensor-01
    sapSystem: PRD
    sapInstance: "00"
    inventory:
      - "10.50.10.20"
      - "10.50.10.21"
    switch:
      mode: span          # span | rspan | erspan
      bothDirections: true
    ingest:
      iface: eth1
    logEvents: true
    decode:
      authorized: false
  probes:
    enabled: true
    interval: 30s
    targets:
      - name: sap-disp
        type: tcp
        endpoint: 10.50.10.20:3200
      - name: sap-web
        type: https
        endpoint: fiori.ejemplo.com:443
      - name: sap-dns
        type: dns
        endpoint: fiori.ejemplo.com
```

```bash
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/ekumetrics-agent
sudo systemctl restart ekumetrics-agent
curl -s http://127.0.0.1:9090/healthz
curl -s http://127.0.0.1:9090/metrics | grep ekms_sap
journalctl -u ekumetrics-agent -e
```

Zeek (`zeek.enabled`) es un sidecar opcional. No alimenta las series `ekms_sap_*`. No hace falta para la POC.

---

## 12. POC recomendada (15 días, solo agente)

| Fase | Días | Resultado |
|------|------|-----------|
| Descubrimiento | 1–2 | Edición SAP, GUI/Fiori/RFC/VDI, VPN, endpoints, puertos |
| Captura controlada | 3 | Tráfico de sesiones conocidas; validar que no es solo ESP/ICA |
| Validación | 4 | Simetría, cifrado, SNC, pérdida de captura |
| Instalación | 5–6 | Agente, `setcap`, scrape Grafana a `:9090` |
| Línea base | 7–11 | Comportamiento por sede, sistema y horario |
| Tableros | 10–13 | Canal vivo, conversaciones (journal), calidad, TLS |
| Validación conjunta | 14 | Correlación con quejas de usuarios |
| Informe | 15 | Alcance real, sizing, sí/no de producción |

### Día 3 (decisivo)

| Lo que aparece | Qué hacer |
|----------------|-----------|
| Solo ESP / UDP 4500 | Mover el espejo al lado interno de la VPN |
| Solo Citrix/RDP | Capturar en la red de los servidores VDI |
| Solo HTTPS | El producto sigue siendo útil como experiencia de red/TLS |
| DIAG/RFC en claro | Se puede evaluar decode **sin** convertirlo en promesa |

### Criterios de éxito (firmables)

- Las sesiones de prueba a `32NN` / `33NN` / `443` aparecen en `:9090` y en el journal.
- Clasificación por puerto (no semántica DIAG/RFC).
- Bidireccionalidad: bytes sent y received > 0.
- Cero payload persistido.
- Se puede responder: qué sede (YAML), qué IP (journal), qué canal, desde cuándo, qué síntoma TCP.

### Criterios que no se firman

Sustituir Cloud ALM / HANA Cockpit / SolMan. Usuario SAP o tcode desde el cable. Causa interna de HANA. 95 % de “clasificación de protocolo” estilo dissector.

---

## 13. Información que hace falta del cliente

1. Edición: Public o Private / RISE.
2. Uso de SAP GUI, Fiori, WebGUI, RFC, OData, VDI o Citrix.
3. FQDN, IP, puertos, SAProuter, balanceadores.
4. Diagrama de VPN, NAT, proxy, SD-WAN y punto de salida.
5. VLAN y subredes de usuarios.
6. Punto para SPAN/TAP (ambos sentidos) y velocidad del enlace.
7. Confirmación de SNC / TLS / HTTPS.
8. Política de privacidad y retención.
9. Autorización de captura de **metadatos** (sin payload).
10. Autorización de sondas sintéticas (TCP/HTTPS/DNS), si se usan.
11. DHCP/NAC/VPN/CMDB solo si quieren correlacionar IP con persona.

No hace falta usuario SAP, RFC ni software en el SID.

---

## 14. Tableros sugeridos (Grafana sobre `:9090`)

1. **Canal vivo** — intentos, establecimientos, `sessions_active`, licencia.
2. **Mix de componentes** — sesiones por `protocol`.
3. **Conversaciones** — journal (`src_ip`, `dst_ip`, puerto). No series Prometheus por IP.
4. **Calidad** — setup p50/p95, retrans, RST por origen, dup ACK, OOO, zero-window.
5. **TLS** — versión, SNI, cipher, handshake.
6. **Salud de la sonda** — `capture_packets` vs `capture_drops`.
7. **Fuera de catálogo** — `unexpected_port_packets`, `new_endpoints`.

---

## 15. Código de referencia

| Pieza | Ruta |
|-------|------|
| Puertos | `pkg/ingest/ports.go` |
| Filtro de hosts / puertos inesperados | `pkg/ingest/gate.go` |
| Handshake TLS | `pkg/ingest/tlshello.go` |
| Sesiones y métricas | `pkg/sap/session/` |
| Captura Linux y drops | `pkg/ingest/live_linux.go`, `packetstats_linux.go` |
| Arranque del módulo | `pkg/modules/sap/module.go` |
| Sondas | `pkg/modules/probes/` |

---

## 16. Lo que este módulo no hace

- No sustituye SAP Cloud ALM, HANA Cockpit ni Solution Manager; los complementaría solo con una integración oficial futura.
- No reconstruye una transacción a partir de tráfico cifrado.
- No pone IP de cliente como label de Prometheus.
- No implementa aún escritura de `pcapDump` (el interruptor solo registra en log).
- Zeek opcional no exporta `capture_loss` a Prometheus; use `ekms_sap_capture_drops_total`.
- J2EE 50000/50001 fuera de filtro.
- Dup ACK / OOO / retransmisión son heurísticas de seq/ack, no un dissector TShark completo.

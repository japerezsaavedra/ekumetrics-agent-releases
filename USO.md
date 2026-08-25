# Ekumetrics Agent

Un proceso en el servidor. Un fichero de configuración. Usted decide qué se recolecta: cada señal tiene un interruptor. Lo que no active no se inicia.

Esta guía describe **qué obtiene** y **cómo se configura**. Rol del agente (Servidor, NOC, Sensor): [ROL.md](ROL.md). En el servidor instalado: `man ekumetrics-agent`. El canal SAP está en [SAP.md](SAP.md).

## Dónde están los datos

El agente recolecta aunque no haya plataforma detrás. Con destino, empuja las señales activas. Sin destino, los datos quedan en el propio host para que Grafana, Mimir u otro recolector los lean.

| Dirección | Contenido | Cuándo existe |
|-----------|-----------|---------------|
| `http://127.0.0.1:9090/healthz` | Salud del proceso | Si el proceso responde |
| `http://127.0.0.1:9090/metrics` | Identidad del agente y canal SAP | Siempre |
| `http://127.0.0.1:9090/inventory` | Activos conocidos y sugeridos | Siempre (lista vacía si no hay semillas ni talkers) |
| `http://127.0.0.1:9090/flows` | Metadatos NetFlow v5 (top talkers, últimos flujos) | `ingest.netflow` activo |
| `http://127.0.0.1:8889/metrics` | Servidor, procesos, aplicaciones y SNMP | Si hay algún módulo de métricas activo |
| `:4317` (gRPC) y `:4318` (HTTP) | Recepción de métricas, logs o trazas que envían las aplicaciones | Si activa recepción OTLP |

`environment` y `site` viajan en todas las señales (`deployment.environment`, `host.site`).

## Qué puede obtener

| Ámbito | Qué ve | Interruptor |
|--------|--------|-------------|
| Este servidor | CPU, memoria, discos, filesystems, load e interfaces | `metrics.host` (activo de fábrica) |
| Procesos | CPU, memoria y E/S de procesos que usted nombre | `metrics.processes` |
| Aplicaciones (Prometheus) | Lo que cada app publica en `/metrics` | `metrics.scrape` |
| Aplicaciones (OTLP) | Métricas que la app envía al agente | `metrics.otlp` |
| Equipos de red | Uptime, ifAdmin/ifOper, speed, octetos, paquetes, errores y descartes; CPU/memoria del equipo con `host-mib` | `snmp.devices` |
| Syslog del sitio | Eventos UDP/TCP que los equipos envían al agente | `ingest.syslog` |
| Flujos | NetFlow v5 (metadatos; sin payload): IPs, puertos, proto, bytes, paquetes, duración | `ingest.netflow` |
| Traps SNMP | `coldStart` (info), `linkDown` (warning); apagado de fábrica | `ingest.traps` |
| Inventario sugerido | Semillas + ARP/LLDP/CDP/ENTITY; ID `site/<site>/ip/<ip>`; no escribe CMDB | `discovery.passive` |
| Conectividad | Latencia y pérdida ICMP o TCP | `probes` |
| Logs de fichero | Líneas nuevas de rutas que declare | `logs` + `files` |
| journald | Diario de systemd (Linux) | `logs` + `journald` |
| Event Log | Canal Application (Windows) | `logs` + `windowsEvent` |
| Logs de aplicaciones | Logs OTLP que la app envía al agente | `logs` + `otlp` |
| Trazas | Trazas OTLP de apps ya instrumentadas | `traces` |
| SAP (red) | Sesiones hacia componentes SAP: quién habla con quién, cuánto, RTT, retransmisiones | `sap` + `channel` |
| SAP (protocolo) | Clasificación DIAG, RFC y demás **solo en claro** y con permiso | `sap.decode.authorized` |
| SAP (trabajo) | Transacción y usuario seudonimizado desde un fichero autorizado | `sap.work` |

El agente **no** instrumenta aplicaciones, **no** barre la red (el descubrimiento activo está apagado), **no** descifra SNC/TLS/HTTPS y **no** sustituye SAP Solution Manager ni una CMDB.


## Configuración

Ruta en producción: `/etc/ekumetrics-agent/agent.yaml`. Tras editar:

```bash
sudo systemctl restart ekumetrics-agent
```

No hay recarga en caliente. Active solo lo que vaya a usar.

El fichero tiene tres zonas. No declare el mismo equipo en dos sitios.

| Zona | Claves | Pregunta |
|------|--------|----------|
| Infra | `agent`, `export`, `receive` | ¿Quién soy y a dónde envío? |
| Este host y el sitio | `modules` | ¿Qué recojo en esta máquina y qué escucho? |
| Equipos y apps | `snmp`, `databases`, `queues`, `icewarp` | ¿Qué hay alrededor? |

Los equipos SNMP se declaran en `snmp.devices`. El canal SAP (`mode: sensor`) está en [SAP.md](SAP.md), no en el YAML de un agente Servidor.

### Identidad

```yaml
agent:
  environment: production
  site: planta-norte
  tenantId: cliente
  agentId: agent-planta-norte-01
  mode: site
  metricsAddr: ":9090"
```

| Campo | Uso |
|-------|-----|
| `environment` | Entorno (production, qa, …). Sale en `ekms_agent_info` y en los datos exportados |
| `site` / `tenantId` / `agentId` | Sitio, tenant y agente hacia Ekumetrics Platform |
| `mode` | `site` (Servidor), `central` (NOC), `sensor` (SPAN) o `endpoint`. Ver [ROL.md](ROL.md) |
| `metricsAddr` | Identidad, `/healthz`, `/inventory`, `/flows` y SAP |

En `:9090/metrics` verá:

| Serie | Significado |
|-------|-------------|
| `ekms_agent_info` | Agente vivo (`environment`, `site`, `version`) |
| `ekms_agent_identity` | `tenant_id`, `site_id`, `agent_id`, `mode` |
| `ekms_agent_module_enabled` | `1` si ese módulo está encendido |

### Destino (plataforma)

```yaml
export:
  otlp:
    endpoint: "otel-backend:4317"
    insecure: true
    caFile: /etc/ekumetrics-agent/ca.pem
    certFile: /etc/ekumetrics-agent/agent.pem
    keyFile: /etc/ekumetrics-agent/agent-key.pem
  buffer:
    dir: ""
    maxItems: 10000
    compress: false
```

Un solo destino. Vacío: recolecta y no reenvía. `insecure: true` solo si el destino no usa TLS. En producción use `insecure: false` y certificados (TLS 1.2, mTLS si hay `certFile` y `keyFile`).

Los eventos de inventario, flujos y traps se guardan en disco (store-and-forward) y se reenvían a `/v1/ekms/events` conservando el timestamp original. Si no hay destino, la cola sigue persistiendo. Si el enlace cae, reintenta con backoff y prioriza `warning`/`error`.

| Señal | Destino habitual |
|-------|------------------|
| Trazas | Tempo |
| Logs | Loki (o un receptor intermedio) |
| Métricas | Mimir, o scrape de `:8889` y `:9090` |

Para Loki + Tempo + Mimir, apunte a un receptor que reparta por tipo de señal.

### Recepción OTLP (aplicaciones → agente)

```yaml
receive:
  otlp:
    grpc: ":4317"
    http: ":4318"
```

Esos puertos **solo se abren** si activa `metrics.otlp`, `logs.otlp` o `traces`. La aplicación debe enviar al host del agente. El agente no inyecta agentes ni SDKs.

---

### Servidor (sistema operativo)

Activo de fábrica. Intervalo: 30 segundos. Lectura: `:8889/metrics`.

```yaml
modules:
  metrics:
    host:
      enabled: true
```

| Qué obtiene | Series típicas en `:8889` |
|-------------|---------------------------|
| Tiempo de CPU por estado | `system_cpu_time_seconds_total` |
| Memoria usada | `system_memory_usage_bytes` |
| Espacio e inodes de cada filesystem | `system_filesystem_usage_bytes`, `system_filesystem_inodes_usage` |
| I/O de disco | `system_disk_io_bytes_total`, `system_disk_operations_total` |
| Load 1 / 5 / 15 minutos | `system_cpu_load_average_1m`, `_5m`, `_15m` |
| Tráfico, paquetes y errores por NIC | `system_network_io_bytes_total`, `system_network_packets_total`, `system_network_errors_total` |

Es el mínimo recomendado en cada servidor.

### Procesos

```yaml
    processes:
      enabled: true
      names: ["nginx", "postgres", "sapstartsrv"]
```

| Qué obtiene | Series típicas |
|-------------|----------------|
| CPU del proceso | `process_cpu_time_seconds_total` |
| Memoria residente y virtual | `process_memory_usage_bytes`, `process_memory_virtual_bytes` |
| E/S a disco | `process_disk_io_bytes_total` |

`names` vacío con `enabled: true` cubre **todos** los procesos y genera muchas series. En producción, liste los que importan. Los valores de `names` se interpretan como expresión regular.

### Aplicaciones con `/metrics` (Prometheus)

```yaml
    scrape:
      enabled: true
      jobs:
        - job: facturacion
          targets: ["127.0.0.1:8080"]
```

El agente visita cada destino cada 15 segundos y toma lo que publique en `/metrics`. Usted declara el inventario: no hay descubrimiento.

`enabled: true` sin `jobs` no inicia nada.

### Métricas OTLP de aplicaciones

```yaml
    otlp:
      enabled: true
```

La aplicación envía métricas a `:4317` o `:4318`. El agente las etiqueta con entorno y sitio y, si hay destino, las reenvía. En local también salen por `:8889`.

### Equipos SNMP (cualquier dispositivo)

Un agente por sitio. Declara **cualquier** equipo que hable SNMP: switch, firewall, impresora, UPS, radio, PLC. v2c o v3. Puerto por defecto **161**. Perfiles: `if-mib`, `host-mib`, `ups`, `icewarp`.

Use el bloque de raíz. `modules.metrics.snmp.targets` sigue funcionando si ya lo tenía.

```yaml
snmp:
  enabled: true
  devices:
    - name: core-sw
      role: switch
      endpoint: "192.168.1.1"
      version: v2c
      community: public
      profiles: [if-mib, host-mib]
      interval: 60s
    - name: fw
      role: firewall
      endpoint: "192.168.1.2"
      version: v3
      user: otel
      securityLevel: auth_priv
      authType: SHA
      authPassword: "..."
      privacyType: AES
      privacyPassword: "..."
      profiles: [if-mib]
    - name: ups-sala
      role: ups
      endpoint: "192.168.1.8"
      version: v2c
      community: public
      profiles: [ups]
```

Intervalo: `interval` por dispositivo. Vacío: 60 s, o 120 s si el único perfil es `ups`. El host del agente debe alcanzar UDP/161. Si `discovery.passive` está on, estos equipos son las semillas de ARP/LLDP/CDP.

| Qué obtiene | Serie |
|-------------|--------|
| Tiempo encendido | `snmp_sys_uptime` |
| Estado admin / operativo | `snmp_if_admin`, `snmp_if_oper` |
| Velocidad nominal | `snmp_if_speed` (para calcular utilización) |
| Octetos / paquetes | `snmp_if_io`, `snmp_if_packets` |
| Errores / descartes | `snmp_if_errors`, `snmp_if_discards` |
| CPU y memoria del equipo | `snmp_hr_cpu`, `snmp_hr_memory_size` (perfil `host-mib`) |
| Almacenamiento | `snmp_hr_storage_size`, `snmp_hr_storage_used` |
| UPS (perfil `ups`) | `snmp_ups_battery_status`, `snmp_ups_battery_minutes` |

`enabled: true` sin `devices` no inicia SNMP. SNMPv2c envía la community en claro: use red de gestión o v3.

### Bases de datos y colas

Cuenta de **solo lectura**. No se leen filas ni mensajes. Si el `endpoint` no lleva puerto, se usa el de fábrica.

| `type` | Puerto | Qué mira |
|--------|--------|----------|
| `postgresql` | 5432 | conexiones, tuplas, locks, réplica |
| `mysql` | 3306 | hilos, buffer pool, réplica |
| `redis` | 6379 | memoria, evictions, hits |
| `mongodb` | 27017 | ops, conexiones, réplica |
| `kafka` | 9092 | brokers, topics, consumers |
| `rabbitmq` | 15672 (API) | colas y consumidores |
| `nats` | 8222 `/metrics` | scrape de monitoring |

```yaml
databases:
  enabled: true
  targets:
    - name: pg-sitio
      type: postgresql
      endpoint: "127.0.0.1"
      username: ekms_ro
      passwordFile: /etc/ekumetrics-agent/pg.pw
```

Las trazas las emite la **aplicación**, no el motor (`traces.enabled`).

### IceWarp

Solo IceWarp Server. SNMP en **1161** (no 161) y probes de puertos de correo. No lee buzones.

```yaml
icewarp:
  enabled: true
  targets:
    - name: mail-sitio
      endpoint: "192.168.1.20"
      snmp:
        enabled: true
        port: "1161"
        community: public
      probe: true
```

### Syslog del sitio

Los equipos envían syslog al agente (no lee `/var/log/syslog` remoto).

```yaml
  ingest:
    syslog:
      enabled: true
      listen: ":5514"
      protocol: udp
      rfc: rfc3164
```

NetFlow v5 (metadatos, sin payload) y traps:

```yaml
  ingest:
    netflow:
      enabled: true
      listen: ":2055"
    traps:
      enabled: false
      listen: ":9162"
  discovery:
    passive:
      enabled: true
    active:
      enabled: false
      authorized: false
```

El switch debe exportar NetFlow al agente. Cada registro v5 publica IPs, puertos, protocolo, bytes, paquetes, interfaces y duración. La primera vez que aparece un par origen/destino se marca como comunicación nueva. v9/IPFIX se cuentan y no se decodifican. Últimos flujos y top talkers: `http://127.0.0.1:9090/flows`.

Inventario sugerido (no es CMDB): `http://127.0.0.1:9090/inventory` y series `ekms_asset_known` / `ekms_asset_suggested`. El ID de activo es `site/<site_id>/ip/<ip>` y no cambia al reiniciar. El pasivo (v2c) lee `sysName`, ARP, LLDP (nombre e interfaz), CDP, ENTITY (serial/modelo si el equipo los da) y `sysObjectID`. Un activo que deja de verse pasa a `stale` y no se borra. El inventario se guarda en el cache del agente (`EKUMETRICS_CACHE_DIR` o el directorio de cache del usuario). El descubrimiento activo exige `authorized: true` y en esta fase no barre la red.

Traps nacen apagados. `coldStart` y `warmStart` son `info`; `linkDown` y fallos de autenticación son `warning`. Use `:9162` si el proceso no es root.

### Sondas (latencia y pérdida)

```yaml
  probes:
    enabled: true
    interval: 30s
    targets:
      - name: wan
        type: icmp
        endpoint: 8.8.8.8
      - name: sap-disp
        type: tcp
        endpoint: 10.0.0.5:3200
      - name: sap-web
        type: https
        endpoint: fiori.ejemplo.com:443
      - name: sap-dns
        type: dns
        endpoint: fiori.ejemplo.com
```

| Serie en `:9090` | Significado |
|------------------|-------------|
| `ekms_probe_up` | `1` si respondió |
| `ekms_probe_rtt_seconds` | Latencia (en HTTPS es el handshake TLS) |
| `ekms_probe_loss_ratio` | `1` si se perdió |
| `ekms_probe_tls_not_after_timestamp_seconds` | Fin del certificado (solo `https`) |

ICMP usa `ping` del sistema. TCP no necesita privilegios. `https` y `dns` son pruebas **autorizadas** al FQDN que declare; no descifran el tráfico de usuarios. Los conectores de gestores (Strata, FortiAnalyzer, iMaster) no están en esta versión.

### Logs

Interruptor maestro `logs.enabled`. Debe activar **al menos una** fuente.

```yaml
  logs:
    enabled: true
    files: ["/var/log/app/*.log", "/var/log/syslog"]
    journald: true
    windowsEvent: false
    otlp: false
```

| Fuente | Qué obtiene | Requisito |
|--------|-------------|-----------|
| `files` | Líneas nuevas (no el histórico) | Rutas o glob que el proceso pueda leer |
| `journald` | Diario de systemd | Linux; el servicio usa el grupo `systemd-journal` |
| `windowsEvent` | Canal Application | Windows |
| `otlp` | Logs que la app envía a `:4317` / `:4318` | App instrumentada |

Cada línea lleva `service.name`, entorno y sitio. Sin `export.otlp.endpoint` se recolecta y no se reenvía. Con destino, los logs de fichero van a `/v1/logs` (puerto HTTP 4318 del destino). El bundle de Fluent Bit lleva OpenSSL, `libpq` y el resto de `.so` de la imagen oficial; no hay que instalar librerías en el servidor.

### Trazas

```yaml
  traces:
    enabled: true
```

Solo recibe. La aplicación debe estar instrumentada y apuntar al agente. Sin destino las trazas no se persisten en el host (no hay scrape de trazas).

---

### SAP — sesiones de red

Observa el tráfico hacia componentes SAP **sin entrar al sistema**. Por defecto solo metadatos: no payload, no usuario, no transacción.

Hace falta un mirror (SPAN, RSPAN o ERSPAN) o un TAP hacia una **segunda NIC** del sensor, y capacidades de red:

```bash
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/ekumetrics-agent
sudo systemctl restart ekumetrics-agent
```

```yaml
  sap:
    enabled: true
    channel: true
    sensor: sap-sensor-01
    sapSystem: PRD
    sapInstance: "00"
    switch:
      mode: span          # span | rspan | erspan
      bothDirections: true
    ingest:
      iface: eth1
    logEvents: true
```

| Campo | Uso |
|-------|-----|
| `enabled` | Enciende el módulo |
| `channel` | Activa captura (interfaz o fichero PCAP) |
| `sensor` | Nombre del sensor en el evento |
| `sapSystem` / `sapInstance` | SID e instancia que etiqueta los datos |
| `switch.mode` | Tipo de mirror |
| `switch.bothDirections` | El mirror debe copiar ida y vuelta |
| `ingest.iface` | NIC de captura (no la de gestión) |
| `ingest.pcap` | En lugar de la NIC: reproduce un PCAP |
| `ingest.pcapRealtime` | Respeta tiempos del PCAP |
| `logEvents` | Escribe cada sesión en JSON (stdout / journal) |

**Qué obtiene en cada sesión** (`logEvents: true`):

| Campo | Significado |
|-------|-------------|
| `timestamp` | Cierre o expiración de la sesión |
| `sensor` | Sensor que la vio |
| `protocol` | Clasificación por puerto (DIAG, RFC, Message Server, Router, HANA, HTTP, …) |
| `src_ip` / `src_port` | Origen |
| `dst_ip` / `dst_port` | Destino |
| `duration_ms` | Duración de la conexión TCP (no de una transacción) |
| `bytes_sent` / `bytes_received` | Volumen |
| `packets_sent` / `packets_received` | Segmentos SYN, datos, FIN o RST |
| `tcp_rtt_ms` | Tiempo de establecimiento TCP (SYN→SYN-ACK). No es logon SAP |
| `retransmissions` / `resets` | Calidad del enlace |
| `dup_ack` / `out_of_order` / `zero_window` | Señales TCP (no pérdida de captura) |
| `reset_origin` | `client` o `remote` si hubo RST |
| `outcome` | `established`, `syn_timeout`, `rejected` o `unknown` |
| `encrypted` | Puerto HTTPS o ClientHello/ServerHello visto |
| `tls_version` / `tls_sni` / `tls_cipher` | Visible en el handshake; no descifra el resto |
| `tls_handshake_ms` | ClientHello → ServerHello |
| `sap_system` / `sap_instance` | SID e instancia configurados |

**Qué obtiene en `:9090`:**

| Serie | Significado |
|-------|-------------|
| `ekms_sap_sessions_total` | Sesiones cerradas o expiradas (`protocol`, `encrypted`, `sap_system`) |
| `ekms_sap_tcp_attempts_total` | SYN cliente hacia un puerto SAP |
| `ekms_sap_tcp_established_total` | SYN-ACK visto |
| `ekms_sap_tcp_syn_timeout_total` | SYN sin respuesta (30 s) |
| `ekms_sap_tcp_rejected_total` | RST remoto sin establecimiento |
| `ekms_sap_tcp_setup_seconds` | Histograma SYN→SYN-ACK (p50/p95/p99 en Grafana) |
| `ekms_sap_session_duration_seconds` | Histograma de duración TCP |
| `ekms_sap_sessions_active` | Sesiones abiertas ahora |
| `ekms_sap_session_bytes_total` | Bytes (`direction`: sent / received) |
| `ekms_sap_session_packets_total` | Segmentos por dirección |
| `ekms_sap_session_retransmissions_total` | Retransmisiones aparentes |
| `ekms_sap_session_resets_total` | RST (`origin`: client / remote) |
| `ekms_sap_capture_packets_total` | TCP a puertos SAP (o a host SAP por otro puerto) |
| `ekms_sap_capture_drops_total` | Descartes del socket de captura (toda la NIC) |
| `ekms_sap_tcp_dupack_total` | ACK duplicados |
| `ekms_sap_tcp_outoforder_total` | Segmentos fuera de orden |
| `ekms_sap_tcp_zerowin_total` | Ventana TCP cero |
| `ekms_sap_tls_handshakes_total` | Handshake TLS visible (`version`) |
| `ekms_sap_tls_handshake_seconds` | Histograma ClientHello→ServerHello |
| `ekms_sap_tls_sni_total` | SNI del ClientHello |
| `ekms_sap_tls_cipher_total` | Cipher del ServerHello |
| `ekms_sap_unexpected_port_packets_total` | Puerto fuera de catálogo hacia un host SAP conocido |
| `ekms_sap_new_endpoints_total` | Primera vez que aparece un IP:puerto SAP |

Con SNC, TLS o HTTPS se ve **quién, a dónde, cuándo, cuánto y la calidad del enlace**. No se ve usuario, transacción, SQL ni payload.

Detalle de red y checklist del cliente: [SAP.md](SAP.md).

### SAP — protocolo en claro (opcional)

Solo con autorización expresa. No descifra.

```yaml
    decode:
      authorized: true
    inventory: ["10.50.10.20", "10.50.10.21"]  # IPs SAP; sin esto solo se aprenden al ver un puerto 32NN/33NN/…
```

| Qué obtiene | Cuándo |
|-------------|--------|
| Tipo de protocolo (DIAG, RFC, Message Server, Gateway, Router, HANA, HTTP, SNC, …) | Tráfico en claro |
| Conteos y tamaños de mensaje, errores, tramas cifradas o malformadas | Series `ekms_sap_protocol_*` en `:9090` |

`inventory` marca IPs que usted confirma como SAP. Sin `authorized` el agente no abre el payload.

### SAP — trabajo autorizado (usuario / transacción)

No sale del cable. Entra por un JSONL que usted genera (STAD, audit u otra fuente con permiso).

```yaml
    work: true
    workFile: /var/lib/ekumetrics/sap-work.jsonl
```

Cada línea:

```json
{"timestamp":"2026-08-18T10:30:00Z","environment":"production","sap_system":"PRD","user_hash":"a1b2c3","tcode":"VA01","work_type":"dialog","status":"ok","duration_ms":420,"source":"stad"}
```

El usuario va **hasheado**, no en claro.

| Serie en `:9090` | Significado |
|------------------|-------------|
| `ekms_sap_work_transactions_total` | Pasos por transacción, tipo y estado |
| `ekms_sap_work_step_duration_seconds` | Duración del paso |
| `ekms_sap_work_user_steps_total` | Pasos por `user_hash` |

---

## Instalación

Linux x86_64 (Debian, Ubuntu, Rocky, AlmaLinux, RHEL). Paquete de la [release](https://github.com/japerezsaavedra/ekumetrics-agent/releases). Si ya corre el binario a mano, deténgalo para liberar `:9090`.

```bash
# Debian / Ubuntu
sudo apt-get install -y ./ekumetrics-agent_*_amd64.deb

# Rocky / AlmaLinux / RHEL
sudo dnf install -y ./ekumetrics-agent-*.x86_64.rpm

# Cualquier Linux x86_64
tar -xzf ekumetrics-agent-*-linux-amd64.tar.gz
cd ekumetrics-agent-*-linux-amd64
sudo ./install.sh
```

| Ruta | Uso |
|------|-----|
| `/usr/bin/ekumetrics-agent` | Proceso |
| `/etc/ekumetrics-agent/agent.yaml` | Configuración (no se pisa al actualizar; se anaden bloques de módulos nuevos si faltan; `dpkg` no pregunta) |
| `ekumetrics-agent.service` | Arranque automático; si cae, vuelve a levantar |
| `/usr/share/man/man8/ekumetrics-agent.8` | Manual (`man ekumetrics-agent`) |
| `/usr/share/doc/ekumetrics-agent/USO.md` | Guía de uso (esta) |
| `/usr/share/doc/ekumetrics-agent/ROL.md` | Roles site / central / sensor |
| `/usr/share/doc/ekumetrics-agent/SAP.md` | Canal SAP |

```bash
sudo systemctl status ekumetrics-agent
curl -s http://127.0.0.1:9090/healthz
journalctl -u ekumetrics-agent -e
```

Parada: `sudo systemctl stop ekumetrics-agent`.

## Comprobar que recolecta

| Pregunta | Cómo |
|----------|------|
| ¿El proceso vive? | `curl -s http://127.0.0.1:9090/healthz` responde `ok` |
| ¿Qué módulos están on? | `ekms_agent_module_enabled` en `:9090/metrics` |
| ¿Hay CPU, disco, SNMP, apps? | `:8889/metrics` |
| ¿Hay sesiones SAP? | `ekms_sap_sessions_total` o `ekms_sap_tcp_attempts_total` en `:9090/metrics` |
| ¿Hay activos sugeridos? | `http://127.0.0.1:9090/inventory` |
| ¿Hay flujos? | `http://127.0.0.1:9090/flows` (si NetFlow está on) |
| ¿Llega a la plataforma? | `export.otlp.endpoint` informado; si está vacío, scrapee el agente |

## Operación

- Un agente por servidor. Los equipos de red se declaran en `snmp.devices` de ese agente.
- Cada módulo genera series o abre puertos. No active “por si acaso”.
- No declare el mismo dispositivo en `snmp.devices` y en `modules.metrics.snmp`.
- Al detener el servicio se detiene toda la recolección.

## Si falta un dato

| Lo que ve | Causa habitual |
|-----------|----------------|
| Grafana vacío | No hay destino y nadie scrapea `:8889` / `:9090` |
| Host sin series | El módulo está en `false` |
| Procesos o SNMP vacíos | `names`, `jobs` o `snmp.devices` vacíos |
| Logs sin efecto | `enabled: true` sin `files`, `journald`, `windowsEvent` ni `otlp` |
| Syslog sin eventos | `ingest.syslog` off, puerto ocupado o los equipos no apuntan al agente |
| Sondas en 0 | `probes` sin `targets`, ICMP sin `ping`, o el destino no responde |
| Trazas ausentes | La app no está instrumentada o no apunta a `:4317` / `:4318` |
| SAP sin sesiones | Mirror de un solo sentido, NIC incorrecta o faltan capacidades de red |
| SAP sin usuario ni tcode | Tráfico cifrado, o no hay fichero de trabajo autorizado |

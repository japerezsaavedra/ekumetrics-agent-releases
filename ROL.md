# Rol del Ekumetrics Agent

El agente recolecta en el borde. **Ekumetrics Platform** correlaciona, guarda CMDB, tickets, HolmesGPT y la gestión remota.

Cinco funciones: descubrir (sugerir, no pisar CMDB), recolectar, probar, remediar con control, enviar datos normalizados a Platform.

El producto es **Ekumetrics Agent**. La identidad (`tenant_id`, `site_id`, `agent_id`) la pone el despliegue. No hay cliente ni sede fijos en el binario.

## Modos de instalación

El valor YAML no cambia. En la plataforma el nombre visible es el de la primera columna.

| Tipo | YAML | Qué hace |
|------|------|----------|
| Servidor | `site` | El propio host y, si se activa, la red de la sede: SNMP, syslog, NetFlow, traps, probes, inventario pasivo. **Un servidor usa este tipo.** Un proceso por zona, no uno por equipo. |
| NOC | `central` | APIs de gestores (Strata, FortiAnalyzer, iMaster, FMC). Reservado. |
| Sensor | `sensor` | Observación pasiva en SPAN/TAP (hoy: canal SAP). Sin payload salvo autorización. |
| Endpoint | `endpoint` | Métricas y logs del PC o portátil. No es MDM. |

`agent.mode` elige el modo. Detalle operativo de la plataforma: `ekumetrics-platform/docs/tenants-sitios-agentes.md`. El descubrimiento **activo** nace apagado y exige `authorized: true`. En esta fase, aunque se autorice, no barre la red.

## Fase actual (B + C)

El agente es capaz en sede: inventario pasivo usable, SNMP operacional, flujos v5 como metadatos, envelope común, cola store-and-forward y export con mTLS.

| Incluido | Aún no |
|----------|--------|
| Identidad por sede y envelope (`tenant_id`, `site_id`, `agent_id`, `asset_id`) | Runbooks y remediación |
| Inventario sugerido persistente (`/inventory`): MAC, OUI, serial, vecinos, stale | Cruce real con CMDB |
| Pasivo SNMP v2c: ARP, LLDP, CDP, ENTITY, `sysObjectID` | Walk SNMP v3 (la semilla v3 sí se declara) |
| NetFlow v5 en `/flows` (sin payload); v9/IPFIX solo se cuentan | IPFIX/sFlow/streaming |
| SNMP `if-mib` (admin/oper/speed/io/paquetes), `host-mib`, `ups` | MIB de fabricante, traps con parsing de vendor |
| Traps UDP opcionales (`ingest.traps`, apagados de fábrica) | Vault, rotación automática de certificados |
| Cola persistente de envelope y reenvío con timestamp original | Runbooks y remediación |
| Export OTLP con TLS 1.2 y mTLS opcional | DNS/HTTP/TLS probes, certificados, Holmes |
| Probes ICMP/TCP | IPFIX/sFlow, conectores central |

Detalle de operación: [USO.md](USO.md).

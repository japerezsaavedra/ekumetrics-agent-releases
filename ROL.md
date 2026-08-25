# Rol del Ekumetrics Agent

El agente recolecta en el borde. Cada instalación elige un modo.

La identidad la define el despliegue:

| Término | YAML / señal | Qué es |
|---------|--------------|--------|
| Tenant | `tenantId` → `tenant_id` | El cliente (la organización) |
| Sitio | `site` → `site_id` | Una planta, oficina o zona de ese tenant |
| Agente | `agentId` → `agent_id` | El proceso que reporta |

Un tenant tiene varios sitios. Un sitio no se comparte entre tenants. El modo `site` en el YAML es el tipo **Servidor**, no el sitio.

## Modos

| Tipo | YAML | Qué hace |
|------|------|----------|
| Servidor | `site` | El propio host y, si se activa, la red del sitio: SNMP, syslog, NetFlow, traps, probes e inventario pasivo. Un proceso por servidor, no uno por equipo de red. |
| Sensor | `sensor` | Observación pasiva en SPAN/TAP (canal SAP). Sin payload salvo autorización. |
| NOC | `central` | APIs de gestores de red. Reservado. |
| Endpoint | `endpoint` | Métricas y logs del PC o portátil. No es MDM. |

`agent.mode` elige el modo. El descubrimiento activo nace apagado. Detalle de operación: [USO.md](USO.md). El canal SAP: [SAP.md](SAP.md).

# Rol del Ekumetrics Agent

El agente recolecta en el borde. Cada instalación elige un modo. La identidad (`tenant_id`, `site_id`, `agent_id`) la define el despliegue.

## Modos

| Tipo | YAML | Qué hace |
|------|------|----------|
| Servidor | `site` | El propio host y, si se activa, la red de la sede: SNMP, syslog, NetFlow, traps, probes e inventario pasivo. Un proceso por servidor, no uno por equipo de red. |
| Sensor | `sensor` | Observación pasiva en SPAN/TAP (canal SAP). Sin payload salvo autorización. |
| NOC | `central` | APIs de gestores de red. Reservado. |
| Endpoint | `endpoint` | Métricas y logs del PC o portátil. No es MDM. |

`agent.mode` elige el modo. El descubrimiento activo nace apagado. Detalle de operación: [USO.md](USO.md). El canal SAP: [SAP.md](SAP.md).

# Prompts utilizados

## 1. Revisión de arquitectura

**Rol:**\
Actúa como Cloud Engineer Senior con experiencia en AWS multi-account,
Cloud WAN, Terraform, CI/CD, seguridad y DRP.

**Contexto:**\
Estoy resolviendo una prueba técnica para una compañía con AWS
Organizations, cuentas hub/dev/qat/prd, Terraform, Azure DevOps con OIDC
y AWS Cloud WAN multi-región.

**Objetivo:**\
Analiza la propuesta y detecta riesgos de arquitectura, seguridad,
operación, resiliencia y costo.

**Restricciones:**\
- No sobrearquitecturar. - Priorizar soluciones reversibles y
auditables. - No asumir capacidades de AWS sin validarlas. - Diferenciar
hechos, hipótesis y trade-offs.

**Salida:**\
1. Problemas detectados. 2. Recomendación. 3. Trade-offs. 4. Riesgos. 5.
Qué debería validar en documentación oficial.

------------------------------------------------------------------------

## 2. Diseño de pipeline Terraform

**Rol:**\
Actúa como DevOps/Cloud Engineer Senior especializado en Terraform y
Azure DevOps.

**Contexto:**\
El pipeline usa OIDC hacia una cuenta Hub y desde allí asume roles
cross-account en dev, qat y prd.

**Objetivo:**\
Diseña una estrategia de promoción dev → qat → prd manteniendo el mismo
código/artefacto y evitando que dev pueda desplegar en prd.

**Restricciones:**\
- Sin access keys estáticas. - Least privilege. - Aprobación manual
antes de producción. - State aislado. - Terraform reusable.

**Salida:**\
Pipeline lógico, modelo IAM, estrategia de state y controles de
seguridad.

------------------------------------------------------------------------

## 3. Cloud WAN / routing

**Rol:**\
Actúa como Network/Cloud Architect senior especializado en AWS Cloud WAN
y BGP.

**Contexto:**\
Existe Cloud WAN multi-región, segmentación por ambiente, Direct Connect
primario, VPN backup y FortiGate para inspección.

**Objetivo:**\
Analiza failover, segmentación, service insertion, routing policies, BGP
y posibles causas de blackhole/asimetría.

**Restricciones:**\
- No usar TGW Peering. - No usar VPN inter-region para conectar
regiones. - El tráfico crítico debe permanecer inspeccionado. - Evitar
routing asimétrico.

**Salida:**\
1. Diseño recomendado. 2. Flujo de rutas. 3. Mecanismo de failover. 4.
Riesgos. 5. Pruebas que demostrarían que el diseño funciona.

------------------------------------------------------------------------

## 4. Troubleshooting

**Rol:**\
Actúa como SRE/Cloud Engineer senior.

**Contexto:**\
Existe una aplicación detrás de ALB con latencia intermitente y CPU
normal.

**Objetivo:**\
Construye un árbol de diagnóstico que permita diferenciar problemas de
red, aplicación, base de datos y capacidad.

**Restricciones:**\
No escalar recursos sin evidencia.

**Salida:**\
Orden de diagnóstico, métricas, logs, hipótesis y criterios para
descartar cada una.

------------------------------------------------------------------------

## 5. FinOps

**Rol:**\
Actúa como Cloud FinOps Engineer Senior.

**Contexto:**\
El gasto de AWS aumenta de forma sostenida.

**Objetivo:**\
Diseña una metodología para encontrar oportunidades de reducción de
costo sin degradar disponibilidad o seguridad.

**Restricciones:**\
Distinguir ahorro real de simple transferencia de costo.

**Salida:**\
Prioridades, métricas, herramientas, quick wins, compromisos Savings
Plans/RI y riesgos.

------------------------------------------------------------------------

## 6. Revisión final de la respuesta

**Rol:**\
Actúa como entrevistador técnico de un proceso para Senior Cloud
Engineer.

**Objetivo:**\
Evalúa la siguiente respuesta como si fuera presentada por un candidato.

**Criterios:**\
- profundidad técnica; - claridad; - criterio; - trade-offs; -
seguridad; - costos; - resiliencia; - operabilidad; - ausencia de
sobreingeniería.

**Salida:**\
1. Fortalezas. 2. Debilidades. 3. Preguntas que probablemente haría el
entrevistador. 4. Correcciones prioritarias. 5. Veredicto general.

# Guía de diagramas

Los diagramas pueden construirse en Draw.io/diagrams.net siguiendo las
siguientes capas.

## 01 --- CI/CD

**Actores** - Developer - Git repository - Azure DevOps - OIDC - Hub
account - Dev/QAT/PRD accounts

**Flujo** Developer → PR → Quality gates → Build → Artifact → OIDC → Hub
→ Cross-account role → Dev → QAT → Approval → PRD

------------------------------------------------------------------------

## 02 --- Cloud WAN

**Componentes** - Cloud WAN core network - us-east-1 - us-west-2 -
Dev/QAT/PRD segments - Hybrid segment - Inspection segment - VPC
attachments - DX attachment - VPN attachment - FortiGate

------------------------------------------------------------------------

## 03 --- Hybrid failover

On-prem → DX → Cloud WAN\
On-prem → VPN → Cloud WAN

Marcar DX como primary y VPN como backup.

Añadir BGP/BFD y representar explícitamente el retiro de rutas durante
la falla.

------------------------------------------------------------------------

## 04 --- Service insertion

PRD VPC → Cloud WAN → Inspection VPC → FortiGate → Cloud WAN → destino.

Para north-south:

PRD → FortiGate → Internet/On-prem.

Para east-west:

PRD → FortiGate → otra VPC/segment.

------------------------------------------------------------------------

## 05 --- Regional DR

Representar:

-   us-east-1 / primary
-   us-west-2 / DRP
-   Cloud WAN entre ambos
-   workloads PRD
-   datos replicados
-   inspection VPC por región
-   Route 53 / Global Accelerator como entrada, según la decisión final.

------------------------------------------------------------------------

## 06 --- Arquitectura integradora

Mantener máximo 5 capas:

1.  Users / Internet
2.  Edge: Route 53 + CloudFront + WAF
3.  Application: API Gateway/ALB + Lambda/EKS
4.  Cloud WAN + Inspection + Hybrid
5.  Data: RDS/DynamoDB/S3 + On-prem core

El diagrama ejecutivo no debe convertirse en un inventario de recursos.

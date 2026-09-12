# Technical Assessment --- Senior Cloud Engineer

> Propuesta de resolución para la prueba técnica de Ingeniero Senior
> Cloud.
>
> **Enfoque:** AWS multi-account, Terraform, Azure DevOps/OIDC, Cloud
> WAN, seguridad, resiliencia, DRP y FinOps.
>
> **Criterio:** priorizar soluciones seguras, reversibles, auditables y
> operables, evitando sobrearquitectura.

------------------------------------------------------------------------

## 0. Contexto y principios de diseño

La prueba establece una organización AWS multi-cuenta con segmentación
`hub / dev / qat / prd`, Terraform con state en S3 y Azure DevOps
autenticado mediante OIDC. El flujo de despliegue parte de un rol OIDC
en la cuenta Hub y posteriormente asume un rol de despliegue en la
cuenta destino. La red utiliza AWS Cloud WAN con Virginia (`us-east-1`)
como región primaria y Oregon (`us-west-2`) como DRP, además de
FortiGate para inspección y Direct Connect + Site-to-Site VPN para
conectividad híbrida. \[Fuente: prueba, págs. 1 y 7-9\]

Mi criterio general sería:

1.  **El pipeline debe promover artefactos, no recompilar código para
    cada ambiente.**
2.  **Terraform debe tener un único código reutilizable y configuración
    por ambiente separada.**
3.  **Producción debe ser una frontera de seguridad, no solo otra
    variable del pipeline.**
4.  **Los cambios de red y seguridad deben ser centralizados,
    versionados y auditables.**
5.  **DRP se diseña a partir de RTO/RPO, no de una lista de servicios.**
6.  **Observabilidad debe permitir demostrar si un cambio realmente
    mejoró disponibilidad, rendimiento o costo.**
7.  **FinOps debe medir costo unitario y costo total, no solamente la
    factura.**

------------------------------------------------------------------------

# Bloque 1 --- Aprovisionamiento y Automatización

## Caso 1.1 --- Promoción dev → qat → prd

### 1. Pipeline de Azure DevOps

Separaría claramente **build**, **infraestructura** y **promoción**.

``` text
Developer
   |
   v
Pull Request
   |
   +--> Tests / SAST / dependency scan
   |
   v
Build immutable artifact
   |
   v
DEV
   |
   +--> smoke/integration tests
   |
   v
QAT
   |
   +--> functional/regression tests
   |
   v
Manual approval + change controls
   |
   v
PRD
```

El mismo commit y, cuando aplique, el mismo artefacto construido deben
avanzar entre ambientes. No recompilaría la Lambda en cada stage porque
introduce la posibilidad de que `dev`, `qat` y `prd` estén ejecutando
artefactos distintos.

Para Terraform:

``` text
validate
   -> fmt
   -> tflint/checkov
   -> terraform plan
   -> apply DEV

DEV validation
   -> plan QAT
   -> approval
   -> apply QAT

QAT validation
   -> plan PRD
   -> manual approval
   -> apply PRD
```

En producción utilizaría **Azure DevOps Environment approvals/checks**,
de modo que la aprobación no dependa únicamente de un `if` dentro del
YAML.

### 2. Diferencias entre ambientes

No duplicaría módulos Terraform. Separaría:

``` text
terraform/
├── modules/
│   ├── lambda/
│   ├── api-gateway/
│   └── rds/
└── environments/
    ├── dev/
    ├── qat/
    └── prd/
```

Los módulos contienen infraestructura reusable; los ambientes contienen
solamente composición y valores.

Ejemplo:

``` hcl
module "rds" {
  source = "../../modules/rds"

  environment         = var.environment
  instance_class      = var.rds_instance_class
  subnet_ids          = var.private_subnet_ids
  backup_retention    = var.backup_retention
  deletion_protection = var.deletion_protection
}
```

Los valores deberían provenir de variables/versionados de configuración,
no de `if var.environment == "prd"` repartidos por todo el código.

También establecería defaults seguros y overrides mínimos:

``` text
common configuration
       |
       +--> dev.tfvars
       +--> qat.tfvars
       +--> prd.tfvars
```

Tags mínimos comunes:

``` text
Application
Environment
Owner
CostCenter
ManagedBy=Terraform
DataClassification
```

### 3. Control OIDC hub → deploy cross-account

El control crítico debe estar en **IAM de la cuenta destino**, no
solamente en Azure DevOps.

Para cada ambiente utilizaría roles diferentes:

``` text
Azure DevOps OIDC
       |
       v
Hub: role/ado-terraform
       |
       +----> Dev account: role/deploy-dev
       |
       +----> QAT account: role/deploy-qat
       |
       +----> PRD account: role/deploy-prd
```

La trust policy del `deploy-dev` debe aceptar únicamente el
principal/rol del pipeline autorizado para dev.

El rol de dev **no debe tener `sts:AssumeRole` sobre `deploy-prd`**.

Además:

-   SCP de AWS Organizations como segunda barrera.
-   IAM Access Analyzer para revisar accesos externos.
-   Separación de roles de plan/apply cuando sea viable.
-   PRD protegido por approval + IAM + SCP.
-   CloudTrail para auditar `AssumeRole` y cambios.

La idea es que un error de YAML no pueda convertirse en un despliegue de
producción.

### 4. Terraform state

Preferiría un state remoto S3 por ambiente y, idealmente, por unidad
lógica:

``` text
s3://company-terraform-state/
├── dev/service-a/terraform.tfstate
├── qat/service-a/terraform.tfstate
└── prd/service-a/terraform.tfstate
```

Con:

-   bucket dedicado de state;
-   versionado;
-   cifrado KMS;
-   bloqueo/concurrencia soportada por el backend utilizado;
-   acceso restringido exclusivamente a roles Terraform;
-   logging/auditoría;
-   separación de permisos entre ambientes.

Para producción prefiero **cuentas separadas** y estados separados antes
que Terraform workspaces como frontera de seguridad. Los workspaces
pueden ser útiles, pero no deben sustituir una separación real de
cuentas/credenciales.

------------------------------------------------------------------------

## Caso 1.2 --- Módulo S3 reutilizable

### 1. Creación y publicación

Convertiría los 200 líneas en:

``` text
terraform-aws-secure-s3/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── README.md
├── examples/
└── tests/
```

Lo publicaría inicialmente en un **Terraform Private Registry** de la
organización. Si la compañía no tiene uno, Git privado con releases
versionadas es una alternativa razonable.

El módulo debe abstraer la implementación, no ocultar decisiones
importantes.

### 2. Versionado

Aplicaría Semantic Versioning:

``` text
1.0.0  -> primera versión estable
1.1.0  -> nueva funcionalidad compatible
1.1.1  -> bugfix
2.0.0  -> breaking change
```

Los consumidores fijan una versión:

``` hcl
module "bucket" {
  source  = "registry.company.com/platform/secure-s3/aws"
  version = "~> 1.2"
}
```

No permitiría que un `terraform init` en producción descargue
implícitamente una versión futura incompatible.

### 3. Migración sin recrear buckets

La migración debe ser una operación de **state**, no de infraestructura.

Primero hago que el módulo reproduzca exactamente la infraestructura
existente.

Después:

``` text
existing resource
       |
       v
module resource address
       |
       v
terraform state mv
       |
       v
terraform plan
       |
       +--> 0 destroy / 0 unexpected create
```

Usaría `terraform state mv` o bloques `moved {}` según la estrategia y
versión de Terraform.

No aceptaría el cambio hasta obtener un `plan` que demuestre que el
bucket existente permanece administrado por Terraform y que no hay
recreación.

------------------------------------------------------------------------

## Caso 1.3 --- Drift en producción

### 1. Diagnóstico

Primero:

``` bash
terraform plan
terraform state show <resource>
```

Compararía:

``` text
Terraform configuration
        |
Terraform state
        |
AWS actual resource
```

Para un Security Group revisaría específicamente:

-   reglas ingress;
-   reglas egress;
-   CIDRs;
-   security-group references;
-   puertos/protocolos;
-   quién realizó el cambio;
-   timestamp.

CloudTrail ayuda a identificar el cambio manual.

### 2. ¿Revertir o incorporar?

No asumiría automáticamente que el código tiene la razón.

Preguntaría:

1.  ¿El cambio fue autorizado?
2.  ¿Solucionaba un incidente?
3.  ¿Existe ticket/change?
4.  ¿La regla viola seguridad?
5.  ¿Es una modificación que debería existir arquitectónicamente?

**Si fue accidental/no autorizado:** revierto mediante Terraform.

**Si fue válido:** incorporo el cambio al código y elimino el drift.

La fuente de verdad final debe ser Terraform.

### 3. Prevención

-   IAM: quitar permisos de modificación directa en PRD.
-   SCP/permission boundaries para operaciones sensibles.
-   Terraform como único canal de cambio.
-   PRs obligatorios.
-   `terraform plan` como evidencia del cambio.
-   CloudTrail + alertas.
-   AWS Config para detectar configuraciones no conformes.
-   Runbook de break-glass con acceso temporal y auditado.

------------------------------------------------------------------------

# Bloque 2 --- Operación, Resiliencia y Soporte

## Caso 2.1 --- Latencia intermitente detrás de ALB

No empezaría escalando EC2. El CPU normal solamente descarta una
hipótesis.

### Proceso

``` text
User
 |
ALB
 |
EC2 / App
 |
RDS / dependencies
```

Primero compararía:

-   ALB `TargetResponseTime`
-   `RequestCount`
-   `HTTPCode_ELB_5XX_Count`
-   `HTTPCode_Target_5XX_Count`
-   healthy/unhealthy hosts
-   `TargetConnectionErrorCount`
-   access logs del ALB
-   latencia por endpoint
-   EC2 CPU, memory, network, disk
-   RDS CPU, connections, latency, IOPS, locks
-   logs y métricas de aplicación

Después correlacionaría por timestamp.

### Posibles causas

  -----------------------------------------------------------------------
  Causa                               Cómo la descartaría
  ----------------------------------- -----------------------------------
  Query lenta / lock RDS              Performance Insights, slow queries,
                                      connections, locks

  Saturación de connection pool       métricas de app + RDS connections

  Problema de una instancia           latencia por target/health

  Dependencia externa lenta           traces/logs y timeouts

  Network/NAT/DNS                     VPC Flow Logs, DNS metrics, NAT
                                      metrics

  Garbage collection / runtime        application metrics y profiling

  Escalado tardío                     RequestCountPerTarget + ASG metrics

  ALB/target timeout                  access logs y target response time
  -----------------------------------------------------------------------

### Diferenciar el dominio

**Red:** paquetes/flows, DNS, conexiones y latencia de red.

**DB:** consultas, locks, conexiones, IOPS y waits.

**Aplicación:** endpoint concreto, errores, GC, threads, pool.

**Escalado:** latencia aumenta mientras la demanda supera capacidad
disponible.

### Prevención

Dejaría:

-   dashboard de golden signals;
-   alarmas de p95/p99;
-   ALB access logs;
-   trazabilidad distribuida;
-   métricas por endpoint;
-   RDS Performance Insights;
-   alarmas por target unhealthy;
-   correlation/request IDs.

El objetivo no es detectar "CPU alto", sino detectar **latencia
degradada y dónde se genera**.

------------------------------------------------------------------------

## Caso 2.2 --- Resiliencia y DRP

### 1. Prioridad impacto/esfuerzo

  Cambio                           Impacto     Esfuerzo
  --------------------------- ------------ ------------
  EC2/ALB Multi-AZ                    Alto   Bajo/Medio
  RDS Multi-AZ                        Alto         Bajo
  Backups + restore probado           Alto         Bajo
  Auto Scaling                        Alto        Medio
  S3 versioning/lifecycle       Medio/Alto         Bajo
  Observabilidad                      Alto        Medio
  DR cross-region                 Muy alto         Alto
  Active-active                   Muy alto     Muy alto

Primero eliminaría el **single point of failure regional/AZ** antes de
invertir en active-active.

### 2. RTO/RPO

Para una plataforma financiera crítica propondría inicialmente:

``` text
RTO regional: 30 minutos
RPO datos críticos: <= 5 minutos
```

No son valores universales; deben validarse con negocio.

Los mediría mediante ejercicios reales:

``` text
failure injected
    |
timestamp T0
    |
restore/failover
    |
service recovered T1

RTO = T1 - T0
RPO = máximo dato perdido
```

### 3. Prueba DRP

No probaría primero en producción.

Usaría:

-   ambiente DR aislado;
-   restauración de backups;
-   pruebas de conectividad;
-   failover controlado;
-   synthetic transactions;
-   validación de datos;
-   medición RTO/RPO;
-   runbook versionado.

Después realizaría ejercicios controlados sobre producción, con change
window y criterios claros de abort.

------------------------------------------------------------------------

## Caso 2.3 --- Uso de IA para stack trace

### Prompt utilizado

``` text
Rol: actúa como ingeniero senior de AWS especializado en troubleshooting de Lambda.

Contexto:
Tengo el siguiente stack trace: <STACK_TRACE>.
La función corre en <RUNTIME>, está integrada con <SERVICIOS>,
y el incidente comenzó aproximadamente en <TIMESTAMP>.

Objetivo:
Identificar hipótesis de causa raíz y proponer un proceso de diagnóstico.

Restricciones:
- No asumir información que no esté disponible.
- No proponer cambios destructivos.
- Separar hipótesis de hechos observables.
- Priorizar acciones reversibles y seguras.

Salida:
1. Interpretación del error.
2. Hipótesis ordenadas por probabilidad.
3. Evidencia que necesito para confirmar/descartar cada una.
4. Comandos/consultas de diagnóstico.
5. Riesgos de cada posible solución.
6. Solución recomendada y cómo validarla antes de producción.
```

### Validación

La IA es un acelerador de hipótesis, no una fuente de verdad.

Validaría contra:

-   documentación AWS;
-   métricas reales;
-   logs;
-   código;
-   configuración;
-   reproducción en entorno no productivo;
-   tests.

Nunca aplicaría directamente un comando destructivo sugerido por IA.

### Riesgos

-   alucinaciones;
-   recomendaciones incompatibles con la versión;
-   exposición de secretos/datos;
-   cambios excesivos;
-   falsa confianza;
-   ignorar restricciones arquitectónicas.

------------------------------------------------------------------------

# Bloque 3 --- Gobernanza, Seguridad y FinOps

## Caso 3.1 --- `admin-everything`

### Estrategia

No reemplazaría inmediatamente `Action: "*"`.

Primero observaría el uso real:

``` text
CloudTrail
   |
IAM Access Analyzer
   |
Observed actions
   |
Candidate policy
   |
Review
   |
Canary
   |
Least privilege
```

Mantendría temporalmente el rol actual mientras construyo una política
de transición.

Después:

1.  Identificar Lambdas que usan el rol.
2.  Separar roles por workload.
3.  Capturar actividad suficiente.
4.  Generar política candidata.
5.  Añadir recursos específicos.
6.  Probar en dev/qat.
7.  Promover gradualmente.
8.  Retirar el rol excesivamente permisivo.

IAM Access Analyzer puede generar políticas a partir de actividad
observada en CloudTrail.

### Guardrails

-   IAM Access Analyzer.
-   AWS Config.
-   SCP.
-   Permission Boundaries.
-   policy validation.
-   Terraform checks.
-   CI/CD con políticas como código.
-   detección de `Action:*` y `Resource:*`.

------------------------------------------------------------------------

## Caso 3.2 --- Secretos hardcodeados

### Inmediato

1.  Revocar/rotar credenciales.
2.  Identificar repositorios, commits y logs afectados.
3.  Invalidar credenciales comprometidas.
4.  Verificar CloudTrail.
5.  Eliminar secretos del código.
6.  Si fueron publicados en repositorios, tratar el secreto como
    comprometido aunque luego se elimine el commit.

### Diseño objetivo

``` text
Application
    |
    | IAM
    v
Secrets Manager
    |
    v
RDS
```

La aplicación obtiene el secreto en runtime.

Terraform no debería recibir el password como variable sensible desde el
pipeline si puede evitarse. Terraform crea/configura el secreto, pero el
valor debe gestionarse fuera del código.

### Rotación

Para RDS usaría Secrets Manager y, cuando el patrón sea compatible,
rotación de usuarios alternos para minimizar downtime.

``` text
User A active
     |
rotate
     v
User B updated
     |
validate
     v
B active
```

La aplicación debe tolerar renovación de conexiones.

------------------------------------------------------------------------

## Caso 3.3 --- FinOps

### Metodología

``` text
1. Cost Explorer
2. CUR / billing data
3. Cost allocation tags
4. AWS Organizations
5. Cost Categories
6. Trusted Advisor
7. Compute Optimizer
8. Rightsizing
9. Savings Plans / RI analysis
10. Unit economics
```

Primero identificaría **qué creció**, no qué servicio parece caro.

Analizaría:

``` text
Account
 -> Service
   -> Region
     -> Workload
       -> Environment
         -> Owner
```

### Savings Plan / RI

Antes de comprometer capital analizaría:

-   utilización histórica;
-   demanda mínima;
-   crecimiento;
-   estabilidad del workload;
-   duración esperada;
-   flexibilidad;
-   cobertura actual;
-   on-demand spend;
-   términos de pago.

No compraría basándome en el pico.

### Cinco palancas

  -----------------------------------------------------------------------
  Palanca                 Cuándo aplica           Riesgo
  ----------------------- ----------------------- -----------------------
  Rightsizing             recursos                degradación
                          sobredimensionados      

  Scheduling              dev/qat no 24x7         olvidar apagar

  Storage lifecycle       datos fríos             recuperación más lenta

  Savings Plans           consumo estable         sobrecompromiso

  Arquitectura            cargas variables        cambios de diseño
  serverless/managed                              
  -----------------------------------------------------------------------

Una reducción real debe verse en el **costo total de AWS** o en el costo
unitario.

Mover un costo de EC2 a otra cuenta no es ahorro.

------------------------------------------------------------------------

# Bloque 4 --- Evolución y Deuda Técnica

## Caso 4.1 --- Migración de generación de DB

### Plan

``` text
Assessment
   |
Compatibility
   |
Benchmark
   |
Non-prod migration
   |
Production rehearsal
   |
Backup
   |
Change window
   |
Migration
   |
Validation
   |
Monitor
```

Validaría:

-   engine/version;
-   extensiones;
-   drivers;
-   parámetros;
-   CPU;
-   memoria;
-   IOPS;
-   throughput;
-   conexiones;
-   latencia;
-   storage;
-   backups;
-   maintenance window;
-   compatibilidad de IaC.

### Ahorro real

Compararía mínimo:

``` text
$/month
$/transaction
p95 latency
CPU utilization
IOPS
throughput
error rate
```

No basta con que la instancia nueva tenga un precio menor.

### Rollback

Antes del cambio:

-   snapshot/backup validado;
-   versión anterior disponible;
-   procedimiento de restore;
-   rollback criteria;
-   ventana de cambio;
-   responsable de decisión.

------------------------------------------------------------------------

## Caso 4.2 --- ETL caro y con errores

No atacaría primero el síntoma "el job cuesta mucho".

Mediría:

``` text
Input GB
Output GB
Execution time
Retries
Failed records
Reprocessing
Compute utilization
$/run
$/GB processed
```

Puede que el costo sea causado por:

``` text
error
 -> retry
 -> reprocess
 -> more compute
 -> higher cost
```

### Estrategia

No apagaría el job productivo.

Haría:

1.  baseline;
2.  observabilidad;
3.  análisis de errores;
4.  prueba sobre muestra;
5.  optimización;
6.  ejecución paralela/canary;
7.  comparación;
8.  migración;
9.  rollback preparado.

------------------------------------------------------------------------

# Bloque 5 --- Ingeniería de Datos

## Caso 5.1 --- Data Lake

Propondría:

``` text
                    +----------------+
                    |      S3        |
                    | Parquet/Iceberg|
                    +-------+--------+
                            |
                            v
                    +---------------+
                    | Glue Catalog  |
                    +-------+-------+
                            |
             +--------------+--------------+
             |                             |
             v                             v
         Athena                      Redshift
      ad-hoc/query               BI/warehouse
```

### ¿Cuándo usar cada uno?

**Glue Catalog:** metadatos y catálogo.

**Glue Crawlers:** útiles para descubrir/escanear esquemas, pero no
necesariamente los usaría como mecanismo permanente si el esquema puede
gestionarse explícitamente.

**Athena:** consultas SQL serverless directamente sobre S3.

**Redshift:** workloads analíticos repetitivos, alta concurrencia,
modelos warehouse/BI y necesidades de rendimiento predecible.

### Optimización

-   Parquet en lugar de CSV/JSON.
-   compresión;
-   particionamiento por columnas de alta selectividad y baja
    cardinalidad razonable;
-   evitar particiones excesivas;
-   compactación de archivos pequeños;
-   Iceberg para tablas que requieran evolución/transaccionalidad;
-   limitar `SELECT *`;
-   workgroups/limites de consulta.

La regla práctica: **menos bytes leídos = menor costo y normalmente
menor latencia.**

### Gobierno

Aplicaría:

``` text
IAM
+
Lake Formation
+
S3 policies
+
KMS
+
CloudTrail
```

Separaría acceso por dominios/datos y aplicaría mínimo privilegio.

------------------------------------------------------------------------

## Caso 5.2 --- Lifecycle y retención

Ejemplo conceptual:

``` text
0 - 30d     -> S3 Standard
30 - 90d    -> Intelligent-Tiering / IA según patrón
90 - 365d   -> Glacier
> 365d      -> Deep Archive
```

Los tiempos reales dependen de:

-   frecuencia de lectura;
-   requisitos regulatorios;
-   tiempo de recuperación aceptable;
-   mínimo de permanencia;
-   costo de retrieval;
-   tamaño de objetos.

No movería todo automáticamente a Deep Archive solo porque sea barato.

### AWS Backup

Diseñaría:

``` text
Workload
   |
Backup Plan
   |
Daily/weekly/monthly
   |
Cross-account copy
   |
Cross-region copy
   |
Vault Lock
```

Para datos regulatorios utilizaría un vault protegido contra eliminación
según el nivel de inmutabilidad requerido.

El cumplimiento se valida también con **restore tests**, no solamente
comprobando que el backup "existe".

------------------------------------------------------------------------

# Bloque 6 --- Redes, Cloud WAN y DRP Cross-Region

## Caso 6.1 --- DX → VPN

La prioridad debe resolverse mediante **política de routing**, no
esperando que dos caminos iguales "se comporten bien".

``` text
                  +------------------+
                  |    Cloud WAN     |
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
          DX attachment             VPN attachment
          preferred                backup path
```

Usaría BGP y políticas de routing para que el camino DX sea preferido y
la VPN quede como backup.

Cloud WAN soporta routing policies sobre attachments BGP como Direct
Connect y Site-to-Site VPN.

Para detección rápida de caída de DX habilitaría BFD donde sea
soportado/configurable. En Direct Connect, BFD puede proporcionar
detección más rápida que depender solamente del hold timer BGP.

No prometería un RTO de "X segundos" sin probarlo: el tiempo real
depende de detección, retirada de rutas, propagación y convergencia.

### Segmentación

Crearía un segmento de conectividad híbrida:

``` text
prd
 |
share route
 |
hybrid/onprem
 |
DX primary
VPN backup
```

Si la política exige acceso explícito desde `prd`, `shrprd` y `epsprd`,
lo declararía en la política de Cloud WAN. No asumiría transitividad
implícita.

### Validación

Haría una prueba controlada:

1.  medir tráfico por DX;
2.  registrar rutas/BGP;
3.  provocar caída controlada del DX;
4.  observar retiro de rutas;
5.  verificar selección VPN;
6.  ejecutar transacciones end-to-end;
7.  medir packet loss;
8.  medir tiempo de convergencia;
9.  restaurar DX;
10. comprobar retorno sin flapping.

------------------------------------------------------------------------

## Caso 6.2 --- FortiGate e inspección

La regla debe ser **fail closed**, no fail open.

Cloud WAN service insertion permite:

-   `send-via` para tráfico east-west;
-   `send-to` para tráfico north-south.

El network function group debe tener attachments válidos. Si se define
una inserción sin attachment correspondiente, el tráfico puede terminar
siendo descartado.

### Arquitectura

``` text
VPC PRD
   |
Cloud WAN
   |
Inspection segment
   |
FortiGate HA
   |
Cloud WAN
   |
Destination
```

Para DR:

``` text
us-east-1                    us-west-2
FortiGate                    FortiGate
   |                            |
   +-------- Cloud WAN ---------+
```

Preferiría inspection VPC/appliances disponibles en ambas regiones
cuando el requisito sea mantener inspección durante failover regional.

### BGP peer configurado pero no Established

Un peer configurado no significa que exista forwarding seguro.

Si una parte del diseño espera dos caminos y uno está
`Idle/Active/never-established`, puede producir:

-   pérdida de rutas;
-   convergencia incompleta;
-   asimetría;
-   blackhole;
-   falsa percepción de HA.

Por eso la salud debe comprobar **BGP Established + rutas esperadas +
dataplane**, no solamente estado del appliance.

### BLACKHOLE

Diagnóstico:

``` text
Cloud WAN route table
       |
segment policy
       |
service insertion
       |
network function group
       |
FortiGate attachment
       |
BGP
       |
route advertisement
       |
next hop
       |
dataplane
```

Revisaría primero el último cambio de política, attachment, BGP, rutas
propagadas y estado del firewall.

------------------------------------------------------------------------

## Caso 6.2.1 --- Egreso a Internet

Default deny.

Permitiría únicamente puertos/protocolos necesarios.

Como baseline:

``` text
HTTPS 443  -> permitido
HTTP 80    -> solo si existe una necesidad explícita
DNS        -> preferiblemente hacia resolvers controlados
NTP        -> hacia servicio autorizado
```

No permitiría `0-65535` ni acceso directo desde workloads a Internet.

Además:

-   DNS filtering;
-   URL/category filtering si el firewall lo soporta;
-   logging;
-   egress allowlist para workloads críticos;
-   NAT centralizado/inspeccionado según arquitectura.

------------------------------------------------------------------------

## Caso 6.3 --- Virginia → Oregon

La prueba pide evitar TGW Peering y VPN inter-region.

Usaría **Cloud WAN como core network multi-region**. Cloud WAN propaga
rutas entre core network edges y permite políticas de segmentación por
región.

El segmento `prd` debe existir/estar habilitado en ambas regiones.

``` text
                Cloud WAN
          +-------------------+
          |                   |
       us-east-1          us-west-2
        PRD edge            PRD edge
          |                   |
       Primary             DRP
```

### Detección y redirección

La detección debe ser desde la aplicación/servicio y la red:

-   health checks;
-   CloudWatch;
-   AWS Health/EventBridge;
-   synthetic checks.

La redirección depende del tipo de endpoint:

-   Route 53 health checks/failover para DNS;
-   Global Accelerator si se necesita failover de entrada con IPs
    estáticas y rápida convergencia;
-   políticas Cloud WAN para routing interno.

No asumiría que "Cloud WAN detecta que una aplicación está caída" por sí
solo.

### dev/qat

No compartiría el segmento productivo con dev/qat.

El DRP regional debe tener recursos mínimos para PRD y no mantener
capacidad completa para ambientes no productivos.

------------------------------------------------------------------------

## Caso 6.4 --- Estrategia DRP por servicio

  ------------------------------------------------------------------------
  Servicio          Estrategia         RTO/RPO esperado  Costo
  ----------------- ------------------ ----------------- -----------------
  Aurora/RDS        warm standby /     bajo/minutos      medio/alto
                    replica                              
                    cross-region                         

  EKS               pilot light +      minutos           bajo/medio
                    IaC + imágenes                       

  EC2               AMI + IaC +        decenas de min    bajo
                    backups                              

  Lambda            artefacto + IaC en minutos           bajo
                    DR                                   

  DynamoDB          Global Tables si   muy bajo          alto
                    el negocio lo                        
                    exige                                

  S3                versioning +       minutos           medio
                    replication según                    
                    criticidad                           

  S3 no crítico     backup/lifecycle   horas             bajo
  ------------------------------------------------------------------------

No todo debe ser active-active. El patrón depende de RTO/RPO y costo.

### Base on-premise con réplicas AWS

La réplica de Oregon debe depender de conectividad independiente de
Virginia.

Preferiría:

``` text
On-prem
   |
DX
   |
Cloud WAN
   +---- us-east-1
   |
   +---- us-west-2
```

y VPN como respaldo según el diseño.

No diseñaría:

``` text
On-prem -> Virginia -> Oregon
```

porque convierte Virginia en dependencia del DRP.

### Onboarding automático de cuentas

Usaría una combinación de:

``` text
AWS Organizations
       |
Account vending / baseline
       |
Tags
       |
Cloud WAN attachment policy
       |
Segment assignment
       |
Network function group
       |
Inspection
       |
Hybrid connectivity
```

Cloud WAN attachment policies permiten asociar attachments a segmentos
mediante tags/políticas.

La cuenta nueva debe pasar por un pipeline de onboarding, no por
configuración manual.

### Prueba DRP

-   tabletop exercise;
-   restore test;
-   failover de aplicación;
-   failover de red;
-   synthetic transactions;
-   validación de datos;
-   medición real de RTO/RPO;
-   evidencias almacenadas;
-   postmortem.

------------------------------------------------------------------------

# Bloque 7 --- Escenario Integrador

## Arquitectura propuesta

``` text
                         Internet
                            |
                       Route 53
                            |
                       CloudFront
                            |
                          WAF
                            |
                         ALB/API
                            |
                 +----------+----------+
                 |                     |
              Lambda                  EKS
                 |                     |
                 +----------+----------+
                            |
                       Data layer
                     /            \
                   RDS           DynamoDB
                     |
                     v
                 Cloud WAN
                     |
              Inspection VPC
                FortiGate
                     |
          +----------+----------+
          |                     |
       Internet             On-prem Core
                               |
                         Direct Connect
                               |
                         VPN backup
```

### Compute

Para el nuevo producto empezaría con **Lambda + API Gateway** si el
workload es event-driven/HTTP y tiene picos variables.

EKS/Fargate solamente donde exista una necesidad real de:

-   runtime específico;
-   workloads largos;
-   networking/control especializado;
-   patrones que Lambda no cubra bien.

Esto controla costos y reduce operación.

### Datos

RDS/Aurora para el modelo relacional del producto.

DynamoDB si existen patrones de acceso clave-valor de alta escala.

S3 para objetos/data lake.

No convertiría todo en una base de datos distribuida solamente por
anticipar picos.

------------------------------------------------------------------------

## Red

La nueva cuenta:

``` text
New AWS Account
      |
VPC attachment
      |
Cloud WAN
      |
PRD segment
      |
Inspection
      |
Hybrid segment
      |
DX / VPN
      |
On-prem core
```

El acceso a Cloud WAN debe ser consecuencia de una política/tag y del
baseline de la cuenta.

------------------------------------------------------------------------

## IaC + CI/CD

``` text
Git
 |
PR
 |
Security / Quality
 |
Terraform plan
 |
Build artifact
 |
OIDC
 |
Hub role
 |
Cross-account deploy role
 |
DEV
 |
QAT
 |
Approval
 |
PRD
```

Separaría repositorios:

``` text
product-app
terraform-product
terraform-modules
cloudwan-policy
```

No pondría todo en un único repositorio gigante.

------------------------------------------------------------------------

## Seguridad

### IAM

-   roles específicos por workload;
-   no usuarios con access keys;
-   OIDC;
-   least privilege;
-   SCP;
-   permission boundaries;
-   separación de roles.

### KMS

Claves administradas por dominio/cuenta según necesidad:

``` text
S3
RDS
Secrets Manager
CloudWatch Logs
Backups
```

Evitaría crear una KMS key diferente por cada objeto sin justificación.

### WAF

Rulesets administrados + reglas específicas de aplicación.

Rate limiting para endpoints sensibles.

### Secrets

Secrets Manager + IAM runtime.

No secrets en:

-   Git;
-   Terraform `.tf`;
-   pipeline variables no protegidas;
-   logs.

### Network inspection

Service insertion con `send-via` / `send-to`.

Fail closed para rutas que requieran inspección.

------------------------------------------------------------------------

# Observabilidad desde día 1

## Application

-   request rate;
-   p50/p95/p99;
-   error rate;
-   timeout;
-   saturation;
-   correlation ID.

## ALB

-   target response time;
-   4xx/5xx;
-   unhealthy hosts;
-   request count.

## RDS

-   CPU;
-   connections;
-   IOPS;
-   storage;
-   latency;
-   locks;
-   replication lag.

## Cloud WAN

-   attachment status;
-   route propagation;
-   routing policy;
-   segment state.

## BGP/DX/VPN

-   BGP session state;
-   routes;
-   packet loss;
-   latency;
-   tunnel state;
-   DX interface state.

## FortiGate

-   HA state;
-   BGP peers;
-   throughput;
-   sessions;
-   drops;
-   CPU/memory;
-   packet loss.

------------------------------------------------------------------------

# FinOps desde el diseño

Definiría:

``` text
CostCenter
Application
Environment
Owner
Product
DataClassification
ManagedBy
```

y revisaría mensualmente:

1.  top cost drivers;
2.  variación mensual;
3.  costo por ambiente;
4.  idle resources;
5.  NAT/DX/data transfer;
6.  storage growth;
7.  Savings Plan coverage;
8.  RDS/EC2 utilization;
9.  costo unitario por transacción;
10. forecast.

------------------------------------------------------------------------

# Preguntas de profundización

## ¿Qué se rompe primero si el tráfico se duplica?

No asumiría que será EC2.

Revisaría:

``` text
WAF
CloudFront
ALB
Lambda concurrency
API Gateway throttling
RDS connections
NAT throughput
FortiGate sessions
DX bandwidth
external dependencies
```

Prevención:

-   load testing;
-   quotas revisadas;
-   autoscaling;
-   reserved concurrency;
-   connection pooling;
-   caching;
-   rate limiting;
-   capacity planning.

------------------------------------------------------------------------

## ¿NAT Gateway empieza a costar demasiado?

Primero mediría si el costo viene de:

-   cantidad de gateways;
-   horas;
-   GB procesados;
-   tráfico innecesario;
-   tráfico hacia AWS services que podría usar VPC endpoints.

No eliminaría NAT sin entender el flujo.

Para tráfico hacia servicios AWS evaluaría **Gateway/Interface VPC
Endpoints** cuando sea económicamente justificable.

------------------------------------------------------------------------

## ¿DX falla?

``` text
BFD/BGP detects failure
        |
DX routes withdrawn
        |
VPN becomes preferred
        |
Cloud WAN converges
        |
application continues
```

El tiempo debe medirse con un test, no prometerse.

------------------------------------------------------------------------

## ¿Cómo hereda una cuenta nueva todo automáticamente?

``` text
AWS Account created
       |
Account baseline
       |
Tags / IAM / logging
       |
VPC standard
       |
Cloud WAN attachment
       |
Attachment policy
       |
Segment
       |
Inspection
       |
Hybrid
       |
DRP
```

Esto debe ser un producto interno de plataforma, no una lista de pasos
manuales.

------------------------------------------------------------------------

## Cambio productivo viernes 5 PM

Mi primera opción sería **no hacerlo** si no existe una razón
operacional.

Si es inevitable:

1.  cambio pequeño;
2.  peer review;
3.  plan Terraform;
4.  backup/snapshot;
5.  change approval;
6.  ventana definida;
7.  rollback probado;
8.  monitoreo activo;
9.  canary/progressive rollout;
10. validación;
11. cierre documentado.

No aprobaría un cambio solamente porque "el cambio es sencillo".

------------------------------------------------------------------------

# Falla regional Virginia --- secuencia objetivo

Ejemplo con un objetivo de RTO de \~30 minutos para el producto
completo:

``` text
T+00
Falla de Virginia

T+00-02
Synthetic checks / health signals detectan degradación

T+02-05
Se confirma que no es un incidente aislado de aplicación

T+05
Route 53 / Global Accelerator comienza failover de entrada

T+05-10
Cloud WAN mantiene/propaga rutas hacia Oregon

T+05-10
Oregon utiliza compute y datos DRP

T+10-20
Validación de servicios, conectividad y transacciones

T+20-30
Servicio declarado operativo bajo DRP
```

El valor real debe validarse mediante un ejercicio DRP. El objetivo no
es decir "30 minutos" porque suena bien, sino demostrarlo con
mediciones.

------------------------------------------------------------------------

# Diagramas recomendados para Draw.io

## Diagrama 1 --- Landing Zone + Cloud WAN

Componentes:

``` text
AWS Organization
├── Hub / Network
├── Dev
├── QAT
├── PRD
└── DRP

Cloud WAN
├── Dev segment
├── QAT segment
├── PRD segment
├── Shared services
├── Hybrid
└── Inspection
```

## Diagrama 2 --- CI/CD

``` text
Developer
  -> Git
  -> PR
  -> Security gates
  -> Build
  -> Artifact
  -> Azure DevOps
  -> OIDC Hub
  -> Cross-account
  -> DEV
  -> QAT
  -> Approval
  -> PRD
```

## Diagrama 3 --- Hybrid connectivity

``` text
On-prem
   |
+--+----------------+
|                   |
DX                VPN
| Primary          | Backup
+--------+----------+
         |
      Cloud WAN
         |
   Hybrid segment
```

## Diagrama 4 --- Inspection

``` text
PRD
 |
Cloud WAN
 |
Inspection VPC
 |
FortiGate HA
 |
Cloud WAN
 |
Destination
```

## Diagrama 5 --- Regional DR

``` text
              Cloud WAN
            /           \
       us-east-1       us-west-2
        Primary            DRP
       PRD VPC            PRD VPC
          |                  |
      FortiGate          FortiGate
          |                  |
        Data             Replicated data
```

## Diagrama 6 --- Escenario integrador

Usaría un diagrama de máximo 3 niveles:

``` text
Users
 |
Edge
 |
Application
 |
Cloud WAN / Security
 |
Data / On-prem
```

No intentaría representar cada subnet, route table o security group en
el diagrama ejecutivo. Esos detalles van en diagramas específicos.

------------------------------------------------------------------------

# Decisiones y trade-offs

  ---------------------------------------------------------------------------
  Decisión                Motivo                      Trade-off
  ----------------------- --------------------------- -----------------------
  Multi-account           aislamiento y seguridad     mayor gobierno

  Cloud WAN               operación                   costo y complejidad
                          multi-región/segmentación   

  Lambda para carga       elasticidad y menor         límites/runtime
  variable                operación                   

  RDS/Aurora              modelo relacional           dependencia regional

  Terraform modules       estandarización             gobierno de versiones

  OIDC                    elimina secrets estáticos   configuración inicial

  Fail closed             evita bypass de inspección  posible
                                                      indisponibilidad
                                                      durante falla

  Warm standby            buen equilibrio DR/costo    costo continuo

  Pilot light             bajo costo                  RTO mayor

  Active-active           RTO mínimo                  alta complejidad/costo
  ---------------------------------------------------------------------------

------------------------------------------------------------------------

# Criterios de validación antes de producción

Para cualquier cambio crítico aplicaría:

``` text
Security
   +
Functional
   +
Performance
   +
Observability
   +
Rollback
   +
Cost impact
   =
Production readiness
```

Una implementación no está terminada cuando "funciona"; está terminada
cuando puedo **demostrar que funciona, detectar cuándo deja de funcionar
y volver atrás de forma controlada**.

------------------------------------------------------------------------

# Prompts utilizados con IA

La prueba explícitamente permite y espera uso de IA. Estos son los
prompts que utilizaría para acelerar análisis, no para delegar la
decisión técnica.

Ver `PROMPTS.md`.

------------------------------------------------------------------------

# Referencias técnicas

La resolución toma como base el contexto de la prueba y contrasta
aspectos sensibles con documentación oficial de AWS y HashiCorp,
especialmente:

-   AWS Cloud WAN y segment actions.
-   Cloud WAN service insertion.
-   Direct Connect + Cloud WAN.
-   Site-to-Site VPN/BGP.
-   IAM Access Analyzer.
-   Secrets Manager rotation.
-   AWS Backup/Vault Lock.
-   Terraform modules/version constraints.

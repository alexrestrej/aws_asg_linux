# Plantillas AWS para infraestructura Linux

Este proyecto contiene plantillas AWS CloudFormation para componentes de una
plataforma Linux, configuraciones de parámetros y pipelines Azure Pipelines.
Los datos identificables del entorno original se sustituyeron por marcadores
genéricos. Antes de desplegar, complete los marcadores y revise la configuración
para la cuenta y el entorno de destino.

## Estructura

| Ruta | Contenido |
| --- | --- |
| `cloudformation/` | Plantillas CloudFormation para ALB, ALB externo, ASG, CloudWatch, DMS, EC2, IAM, KMS, NLB, RDS, migración y réplica de RDS, S3, SFTP, grupos de seguridad, Secrets Manager y WAF. |
| `config/` | Archivos JSON de parámetros de despliegue. |
| `chaos/` | Definiciones de experimentos de AWS Fault Injection Service y pipeline para publicarlas como artefactos. |
| `azure-pipeline.yaml` | Pipeline manual que despliega un recurso seleccionado sucesivamente en desarrollo, QA y producción. |
| `chaos/azure-pipeline.yaml` | Pipeline manual que publica los experimentos locales. |

No hay un archivo `config/params-rdsref.json`; por eso `rdsref` no está entre
las opciones del pipeline. `params-rds.json` no se asume compatible con esa
plantilla. Revise y cree un archivo de parámetros específico antes de añadir
ese recurso al pipeline.

`template-iam.yaml` también requiere revisión antes de usarse: los recursos del
rol de EC2 están comentados, pero la política restante hace referencia a ese
rol. Por ello, IAM no está entre las opciones desplegables del pipeline.

## Pipeline de infraestructura

El pipeline `azure-pipeline.yaml` no tiene ejecución automática (`trigger:
none`). Al iniciarlo manualmente, seleccione un recurso CloudFormation. El
pipeline despliega ese recurso en las tres etapas secuenciales:

1. `dev`
2. `qa`
3. `pdn` (producción)

Opciones disponibles: `s3`, `sm`, `sg`, `alb`, `asg`, `kms`, `rds`, `rdsmig`,
`cw`, `waf`, `albE`, `dms`, `rdsrep`, `ec2`, `nlb` y `sftp`. Cada opción usa
la plantilla `cloudformation/template-<recurso>.yaml` y su correspondiente
`config/params-<recurso>.json`. El pipeline despliega un recurso por ejecución;
no organiza automáticamente el orden de dependencias entre stacks.

Configura en Azure Pipelines las variables:

- `appCode` y `projectName`.
- Para cada ambiente, `AWS_ACCESS_KEY_ID_<AMBIENTE>`,
  `AWS_SECRET_ACCESS_KEY_<AMBIENTE>`, `AWS_SESSION_TOKEN_<AMBIENTE>` y
  `AWS_REGION_<AMBIENTE>`, usando los sufijos `DEV`, `QA` y `PROD`.

El pipeline sustituye `#{env}#`, `#{application-code}#` y
`#{project-name}#` en los JSON, pero no comprueba que los demás marcadores hayan
sido completados. No completa automáticamente los marcadores que estén
directamente en las plantillas CloudFormation.

## Preparación de los parámetros y las plantillas

Antes de desplegar:

1. Completa los valores `#{...}#` de `config/params-*.json` con valores adecuados
   para la aplicación y para cada ambiente.
2. Sustituye todos los valores `YOUR_*` en `cloudformation/`. Incluyen CIDR,
   cuentas AWS, roles, nombres de políticas, ARN de servicios compartidos,
   buckets, dominios, certificados, parámetros de Systems Manager Parameter
   Store y alarmas de CloudWatch.
3. Revisa que los stacks importados o servicios externos existan y que sus
   identificadores coincidan: exportaciones CloudFormation, VPC y subredes,
   roles IAM, claves KMS, buckets, certificados ACM, parámetros de Systems
   Manager, tópicos SNS, funciones Lambda y servicios de protección.
4. Comprueba los permisos de la identidad AWS configurada en el pipeline. Los
   despliegues solicitan `CAPABILITY_IAM` y `CAPABILITY_NAMED_IAM`.
5. Confirma los valores y dependencias de un ambiente antes de permitir que la
   ejecución secuencial continúe al siguiente.

Los marcadores de red no son rangos válidos para operar. Sustitúyelos por CIDR
aprobados y de mínimo alcance. Las direcciones públicas de proveedores que
permanecen en las reglas del WAF deben revisarse contra la documentación actual
del proveedor antes de utilizarlas. El identificador `127311923021` que queda
en la política del bucket de logs corresponde al servicio de entrega de logs de
ALB de AWS; confírmalo con la región de destino.

Las alarmas y métricas preconfiguradas en `cloudformation/template-cw.yaml`
incluían referencias a recursos de una instalación previa. Sus nombres e
identificadores se generalizaron; actualiza esas referencias para que los
dashboards y widgets apunten a alarmas, instancias, certificados, balanceadores
y grupos de destino reales.

## Experimentos de caos

`chaos/azure-pipeline.yaml` publica el contenido local de `chaos/` como artefacto
y no ejecuta experimentos ni depende de otro repositorio. Los experimentos
incluyen acciones que pueden terminar instancias EC2, cambiar subredes de
clusters o forzar failover/reinicio de bases de datos. Revisa los objetivos,
permisos, duración y plan de recuperación; ejecútalos únicamente en cuentas
autorizadas y con una ventana aprobada.

Los archivos de experimentos contienen marcadores propios y algunos valores
numéricos sin resolver, por lo que no son JSON ejecutable hasta completar esos
valores. La canalización solo los publica; no los sustituye ni los valida.

## Despliegue manual

Después de completar los marcadores y las dependencias, un stack puede
desplegarse con AWS CLI, por ejemplo:

```bash
aws cloudformation create-stack \
  --template-file cloudformation/template-iam.yaml \
  --stack-name <codigo>-<proyecto>-dev-iam \
  --parameters file://config/params-iam.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <region>
```

Adapta el comando al recurso y al estado del stack. Despliega primero los stacks
que exportan valores consumidos por otros recursos. El ejemplo crea un stack
nuevo; usa el procedimiento de actualización adecuado si el stack ya existe.

## Alcance

Los pipelines mantienen un flujo básico de publicación/despliegue en tres
ambientes. No incluyen validaciones, análisis, pruebas, aprobaciones ni
integraciones con repositorios o plantillas externas. No se ha ejecutado ningún
despliegue AWS como parte de la generalización de este proyecto.

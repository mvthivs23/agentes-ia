# Lección 09 — Cloud Architecture Fundamentals

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Cada breach moderno termina en una consola cloud. El perímetro que te enseñaron a atacar hace una década — un bastion host endurecido en un rack de colocación — fue reemplazado por un documento de policy de IAM, una route table de VPC y un endpoint de metadata en `169.254.169.254`. Cuando comprometes una aplicación moderna, el botín ya no son hashes locales de `/etc/shadow`; es un conjunto de credenciales STS de corta vida, una key JSON de service account o un token de managed identity que abre todo el tenant.

Esta lección mapea el terreno. Recorre los tres grandes proveedores lado a lado, instala las CLIs en Kali 2026, profundiza en IAM de cada proveedor, cubre los primitivos de red (VPC, VNet, subnets, endpoints), los servicios de metadata que convierten cualquier SSRF en un compromiso total, las misconfigurations de storage que filtran petabytes, la superficie de ataque serverless, las bases de datos managed, los orquestadores de containers, los stores de secrets y el logging que dejas atrás mientras enumeras. Cierra con un escenario de punta a punta — un SSRF sin authentication que termina con credenciales de base de datos en texto plano extraídas de un backup de S3 — y un conjunto de ejercicios que te obligan a reproducir cada etapa.

Al final vas a leer un diagrama de arquitectura cloud como ahora lees un scan de Nmap, y vas a saber exactamente qué comando de Kali mapea a qué primitivo cloud.

## 1. Los tres grandes proveedores lado a lado

Amazon Web Services (AWS), Google Cloud Platform (GCP) y Microsoft Azure controlan aproximadamente dos tercios del mercado global de cloud entre los tres. Cada uno nombra el mismo primitivo fundamental de forma distinta, y esa es una fuente principal de confusión cuando caes en un host comprometido y no puedes decir si las credenciales que encontraste te dan acceso a EC2, a Compute Engine o a una Azure Virtual Machine.

### 1.1 La tabla de equivalencias

| Concepto | AWS | GCP | Azure |
|---|---|---|---|
| Virtual machine | EC2 instance | Compute Engine VM | Virtual Machine |
| Object storage | S3 bucket | Cloud Storage bucket (GCS) | Blob Storage container |
| Block storage | EBS volume | Persistent Disk | Managed Disk |
| Identity for workloads | IAM Role | Service Account | Managed Identity |
| Identity for humans | IAM User | Google Account / Workspace | Entra ID (Azure AD) user |
| Virtual network | VPC | VPC | VNet |
| Subnet | Subnet | Subnet | Subnet |
| Function as a Service | Lambda | Cloud Functions | Azure Functions |
| Container orchestration | EKS | GKE | AKS |
| Managed containers | ECS / Fargate | Cloud Run | ACI / Container Apps |
| Managed relational DB | RDS / Aurora | Cloud SQL | Azure SQL |
| Managed NoSQL | DynamoDB | Firestore / Bigtable | Cosmos DB |
| Secrets store | Secrets Manager | Secret Manager | Key Vault |
| Configuration store | Parameter Store | Runtime Config | App Configuration |
| Audit log | CloudTrail | Cloud Audit Logs | Activity Log |
| Network flow log | VPC Flow Logs | VPC Flow Logs | NSG Flow Logs |
| Anomaly detection | GuardDuty | Security Command Center | Microsoft Defender for Cloud |
| Metadata endpoint | 169.254.169.254 (IMDS) | metadata.google.internal | 169.254.169.254 (IMDS) |
| Assume role primitive | `sts:AssumeRole` | `iam.serviceAccounts.getAccessToken` | `Get-AzAccessToken` on MI |

### 1.2 Lo que la equivalencia esconde

Los primitivos se ven uno a uno en el papel. En la práctica, los modelos de confianza difieren lo suficiente como para importarle a un atacante:

- AWS IAM es deny-by-default y aditivo; los roles los asumen los principals a través de una trust policy que nombra quién puede asumirlos. La cadena es explícita.
- GCP IAM está ligado a recursos (org, folder, project, resource) y hereda hacia abajo en la jerarquía. Un principal que es `roles/owner` a nivel de folder es dueño de cada project que el folder contenga alguna vez, para siempre, hasta que se desligue.
- Azure IAM (RBAC) es basado en scope como GCP, pero está montado encima de Entra ID (antes Azure AD), que maneja la identidad en sí. Dos directorios, un plano.

Entender estos tres modelos de confianza es la diferencia entre un reporte válido de privesc y un falso positivo.

## 2. Instalar y configurar las CLIs en Kali 2026

Kali 2026 trae Python 3.12 y repositorios de paquetes actuales. Las tres CLIs de proveedor se instalan por la distribución o por los instaladores del vendor. En la práctica mantienes las tres configuradas en tu caja de engagement porque rara vez sabes de antemano qué proveedor usa un cliente, y los entornos multi-cloud son la norma.

### 2.1 AWS CLI v2

```bash
# Instala desde el repositorio de la distribución:
sudo apt update
sudo apt install -y awscli
aws --version
# aws-cli/2.17.x Python/3.12.x Linux/6.10-kali

# O instala directo desde Amazon (siempre la última):
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install --update

# Configura un profile con nombre — nunca uses el profile default en un engagement real:
aws configure --profile client-prod
# AWS Access Key ID:     AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name:   us-east-1
# Default output format: json
```

Las credenciales caen en `~/.aws/credentials`:

```ini
[client-prod]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Y la config en `~/.aws/config`:

```ini
[profile client-prod]
region = us-east-1
output = json
```

Invoca un profile con `--profile client-prod` en cada comando, o exporta `AWS_PROFILE=client-prod` para la shell.

### 2.2 Google Cloud SDK

```bash
# Agrega el repositorio apt del Cloud SDK:
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | \
  sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

sudo apt update
sudo apt install -y google-cloud-sdk

gcloud version
# Google Cloud SDK 490.x
# alpha 2026.x
# beta 2026.x

# Autentica con una key de service account (típico de un pentester que encontró una key JSON):
gcloud auth activate-service-account --key-file=/loot/sa-key.json

# O define una configuration de autorización con nombre:
gcloud config configurations create client-prod
gcloud config set account pentester@client.iam.gserviceaccount.com
gcloud config set project client-prod-01
gcloud config configurations activate client-prod
```

### 2.3 Azure CLI

```bash
# Instala con el script de instalación de Microsoft:
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az version
# {
#   "azure-cli": "2.66.x",
#   "azure-cli-core": "2.66.x"
# }

# Login interactivo por device-code (abre un navegador):
az login --use-device-code

# O login de service principal cuando tienes un client secret:
az login --service-principal \
  -u 00000000-0000-0000-0000-000000000000 \
  -p 'sp-client-secret' \
  --tenant 11111111-1111-1111-1111-111111111111

# Cambia entre subscriptions:
az account list -o table
az account set --subscription "Client Production"
```

### 2.4 Mantén los profiles aislados

Nunca mezcles las credenciales de un cliente con tus cuentas cloud personales. Un solo comando perdido contra el profile equivocado puede facturarle al cliente miles o disparar su detección. Usa aliases de shell:

```bash
alias awsp='AWS_PROFILE=client-prod aws'
alias gcp-p='CLOUDSDK_ACTIVE_CONFIG_NAME=client-prod gcloud'
alias az-p='az --subscription "Client Production"'
```

## 3. AWS IAM en profundidad

IAM es todo el modelo de seguridad de AWS. Cada llamada de API pasa por él, cada recurso está protegido por él, y cada privilege escalation en AWS es una misconfiguration de policy de IAM.

### 3.1 Los cuatro principals

| Principal | Descripción | Credencial |
|---|---|---|
| IAM User | Identidad humana de larga vida dentro de una account | Access key + secret |
| IAM Group | Un contenedor de users; las policies se adjuntan al group | N/A — hereda por membership |
| IAM Role | Identidad asumible para workloads o cross-account | Credenciales STS temporales |
| Federated user | Identidad de un IdP externo (SAML, OIDC) | Credenciales STS temporales |

### 3.2 Inline vs managed policies

- Las **inline policies** están embebidas directo en un solo user, group o role. Son invisibles salvo que las listes de forma explícita por principal.
- Las **managed policies** son objetos independientes y se pueden adjuntar a muchos principals. Las AWS-managed policies (`arn:aws:iam::aws:policy/...`) las cura AWS; las customer-managed policies viven bajo el ARN propio de la account.

Cuando auditas una account, enumera siempre las dos. Los atacantes esconden privilegios en inline policies porque una adición de una sola línea a una inline policy se escapa de la mayoría de los checks por defecto.

### 3.3 Trust policy vs permissions policy

Un role tiene dos documentos de policy:

- La **trust policy** responde quién puede asumir este role. Aparece en el role mismo y usa la action `sts:AssumeRole` en una policy donde el elemento `Principal` nombra accounts, users, roles o services.
- La **permissions policy** responde qué puede hacer el role una vez asumido. Se ve como cualquier otra policy de IAM.

Un finding típico de pentester es una trust policy que nombra `"Principal": {"AWS": "*"}` — cualquiera en cualquier account de AWS puede asumir el role. Combinada con una permissions policy que otorga `s3:*`, eso es un data breach completo expuesto a internet público.

### 3.4 Condition keys

Las condition keys acotan cuándo aplica una policy. Las importantes para un red teamer:

- `aws:SourceIp` — restringe por IP de origen. Se bypasea de forma trivial si caes dentro de la VPC del target.
- `aws:MultiFactorAuthPresent` — exige MFA en la session que llama.
- `aws:PrincipalOrgID` — restringe a principals de una AWS Organization específica.
- `aws:SourceVpce` — restringe a llamadas que llegan por un VPC endpoint específico.

La ausencia de estas en una action sensible es en sí el finding.

### 3.5 Primeros comandos después de un compromiso

En el momento en que caes con credenciales, corres:

```bash
# ¿Quién soy?
aws sts get-caller-identity --profile loot
# {
#   "UserId": "AIDAEXAMPLEUSERID",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/backup-service"
# }

# ¿Qué puedo hacer? — prueba las APIs autoritativas de la account:
aws iam list-account-aliases --profile loot
aws iam list-users --profile loot
aws iam list-roles --profile loot
aws iam list-attached-user-policies --user-name backup-service --profile loot
aws iam list-user-policies --user-name backup-service --profile loot

# Enumera las policies adjuntas a tu propia identidad:
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/backup-service \
  --action-names s3:GetObject ec2:RunInstances iam:PassRole \
  --profile loot
```

La llamada `simulate-principal-policy` es la forma más limpia de chequear permisos individuales sin generar un `AccessDenied` sobre la action real del target.

## 4. Caminos de privilege escalation en IAM

Rhino Security Labs catalogó los caminos clásicos de privesc en AWS hace años; siguen funcionando porque explotan composiciones de permisos legítimos y no bugs de software.

### 4.1 `iam:PassRole` + `ec2:RunInstances`

Si tu principal puede pasar cualquier role a una instancia EC2 nueva y puede lanzar una instancia, puede adjuntar un instance profile administrativo y leer las credenciales de ese role en el IMDS. Los comandos y el script de user-data no se incluyen.

### 4.2 `iam:CreateAccessKey` sobre un user privilegiado

Si puedes crear access keys sobre un user con más privilegio que el tuyo, esa key nueva es el user privilegiado. Los users tienen como máximo dos access keys activas: si ya hay dos, la llamada falla. Los comandos no se incluyen.

### 4.3 Cadenas de `sts:AssumeRole`

Las cadenas cross-account o cross-role son el camino menos monitoreado. Si puedes asumir el role A, el role A puede asumir el role B en otra account, y el role B es poderoso, cada eslabón se ve legítimo por separado. Los comandos no se incluyen.

Catálogo de permisos que componen una escalada:

| Vector | Permiso requerido |
|---|---|
| `iam:PassRole` + `ec2:RunInstances` | Ambos |
| `iam:CreateAccessKey` | Sobre cualquier user privilegiado |
| `iam:CreateLoginProfile` | Sobre cualquier user privilegiado sin acceso a consola |
| `iam:UpdateLoginProfile` | Resetear una password de consola en un user privilegiado |
| `iam:AttachUserPolicy` | Adjuntarte admin a ti mismo |
| `iam:PutUserPolicy` | Admin inline sobre ti mismo |
| `iam:CreatePolicyVersion` + `iam:SetDefaultPolicyVersion` | Reescribir una customer-managed policy |
| `lambda:UpdateFunctionCode` + role adjunto | Sobrescribir una function que corre como admin |
| `codebuild:UpdateProject` + role adjunto | Lo mismo, en CodeBuild |

## 5. Enumeración desde Kali — enumerate-iam y pacu

Dos tools dominan la enumeración de AWS del lado atacante: enumerate-iam, que prueba qué acciones de una credencial tienen éxito, y pacu, el framework modular de Rhino Security Labs. Las dos dejan ráfagas de `AccessDenied` en CloudTrail. La instalación, las keys de ejemplo y los módulos de explotación no se incluyen. Cada acción de Pacu queda en la base SQLite de la session bajo `~/.local/share/pacu/`.

### 5.1 Impacto en la detección

Las dos tools disparan ráfagas de `AccessDenied` en CloudTrail. Si GuardDuty está habilitado, van a dispararse estos findings:

- `Recon:IAMUser/UserPermissions` — muchos checks fallidos de permisos de IAM.
- `Recon:IAMUser/MaliciousIPCaller` — enumeración desde un exit de Tor o una IP de scanner conocida.
- `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` — credenciales de un role de EC2 usadas desde fuera de AWS.

## 6. GCP IAM

GCP IAM es jerárquico por recurso. Los bindings aplican a nivel de Organization, Folder, Project o recurso individual, y heredan hacia abajo.

### 6.1 La jerarquía

```text
Organization (client.io)
   |
   +-- Folder: Engineering
   |     |
   |     +-- Project: eng-dev
   |     +-- Project: eng-prod
   |
   +-- Folder: Data
         |
         +-- Project: analytics-prod
```

Un binding de `roles/owner` en el nodo Organization es total: es dueño de cada folder, cada project y cada recurso, para siempre. Los atacantes cazan el binding de mayor scope que puedan lograr porque todo lo de abajo hereda.

### 6.2 Roles primitive, predefined y custom

| Tipo de role | Ejemplo | Notas |
|---|---|---|
| Primitive (legacy) | `roles/owner`, `roles/editor`, `roles/viewer` | Groseramente amplios; a nivel de project. |
| Predefined | `roles/compute.admin`, `roles/storage.objectViewer` | Curados por Google; acotados a un service. |
| Custom | `projects/eng-prod/roles/customBackupReader` | Definidos por el cliente con permisos explícitos. |

`roles/editor` tiene un nombre engañoso — otorga acceso de escritura a casi cada recurso de un project y es un target común de privesc.

### 6.3 Service accounts y archivos de key

Una service account es una identidad para workloads. Tiene su propia dirección de email de la forma `sa-name@project-id.iam.gserviceaccount.com`, sus propios bindings de IAM, y puede tener hasta diez keys administradas por el usuario. Un archivo JSON de key de service account — filtrado a un repo de Git, embebido en una app mobile o dejado en un bastion host — es una credencial de larga vida, sin rotar.

```bash
# Enumera service accounts de un project:
gcloud iam service-accounts list --project client-prod-01

# Enumera bindings de IAM de un project:
gcloud projects get-iam-policy client-prod-01

# Enumera bindings de un folder:
gcloud resource-manager folders get-iam-policy 987654321000

# Enumera bindings de la org:
gcloud organizations get-iam-policy 123456789000

# Impersona una service account sobre la que tienes bindings:
gcloud auth print-access-token \
  --impersonate-service-account=admin-sa@client-prod-01.iam.gserviceaccount.com
```

El permiso `iam.serviceAccounts.getAccessToken` es el equivalente en GCP de `sts:AssumeRole`. Si puedes llamarlo contra una SA más privilegiada, tienes el poder de esa SA.

### 6.4 Vectores comunes de privesc en GCP

| Permiso | Escalada |
|---|---|
| `iam.serviceAccounts.getAccessToken` | Acuñar tokens de una SA más privilegiada. |
| `iam.serviceAccounts.actAs` + `compute.instances.create` | Adjuntar una SA privilegiada a una VM nueva. |
| `iam.serviceAccountKeys.create` | Acuñar una key de larga vida para una SA privilegiada. |
| `iam.roles.update` | Agregar permisos a un custom role que tienes. |
| `deploymentmanager.deployments.create` | Desplegar usando la SA de Deployment Manager, que a menudo es `roles/editor`. |
| `cloudfunctions.functions.update` + SA adjunta | Sobrescribir una function que corre como una SA privilegiada. |

## 7. Azure IAM

Azure separa la identidad (Entra ID, antes Azure AD) de la autorización (RBAC sobre recursos de Azure).

### 7.1 Jerarquía de scope

```text
Management Group (root)
   |
   +-- Subscription: Client Production
         |
         +-- Resource Group: rg-web-prod
         |     |
         |     +-- Storage Account: cldstoreprod01
         |     +-- Virtual Machine: web-01
         |
         +-- Resource Group: rg-data-prod
```

Las asignaciones de role en cualquier nivel heredan hacia abajo. Owner en una subscription es dueño de cada resource group y cada recurso para siempre.

### 7.2 Roles built-in vs custom

- **Built-in:** Owner, Contributor, Reader, User Access Administrator, más roles específicos de servicio como Storage Blob Data Contributor.
- **Custom:** definición JSON con `actions`, `notActions`, `dataActions` y `assignableScopes`.

Contributor es poderoso — puede crear, actualizar y borrar cualquier cosa, pero de forma crucial no puede modificar RBAC. User Access Administrator es el role que otorga poder de RBAC. Contributor + User Access Administrator = Owner de facto.

### 7.3 Managed identities

Una managed identity es el primitivo de Azure equivalente al role de IAM para workloads. Dos sabores:

- **System-assigned** — ligada al tiempo de vida de un recurso (VM, Function App, Container App). Se borra cuando se borra el recurso.
- **User-assigned** — una identidad independiente que se puede adjuntar a muchos recursos.

Las dos se representan en Entra ID como service principals y reciben tokens desde IMDS.

### 7.4 Privileged Identity Management (PIM)

PIM hace que los roles sean elegibles en lugar de estar siempre activos. Un admin activa el role por un tiempo acotado (típicamente 1–8 horas), a menudo con MFA y un workflow de aprobación. En un engagement maduro vas a ver casi cada role de alto privilegio marcado solo como elegible; busca asignaciones de role permanentes como finding.

### 7.5 Enumeración desde Kali

```bash
# ¿Qué identidad soy?
az account show

# Lista cada subscription que puedo ver:
az account list -o table

# Vuelca las asignaciones de role de la subscription actual:
az role assignment list --all -o table

# Vuelca las asignaciones de role de un principal específico:
az role assignment list --assignee myprincipal@client.onmicrosoft.com --all

# Lista definiciones de custom roles:
az role definition list --custom-role-only true -o table

# Lista managed identities:
az identity list -o table

# Para una enumeración más profunda, usa ROADrecon o AzureHound (los dos en el skill pentest-cloud):
sudo apt install -y bloodhound
# AzureHound recolecta tanto AAD como Azure RBAC y alimenta a BloodHound.
```

## 8. Fundamentos de VPC / VNet

Las redes cloud son software-defined. Una VPC (AWS/GCP) o una VNet (Azure) es un rango de IP privada que tallas en subnets, enrutas con una route table y conectas al exterior a través de gateways.

### 8.1 Los objetos centrales

| Objeto | Propósito | Interés del pentester |
|---|---|---|
| VPC / VNet | El espacio de IP privada | Define el blast radius de un pivot interno. |
| Subnet | Una rebanada de la VPC ligada a una availability zone | La clasificación public vs private importa. |
| Route Table | Envía tráfico a un target (IGW, NAT, peering, VPC endpoint) | Modificarla puede redirigir tráfico a través de tu instancia. |
| Internet Gateway (IGW) | Egress e ingress a internet público | Solo las subnets public tienen una ruta default hacia él. |
| NAT Gateway | Internet solo de salida desde subnets private | Los hosts private igual hacen egress; útil para canales de beacon. |
| Security Group | Firewall L4 stateful ligado a una ENI | El egress `0.0.0.0/0` es común y permisivo. |
| NACL | ACL stateless a nivel de subnet | Se usa poco; se pasa por alto cuando está seteada. |
| VPC Peering | Ruta directa entre dos VPCs | Camino de movimiento lateral entre accounts. |
| VPC Endpoint / PrivateLink | Ruta privada hacia un servicio de AWS | Bypasea condiciones `aws:SourceIp`. |
| Transit Gateway | Conectividad hub-and-spoke | Un compromiso, muchas VPCs. |

### 8.2 Puntos de pivot

Una vez que hay shell en una VPC, el IMDS revela la subnet y el CIDR, y un VPC endpoint de S3 o DynamoDB abre buckets que desde internet parecen cerrados si la policy usa `aws:SourceVpce`. El barrido de la VPC no se incluye.

## 9. Servicios de metadata

Cada VM cloud expone un endpoint de metadata que el guest OS puede consultar para detalles de la instancia y, de forma crítica, para credenciales ligadas al role o a la managed identity de la instancia. Llegar a este endpoint desde un SSRF es la escalada cloud más común de internet.

### 9.1 AWS — IMDSv1 vs IMDSv2

IMDSv1 responde cualquier HTTP GET a `http://169.254.169.254/latest/meta-data/...`. No hay token de sesión. Un SSRF que solo hace GET devuelve el nombre del role y, en el path de ese role, las credenciales. Las peticiones no se incluyen.

IMDSv2 exige un PUT para obtener un session token de corta vida, enviado en `X-aws-ec2-metadata-token` en los GET siguientes. Derrota a la mayoría de los SSRF porque el cliente HTTP afectado típicamente no puede emitir un PUT ni setear headers arbitrarios.

IMDSv1 sigue permitido en muchas accounts de producción porque forzar IMDSv2 rompe SDKs legacy. En un assessment, `HttpTokens: optional` en una instancia significa que IMDSv1 sigue activo ahí.

### 9.2 GCP — header `Metadata-Flavor`

El servicio de metadata de Google está en `http://metadata.google.internal/` (o `169.254.169.254`) y exige el header `Metadata-Flavor: Google`. Históricamente también honraba `X-Google-Metadata-Request: True`. Cualquier SSRF que permita fijar un header arbitrario obtiene el access token de la service account adjunta, o un identity token JWT si se pide con `audience`. Las peticiones no se incluyen.

### 9.3 Azure — `Metadata: true`

Azure IMDS está en `http://169.254.169.254/metadata/` y exige `Metadata: true` y una versión de API. Con ese header, el path de OAuth de la managed identity devuelve un token para el resource que se pida (por ejemplo `management.azure.com`). La petición no se incluye. Cualquier SSRF que controle headers obtiene ese token.

### 9.4 Conclusión defensiva que deberías reportar

Cuando encuentras una instancia con IMDSv1 habilitado y demuestras SSRF, reporta tanto el SSRF como la configuración de IMDS. Forzar IMDSv2 es un fix de un comando y cada finding que archives debería incluirlo como remediación:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0abcdef1234567890 \
  --http-tokens required \
  --http-endpoint enabled
```

## 10. Misconfiguration de object storage

Los buckets de object storage públicos siguen siendo la mayor fuente individual de data breaches cloud. Cada proveedor ofrece una variante del mismo control, y los tres ahora salen seguros por defecto — pero los buckets históricos y las misconfigurations de los equipos de aplicación están por todas partes.

### 10.1 AWS S3

Un bucket de S3 es público si cualquiera de estas es verdadera:

- La ACL del bucket otorga `AllUsers` o `AuthenticatedUsers`.
- La bucket policy otorga `Principal: "*"` en actions de lectura.
- El acceso público no está bloqueado a nivel de account o de bucket (Block Public Access).

Un bucket es público si la ACL otorga `AllUsers` o `AuthenticatedUsers`, si la policy usa `Principal: "*"`, o si Block Public Access no está activo en la account o en el bucket. El fuente cita s3scanner para descubrirlo. El volcado de objetos no se incluye.

En una account propia, la auditoría es `get-bucket-acl`, `get-bucket-policy` y `get-public-access-block`. No listes ni copies el contenido.

### 10.2 GCP Cloud Storage

GCS soporta dos modos de control de acceso:

- **Fine-grained (legacy)** — ACLs por objeto, más IAM.
- **Uniform bucket-level access** — solo IAM; las ACLs por objeto están deshabilitadas.

El default es uniform. Los buckets públicos son los que tienen un binding de IAM de `allUsers` o `allAuthenticatedUsers` en `roles/storage.objectViewer` o superior. El fuente cita `gsutil ls -L` y gcp-scanner, que lista recursos públicos de una organización. El escaneo y el volcado no se incluyen.

### 10.3 Azure Blob Storage

Los containers de blobs de Azure tienen tres niveles de acceso público: off, blob (lectura pública de blobs individuales si conoces el nombre) y container (listado público). El fuente cita MicroBurst, que adivina nombres DNS contra `<name>.blob.core.windows.net` y `<name>.file.core.windows.net`. La enumeración no se incluye.

### 10.4 Leer primero los buckets correctos

Cuando tienes una lista de buckets, prioriza por nombre:

- `backup`, `backups`, `db-backup`, `mysql-dump`, `snapshot`
- `terraform`, `tfstate`, `state`, `infra`, `cicd`
- `secrets`, `creds`, `dev-envs`, `env`, `configs`
- `logs`, `access-logs`, `cloudtrail`, `waf`

Los archivos de state de Terraform (`terraform.tfstate`) son especialmente lucrativos — contienen cada secret que Terraform administró, incluidas passwords de base de datos y API keys, en texto plano.

## 11. Arquitectura serverless

Las plataformas serverless — AWS Lambda, GCP Cloud Functions, Azure Functions — abstraen el host, pero no abstraen la superficie de ataque. Entender qué queda es todo el punto de esta sección.

### 11.1 Qué atacas

| Superficie | Descripción |
|---|---|
| Deployment package | El zip o la imagen de container que corre la plataforma. Dependency-confusion en `requirements.txt` o `package.json` compromete cada invocación. |
| Environment variables | Lugar conveniente para guardar URLs de DB, API keys, tokens. Cualquier inyección de código las lee vía `os.environ`. |
| Execution role / SA | La identidad de la function para las llamadas que hace a otros servicios cloud. A menudo sobreprivilegiada. |
| Triggers | Endpoints HTTP, fuentes de eventos (eventos de S3, SQS, Pub/Sub). El input controlado por el atacante fluye por ellos. |
| Persistencia de cold-start | Los archivos escritos en `/tmp` persisten entre invocaciones en la misma instancia de container — útil tanto para offense como para defense. |

### 11.2 Enumerar Lambda desde Kali

```bash
# Lista functions:
aws lambda list-functions --profile loot | jq '.Functions[].FunctionName'

# Obtén el código y el environment de una function objetivo:
aws lambda get-function --function-name public-api-handler --profile loot
# {
#   "Configuration": { "Environment": { "Variables": { "DB_URL": "postgres://...", ... } } },
#   "Code": { "Location": "https://prod-04-2014-tasks.s3...", ... }
# }

# Descarga y descomprime el código:
curl -o /tmp/fn.zip "$(aws lambda get-function --function-name public-api-handler --query 'Code.Location' --output text --profile loot)"
unzip /tmp/fn.zip -d /tmp/fn

# Grep de lo interesante:
grep -rEi 'password|secret|token|key' /tmp/fn
```

La misma forma funciona para GCP y Azure — `gcloud functions describe` y `az functionapp show` respectivamente.

### 11.3 Scope creep del execution role

Los developers otorgan `roles/editor`, Contributor o `AmazonS3FullAccess` a una function «temporalmente» para un launch, y se queda por años. Cada vulnerabilidad de ejecución de código en la function hereda ese privilegio. Audita los roles de functions con la misma agresividad con la que auditas los roles de users.

## 12. Bases de datos managed

Las bases de datos managed le mueven la carga operativa al proveedor, pero la superficie de ataque sigue ahí.

### 12.1 Los sospechosos habituales

| Servicio | Misconfiguration común |
|---|---|
| RDS / Aurora | Snapshot compartido con todas las accounts; public accessibility on. |
| Cloud SQL | Red autorizada de la instancia `0.0.0.0/0`. |
| Cosmos DB | Firewall Allow access from all networks; primary key en el código. |
| DynamoDB | Policy de IAM demasiado amplia; backups de PITR legibles. |

### 12.2 Misconfiguration de snapshot sharing

Los snapshots de RDS se pueden marcar como públicos. Cualquiera en cualquier account de AWS puede entonces restaurar tu base de datos en su propia account y consultarla con calma.

```bash
# Desde tu propia account, caza snapshots compartidos públicamente:
aws rds describe-db-snapshots \
  --snapshot-type public \
  --include-public \
  --query 'DBSnapshots[?contains(DBSnapshotArn, `target-account-id`)]'

# Si el target es tu propio cliente, esta llamada también enumera sus leaks:
aws rds describe-db-snapshots \
  --snapshot-type public \
  --include-public --profile loot
```

### 12.3 Instancias alcanzables públicamente

Cualquier base de datos managed con un endpoint público y un security group demasiado permisivo es una exposición directa. Haz Nmap del CIDR expuesto:

```bash
nmap -Pn -p 3306,5432,1433,27017,6379 -sS -T4 <db-endpoint-cidr>
```

Combínalo con credenciales que encontraste en un `.env` filtrado, en un environment de Lambda o en un backup de S3.

## 13. Servicios de containers

Las plataformas de containers managed son cloud-native en cada capa, y su superficie de ataque es un superconjunto de Kubernetes más la identidad cloud de abajo.

### 13.1 El paisaje

| Servicio | Proveedor | Notas |
|---|---|---|
| ECS | AWS | Task roles por task; las tasks llegan a IMDS a través de un endpoint de metadata de task. |
| EKS | AWS | Kubernetes completo; los nodes tienen roles de IAM, los pods pueden tener IRSA o Pod Identity. |
| Fargate | AWS | Runtime de containers serverless bajo ECS/EKS. |
| GKE | GCP | Kubernetes; los nodes tienen node SA; Workload Identity liga pods a una SA. |
| Cloud Run | GCP | Containers serverless; SA por servicio. |
| AKS | Azure | Kubernetes; identidad de kubelet, managed identity de pod o Workload Identity. |
| ACI | Azure | Containers serverless; MI por container group. |

### 13.2 Robo de task role en ECS vía SSRF a IMDS

Las tasks de ECS obtienen sus credenciales de un endpoint de metadata de task, no de IMDS directo. La URL del endpoint se expone al container en la variable de entorno `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`:

```bash
# Dentro del container:
env | grep AWS
# AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/8bd...

# Trae las credenciales de la task:
curl "http://169.254.170.2${AWS_CONTAINER_CREDENTIALS_RELATIVE_URI}"
# {
#   "AccessKeyId":"ASIA...","SecretAccessKey":"...","Token":"...",
#   "Expiration":"2026-...","RoleArn":"arn:aws:iam::...:role/ecs-task-role"
# }
```

Un SSRF en una aplicación web containerizada que llega a `169.254.170.2` devuelve credenciales del task role. El mismo truco aplica a Fargate.

### 13.3 Credenciales de pods en EKS

Los pods que corren bajo IRSA (IAM Roles for Service Accounts) tienen un JWT proyectado en `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`. Cualquier break-out de container lo lee y llama a `aws sts assume-role-with-web-identity` desde cualquier lugar:

```bash
aws sts assume-role-with-web-identity \
  --role-arn "$AWS_ROLE_ARN" \
  --role-session-name pivot \
  --web-identity-token file://$AWS_WEB_IDENTITY_TOKEN_FILE
```

## 14. Servicios de secrets en cloud

Cada proveedor entrega un store de secrets managed. Si tienes un role que los lee, tienes cada secret.

### 14.1 AWS Secrets Manager y Parameter Store

```bash
# Lista y lee cada secret de la account:
aws secretsmanager list-secrets --profile loot | jq '.SecretList[].Name'
aws secretsmanager get-secret-value --secret-id prod/db/master --profile loot
# {
#   "SecretString": "{\"username\":\"admin\",\"password\":\"...\"}"
# }

# Parameter Store — SSM:
aws ssm describe-parameters --profile loot
aws ssm get-parameters-by-path --path / --recursive --with-decryption --profile loot
```

Parameter Store a menudo es más permisivo que Secrets Manager porque empezó como un store de config y se volvió un store de secrets por convención.

### 14.2 GCP Secret Manager

```bash
gcloud secrets list --project client-prod-01
gcloud secrets versions access latest --secret=db-password --project client-prod-01
```

### 14.3 Azure Key Vault

```bash
# Lista vaults:
az keyvault list -o table

# Lista y lee secrets de un vault:
az keyvault secret list --vault-name prod-kv -o table
az keyvault secret show --vault-name prod-kv --name db-connection-string
```

El acceso al vault lo controlan o las access policies legacy o Azure RBAC sobre el vault. El role Key Vault Administrator es el equivalente a ownership. Key Vault Secrets User otorga solo lectura.

## 15. Logging y detección

Todo lo que escribes genera líneas de log. Asume que cada llamada de API queda capturada; planifica tu enumeración alrededor de esa suposición.

### 15.1 Las superficies de log

| Proveedor | Auditoría del control plane | Acceso del data plane | Red |
|---|---|---|---|
| AWS | CloudTrail (management + data events) | S3 Access Logs, data events de CloudTrail | VPC Flow Logs |
| GCP | Cloud Audit Logs (Admin Activity + Data Access) | Access logs de GCS | VPC Flow Logs |
| Azure | Activity Log + Sign-in Logs de Entra ID | Diagnostic logs de Storage | NSG Flow Logs |

Sistemas de detección montados encima:

- **AWS:** GuardDuty (detección de anomalías), Detective (investigaciones en grafo), Security Hub (agregación).
- **GCP:** Security Command Center, Event Threat Detection.
- **Azure:** Microsoft Defender for Cloud, Microsoft Sentinel (SIEM).

### 15.2 Qué enciende la detección

`aws sts get-caller-identity` no está monitoreado en muchas accounts porque es un chequeo de identidad. Las llamadas `aws iam list-*` son de baja señal de forma individual, pero cientos en un minuto desde un solo principal encienden un SIEM afinado. GuardDuty marca el volumen como `Recon:IAMUser/UserPermissions` y las credenciales de un role de EC2 usadas fuera de AWS como `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS`. Las lecturas de IMDS no aparecen en CloudTrail.

## 16. Ejemplo de punta a punta — SSRF a data breach

Una cadena realista única, desde un request web sin authentication hasta un dump de credenciales de base de datos en texto plano.

### 16.1 Target y setup

El target es una app web de Node.js que fetchea un thumbnail de preview desde una URL que suministra el usuario. El endpoint `POST /preview` acepta un body JSON `{ "url": "..." }` y devuelve los bytes fetcheados. La aplicación corre en EC2 con `HttpTokens: optional` — IMDSv1 está habilitado.

### 16.2 Qué consigue la cadena

`POST /preview` acepta una URL y devuelve los bytes. Con IMDSv1, esa URL puede ser el path de credenciales del role de la instancia. El role del ejemplo, `backup-writer-role`, puede listar y leer un bucket de backups que contiene un dump SQL de producción, y en ese dump aparecen una secret key de Stripe y una de email. Las peticiones, la exportación de las credenciales y la descarga no se incluyen.

La remediación es: forzar IMDSv2, restringir el SSRF, ajustar el role y cifrar los backups con una key KMS administrada por el cliente que el role no pueda descifrar.

### 16.3 Cómo se ve en CloudTrail

Las llamadas a la API de AWS quedan en CloudTrail con la IP de origen. Las lecturas de IMDS no son eventos de CloudTrail. GuardDuty dispara `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` cuando las credenciales del role se usan desde fuera de AWS.

## 17. Ejercicios

### 17.1 Ejercicios guiados

#### Ejercicio 1 — Auditar una policy de IAM por privesc de `iam:PassRole`

Dada la siguiente inline policy adjunta a `arn:aws:iam::123456789012:user/deployer`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "ec2:RunInstances", "Resource": "*" },
    { "Effect": "Allow", "Action": "iam:PassRole", "Resource": "*" },
    { "Effect": "Allow", "Action": "iam:ListInstanceProfiles", "Resource": "*" }
  ]
}
```

Explica la cadena: `iam:PassRole` sin acotar más `ec2:RunInstances` permite adjuntar un instance profile administrativo. No escribas el comando que lanza la instancia. Propón un fix de policy que mantenga el deployment funcionando: acota `iam:PassRole` a un role de deploy de bajo privilegio vía `Resource` y una `Condition` sobre `iam:PassedToService`.

#### Ejercicio 2 — Enumerar una account de lab de S3

Levanta una account personal de AWS (o usa una account de lab asignada). Crea tres buckets de S3:

- `<your-alias>-public-lab` con una bucket policy que otorga lectura a `Principal: "*"`.
- `<your-alias>-authusers-lab` con una ACL que otorga lectura a `AuthenticatedUsers`.
- `<your-alias>-private-lab` con settings por defecto.

Desde un role de auditoría de la misma account, documenta qué modelo de acceso expone cada bucket usando `get-bucket-acl`, `get-bucket-policy` y `get-public-access-block`. No listes ni copies objetos.

#### Ejercicio 3 — Recorrer un camino de token de EKS

Despliega un cluster mínimo de EKS. Crea una service account con una annotation de IRSA hacia un role que solo tenga lo que el pod necesita. Anota la trust policy (el OIDC del cluster y la condition sobre el subject de la SA) y confirma que un principal de fuera del cluster no puede asumir ese role. No extraigas el token ni lo uses fuera del pod.

### 17.2 Ejercicios independientes

#### Ejercicio 4 — Matriz de IMDS entre proveedores

Arma una matriz de cada endpoint de IMDS (path, method, headers obligatorios) para AWS IMDSv1, AWS IMDSv2, GCP y Azure. Para cada fila anota el control que lo cierra (`HttpTokens: required`, `Metadata-Flavor`, `Metadata: true`, o bloquear el enlace). No escribas la petición ni un payload de SSRF.

#### Ejercicio 5 — Cadena de `iam.serviceAccounts.getAccessToken` en GCP

Diseña en papel un project de GCP con tres service accounts — A, B y C — donde A puede impersonar a B, B puede impersonar a C, y C tiene `roles/owner`. Propón el conjunto mínimo de roles que hay que quitar para romper la cadena sin romper el workload previsto de B. No escribas los comandos de los saltos.

#### Ejercicio 6 — Extracción de secrets en serverless

Describe una function de Lambda que lee `DB_URL` del entorno y cuyo role además tiene `secretsmanager:GetSecretValue`. Anota las dos exposiciones (la variable y el permiso) y la remediación: el handler lee el secret por IAM, la variable no lleva la credencial y el role no puede leer otros secrets. No introduzcas una inyección ni invoques la function para leerlos.

## 18. Resumen y lección siguiente

La infraestructura cloud es código, e IAM es el sistema operativo. Cada proveedor entrega primitivos análogos — compute, storage, identity, network — con nombres distintos y modelos de confianza ligeramente distintos. En un engagement moderno vas a pivotar de un foothold de app web a una consola cloud mucho más rápido de lo que vas a pivotar de un domain user a Domain Admin. Eso es porque las misconfigurations de IAM se componen: un solo `iam:PassRole` o `iam.serviceAccounts.getAccessToken` combinado con el permiso adyacente correcto es total.

La detección es real. GuardDuty, Security Command Center y Defender for Cloud cubren las técnicas clásicas: ráfagas de `AccessDenied`, credenciales de instancia usadas fuera de AWS y buckets públicos. El control que cierra la cadena de esta lección es IMDSv2, un role acotado y el cifrado del backup con una key que ese role no puede usar.

Lección siguiente: Módulo 04, Lección 10 — Kubernetes for Attackers. Tomamos las abstracciones de containers introducidas acá y profundizamos en la superficie de la API de Kubernetes: kube-apiserver, tokens de service-account, RBAC, admission controllers, caminos de escape de `kubectl exec` y el audit trail que Kubernetes deja cuando enumeras. Las distinciones EKS/GKE/AKS cubiertas en la sección 13 se vuelven el punto de partida.

# Lección 10 — Infrastructure as Code and Configuration

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Infrastructure as Code (IaC) convirtió las operaciones en ingeniería de software. Cada regla de firewall, cada policy de IAM, cada subnet de red y cada bucket de storage cloud se expresa ahora como texto en un repositorio git. Esa transformación es un regalo para los defensores porque hace la infraestructura auditable y reproducible — pero es un regalo todavía más grande para los atacantes, porque un solo `git clone` te da el plano de todo el entorno del target.

Los repositorios de IaC filtran. Filtran por accidente en GitHub público. Filtran por wikis internos demasiado permisivos. Filtran por buckets de S3 mal configurados que los espejan para CI. Filtran por archivos de state que los developers commitean sin pensarlo. Cuando encuentras uno durante un engagement, no encontraste una vulnerabilidad: encontraste la descripción de cada vulnerabilidad que el equipo de operaciones escribió y pusheó.

Esta lección trata IaC como superficie de ataque. Cubrimos las dos tools dominantes — Terraform y Ansible — más el ecosistema alrededor: user-data de cloud-init, imágenes golden de Packer, Pulumi y CDK, detección de drift, policy-as-code y la supply chain de módulos de la comunidad. Cada sección incluye comandos reales de Kali 2026 y patrones concretos para cazar. Al final vas a poder entrar a un repo de cliente, correr tres comandos y devolver un reporte de todo lo que está mal en su postura cloud — antes de tocar producción.

## 1. IaC como superficie de ataque — el código fuente es la verdad de terreno

IaC es código fuente, y el código fuente filtra. Esa sola frase debería dar forma a cómo abordas cada engagement que incluye un componente de infraestructura.

### 1.1 Qué contienen los repositorios de IaC

Un repositorio típico de operaciones guarda:

- Topología de red (CIDRs de VPC, layouts de subnet, relaciones de peering)
- Reglas de firewall y security groups (qué puertos están abiertos hacia quién)
- Policies de IAM (quién puede hacer qué sobre qué recurso)
- Backends de secrets y sus configuraciones
- Scripts de bootstrap que corren en cada server nuevo
- Definiciones de pipelines de CI/CD
- A veces: credenciales reales, tokens o private keys commiteadas por error

Compara esa lista con lo que te da un pentest externo tradicional: un hostname y un documento de scope. El repo de IaC es el mapa; el pentest es el terreno. Una vez que tienes el mapa puedes planear cada movimiento siguiente.

### 1.2 El cambio del modelo de amenaza

```text
Infra tradicional:
  attacker → recon → scan → discover services → probe → exploit

Infra habilitada por IaC:
  attacker → git clone → grep -r "0.0.0.0/0" → know every exposure
```

> **Pentester Insight:** Cada repositorio de IaC debería modelarse como amenaza como si fueran las joyas de la corona — porque operacionalmente lo son. Durante engagements de red team pregunta temprano si el cliente tiene un repo interno de infraestructura o de terraform-live, y si es accesible desde las laptops de developers que ya comprometiste. Ese solo artefacto a menudo vuelve trivial el resto del engagement.

### 1.3 Dónde vive IaC durante un test

| Ubicación | Contenido habitual | El acceso exige |
|---|---|---|
| GitHub público | Módulos open-source, errores ocasionales | Nada |
| GitHub/GitLab interno | Definiciones de infraestructura viva | Credenciales de developer |
| Artefactos de pipeline CI/CD | Plans compilados, snapshots de state | Acceso al runner de CI |
| Buckets backend de S3/GCS | Archivos de state de Terraform | Permiso de lectura cloud |
| Workstations de developers | Carpetas locales `.terraform/` | Compromiso de endpoint |
| Sistemas de backup | State y plans históricos | Derechos de restore de backup |

## 2. Fundamentos de Terraform

Terraform, de HashiCorp, es el estándar de facto para infraestructura declarativa multi-cloud. Escribes archivos HCL (HashiCorp Configuration Language) que describen el estado deseado; Terraform los reconcilia con la realidad.

### 2.1 Instalar Terraform en Kali 2026

```bash
# Kali 2026 trae terraform en los repositorios de la comunidad:
sudo apt update
sudo apt install terraform

# Verifica:
terraform version
# Terraform v1.9.5
# on linux_amd64

# Si necesitas una versión específica, usa tfenv:
sudo apt install tfenv
tfenv install 1.9.5
tfenv use 1.9.5
```

### 2.2 Panorama de la sintaxis HCL

Un archivo mínimo de Terraform muestra cada constructo central:

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# variables.tf
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_name" {
  type = string
}

# data.tf — lookups de solo lectura
data "aws_caller_identity" "current" {}

# main.tf — recursos declarativos
resource "aws_s3_bucket" "logs" {
  bucket = var.bucket_name
  tags = {
    Owner = data.aws_caller_identity.current.account_id
  }
}

# outputs.tf
output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}
```

### 2.3 El workflow de Terraform

```bash
# Inicializa el directorio de trabajo (descarga providers, configura el backend):
terraform init

# Muestra qué va a cambiar Terraform sin cambiar nada:
terraform plan -out=tfplan

# Aplica el plan:
terraform apply tfplan

# Derriba todo:
terraform destroy

# Formatea el código al estilo canónico:
terraform fmt -recursive

# Valida sintaxis y referencias internas:
terraform validate
```

### 2.4 Módulos

Los módulos son paquetes reutilizables de Terraform. Un módulo tiene sus propios archivos `.tf` y se puede llamar desde otras configuraciones:

```hcl
module "network" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
}
```

Los módulos pueden venir del disco local, de repositorios git, del Terraform Registry público o de registries privados. Cada source de módulo es un input de supply chain en el que tienes que confiar.

### 2.5 Backends

Un backend le dice a Terraform dónde guardar el state. El backend local por defecto escribe `terraform.tfstate` en el directorio actual — bien para trabajo solo, catastrófico para equipos. Los backends remotos guardan el state en una ubicación compartida y con lock.

## 3. El archivo de state de Terraform — el artefacto más sensible

El archivo más valioso de cualquier entorno administrado por Terraform es `terraform.tfstate`. Es un documento JSON que espeja toda la infraestructura — y al hacerlo guarda cada atributo de cada recurso, incluidos los atributos marcados como sensitive, en texto plano.

### 3.1 Qué contiene de verdad el state

El state guarda en texto plano atributos que el HCL marcó como sensitive. En una instancia de base de datos aparecen `password` y `master_password`. También aparecen `secret_string`, `private_key_pem` y `access_key`. Quien puede leer el state tiene el snapshot de esas credenciales. No vuelques un state de producción.

### 3.2 State local — qué no hacer

```bash
# El backend por defecto si no hay nada configurado:
ls -la
# -rw-r--r-- 1 user user  14523 terraform.tfstate
# -rw-r--r-- 1 user user   8901 terraform.tfstate.backup

# .gitignore DEBE incluir:
#   terraform.tfstate
#   terraform.tfstate.backup
#   *.tfvars
#   .terraform/
```

La cantidad de engagements de cliente en los que encontré `terraform.tfstate` commiteado a un repositorio, en una laptop de developer o sentado en un drive compartido sin protección no es chica.

### 3.3 Backends remotos — S3 + DynamoDB

El patrón canónico de AWS combina un bucket de S3 para storage con una tabla de DynamoDB para locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tf-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-tf-locks"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:111122223333:key/mrk-abc"
  }
}
```

El locking evita que dos ingenieros corran `terraform apply` al mismo tiempo. El cifrado evita que una exfiltración de objetos de S3 sea una pérdida total.

### 3.4 Otros backends remotos

```hcl
# Google Cloud Storage:
backend "gcs" {
  bucket = "acme-tf-state"
  prefix = "prod/network"
}

# Azure Blob:
backend "azurerm" {
  resource_group_name  = "tfstate-rg"
  storage_account_name = "acmetfstate"
  container_name       = "state"
  key                  = "prod.terraform.tfstate"
}

# Terraform Cloud / HCP:
cloud {
  organization = "acme"
  workspaces {
    name = "prod-network"
  }
}
```

### 3.5 Cazar archivos de state durante un engagement

```bash
# En una workstation de developer comprometida:
find / -name "terraform.tfstate*" 2>/dev/null
find / -name ".terraform" -type d 2>/dev/null

# En un repositorio git (incluida la historia):
git log --all --full-history -- '**/terraform.tfstate'
git log --all -p | grep -A2 '"password"'

# En un bucket de S3 que puedes listar:
aws s3 ls s3://acme-tf-state/ --recursive | grep tfstate

# Extrae secrets de un archivo de state capturado:
jq -r '.resources[] | .instances[]?.attributes | to_entries[] | select(.value | tostring | test("(?i)pass|secret|key|token"))' terraform.tfstate
```

> **Pentester Insight:** Un solo `terraform.tfstate` robado de una empresa de tamaño medio puede entregarte passwords de RDS a nivel de dominio, access keys de IAM, aliases de KMS con sus policies adjuntas y cada tag de EC2 que revela qué host corre qué workload sensible. Trata el robo de state como un objetivo de máxima prioridad.

## 4. Leer Terraform para la postura de seguridad

Antes de correr cualquier scanner, aprende a leer HCL con ojos de pentester. Los scanners se pierden el contexto; los humanos no.

### 4.1 Wildcards de IAM

```hcl
# El desastre clásico — acceso administrativo a todo:
resource "aws_iam_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}
```

Patrón de grep:

```bash
grep -rEn '"Action"\s*[:=]\s*"?\*' .
grep -rEn '"Resource"\s*[:=]\s*"?\*' .
grep -rEn 'iam:\*|s3:\*|ec2:\*' .
```

### 4.2 Buckets de S3 públicos

```hcl
resource "aws_s3_bucket_public_access_block" "bad" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_acl" "worse" {
  bucket = aws_s3_bucket.data.id
  acl    = "public-read"
}
```

Patrón de grep:

```bash
grep -rEn 'acl\s*=\s*"public' .
grep -rEn 'block_public_(acls|policy)\s*=\s*false' .
grep -rEn 'publicly_accessible\s*=\s*true' .
```

### 4.3 Buckets sin versionado — riesgo de ransomware y borrado

```hcl
resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  versioning_configuration {
    status = "Suspended"   # o ausente por completo
  }
}
```

Patrón de grep:

```bash
grep -rEn 'versioning\s*\{[^}]*enabled\s*=\s*false' .
grep -rEn 'status\s*=\s*"Suspended"' .
```

### 4.4 Security groups abiertos al mundo

```hcl
resource "aws_security_group_rule" "ssh_world" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

Patrón de grep:

```bash
grep -rEn 'cidr_blocks\s*=\s*\["0\.0\.0\.0/0"' .
grep -rEn '::/0' .
grep -rEn 'source_ranges\s*=\s*\["0\.0\.0\.0/0"' .   # GCP
```

### 4.5 Storage sin cifrar

```bash
grep -rEn 'storage_encrypted\s*=\s*false' .
grep -rEn 'encryption\s*=\s*"NONE"' .
grep -rEn 'kms_key_id' .   # inverso — encuentra lo que SÍ está cifrado
```

### 4.6 Credenciales hardcodeadas

```bash
grep -rEnI '(access_key|secret_key|password|token)\s*=\s*"[A-Za-z0-9+/=]{16,}"' .
grep -rEnI 'AKIA[0-9A-Z]{16}' .
grep -rEnI 'AIza[0-9A-Za-z_\-]{35}' .
```

## 5. Scanners exclusivos de Terraform

La revisión manual escala mal. Los scanners hechos a propósito corren en segundos y cazan los patrones conocidos.

### 5.1 tfsec

```bash
# Instala (Kali 2026):
sudo apt install tfsec

# O por el instalador oficial:
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Escanea un directorio:
tfsec .

# Output típico:
#   Result #1 HIGH Bucket does not have versioning enabled
#     ./s3.tf:12-18
#      12 | resource "aws_s3_bucket" "data" {
#      13 |   bucket = "acme-data"
#      14 | }
#     ID:  aws-s3-enable-versioning
#     Impact: Deleted or modified data would not be recoverable
#     Resolution: Enable versioning

# Output JSON para pipelines:
tfsec . --format json > tfsec-report.json

# Excluye findings aceptables conocidos:
tfsec . --exclude aws-s3-enable-versioning,aws-vpc-no-public-ingress-sgr
```

### 5.2 checkov

```bash
# Instala vía pip (se recomienda un entorno aislado):
pipx install checkov

# Escanea Terraform, CloudFormation, manifiestos de Kubernetes y más:
checkov -d .

# Enfócate en un solo framework:
checkov -d . --framework terraform

# SARIF para code scanning de GitHub:
checkov -d . -o sarif --output-file-path checkov.sarif

# Salta checks que ya sabes que son seguros:
checkov -d . --skip-check CKV_AWS_18,CKV_AWS_144
```

### 5.3 terrascan

```bash
sudo apt install terrascan
terrascan scan -t aws -i terraform -d .

# Los policy packs cubren AWS, Azure, GCP, Kubernetes, Docker, GitHub.
```

### 5.4 regula

```bash
# Regula usa reglas de OPA/Rego por debajo:
curl -L https://github.com/fugue/regula/releases/latest/download/regula_linux_amd64.tar.gz | tar xz
sudo mv regula /usr/local/bin/

regula run .
regula run --format json .
```

### 5.5 Combinar scanners

```bash
# Una etapa de CI en profundidad de defensa:
tfsec . --format json > out/tfsec.json
checkov -d . -o json > out/checkov.json
terrascan scan -t aws -i terraform -d . -o json > out/terrascan.json

# Después alimenta los tres a un script de triage — scanners distintos cazan
# issues distintos. Nunca te apoyes en uno solo.
```

## 6. Fundamentos de Ansible

Ansible es la contraparte imperativa de Terraform. Donde Terraform declara el estado deseado, Ansible ejecuta tareas ordenadas. Ansible no exige un agente — corre sobre SSH (o WinRM para Windows) usando Python en el target.

### 6.1 Instalar Ansible en Kali 2026

```bash
sudo apt install ansible ansible-lint

ansible --version
# ansible [core 2.17.4]
#   config file = /etc/ansible/ansible.cfg
#   python version = 3.12.5
```

### 6.2 Inventory

Un inventory lista los hosts que Ansible administra:

```ini
# inventory.ini
[web]
web01.acme.local ansible_user=deploy
web02.acme.local ansible_user=deploy

[db]
db01.acme.local ansible_user=postgres

[all:vars]
ansible_ssh_private_key_file=~/.ssh/acme_deploy
```

O YAML:

```yaml
all:
  children:
    web:
      hosts:
        web01.acme.local:
        web02.acme.local:
    db:
      hosts:
        db01.acme.local:
```

Los inventories dinámicos consultan APIs cloud en runtime:

```bash
ansible-inventory -i aws_ec2.yaml --graph
```

### 6.3 Playbooks

Un playbook es un archivo YAML de tareas contra hosts:

```yaml
# site.yml
- name: Configure web servers
  hosts: web
  become: true
  vars:
    app_version: "2.4.1"
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

  handlers:
    - name: restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

Córrelo:

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --check       # dry-run
ansible-playbook -i inventory.ini site.yml --diff        # muestra diffs de archivos
ansible-playbook -i inventory.ini site.yml --limit web01 # un solo host
```

### 6.4 Roles

Los roles empaquetan tareas, handlers, templates y defaults relacionados:

```text
roles/
  nginx/
    tasks/main.yml
    handlers/main.yml
    templates/nginx.conf.j2
    defaults/main.yml
    vars/main.yml
    meta/main.yml
```

### 6.5 Ansible Vault

Los archivos de variables cifrados usan AES-256 con una key derivada de una passphrase:

```bash
ansible-vault create group_vars/prod/secrets.yml
ansible-vault edit  group_vars/prod/secrets.yml
ansible-vault view  group_vars/prod/secrets.yml
ansible-vault encrypt_string 'super-secret' --name db_password
```

Los playbooks corren contra vars en vault usando `--ask-vault-pass` o un archivo de password.

## 7. Revisión de seguridad de playbooks de Ansible

Los playbooks de Ansible se ejecutan como root en el target. Cada revisión es, por lo tanto, una revisión de supply chain.

### 7.1 Sinks de inyección de shell

```yaml
# PELIGROSO — el input del usuario fluye a una shell:
- name: Fetch user data
  ansible.builtin.shell: "curl {{ user_url }}"

- name: Rebuild
  ansible.builtin.command: "make {{ user_target }}"

# MÁS SEGURO — módulo con argumentos estructurados:
- name: Fetch user data
  ansible.builtin.get_url:
    url: "{{ user_url }}"
    dest: /tmp/data
```

Patrones de grep:

```bash
grep -rEn 'ansible\.builtin\.(shell|command|raw):' roles/ playbooks/
grep -rEn '^\s*shell:\s*' .
grep -rEn '{{[^}]*}}' . | grep -Ei 'shell|command'
```

### 7.2 Disciplina de `no_log`

Las tareas que tocan secrets deben marcarse `no_log: true`, o el secret va a caer en `/var/log/ansible.log` y en cualquier log de job de CI:

```yaml
- name: Set DB password
  community.postgresql.postgresql_user:
    name: app
    password: "{{ vault_db_password }}"
  no_log: true      # obligatorio
```

Haz grep del inverso — encuentra tareas que tocan secrets y NO están marcadas:

```bash
grep -rEln '(password|token|secret|api_key)' roles/ | while read f; do
  grep -L 'no_log:\s*true' "$f"
done
```

### 7.3 Archivos legibles por el mundo

```yaml
# Error común:
- name: Write kubeconfig
  ansible.builtin.copy:
    content: "{{ kubeconfig }}"
    dest: /etc/kubernetes/admin.conf
    mode: '0644'          # legible por el mundo — debería ser 0600
```

```bash
# Busca modos permisivos en paths sensibles:
grep -rEn "mode:\s*['\"]0?6[46]4['\"]" roles/ playbooks/ | \
  grep -Ei 'kube|secret|token|password|\.key|\.pem'
```

### 7.4 Secrets en texto plano en `group_vars`

```bash
# Cualquier YAML sin cifrar en group_vars es una bandera roja si contiene keys con pinta de secret:
grep -rEln 'password:|token:|secret:|api_key:' group_vars/ host_vars/ | while read f; do
  head -1 "$f" | grep -q '^\$ANSIBLE_VAULT' || echo "UNENCRYPTED: $f"
done
```

### 7.5 ansible-lint

```bash
ansible-lint site.yml
ansible-lint --profile production roles/

# Las reglas `no-log-password` y `command-instead-of-shell` cazan
# muchos de los patrones de arriba de forma automática.
```

## 8. Ansible Vault — cifrado hecho mal

Vault es tan fuerte como la passphrase y el lugar donde vive.

### 8.1 El pecado capital

La falla más común de Vault es commitear el archivo cifrado Y la password al mismo repositorio:

```text
repo/
  group_vars/prod/secrets.yml    # cifrado con ansible-vault
  .vault_pass                     # passphrase en texto plano — DESASTRE
```

Caza este patrón:

```bash
find . -name '.vault_pass*' -o -name 'vault_password*' -o -name '*.vault_key' 2>/dev/null
grep -rEln '^\$ANSIBLE_VAULT;' . > encrypted.txt
grep -rEln 'ANSIBLE_VAULT_PASSWORD' .
```

### 8.2 Estructura del archivo de Vault

Un archivo cifrado siempre empieza con una línea de header:

```text
$ANSIBLE_VAULT;1.1;AES256
36393464653734653733...   (hex ciphertext)
```

La etiqueta `1.1` usa AES-256-CTR con un tag HMAC-SHA256. La etiqueta `1.2` (Ansible 2.4+) agrega etiquetas por vault para que varios secrets puedan coexistir.

### 8.3 Integrar un backend de secrets de verdad

En producción la passphrase debería venir de HashiCorp Vault, AWS Secrets Manager o un servicio similar:

```ini
# ansible.cfg
[defaults]
vault_password_file = /usr/local/bin/vault-pw.sh
```

```bash
# vault-pw.sh
#!/bin/sh
vault kv get -field=password secret/ansible/prod
```

El script trae la passphrase desde HashiCorp Vault en el momento de correr. La passphrase nunca vive en disco.

### 8.4 Passphrase débil

Un archivo de Vault cifrado con una passphrase corta se puede atacar offline. El fuente nombra `ansible2john.py`, John y hashcat modo 16900. El procedimiento no se incluye. En el historial de Git quedan ciphertexts viejos aunque la passphrase se haya rotado: por eso el archivo cifrado y la passphrase no van al repositorio, y la passphrase sale de un backend de secrets (sección 8.3).

## 9. Cloud-init y user-data

Cada VM cloud moderna arranca ejecutando un blob de user-data suministrado en el momento del launch. Ese blob típicamente contiene directivas de cloud-init o un script de shell — y a menudo contiene credenciales de provisioning.

### 9.1 Ejemplo de YAML de cloud-init

```yaml
#cloud-config
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3Nz... ops@acme

write_files:
  - path: /etc/consul/consul.hcl
    permissions: '0600'
    content: |
      encrypt = "K5G4t3XmA..."
      acl { tokens { agent = "s3cr3t-token" } }

runcmd:
  - curl -H "X-Auth: $(cat /etc/secrets/api_token)" https://install.acme.io/bootstrap | bash
```

### 9.2 Recuperación por IMDS

El Instance Metadata Service (IMDS) expone user-data a cualquier cosa que corra en la instancia — incluida una aplicación web comprometida:

```bash
# AWS IMDSv2 (default y exigido en accounts modernas):
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/user-data

# GCP:
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/user-data

# Azure:
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/instance/customData?api-version=2021-02-01"
```

### 9.3 SSRF como vector de ataque de IaC

Cualquier SSRF de capa de aplicación que llegue a `169.254.169.254` se vuelve un extractor de IaC. En una instancia EC2 los mismos endpoints entregan:

- `iam/security-credentials/<role-name>` — keys temporales de AWS
- `user-data` — el script de bootstrap
- `hostname`, `local-hostname`, `public-hostname` — para descubrimiento lateral

### 9.4 Cazar user-data del lado del servidor

Si tienes acceso de API para listar instancias:

```bash
# AWS:
aws ec2 describe-instance-attribute --instance-id i-0abc --attribute userData \
  --query 'UserData.Value' --output text | base64 -d

# GCP:
gcloud compute instances describe web01 --zone=us-central1-a \
  --format='value(metadata.items.startup-script)'
```

## 10. Packer — hornear credenciales en imágenes golden

Packer produce imágenes de máquina inmutables (AMIs, imágenes de Compute Engine, templates de VMware) a partir de una definición de build. Si los secrets filtran a la imagen, cada VM lanzada desde ella los expone.

### 10.1 Un archivo de build de Packer

```hcl
source "amazon-ebs" "base" {
  region        = "us-east-1"
  ami_name      = "acme-web-{{timestamp}}"
  instance_type = "t3.medium"
  source_ami    = "ami-0abcdef1234567890"
  ssh_username  = "ec2-user"
}

build {
  sources = ["source.amazon-ebs.base"]

  provisioner "shell" {
    inline = [
      "sudo yum install -y nginx",
      "sudo systemctl enable nginx",
    ]
  }
}
```

### 10.2 Errores comunes de Packer

- Embeber private keys SSH en la imagen construida (`/home/ec2-user/.ssh/id_rsa`)
- Hornear bearer tokens para el registro post-boot
- Dejar scripts de provisioning `/tmp/*.sh` con API keys adentro
- Construir con `-var 'api_key=...'` y olvidar que los logs de Packer redactan solo de forma parcial

### 10.3 Extraer secrets de una AMI sospechosa

```bash
# Lanza una instancia desde la AMI (en una account sandbox) y después:
sudo grep -rEn 'AKIA|BEGIN PRIVATE KEY|api[_-]?key|password' \
  / --include='*.sh' --include='*.conf' --include='*.env' 2>/dev/null

# Busca en los logs de cloud-init, que a menudo conservan el output de provisioning:
sudo cat /var/log/cloud-init-output.log
sudo cat /var/log/cloud-init.log
```

## 11. Pulumi y CDK — IaC con lenguajes de programación de verdad

Pulumi y AWS CDK te dejan escribir infraestructura en TypeScript, Python, Go o C#. Esa flexibilidad multiplica las formas en que se cuelan secrets y misconfigurations.

### 11.1 Ejemplo de Pulumi (TypeScript)

```typescript
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("data", {
    acl: "public-read",                 // el mismo antipatrón que en HCL
});

const key = new aws.iam.AccessKey("ci", { user: "ci-runner" });
export const secret = key.secret;       // va a caer en el state
```

El state de Pulumi vive en el Pulumi Service o en un backend de S3/GCS/Azure Blob y — como Terraform — contiene cada atributo en texto plano salvo que marques los valores de forma explícita como `Output.secret`.

### 11.2 CDK — el synth ejecuta tu código

CDK sintetiza templates de CloudFormation ejecutando la aplicación del usuario:

```bash
cdk synth
cdk diff
cdk deploy
```

`cdk synth` corre código arbitrario. Una dependencia maliciosa de CDK puede:

- Leer credenciales de AWS desde `~/.aws/credentials`
- Exfiltrar las variables de entorno del developer
- Modificar el template emitido para insertar un role de backdoor

### 11.3 Ángulos de ataque propios del IaC basado en código

- Typosquatting de npm/pip sobre el paquete de IaC (por ejemplo, `@pulumi/aws-extras`)
- Scripts de post-install en dependencias transitivas
- `plan` y `synth` corriendo en CI sin restricciones de egress
- Generación dinámica de templates que tira de URLs externas en el momento del plan

### 11.4 Cazar state de Pulumi

```bash
# Backend local:
find / -name 'Pulumi.*.yaml' -o -name '.pulumi' -type d 2>/dev/null

# Remoto (backend S3):
aws s3 ls s3://acme-pulumi-state/ --recursive
```

## 12. Detección de drift — realidad vs declaración

El drift es la brecha entre lo que IaC dice que existe y lo que existe de verdad en el cloud. El drift es oportunidad para atacantes: una regla de emergencia abierta en la consola y nunca codificada es una regla que el proceso de revisión nunca aprobó.

### 12.1 `terraform plan` como tool de drift

```bash
# Un plan sin cambios prueba el alineamiento:
terraform plan -detailed-exitcode
# Exit code 0 = no changes
# Exit code 1 = error
# Exit code 2 = changes required (i.e., drift)

# Refresca el state sin aplicar:
terraform refresh
terraform plan       # cualquier diff = drift
```

### 12.2 driftctl

```bash
# Instala:
curl -L https://github.com/snyk/driftctl/releases/latest/download/driftctl_linux_amd64 \
  -o /usr/local/bin/driftctl
chmod +x /usr/local/bin/driftctl

# Escanea recursos no administrados:
driftctl scan --from tfstate://terraform.tfstate

# Output:
#   Found unmanaged resources:
#     aws_security_group_rule.abc123 (port 22, 0.0.0.0/0)
#     aws_iam_user.ghostops
```

Los recursos no administrados son oro durante una revisión de red team. Existen sin rastro en papel.

### 12.3 Detección de drift nativa del cloud

```bash
# AWS Config:
aws configservice describe-config-rules
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited

# GCP Config Validator / Security Command Center:
gcloud scc findings list \
  --organization=123456789 \
  --filter='state="ACTIVE" AND category="PUBLIC_BUCKET_ACL"'

# Azure Policy:
az policy state list --filter "ComplianceState eq 'NonCompliant'"
```

### 12.4 Cómo se ve el drift en la práctica

| Síntoma | Causa probable | Valor para el pentester |
|---|---|---|
| Policy de IAM extra adjunta a un role | Grant de acceso de emergencia, nunca removido | Camino de persistencia |
| Regla de security group con `0.0.0.0/0` que no está en Terraform | Fix manual «temporal» | Exposición externa |
| Bucket de S3 extra en la account | Proyecto sombra | Target fresco |
| Key de KMS con una policy pública | Atajo de debugging | Descifrado de datos |

## 13. Filtración de secrets en repositorios de IaC

Los repos de IaC concentran secrets porque concentran configuración. Cada scanner de esta sección pertenece a tu toolkit del primer día de un engagement.

### 13.1 gitleaks

```bash
# Instala en Kali 2026:
sudo apt install gitleaks

# Escanea el working tree:
gitleaks detect --source . --verbose

# Escanea toda la historia de git:
gitleaks detect --source . --log-opts="--all"

# Reporte JSON:
gitleaks detect --source . --report-format json --report-path leaks.json

# Finding de ejemplo:
#   Finding:  AKIAIOSFODNN7EXAMPLE
#   Secret:   wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
#   RuleID:   aws-access-token
#   File:     terraform/live/prod/variables.tf
#   Commit:   9c2a1b3e...
```

### 13.2 trufflehog

```bash
# Instala:
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sudo sh -s -- -b /usr/local/bin

# Escanea un repo git, incluida la historia, y valida los secrets:
trufflehog git file://. --only-verified --json > tf-report.json

# Verified significa que el secret se probó contra el servicio real y funciona.
# Esa es la diferencia entre "false positive" y "active credential".

# Escanea una org de GitHub:
trufflehog github --org=acme --token=$GITHUB_TOKEN --only-verified

# Escanea un bucket de S3:
trufflehog s3 --bucket=acme-backup --only-verified
```

### 13.3 Secret scanning nativo de GitHub

```bash
# Lista alertas de secret scanning vía la API:
gh api /repos/acme/infra/secret-scanning/alerts | jq '.[] | {rule: .secret_type, state, url}'

# La push-protection impide commits que contienen patrones de secrets conocidos.
# Durante un assessment, chequea si la push protection está habilitada a
# nivel de org:
gh api /orgs/acme | jq '.secret_scanning_push_protection_enabled_for_new_repositories'
```

### 13.4 Firmas que vale la pena grepear

```bash
grep -rEn 'AKIA[0-9A-Z]{16}' .                           # AWS access key
grep -rEn 'ASIA[0-9A-Z]{16}' .                           # AWS session key
grep -rEn 'aws_secret_access_key' .
grep -rEn 'AIza[0-9A-Za-z_\-]{35}' .                     # Google API key
grep -rEn 'ya29\.[0-9A-Za-z_\-]+' .                      # Google OAuth token
grep -rEn 'ghp_[A-Za-z0-9]{36}' .                        # GitHub PAT
grep -rEn 'gho_[A-Za-z0-9]{36}' .                        # GitHub OAuth
grep -rEn 'xox[abpr]-[A-Za-z0-9-]{10,}' .                # Slack tokens
grep -rEn 'sk_live_[0-9a-zA-Z]{24,}' .                   # Stripe live
grep -rEn -- '-----BEGIN (RSA|EC|OPENSSH|PRIVATE) KEY-----' .
grep -rEn 'service_account.*json' .
find . -name '*.pem' -o -name '*.key' -o -name 'service_account*.json'
```

### 13.5 La realidad de la rotación

Encontrar un secret es el paso uno. Durante un engagement verifica siempre:

- ¿El secret es válido ahora mismo? (Usa `aws sts get-caller-identity`, `gcloud auth`, etc.)
- ¿Qué tan viejo es? (`git log --follow -p -- path/to/file`)
- ¿Quién más pudo haberlo leído? (Acceso al repo, logs de CI, forks)
- ¿Alguna vez se commiteó a un fork público?

## 14. Policy-as-code — imponer invariantes en el momento del plan

Los scanners encuentran problemas después del hecho. Policy-as-code se niega a dejarlos mergear.

### 14.1 OPA / Rego

Open Policy Agent evalúa reglas Rego contra plans de Terraform en forma JSON:

```bash
# Emite un plan como JSON:
terraform plan -out=tfplan.bin
terraform show -json tfplan.bin > tfplan.json

# Evalúa contra un bundle de policy:
opa eval -d policy/ -i tfplan.json 'data.terraform.deny'
```

Una regla Rego simple contra buckets públicos:

```rego
package terraform.aws

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_acl"
  resource.change.after.acl == "public-read"
  msg := sprintf("Public S3 bucket ACL on %s", [resource.address])
}
```

### 14.2 Sentinel (Terraform Cloud/Enterprise)

Las policies de Sentinel corren dentro del pipeline de Terraform Cloud. Se pueden imponer como advisory, soft-mandatory o hard-mandatory.

### 14.3 Conftest

```bash
sudo apt install conftest
conftest test tfplan.json --policy policy/
```

### 14.4 Kyverno para IaC de Kubernetes

Kyverno impone policies en el momento de la admisión contra manifiestos de Kubernetes, incluido el YAML renderizado por Helm y por Kustomize que producen los pipelines de IaC.

### 14.5 Cobertura en la práctica

Los entornos bien llevados apilan policy-as-code, scanners y detección en runtime. Durante una revisión, pregunta cuáles de las tres capas existen. Los entornos que solo tienen detección en runtime son los más blandos.

## 15. Supply chain — módulos, roles y registries

Cada módulo que consumes es código que ejecutas con privilegios de cloud-admin.

### 15.1 Riesgos del Terraform Registry

```bash
# Lista cada módulo de terceros del que depende un repo:
grep -rEn 'source\s*=' . | grep -Ev 'source\s*=\s*"\.\.?/' | \
  grep -Ev 'source\s*=\s*"terraform-aws-modules/'
```

Ángulos de ataque:

- Typosquat: `hashicrop/aws`, `terraform-aws-modulez/vpc`
- Una cuenta de maintainer comprometida publica una versión maliciosa
- Falta el pin de versión — `module "x" { source = "acme/mod/aws" }` sin cláusula `version` se auto-actualiza

Pinea versiones y verifica hashes donde se pueda:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "= 5.1.2"    # exacta, no "~>"
}
```

### 15.2 Riesgos de Ansible Galaxy

```bash
# Lista roles de la comunidad instalados:
ansible-galaxy role list
```

`requirements.yml` debería pinear versiones y usar URLs completas:

```yaml
- src: https://github.com/geerlingguy/ansible-role-nginx
  version: 3.1.4
```

Los roles de Galaxy se ejecutan como root durante un play. Un role comprometido compromete cada host contra el que corre.

### 15.3 Detectar comportamiento malicioso

```bash
# Busca callbacks de red en el source de módulos de terceros:
grep -rEn 'http.get\|http.post\|http.request' .terraform/modules/
grep -rEn '(url|source|uri)\s*=\s*"https?://' .terraform/modules/

# Busca exec/shell en roles de Ansible:
grep -rEn 'ansible\.builtin\.(shell|command|raw)' ~/.ansible/roles/
```

### 15.4 Registries privados

Terraform Cloud, Artifactory y GitLab soportan registries privados de módulos. Publicar módulos internos ahí — en lugar de copiarlos de registries públicos — recorta de forma dramática la superficie de supply chain.

## 16. Hands-on — auditar un archivo de Terraform vulnerable

Guarda lo siguiente como `/tmp/vulnerable.tf`:

```hcl
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

resource "aws_s3_bucket" "public_data" {
  bucket = "acme-public-data-2026"
}

resource "aws_s3_bucket_public_access_block" "public_data" {
  bucket                  = aws_s3_bucket.public_data.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_acl" "public_data" {
  bucket = aws_s3_bucket.public_data.id
  acl    = "public-read"
}

resource "aws_security_group" "web" {
  name = "web-sg"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_iam_policy" "admin" {
  name = "app-admin"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}

resource "aws_db_instance" "prod" {
  identifier          = "prod-db"
  engine              = "postgres"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  username            = "admin"
  password            = "SuperSecret123!"
  publicly_accessible = true
  storage_encrypted   = false
  skip_final_snapshot = true
}
```

### 16.1 Revisión manual

```bash
cd /tmp
grep -En 'AKIA[0-9A-Z]{16}' vulnerable.tf
# 3:  access_key = "AKIAIOSFODNN7EXAMPLE"

grep -En '"Action"\s*=\s*"\*"|"Resource"\s*=\s*"\*"' vulnerable.tf

grep -En 'cidr_blocks\s*=\s*\["0\.0\.0\.0/0"\]' vulnerable.tf

grep -En 'publicly_accessible\s*=\s*true|storage_encrypted\s*=\s*false' vulnerable.tf

grep -En 'acl\s*=\s*"public-read"|block_public.*=\s*false' vulnerable.tf
```

Findings:

- Access key y secret de AWS hardcodeados en el bloque provider
- Policy de IAM con wildcard (`Action=*` sobre `Resource=*`)
- SSH abierto a internet
- Instancia RDS accesible públicamente, sin cifrar y con password hardcodeada
- Bucket de S3 público con todos los bloqueos de acceso público deshabilitados

### 16.2 Confirmación automatizada

```bash
mkdir -p /tmp/tf-audit && cp /tmp/vulnerable.tf /tmp/tf-audit/
cd /tmp/tf-audit
terraform init -backend=false

tfsec .
checkov -d .
gitleaks detect --source .
```

tfsec va a marcar como mínimo:

- `aws-iam-no-policy-wildcards`
- `aws-s3-block-public-acls`
- `aws-s3-block-public-policy`
- `aws-s3-no-public-buckets`
- `aws-vpc-no-public-ingress-sgr`
- `aws-rds-encrypt-instance-storage-data`
- `aws-rds-no-public-db-access`

### 16.3 Forma segura reescrita

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws",    version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "~> 3.6" }
  }
}

provider "aws" {
  region = "us-east-1"
  # Las credenciales vienen del entorno, de un role de IAM o de SSO — nunca inline.
}

data "aws_vpc" "default" { default = true }

resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "db" {
  name = "prod/postgres/master"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}

resource "aws_s3_bucket" "private_data" {
  bucket = "acme-private-data-2026"
}

resource "aws_s3_bucket_public_access_block" "private_data" {
  bucket                  = aws_s3_bucket.private_data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "private_data" {
  bucket = aws_s3_bucket.private_data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_versioning" "private_data" {
  bucket = aws_s3_bucket.private_data.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = data.aws_vpc.default.id
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]   # solo interno
  }
}

resource "aws_iam_policy" "app" {
  name = "app-least-privilege"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.private_data.arn}/*"
    }]
  })
}

resource "aws_db_instance" "prod" {
  identifier          = "prod-db"
  engine              = "postgres"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  username            = "admin"
  password            = random_password.db.result
  publicly_accessible = false
  storage_encrypted   = true
  kms_key_id          = aws_kms_key.db.arn
  skip_final_snapshot = false
}

resource "aws_kms_key" "db" {
  description         = "RDS encryption key"
  enable_key_rotation = true
}
```

Vuelve a correr los scanners contra la reescritura; deberían devolver cero findings.

## 17. Ejercicios

### 17.1 Ejercicios guiados

#### Ejercicio 1 — Extracción de state de Terraform

Dado un archivo `terraform.tfstate` (crea uno corriendo `terraform apply` sobre la reescritura segura de arriba con `terraform { backend "local" {} }`), extrae:

- Cada password maestra de RDS
- Cada access key de un user de IAM
- Cada `random_password.result` generado

Usa solo `jq`. Verifica comparando con los valores en texto plano que sabes que se generaron.

#### Ejercicio 2 — Crack de Ansible vault

Crea un archivo de vault cifrado con una passphrase débil (`ansible-vault encrypt --ask-vault-pass secrets.yml`, usando una password de `rockyou.txt`). Conviértelo con `ansible2john.py` y crackéalo con John the Ripper. Cronometra la operación. Cambia la passphrase a un string random de 25 caracteres y repite — observa la diferencia de factibilidad.

#### Ejercicio 3 — Inyección de drift

Despliega la reescritura segura de la sección 16.3 en una account sandbox de AWS. Después agrega a mano una regla inbound al security group vía la consola de AWS (puerto 80 desde `0.0.0.0/0`). Corre `terraform plan` y `driftctl scan`. Observa cómo reporta el drift cada tool.

### 17.2 Ejercicios independientes

#### Ejercicio 4 — Barrido de repositorio

Elige cualquier organización pública de GitHub pesada en infraestructura (por ejemplo, uno de tus repos personales o un proyecto open-source de infra conocido). Corre `gitleaks detect --log-opts='--all'` y `trufflehog git`. Categoriza los findings por severidad y por si el secret sigue activo. No intentes usar ninguna credencial encontrada.

#### Ejercicio 5 — Autoría de policy

Escribe una policy OPA Rego que deniegue cualquier plan de Terraform que:

- Cree un bucket de S3 sin cifrado server-side
- Cree una regla de security group con `0.0.0.0/0` en cualquier puerto que no sea 80 o 443
- Adjunte una policy de IAM que contenga `Action = "*"`

Pruébala contra las versiones vulnerable y segura de la sección 16.

#### Ejercicio 6 — Caza de cloud-init

Lanza una instancia EC2 con un script de user-data que contenga un secret obvio. Desde la instancia, recupera el user-data vía IMDSv2. Después, desde una segunda instancia en la misma VPC, intenta llegar al IMDS de la primera instancia vía SSRF (usando una app web chica a tu elección). Documenta qué te gana el SSRF y qué derrota IMDSv2.

## 18. Resumen y lección siguiente

Infrastructure as Code es un arma de doble filo. Para los defensores, hace el entorno auditable, revisable y reproducible. Para los atacantes, colapsa el descubrimiento, el mapeo y la explotación en un solo `git clone`. Cada archivo de state de Terraform es una bolsa de passwords. Cada playbook de Ansible es un payload de shell de root. Cada blob de user-data es un broker de credenciales de bootstrap. Cada dependencia de módulo es un ejecutable sin auditar que corre con privilegios de cloud-admin.

Ideas clave de esta lección:

- Trata cualquier repositorio de IaC como un target de máximo valor; el código fuente es el mapa del terreno
- Los archivos de state de Terraform guardan cada atributo de recurso en texto plano — protégelos como keys
- Lee HCL y YAML con patrones de grep para wildcards, flags públicos y `0.0.0.0/0` antes de correr cualquier scanner
- Corre tfsec, checkov y terrascan juntos; se solapan pero no se duplican
- Ansible Vault es tan seguro como la passphrase y el lugar donde vive — chequea un `.vault_pass` commiteado
- El user-data de cloud-init es un store de secrets de primera clase, alcanzable vía IMDS y SSRF
- Packer y CDK agregan superficies de ejecución de código en el momento del build; audita sus pipelines
- El drift es infraestructura no aprobada — cada regla que no está en Terraform se saltó el proceso de revisión
- gitleaks + trufflehog pertenecen al recon del día uno; valida cada finding antes de reportarlo
- Policy-as-code frena issues en el momento del merge — su ausencia es en sí un finding

La lección siguiente, Lección 11 — Container Orchestration Security, pasa del código que provisiona infraestructura a las plataformas que corren workloads encima: Kubernetes, su superficie de API, misconfigurations de RBAC, admission controllers y movimiento lateral a nivel de cluster. Lleva contigo la mentalidad de esta lección — los manifiestos de Kubernetes también son IaC, y todo lo que aprendiste de scanning, secrets y drift aplica a ellos.

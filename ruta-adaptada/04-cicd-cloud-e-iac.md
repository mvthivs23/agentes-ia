# 04 — CI/CD, cloud e IaC

El runner es código con privilegio. La lección 06 lo dice en la primera sección: cada job ejecuta en infraestructura que ya tiene privilegio ambiente (red, IMDS, rol de la instancia, caches) y privilegio inyectado (registry, claves cloud, token del forge, llaves de deploy, certificados de firma). Quien puede cambiar el workflow, o colar input en el shell del workflow, ejecuta con las dos.

## Lee esto

- `containers-security/06-cicd-pipeline-security.md`
- `containers-security/09-cloud-architecture-fundamentals.md`
- `containers-security/10-infrastructure-as-code-and-configuration.md`
- De la 04 de Kubernetes, la sección 15 (Attack Path 3 — CI/CD Poisoning) solo como el caso que el control de esta lección tiene que hacer caro
- De la 12, el desafío de pipeline únicamente en sus notas de detección y en el stretch que endurece y vuelve a verificar

## Qué es trabajo tuyo

De la 06:

- Las seis fases: Source → Build → Test → Package → Publish → Deploy. En cada fase, qué credencial aparece. Ahí se colocan los gates, no en un stage genérico "security" al final.
- El stack defensivo con el que la lección cierra: commits firmados, SBOM, attestations in-toto y cosign.
- GitLab CI y Jenkins entran si el forge del equipo no es el del ejemplo de GitHub Actions. El mecanismo es el mismo: el archivo del workflow es código.

El ejemplo de `pull_request_target` con título sin sanitizar, el runner self-hosted y el giro desde un token hacia la rama protegida se quedan en la lección. El entregable de esta ruta es el control que esa lección contrapone.

De la 09: la arquitectura cloud donde el runner y el registry viven (red, identidad, metadata). La lees antes de tocar el rol del job.

De la 10:

- Terraform y Ansible, cloud-init, Packer, Pulumi y CDK.
- Drift detection y policy-as-code.
- El supply chain de módulos de la comunidad.
- La promesa concreta del texto: entrar a un repo del cliente, correr los comandos que la lección da, y devolver un informe de la postura cloud sin tocar producción. Ese informe es un entregable de consultoría. Los comandos son los de la lección, no una lista nueva.

La amenaza que la lección describe (el repo de IaC es el mapa, el state file filtra, el bucket replica el repo) es el motivo del review. No hace falta explotarla para presentar el informe.

## Ejercicios

1. **Mapa de secretos.** Sobre un workflow de un repo de lab, anota en qué fase de las seis aparece cada secreto, usando la sección 2 de la lección 06. Si un secreto de deploy está en el job de test, muévelo a la fase que la lección indica.

2. **Stack de cierre.** Aplica, en ese mismo repo de lab, las cuatro piezas que la lección 06 nombra al cerrar: commit firmado, SBOM, attestation in-toto, cosign sobre la imagen del archivo 02 de esta ruta. El gate falla si falta una.

3. **Informe de IaC.** En un repo de Terraform o Ansible de lab, sigue la lección 10 hasta el informe de postura. Incluye policy-as-code y, si la lección lo muestra en esa sección, el drift. El informe no propone cambios en la cuenta de producción.

4. **Nota de detección del pipeline.** Del desafío de CI de la lección 12, escribe solo la nota de detección y aplica el stretch de endurecimiento. La re-verificación que la lección pide es que el hallazgo ya no se concreta. No pegues el workflow envenenado en esta carpeta.

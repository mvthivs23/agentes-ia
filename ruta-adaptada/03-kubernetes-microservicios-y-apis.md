# 03 — Kubernetes, microservicios y APIs

El producto de IA en el cluster es un grafo: gateway, servicios, malla, identidad del pod. Estas lecciones ya recorren ese grafo y dejan la detección y el RBAC al lado del camino de ataque.

## Lee esto

- `containers-security/04-kubernetes-architecture-and-attack-surface.md`
- `containers-security/05-microservices-architecture-patterns.md`
- `containers-security/11-api-gateway-and-authentication-architecture.md`
- `containers-security/07-web-application-architecture-deep-dive.md` para el servicio que está detrás del gateway

La lección 08 no está en el `demo.txt`. Lo que el cierre de la 07 anuncia (escape y superficie de Kubernetes) está en las lecciones 03 y 04.

## Qué es trabajo tuyo

De la 04:

- Componentes del cluster, API server, etcd, kubelet, objetos.
- Secciones 9 y 10: RBAC y ServiceAccounts. Cada pod tiene una. El principio de menor privilegio sale de ahí.
- Sección 11: las misconfigurations que la lección llama el top 10 del pentester. Úsalas como checklist de revisión de manifiestos, no como lista de explotación.
- Sección 16: Detection.
- Los attack paths 1 a 3 (pod hacia cluster-admin, kubeconfig filtrado, envenenamiento de CI) se leen para saber qué evento de la sección 16 correspondería. No se rehacen en esta ruta.
- El lab y los ejercicios de las secciones 17 y 18: quédate con la parte que deja el hallazgo detectado o el permiso recortado.

De la 05:

- Gateway como único punto de paso (sección 4).
- Service mesh, sidecars y mTLS (sección 5). El "control-plane escape" se lee; el control que te llevas es mTLS y el corte de la malla.
- BFF, sync frente a async, gRPC. Importa porque el producto de IA suele tener un servicio de inferencia al lado del API.

De la 11: el gateway y la autenticación. Cruza el enumerador de headers del archivo 01 de esta ruta con lo que esta lección exige en el borde.

De la 07: la arquitectura de la aplicación que el gateway publica. La usas para saber qué dato baja hasta el modelo y qué dato no debería cruzar el borde.

## Ejercicios

1. **ServiceAccount.** En el lab cluster de la sección 3 de la lección 04, revisa el ServiceAccount del pod del servicio de IA contra las secciones 9 a 11. El entregable es el Role o ClusterRole recortado a los verbos que el servicio usa. La sección 16 dice qué se vería si el permiso sobrara.

2. **Gateway.** Dibuja, con los nombres de la lección 05, el camino request → gateway → servicio de inferencia → store. Marca dónde la lección 11 pone la autenticación. Si ese punto no está, el hueco es el de la lección, no uno nuevo.

3. **Detección.** De la sección 16 de la lección 04, elige los controles que cubren los tres attack paths que la lección numera. Escríbelos como nota de detección, en el formato en que la lección 12 del módulo pide las notas de detección (Falco, o la policy de admisión que el propio lab de la 12 nombra: OPA/Gatekeeper). No copies el procedimiento del attack path.

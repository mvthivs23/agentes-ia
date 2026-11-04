# 03 — Kubernetes: RBAC y admission

Fuente: `containers-security/04-kubernetes-architecture-and-attack-surface.md`, secciones 9 a 11 y el laboratorio 17. El laboratorio original escala desde el pod y vuelca Secrets. Esa mitad no se corre aquí. Se corre la mala configuración y la policy que la rechaza.

## Paso 1 — El clúster de laboratorio

```bash
kind create cluster --name attack-lab --image kindest/node:v1.29.4
```

Es el clúster desechable de la sección 17.1. No lo crees en un clúster compartido.

## Paso 2 — El binding que no debe existir

La lección lo crea a propósito para poder bloquearlo:

```bash
kubectl create clusterrolebinding oops \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

La sección 11.2 dice que un `ClusterRoleBinding` de `cluster-admin` hacia `system:serviceaccount:default:default` es el hallazgo crítico habitual: quien controla un pod de `default` es cluster-admin.

Comprueba el binding con la consulta de esa sección:

```bash
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") |
         "\(.metadata.name)  ->  \(.subjects // [] | map(.kind+":"+(.namespace//"-")+"/"+.name) | join(", "))"'
```

`oops` tiene que salir en la lista.

## Paso 3 — La policy que lo rechaza

Instala Kyverno como en la sección 17.2 y aplica la `ClusterPolicy` `forbid-default-sa-cluster-admin` de esa misma sección: `validationFailureAction: Enforce`, match sobre `ClusterRoleBinding`, deny cuando `roleRef.name` es `cluster-admin` y algún subject ServiceAccount se llama `default`.

El mensaje de la policy es el del fuente: `Do not bind cluster-admin to a default ServiceAccount.`

## Paso 4 — Demostrar el rechazo

```bash
kubectl delete clusterrolebinding oops
kubectl create clusterrolebinding oops \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

La salida esperada es el admission webhook de Kyverno denegando el request, con el nombre de la policy. Guarda esa salida. Es la prueba del control, no del ataque.

## Paso 5 — Otras dos policies que la lección ya trae

En la sección 16 hay una `ClusterPolicy` `disallow-privileged` que exige `securityContext.privileged: false`. Aplícala en el mismo clúster y intenta un pod con `privileged: true`. El rechazo es el entregable.

La sección 11.10 nombra el IMDS del nodo (`169.254.169.254`) alcanzable desde los pods. El control que la lección escribe para el cloud está en la lección 09: `aws ec2 modify-instance-metadata-options` con `--http-tokens required`. En kind no hay IMDS. Anota la policy de red o el `http-tokens` como el control, sin hacer la consulta al servicio.

## Cierre

```bash
kind delete cluster --name attack-lab
```

## Entregable

- La salida del `jq` con el binding `oops`.
- El YAML de la policy.
- El error del webhook al recrear el binding.
- El rechazo del pod privilegiado.

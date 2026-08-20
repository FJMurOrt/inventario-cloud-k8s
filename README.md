# Despliegue de mi API del inventario cloud en Kubernetes

Con este proyecto, despliego mi API que desarrollé de inventario cloud, en un clúster de Kubernetes, con réplicas, autohealing, configuración externalizada y autoescalado horizontal.

La imagen que despliego es la que yo mismo desarrollé y desplegué con FastAPI/Docker: [fjmurort/inventario-cloud-api](https://hub.docker.com/r/fjmurort/inventario-cloud-api), construida y publicada automáticamente mediante un pipeline de GitHub Actions. Lo puedes ver en [este repositorio](https://github.com/FJMurOrt/inventario-cloud-fastapi).

---

## ¿Qué he desplegado?

| Objeto | Función |
|--------|---------|
| **Deployment** | 3 réplicas de la API con límites de recursos |
| **Service** | Para exponer la aplicación mediante NodePort y repartir el tráfico entre los pods |
| **ConfigMap** | Para externalizar la configuración y no incluirla en la imagen |
| **HorizontalPodAutoscaler** | Se encarga de escarlar el número de réplicas según el uso de CPU |

---

## Estructura del proyecto

```
inventario-cloud-k8s/
├── yamls/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── hpa.yaml
├── capturas/
└── README.md
```

---

## Requisitos

- Un clúster de Kubernetes (he usado Minikube)
- kubectl configurado
- El addon `metrics-server` habilitado, necesario para el HPA:

```bash
minikube addons enable metrics-server
```

---

## Despliegue

```bash
kubectl apply -f yamls/
```

Y para comprobarlo:

```bash
kubectl get all
```

![1](./capturas/1.png)

En esta salida se ve el Deployment, los pods, el Service con el mapeo `80:30800` y el HPA mostrando el uso actual de CPU frente al objetivo.

---

## Para acceder a la API

```bash
minikube service servicio-inventario
```

La documentación interactiva de la API está disponible en `/docs`.

![2](./capturas/2.png)

![3](./capturas/3.png)

---

## Los tres puertos del Service

| Puerto | Valor | Función |
|--------|-------|---------|
| `nodePort` | 30800 | El puerto de entrada desde fuera del clúster |
| `port` | 80 | El puerto del Service dentro del clúster |
| `targetPort` | 8000 | El puerto donde escucha uvicorn en el contenedor |

---

## Autoescalado

El HorizontalPodAutoscaler mantiene el uso medio de CPU en torno al 60%, con un mínimo de 2 réplicas y un máximo de 6.

```bash
kubectl get hpa
kubectl describe hpa hpa-inventario
```

![4](./capturas/4.png)

Con el clúster en reposo el HPA reduce las réplicas al mínimo, ya que el uso de CPU está muy por debajo del objetivo.

Para que el autoescalado funcione, el Deployment debe definir `resources.requests` de CPU. Es decir, el porcentaje se calcula sobre la CPU que se necesita, no sobre la del nodo.

---

## Autohealing

Si un pod deja de funcionar, el Deployment crea otro automáticamente para mantener el número de réplicas que se hayan definido en el yaml:

```bash
kubectl delete pod <nombre-del-pod>
kubectl get pods
```

![5](./capturas/5.png)

---

## GitOps con ArgoCD

Por otro lado, además del despliegue manual con `kubectl`, he configurado ArgoCD para gestionar el estado del clúster mediante GitOps, de esta manera ArgoCD se encarga de que el clúster refleje el contenido del repositorio.

### Instalación

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Para acceder a la interfaz:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Configuración de la aplicación

| Parámetro | Valor |
|-----------|-------|
| Repositorio | Este mismo repositorio |
| Path | `yamls` |
| Namespace destino | `default` |
| Sync Policy | Automatic, con Prune y Self Heal |

![6](./capturas/6.png)

![7](./capturas/7.png)

### Self Heal en marcha

Con Self Heal activado, cualquier cambio manual que se haga en el clúster se revierte automáticamente para mantener la relación real con el repositorio:

```bash
kubectl scale deployment inventario-api --replicas=5
kubectl get pods
```

Aquí lo que sucede es que en pocos segundos ArgoCD detecta la desviación y devuelve el clúster al estado real del repositorio.

![8](./capturas/8.png)

---

## HPA y GitOps

Al desplegar con ArgoCD hubo un pequeño problema entre el HorizontalPodAutoscaler y la sincronización automática de ArgoCD.

El yaml del Deployment definía `replicas: 3`, pero el HPA reducía las réplicas a 2 al detectar el bajo uso de la CPU. Entonces ArgoCD interpretaba esa diferencia como una diferencia respecto al repositorio y marcaba la aplicación como `OutOfSync`.

La solución ha sido eliminar el campo `replicas` del Deployment. Con el HPA controlando el mínimo de replicas, no es necesario reflejar el número de réplicas el yaml del Deployment. Ya lo lleva a cabo el HPA dentro de los límites de `minReplicas` y `maxReplicas`.

---

## Monitorización con Prometheus y Grafana

Para la parte final dejo laobservabilidad completa al clúster mediante `kube-prometheus-stack`, desplegado y gestionado 100% vía GitOps con ArgoCD.

![13](./capturas/13.png)

![14](./capturas/14.png)

### Arquitectura

- Prometheus + Grafana + Alertmanager desplegados como una aplicación de ArgoCD, y que apuntan directamente al chart oficial de Helm (`prometheus-community/kube-prometheus-stack`)
- La API de inventario (`inventario-cloud-fastapi`) se instrumentó con [`prometheus-fastapi-instrumentator`](https://github.com/trallnag/prometheus-fastapi-instrumentator), exponiendo un endpoint `/metrics` con métricas de latencia, número de requests y códigos de estado
- El `ServiceMonitor` conecta el `Service` de la API con Prometheus y hace scraping automático cada 15s

### Dashboard

Dashboard personalizado en Grafana con 3 paneles:
- Total de peticiones por endpoint
- Tasa de requests por código de estado (2xx, 4xx...)
- Latencia p95 por endpoint

![15](./capturas/15.png)

![16](./capturas/16.png)

![17](./capturas/17.png)

![18](./capturas/18.png)

![19](./capturas/19.png)

![20](./capturas/20.png)

### Problemas encontrados y solucionados

**1. CRDs demasiado grandes para `kubectl apply`**
Los CRDs del chart (`Prometheus`, `Alertmanager`, etc.) superan el límite de 262.144 bytes que permite la sync por defecto de ArgoCD. La solución que se le ha dado es activar `ServerSideApply=true` en las `syncOptions` del aplicacion.yaml.

**2. `ServiceMonitor` sin targets (Prometheus no descubría la API)**
Un problema es que el selector de un `ServiceMonitor` no filtra por las labels de los Pods, sino por las **labels del propio `Service`. El `Service` tenía correctamente el `spec.selector` apuntando a los pods, pero no tenía labels en su `metadata`, así que Prometheus no lo encontraba.
La solución fue añadir añadir entonces al metadata del Service `metadata.labels: app: inventario-api` , además del `spec.selector` que ya tenía.

### Cómo probarlo en local

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```
http://localhost:9090/targets

Y para Grafana
```
minikube service monitoring-grafana -n monitoring
```

## Para eliminar los recursos

```bash
kubectl delete -f yamls/
```
o si te encuentras dentro del directorio

```bash
kubectl delete -f .
```
---

## A tener en cuenta de cara al futuro

**Estado no compartido entre réplicas**
La API usa SQLite dentro del contenedor, por lo que cada pod tiene su propia base de datos, es decir, los datos no se comparten. Lo idea para mejorar esto sería una base de datos externa o con un volumen compartido.

**La configuración no consumida** 
El ConfigMap se inyecta como variables de entorno en el contenedor, pero la aplicación todavía no las lee. Sólo lo incluyo para demostrar el patrón de externalizar la configuración.

---

## Tecnologías que se han usado

- Kubernetes
- ArgoCD
- Minikube
- kubectl
- Docker
- FastAPI

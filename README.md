# MQTT Broker en Kubernetes

Este directorio contiene los manifiestos de Kubernetes para desplegar el broker MQTT Mosquitto, equivalente al setup de Docker Compose del proyecto principal.

## Arquitectura

```
k8s/
├── base/                           # Configuración base
│   ├── namespace.yaml             # Namespace mqtt-broker
│   ├── mosquitto-configmap.yaml   # Configuración de Mosquitto
│   ├── mosquitto-secret.yaml      # Credenciales de usuarios
│   ├── mosquitto-pvc.yaml         # Almacenamiento persistente
│   ├── mosquitto-deployment.yaml  # Deployment principal
│   ├── mosquitto-service.yaml     # Services (ClusterIP y NodePort)
│   └── kustomization.yaml         # Kustomize base
├── overlays/
│   ├── dev/                       # Configuración desarrollo
│   │   ├── deployment-patch.yaml  # Recursos reducidos
│   │   ├── ingress.yaml           # Ingress local
│   │   └── kustomization.yaml     # Kustomize dev
│   └── prod/                      # Configuración producción
│       ├── deployment-patch.yaml  # Recursos aumentados
│       ├── service-patch.yaml     # LoadBalancer
│       ├── ingress.yaml           # Ingress con TLS
│       ├── hpa.yaml               # Horizontal Pod Autoscaler
│       └── kustomization.yaml     # Kustomize prod
```

## Requisitos

- Kubernetes cluster (v1.20+)
- kubectl configurado
- kustomize (incluido en kubectl 1.14+)
- StorageClass "standard" disponible
- (Opcional) NGINX Ingress Controller
- (Opcional) cert-manager para TLS

## Configuración

### Usuarios y Contraseñas

Los usuarios están definidos en `base/mosquitto-secret.yaml` (codificado en base64):
- `acme_desktop`
- `acme_gateway` 
- `acme_server`
- `user`
- `frankgallo`

Para agregar nuevos usuarios:
1. Generar hash con: `mosquitto_passwd -c temp_passwd usuario`
2. Codificar en base64: `base64 temp_passwd`
3. Actualizar el Secret

### Almacenamiento

- **Datos**: 1GB para persistencia del broker
- **Logs**: 500MB para archivos de log
- Usa PersistentVolumeClaims con StorageClass "standard"

### Puertos

- **1883**: MQTT TCP
- **9001**: WebSockets
- **31883**: NodePort MQTT (desarrollo)
- **31001**: NodePort WebSockets (desarrollo)

## Configuración por Entorno

### Desarrollo (`overlays/dev/`)
- 1 réplica
- Recursos mínimos (32Mi RAM, 25m CPU)
- NodePort para acceso directo
- Ingress local (mqtt-dev.local)

### Producción (`overlays/prod/`)
- 2+ réplicas con HPA
- Recursos aumentados (256Mi RAM, 200m CPU)
- LoadBalancer para acceso externo
- Ingress con TLS automático
- Autoescalado basado en CPU/memoria

## Acceso al Broker

### Desarrollo
```bash
# Port forwarding
kubectl port-forward svc/dev-mosquitto-service 1883:1883 -n mqtt-broker

# O usando NodePort
mqtt://localhost:31883
ws://localhost:31001
```

### Producción
```bash
# LoadBalancer IP
kubectl get svc prod-mosquitto-nodeport -n mqtt-broker

# O via Ingress
mqtt://mqtt.yourdomain.com:1883
wss://mqtt.yourdomain.com/mqtt
```

## Monitoreo y Troubleshooting

```bash
# Ver estado general
kubectl get all -n mqtt-broker

# Logs detallados
kubectl logs -f deployment/mosquitto -n mqtt-broker

# Eventos del namespace
kubectl get events -n mqtt-broker --sort-by='.lastTimestamp'

# Describir problemas
kubectl describe pod <pod-name> -n mqtt-broker

# Acceder al contenedor
kubectl exec -it <pod-name> -n mqtt-broker -- sh
```

## Personalización

Para personalizar la configuración:

1. **Modificar base**: Editar archivos en `base/`
2. **Crear overlay**: Nuevo directorio en `overlays/`
3. **Usar patches**: Strategic merge o JSON patches
4. **Variables**: Usar replacements en kustomization.yaml

Ejemplo de nuevo overlay:
```yaml
# overlays/staging/kustomization.yaml
bases:
  - ../../base
namePrefix: staging-
replicas:
  - name: mosquitto
    count: 2
```

## Backup y Restauración

```bash
# Backup de datos
kubectl exec deployment/mosquitto -n mqtt-broker -- tar czf - /mosquitto/data | \
  gzip > mosquitto-backup-$(date +%Y%m%d).tar.gz

# Restaurar datos
kubectl exec -i deployment/mosquitto -n mqtt-broker -- tar xzf - -C / < backup.tar.gz
```

## Seguridad

- Secrets para credenciales (no plaintext)
- RBAC en el namespace
- Network policies para aislar tráfico
- TLS terminación en Ingress
- Non-root containers (UID 1883)

## Soporte

Para problemas específicos de Kubernetes:
1. Revisar eventos: `kubectl get events -n mqtt-broker`
2. Validar configuración: `kubectl describe -n mqtt-broker`
3. Port forward para debug local: `make port-forward-mqtt`
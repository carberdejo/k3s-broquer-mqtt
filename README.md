# MQTT Broker en Kubernetes

Manifiestos de Kubernetes para desplegar el broker MQTT Mosquitto en k3s usando Kustomize.

## Estructura del repositorio

```
dpworld_rtls_iot_mqtt/
├── base/                           # Configuración base (aplica a todos los entornos)
│   ├── namespace.yaml              # Namespace: broker-mqtt-acme
│   ├── mosquitto-configmap.yaml    # mosquitto.conf: listeners, TLS, auth
│   ├── mosquitto-secret.yaml       # Secret mosquitto-password: credenciales de usuarios
│   ├── mosquitto-pvc.yaml          # PVCs: datos (1Gi) y logs (500Mi)
│   ├── mosquitto-deployment.yaml   # Deployment principal
│   ├── mosquitto-service.yaml      # Services (ClusterIP y NodePort)
│   ├── tcp-services-configmap.yaml # ConfigMap para TCP ingress
│   └── kustomization.yaml
├── overlays/
│   ├── dev/                        # Entorno desarrollo
│   │   ├── deployment-patch.yaml   # Recursos reducidos, 1 réplica
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   └── prod/                       # Entorno producción
│       ├── deployment-patch.yaml   # Recursos aumentados, 2+ réplicas
│       ├── service-patch.yaml      # LoadBalancer
│       ├── ingress.yaml
│       ├── hpa.yaml                # Horizontal Pod Autoscaler
│       └── kustomization.yaml
└── certs/                          # Plantilla de referencia (ver sección TLS)
    ├── san.cnf
    ├── ca.key / ca.crt / ca.srl
    ├── server.csr / server.crt / server.key
```

## Namespace

Todos los recursos viven en el namespace `broker-mqtt-acme`.

## Puertos

| Puerto | Protocolo | Descripción |
|--------|-----------|-------------|
| 1883   | MQTT TCP  | Conexión estándar |
| 9002   | WebSockets | Conexión via WebSocket |
| 8883   | MQTT TLS  | Conexión cifrada con certificados |

## Prerequisitos

- k3s / k3d o cualquier cluster Kubernetes (v1.20+)
- `kubectl` configurado apuntando al cluster
- `kustomize` (incluido en `kubectl` 1.14+)
- StorageClass `local-path` disponible (default en k3s)
- Secret `mosquitto-tls` creado manualmente (ver sección TLS)

## Despliegue

```bash
# Desarrollo
kubectl apply -k overlays/dev/

# Producción
kubectl apply -k overlays/prod/
```

## Certificados TLS y el Secret `mosquitto-tls`

### Cómo funciona

El broker Mosquitto expone el puerto 8883 con TLS. Los certificados no viven en archivos del repositorio en tiempo de ejecución — viven en un **Secret de Kubernetes** llamado `mosquitto-tls`.

Un Secret puede almacenar múltiples archivos. Internamente es un mapa clave-valor donde la clave es el nombre del archivo y el valor es el contenido en base64:

```yaml
# Representación interna del Secret mosquitto-tls
apiVersion: v1
kind: Secret
metadata:
  name: mosquitto-tls
  namespace: broker-mqtt-acme
type: Opaque
data:
  ca.crt:     <base64>
  server.crt: <base64>
  server.key:  <base64>
```

El deployment monta ese Secret como directorio en el pod:

```yaml
volumes:
- name: mosquitto-tls
  secret:
    secretName: mosquitto-tls  # nombre del Secret en k8s
    defaultMode: 0400

volumeMounts:
- name: mosquitto-tls
  mountPath: /mosquitto/certs  # cada clave se convierte en un archivo aquí
  readOnly: true
```

Resultado: el pod ve exactamente esto en su filesystem:

```
/mosquitto/certs/
├── ca.crt
├── server.crt
└── server.key
```

Y `mosquitto.conf` los referencia por ruta:

```
listener 8883
cafile   /mosquitto/certs/ca.crt
certfile /mosquitto/certs/server.crt
keyfile  /mosquitto/certs/server.key
```

### Crear el Secret desde terminal

El Secret se crea **una sola vez desde la terminal** con los archivos de certificados. No es parte del `kubectl apply` ni del flujo de Kustomize:

```bash
kubectl create secret generic mosquitto-tls \
  --from-file=ca.crt=<ruta>/ca.crt \
  --from-file=server.crt=<ruta>/server.crt \
  --from-file=server.key=<ruta>/server.key \
  -n broker-mqtt-acme
```

Una vez creado, el Secret persiste en el cluster independientemente del repositorio.

### Sobre la carpeta `certs/`

La carpeta `certs/` en este repositorio es una **plantilla de referencia**: contiene los archivos que se generaron al crear los certificados originales (incluyendo `san.cnf` para saber cómo regenerarlos). **No es necesaria para el despliegue** — el broker solo lee el Secret `mosquitto-tls` que ya está en k8s.

> Los archivos reales (`ca.key`, `server.key`) son claves privadas. No deben subirse a ningún repositorio remoto. Guarda una copia segura fuera del repo por si necesitas recrear el Secret.

### Generar certificados nuevos

Si necesitas regenerar los certificados (nuevo cluster, vencimiento, etc.):

```bash
# 1. Generar CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt \
  -subj "/CN=ACME MQTT CA"

# 2. Generar clave y CSR del servidor (usando san.cnf para los SANs)
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config san.cnf

# 3. Firmar el certificado con la CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 730 -extensions req_ext \
  -extfile san.cnf

# 4. Recrear el Secret en k8s
kubectl delete secret mosquitto-tls -n broker-mqtt-acme
kubectl create secret generic mosquitto-tls \
  --from-file=ca.crt=ca.crt \
  --from-file=server.crt=server.crt \
  --from-file=server.key=server.key \
  -n broker-mqtt-acme
```

## Usuarios MQTT

Los usuarios se definen en el Secret `mosquitto-password` (en `base/mosquitto-secret.yaml`).

Para agregar un usuario nuevo:

```bash
# 1. Generar el hash de la contraseña
mosquitto_passwd -c temp_passwd <usuario>

# 2. Codificar en base64
base64 temp_passwd

# 3. Actualizar base/mosquitto-secret.yaml con el nuevo valor y reaplicar
kubectl apply -k overlays/<entorno>/
```

## Almacenamiento persistente

Dos PVCs con StorageClass `local-path` (default de k3s):

| PVC | Tamaño | Descripción |
|-----|--------|-------------|
| `mosquitto-data-pvc` | 1Gi | Persistencia del broker (retained messages, etc.) |
| `mosquitto-log-pvc`  | 500Mi | Archivos de log de Mosquitto |

Los datos persisten entre reinicios y actualizaciones del pod.

## Troubleshooting

```bash
# Estado general del namespace
kubectl get all -n broker-mqtt-acme

# Logs del broker en tiempo real
kubectl logs -f deployment/mosquitto -n broker-mqtt-acme

# Eventos ordenados por tiempo
kubectl get events -n broker-mqtt-acme --sort-by='.lastTimestamp'

# Detalles de un pod específico
kubectl describe pod <pod-name> -n broker-mqtt-acme

# Shell dentro del contenedor
kubectl exec -it <pod-name> -n broker-mqtt-acme -- sh

# Verificar que el Secret TLS existe
kubectl get secret mosquitto-tls -n broker-mqtt-acme

# Ver las claves del Secret (sin revelar el contenido)
kubectl describe secret mosquitto-tls -n broker-mqtt-acme
```

## Acceso al broker

```bash
# Port forwarding para pruebas locales
kubectl port-forward svc/mosquitto-service 1883:1883 -n broker-mqtt-acme

# Verificar conectividad (requiere mosquitto-clients)
mosquitto_pub -h localhost -p 1883 -u <usuario> -P <password> \
  -t test/ping -m "hello"
```

## Seguridad

- Autenticación obligatoria (`allow_anonymous false`)
- Contraseñas almacenadas como hash en un Secret (no plaintext)
- TLS en el puerto 8883 con certificados propios
- Contenedor corre con UID 1883 (non-root)
- Claves privadas nunca deben estar en repositorios remotos

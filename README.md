# Tinman k3s

Este proyecto despliega una instalación de Kubernetes con k3s para ejecutar Ollama con acceso a GPU NVIDIA y Open WebUI detrás de Traefik con HTTPS mediante cert-manager.


## Arquitectura

- k3s como clúster Kubernetes
- NVIDIA Container Toolkit + runtime de containerd
- RuntimeClass `nvidia` para Pods que necesitan GPU
- DaemonSet `nvidia-device-plugin` para exponer los dispositivos NVIDIA
- Deployment de `ollama` con acceso a GPU
- Deployment de `open-webui` para la interfaz web
- Ingress con Traefik y cert-manager para TLS

## Requisitos previos

Antes de desplegar en k3s necesitas que el host tenga:

- NVIDIA driver instalado correctamente
- CUDA y toolkit de NVIDIA instalados en el host
- `containerd` o el `containerd` que usa k3s configurado para soportar runtime NVIDIA

## 1) Configurar la GPU en el sistema operativo

Comprueba que la GPU esté detectada por el host:

```bash
nvidia-smi
```

Si no usa el zip con la misma versión que el host para instalar el driver

## 2) Preparar el runtime NVIDIA para containerd / k3s

Para que los contenedores puedan ver la GPU, debes instalar el toolkit de NVIDIA:

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

Luego configura el runtime para containerd:

```bash
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

Se puede verificar cuando al menos sale un "1" en el siguiente comando:

```bash
kubectl describe node <nombre-del-nodo-con-gpu> | grep -i nvidia.com/gpu
```

## 3) Instalar k3s

Instala k3s con el runtime del sistema disponible:

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

Comprueba que el nodo esté listo:

```bash
kubectl get nodes
```


## 4) Instalar el plugin de NVIDIA en Kubernetes

Este proyecto incluye el DaemonSet del plugin de dispositivos NVIDIA:

```bash
kubectl apply -f nvidia-device-plugin.yaml
```

Eso permite que Kubernetes vea la GPU como recurso `nvidia.com/gpu`.

## Paso 5: Verificación de la GPU en el Clúster

Verificar asignación de GPU en el nodo:

   ```bash
   kubectl describe node <nombre-del-nodo> | grep -i nvidia.com/gpu
   ```

   *Resultado esperado:* Deberías ver `nvidia.com/gpu: 1` (o la cantidad de GPUs instaladas) en las secciones `Capacity` y `Allocatable`.

Verificar la RuntimeClass:

   ```bash
   kubectl get runtimeclass
   ```

   *Resultado esperado:* Debe aparecer el recurso `nvidia`.

## Paso 6: Despliegue de Ollama y Open WebUI

Una vez validada la disponibilidad de la GPU en Kubernetes, crea el namespace e instancia las aplicaciones:

```bash
# Crear namespace dedicado
kubectl create namespace ollama

# Desplegar Ollama (requiere asignación de GPU y RuntimeClass nvidia)
kubectl apply -f ollama.yaml

# Desplegar interfaz gráfica Open WebUI
kubectl apply -f open-webui.yaml
```

## Paso 7: Configuración de Ingress y Cifrado TLS

Configura cert-manager para la emisión de certificados SSL/TLS y Traefik para el enrutamiento:

```bash
# Registrar el emisor de certificados
kubectl apply -f cluster-issuer.yaml

# Aplicar reglas de Ingress
kubectl apply -f ingress.yaml
```
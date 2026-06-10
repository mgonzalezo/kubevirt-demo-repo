# KubeVirt Demo — Validación con Amazon Q Developer

Demo de validación automática de manifiestos KubeVirt usando Amazon Q Developer en GitHub.

**Sesión:** De VirtualBox a KubeVirt — Essential Week UNAC 2026

---

## Qué hace este repositorio

Contiene manifiestos YAML de KubeVirt (máquinas virtuales declarativas). Cuando se crea un Pull Request, **Amazon Q Developer** revisa automáticamente los archivos YAML y comenta errores de configuración directamente en el PR.

**Costo:** $0 — Amazon Q Developer es gratuito (50 reviews/mes).

---

## Setup: Instalar Amazon Q Developer en este repo

### Paso 1 — Instalar la GitHub App

1. Ir a [github.com/apps/amazon-q-developer](https://github.com/apps/amazon-q-developer)
2. Clic **Install**
3. Seleccionar este repositorio (`kubevirt-demo-repo`)
4. Aceptar los permisos — no requiere cuenta de AWS

### Paso 2 — Verificar que funciona

Una vez instalado, Amazon Q aparece como un reviewer automático en cada Pull Request. No necesitas configurar nada más.

---

## Demo: Encontrar un error en un YAML de KubeVirt

### 1. El YAML correcto (`vm-demo.yaml`)

Este manifiesto crea una VM funcional con CirrOS:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: mi-primera-vm
spec:
  running: true
  template:
    spec:
      domain:
        resources:
          requests:
            memory: 128Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo
```

### 2. El YAML con error intencional (`vm-con-error.yaml`)

Este manifiesto declara una interfaz SR-IOV pero **omite** la entrada correspondiente en `spec.template.spec.networks` y no tiene un `NetworkAttachmentDefinition`:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-sriov-demo
spec:
  running: true
  template:
    spec:
      domain:
        resources:
          requests:
            memory: 128Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          interfaces:
          - name: sriov-net
            sriov: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo
```

**Errores intencionales:**
- `interfaces` declara `sriov-net` con tipo `sriov: {}`, pero no hay entrada en `networks` que lo mapee
- No hay `NetworkAttachmentDefinition` referenciado para la red SR-IOV
- Sin `networks`, KubeVirt no puede asignar la interfaz de red y el Pod `virt-launcher` fallará

### 3. Ejecutar la demo (copiar y pegar)

```bash
# Clonar el repo
git clone https://github.com/<TU_USUARIO>/kubevirt-demo-repo.git
cd kubevirt-demo-repo

# Crear el YAML correcto en main
cat > vm-demo.yaml << 'EOF'
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: mi-primera-vm
spec:
  running: true
  template:
    spec:
      domain:
        resources:
          requests:
            memory: 128Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo
EOF

git add vm-demo.yaml
git commit -m "feat: add base KubeVirt VM manifest"
git push origin main

# Crear branch con el YAML roto
git checkout -b demo/yaml-con-error

cat > vm-con-error.yaml << 'EOF'
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-sriov-demo
spec:
  running: true
  template:
    spec:
      domain:
        resources:
          requests:
            memory: 128Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          interfaces:
          - name: sriov-net
            sriov: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo
EOF

git add vm-con-error.yaml
git commit -m "feat: add SR-IOV VM configuration"
git push origin demo/yaml-con-error

# Crear el Pull Request
gh pr create \
  --title "feat: add SR-IOV VM configuration" \
  --body "Adding a VM with SR-IOV networking for high-performance workloads." \
  --base main \
  --head demo/yaml-con-error
```

### 4. Resultado esperado

Amazon Q Developer revisa el PR automáticamente (~30-60 segundos) y comenta:

- La interfaz `sriov-net` no tiene una entrada correspondiente en `spec.template.spec.networks`
- Falta un `NetworkAttachmentDefinition` para la red SR-IOV
- Sugiere el parche con la sección `networks` correcta

---

## Arquitectura avanzada: Lambda + Amazon Bedrock

Para validación más profunda con IA generativa, puedes crear un pipeline serverless:

```
GitHub PR (webhook)
    └── AWS Lambda (free tier: 1M requests/mes)
         └── Amazon Bedrock — Nova Micro ($0.035/1M tokens)
              prompt: "Eres un experto en KubeVirt. Valida este YAML..."
         └── Publica resultado como comentario en el PR
```

**Costo:** ~$0 con créditos de cuenta nueva ($200 USD).

---

## Recursos

- [KubeVirt Documentation](https://kubevirt.io/user-guide/)
- [Amazon Q Developer](https://aws.amazon.com/q/developer/)
- [OpenShift Virtualization Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/virtualization/about)

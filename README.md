# Despliegue de Infraestructura y Aplicación con Terraform y Ansible en Azure

Este proyecto automatiza la creación de infraestructura en la nube (Azure) utilizando **Terraform** y la configuración del servidor utilizando **Ansible**. Como prueba de concepto, despliega una máquina virtual con Ubuntu, instala Docker y ejecuta un contenedor del clásico juego Super Mario Bros.

Terraform describe el estado deseado y usa el proveedor de Azure para crear, consultar y eliminar esos recursos, mientras que Ansible se encarga del aprovisionamiento interno del sistema operativo.

## Recursos Creados en Azure

- Un grupo de recursos en `canadacentral`.
- Una red virtual y una subred.
- Una IP pública estática.
- Una interfaz de red conectada a la subred y a la IP pública.
- Una máquina virtual Ubuntu 22.04 LTS (`Standard_B1s`).
- Un grupo de seguridad de red (NSG) con acceso SSH (puerto 22) y HTTP/Custom (puerto 80/8787).

## Requisitos

- Una suscripción activa de Azure con permisos para crear recursos.
- [Terraform](https://developer.hashicorp.com/terraform/downloads) 1.1.0 o posterior.
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli), para iniciar sesión desde la terminal.

## Archivos Principales

**Infraestructura (Terraform):**
- `main.tf`: proveedor, recursos, variables de usuario y contraseña, y salida de la IP pública.
- `.terraform.lock.hcl`: versiones verificadas del proveedor; se conserva en Git.
- `terraform.tfvars.example`: ejemplo de configuración de variables, sin una contraseña real.
- `terraform.tfvars`: valores locales; está excluido de Git por seguridad.
- `.terraform/`: plugins descargados por Terraform; se genera localmente y está excluido de Git.
- `terraform.tfstate`: estado local de los recursos administrados; está excluido de Git.

**Configuración (Ansible):**
- `inventory/hosts.ini`: Define la IP de la máquina virtual y los datos de conexión SSH.
- `playbooks/install_docker.yml`: Playbook principal que instala Docker y despliega el contenedor de la aplicación.

## Ejecución Paso a Paso

### 1. Autenticación en Azure
Inicia sesión y, si tienes más de una suscripción, selecciona la que usarás:
```bash
az login
az account set --subscription "ID_O_NOMBRE_DE_SUSCRIPCION"
```

### 2. Configuración de Terraform
Edita en terraform.tfvars con tus credenciales:

cp terraform.tfvars.example terraform.tfvars

Terraform
admin_username = "admin_user"
admin_password = "REEMPLAZA_POR_UNA_CONTRASENA_SEGURA"
(Asegúrate de que terraform.tfvars esté excluido en tu .gitignore).

### 3. Iniciar y Crear la Infraestructura (Terraform)
Ejecuta los siguientes comandos en la raíz del proyecto para descargar los plugins de Azure, verificar los recursos a crear y aplicar los cambios:

```Bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply -auto-approve
```
Al finalizar, Terraform mostrará la IP pública de tu nueva máquina virtual. Cópiala.

### 4. Configurar el Inventario (Ansible)
Abre el archivo inventory/hosts.ini y actualiza la dirección IP con el valor que arrojó Terraform:

Ini, TOML
[azure_vm]
TU_IP_PUBLICA ansible_user=admin_user

### 5. Ejecutar los Playbooks (Ansible)
Aprovisiona el servidor, instala Docker y levanta el contenedor ejecutando el playbook principal:

```Bash
ansible-playbook -i inventory/hosts.ini playbooks/install_docker.yml
```

### 6. Acceder a la Aplicación
Una vez que Ansible termine con éxito, abre tu navegador web e ingresa la IP pública de tu servidor, especificando el puerto 8787:

`http://TU_IP_PUBLICA:8787`

### Limpieza de Recursos (Destrucción)
Para eliminar toda la infraestructura y evitar cargos innecesarios en Azure, ejecuta:

```Bash
terraform plan -destroy
terraform destroy
```
Espera a que Terraform confirme la eliminación completa. No borres manualmente los archivos de estado.

### Aprendizajes y Troubleshooting (Solución de Problemas)
**Sincronización de Puertos (Azure NSG vs Docker):** Los puertos mapeados en los playbooks de Ansible (ej. ports: - "8787:8080") deben estar explícitamente permitidos en el Network Security Group (NSG) de Terraform. Se usó la propiedad destination_port_ranges = ["80", "8787"] para abrir múltiples puertos en una misma regla.

**Diagnóstico SSH (Connection timed out):** Si Ansible falla con un error de Timeout en el puerto 22, indica un bloqueo de red a nivel de firewall en Azure (el NSG no permite la entrada o no está enlazado a la interfaz de red), no un error de contraseña.

**Advertencia REMOTE HOST IDENTIFICATION HAS CHANGED:** Al destruir y recrear máquinas en la nube (terraform destroy y luego apply), el servidor genera una firma criptográfica distinta aunque mantenga la misma IP. Limpia el registro local ejecutando ssh-keygen -R 'TU_IP_PUBLICA' para evitar que SSH bloquee la conexión por seguridad.

**Sintaxis Inline en Terraform:** Al definir reglas de seguridad dentro del recurso azurerm_network_security_group (bloque inline), Terraform hereda implícitamente el contexto. Declarar variables como resource_group_name dentro de ese bloque causa errores de compilación (Unsupported argument).
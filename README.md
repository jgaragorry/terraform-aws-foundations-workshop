# Workshop: Fundamentos de Terraform en AWS desde Cero

<img src="https://img.icons8.com/?size=100&id=kEkT1u7zTDk5&format=png&color=000000" alt="Terraform AWS Banner" width="25"/> Este workshop está diseñado para guiar a principiantes absolutos en la creación de infraestructura en AWS utilizando Terraform. Partiremos desde un entorno limpio en Ubuntu 24.04 LTS (vía WSL2) y construiremos una base sólida aplicando las mejores prácticas de **DevOps, DevSecOps, SRE y FinOps** desde el primer día.

Al finalizar, no solo habrás desplegado una instancia EC2, sino que entenderás el propósito de cada línea de código, cómo estructurar un proyecto modular y seguro, y cómo gestionar los costos de tu infraestructura.

---

### 📚 **Tabla de Contenidos**

1.  [**Objetivos de Aprendizaje**](#-objetivos-de-aprendizaje)
2.  [**Arquitectura del Proyecto**](#-arquitectura-del-proyecto)
3.  [**Prerrequisitos**](#-prerrequisitos)
4.  [**Paso 1: Configuración del Entorno Local**](#-paso-1-configuración-del-entorno-local-wsl2--ubuntu)
5.  [**Paso 2: Estructura y Código del Proyecto Terraform**](#-paso-2-estructura-y-código-del-proyecto-terraform)
    * [Estructura de Ficheros](#estructura-de-ficheros)
    * [Módulo `ec2-instance`](#módulo-ec2-instance)
    * [Archivos Raíz](#archivos-raíz)
6.  [**Paso 3: El Ciclo de Vida de Terraform en Acción**](#-paso-3-el-ciclo-de-vida-de-terraform-en-acción)
7.  [**Paso 4: Verificación y Conexión**](#-paso-4-verificación-y-conexión)
8.  [**Paso 5: Limpieza de Recursos (Práctica Crítica de FinOps)**](#-paso-5-limpieza-de-recursos-práctica-crítica-de-finops)
9.  [**Guía Resumen (How-To) de lo Aprendido**](#-guía-resumen-how-to-de-lo-aprendido)
10. [**Próximos Pasos y Desafíos**](#-próximos-pasos-y-desafíos)

---

### 🎯 **Objetivos de Aprendizaje**

Al completar este workshop, serás capaz de:

* **Configurar un entorno de desarrollo** con AWS CLI y Terraform en Ubuntu.
* **Escribir y estructurar código de Terraform** de forma modular y mantenible.
* **Utilizar variables** para crear infraestructura flexible y reutilizable.
* **Buscar recursos dinámicamente** (como AMIs) para evitar el "hardcoding".
* **Implementar prácticas de seguridad básicas** (DevSecOps) como Grupos de Seguridad y gestión de claves.
* **Aplicar etiquetado (tagging)** para la gobernanza y control de costos (FinOps).
* **Comprender y ejecutar el ciclo de vida de Terraform** (`init`, `plan`, `apply`, `destroy`).
* **Gestionar la limpieza de recursos** para evitar costos inesperados en la nube.

---

### 🏗️ **Arquitectura del Proyecto**

Desplegaremos una arquitectura simple pero robusta en AWS, que consiste en:

```
+--------------------------------------------------+
| Región AWS (ej: us-east-1)                       |
|                                                  |
|  +--------------------------------------------+  |
|  | VPC (Default)                              |  |
|  |                                            |  |
|  |  +------------------+                      |  |
|  |  | Security Group   |                      |  |
|  |  | (Allow SSH)      |                      |  |
|  |  +-------+----------+                      |  |
|  |          |                                 |  |
|  |          | Applies to                      |  |
|  |          |                                 |  |
|  |  +-------v----------+                      |  |
|  |  | EC2 Instance     |                      |  |
|  |  | (Amazon Linux 2) |                      |  |
|  |  | Type: t3.micro   |                      |  |
|  |  | KeyPair: attached|                      |  |
|  |  +------------------+                      |  |
|  |                                            |  |
|  +--------------------------------------------+  |
|                                                  |
+--------------------------------------------------+
       ^
       |
+------+----------------+
| Internet              |
| (Tu IP pública)       |-- SSH (Puerto 22) --> EC2
+-----------------------+
```

---

### ✅ **Prerrequisitos**

Antes de comenzar, asegúrate de tener lo siguiente:

1.  **Una cuenta de AWS:** Si no tienes una, puedes crearla [aquí](https://aws.amazon.com/free/).
    * **Mejor Práctica (Seguridad):** **NO uses tu usuario root.** Crea un usuario IAM con permisos de administrador (`AdministratorAccess`) y genera unas claves de acceso (Access Key ID y Secret Access Key) para ese usuario. [Sigue esta guía oficial](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html).
2.  **WSL2 con Ubuntu 24.04 LTS:** Este workshop está diseñado para Windows Subsystem for Linux. [Guía de instalación de WSL](https://learn.microsoft.com/es-es/windows/wsl/install).
3.  **Un par de claves (Key Pair) de EC2 en AWS:**
    * Ve a la consola de AWS -> EC2 -> Network & Security -> Key Pairs.
    * Crea un nuevo par de claves (ej: `terraform-workshop-key`), tipo `RSA` y formato `.pem`.
    * **Guarda el archivo `.pem` en un lugar seguro.** Lo necesitaremos para conectarnos a la instancia. Te recomiendo moverlo a `~/.ssh/` dentro de tu entorno Ubuntu/WSL.

---

### 🔬 **Paso 1: Configuración del Entorno Local (WSL2 + Ubuntu)**

Abre tu terminal de Ubuntu en WSL y sigue estos pasos.

#### 1.1. Actualizar el Sistema

* **Propósito:** Asegurar que todos los paquetes del sistema operativo estén en su última versión para evitar conflictos.
* **Comando:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
* **Verificación:** El comando debe completarse sin errores.

#### 1.2. Instalar Utilidades Básicas

* **Propósito:** `unzip` es necesario para descomprimir el instalador de AWS CLI, y `curl` para descargarlo. `jq` es un procesador de JSON muy útil para trabajar con la salida de la CLI.
* **Comando:**
    ```bash
    sudo apt install -y unzip curl jq
    ```
* **Verificación:** Ejecuta `unzip --version`, `curl --version` y `jq --version`.

#### 1.3. Instalar AWS CLI v2

* **Propósito:** La Interfaz de Línea de Comandos de AWS nos permite interactuar con nuestra cuenta de AWS desde la terminal. Es esencial para configurar las credenciales que Terraform usará.
* **Comandos:**
    ```bash
      curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
      unzip awscliv2.zip
      sudo ./aws/install
      rm -rf aws awscliv2.zip # Limpiamos los archivos de instalación
    ```
* **Verificación:**
    ```bash
    aws --version
    # Salida esperada (la versión puede variar): aws-cli/2.15.40 Python/3.11.8 Linux/5.15.153.1-microsoft-standard-WSL2 x86_64
    ```

#### 1.4. Configurar Credenciales de AWS CLI

* **Propósito:** Este paso crea un archivo (`~/.aws/credentials`) que almacena de forma segura tus claves de acceso. Terraform buscará automáticamente en esta ubicación para autenticarse con AWS.
* **Comando:**
    ```bash
    aws configure
    ```
* **Acción:** Se te solicitará la siguiente información. Usa las credenciales del usuario IAM que creaste en los prerrequisitos.
    * `AWS Access Key ID`: `[TU_ACCESS_KEY]`
    * `AWS Secret Access Key`: `[TU_SECRET_ACCESS_KEY]`
    * `Default region name`: `us-east-1` (o la región que prefieras).
    * `Default output format`: `json`
* **Verificación:**
    ```bash
    # Este comando debería devolver información sobre tu usuario IAM, no un error.
    aws sts get-caller-identity
    ```

#### 1.5. Instalar Terraform

* **Propósito:** Instalar la herramienta de Infraestructura como Código de HashiCorp.
* **Comandos:**
    ```bash
    # 1. Añadir la clave GPG de HashiCorp
    wget -O- [https://apt.releases.hashicorp.com/gpg](https://apt.releases.hashicorp.com/gpg) | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

    # 2. Añadir el repositorio oficial de HashiCorp
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] [https://apt.releases.hashicorp.com](https://apt.releases.hashicorp.com) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

    # 3. Instalar Terraform
    sudo apt update && sudo apt install terraform -y
    ```
* **Verificación:**
    ```bash
    terraform -version
    # Salida esperada (la versión puede variar): Terraform v1.9.0
    ```

¡Felicidades! Tu entorno local está listo para empezar a construir infraestructura.

---

### 📂 **Paso 2: Estructura y Código del Proyecto Terraform**

Ahora, crearemos la estructura de nuestro proyecto. La modularidad es una práctica clave de DevOps que promueve la reutilización y el mantenimiento del código.

#### Estructura de Ficheros

Crea el siguiente árbol de directorios y ficheros. Puedes usar los comandos `mkdir -p` y `touch`.

```
terraform-aws-foundations-workshop/
├── README.md                 # Este mismo archivo
├── provider.tf               # Configuración del proveedor (AWS)
├── variables.tf              # Variables de entrada para el proyecto raíz
├── terraform.tfvars          # Valores para las variables (no subir a git si contiene secretos)
├── main.tf                   # Lógica principal, donde se llama a los módulos
├── outputs.tf                # Valores de salida (ej: IP de la instancia)
├── data.tf                   # Para buscar datos existentes en AWS (como la AMI)
├── security.tf               # Definición de recursos de seguridad (Security Group)
└── modules/
    └── ec2-instance/
        ├── main.tf           # Lógica del módulo (crea la instancia EC2)
        ├── variables.tf      # Variables de entrada para el módulo
        └── outputs.tf        # Valores de salida del módulo
```

Ahora, vamos a llenar cada archivo con el código mejorado.

---

#### Módulo `ec2-instance`

Este módulo es una "caja negra" reutilizable cuyo único propósito es crear una instancia EC2.

**`modules/ec2-instance/variables.tf`**
```hcl
# Propósito: Definir todas las variables de entrada que este módulo necesita para funcionar.

variable "instance_name" {
  description = "Nombre para la instancia EC2 y etiqueta 'Name'."
  type        = string
}

variable "instance_type" {
  description = "Tipo de instancia EC2 (ej: t3.micro)."
  type        = string
  default     = "t3.micro"
}

variable "ami_id" {
  description = "ID de la Amazon Machine Image (AMI) a utilizar."
  type        = string
}

variable "key_name" {
  description = "Nombre del Key Pair de EC2 para permitir el acceso SSH."
  type        = string
}

variable "vpc_security_group_ids" {
  description = "Lista de IDs de Security Groups a asociar a la instancia."
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Mapa de etiquetas adicionales para aplicar a la instancia."
  type        = map(string)
  default     = {}
}
```

**`modules/ec2-instance/main.tf`**
```hcl
# Propósito: Contiene la lógica principal para crear el recurso, en este caso, una aws_instance.

resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  vpc_security_group_ids = var.vpc_security_group_ids

  # Mejor Práctica (FinOps/Gobernanza): El etiquetado es crucial.
  # Usamos 'merge' para combinar etiquetas por defecto con las personalizadas.
  tags = merge(
    {
      "Name" = var.instance_name
    },
    var.tags
  )

  # Mejor Práctica (SRE): El ciclo de vida previene la destrucción accidental.
  # Para un workshop, lo dejamos en 'false', pero en producción sería 'true'.
  lifecycle {
    prevent_destroy = false
  }
}
```

**`modules/ec2-instance/outputs.tf`**
```hcl
# Propósito: Exponer atributos de los recursos creados por este módulo para que puedan ser usados en el proyecto raíz.

output "instance_id" {
  description = "ID de la instancia EC2 creada."
  value       = aws_instance.this.id
}

output "instance_public_ip" {
  description = "Dirección IP pública de la instancia EC2."
  value       = aws_instance.this.public_ip
}
```

---

#### Archivos Raíz

Estos archivos orquestan el despliegue y configuran el entorno general.

**`provider.tf`**
```hcl
# Propósito: Definir los proveedores de nube (en este caso, AWS) y sus versiones requeridas.
# Mejor Práctica (DevOps): Fijar las versiones del proveedor y de Terraform asegura la consistencia entre ejecuciones.

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.3.0"
}

provider "aws" {
  region = var.aws_region
}
```

**`variables.tf`**
```hcl
# Propósito: Definir las variables de entrada para todo el proyecto.

variable "aws_region" {
  description = "Región de AWS donde se desplegarán los recursos."
  type        = string
  default     = "us-east-1"
}

variable "instance_name" {
  description = "Nombre para la instancia EC2 del workshop."
  type        = string
  default     = "terraform-workshop-instance"
}

variable "key_pair_name" {
  description = "Nombre del Key Pair existente en AWS para el acceso SSH."
  type        = string
}

variable "allowed_ssh_cidr" {
  description = "Bloque CIDR de IP permitidas para conectarse por SSH. Por defecto, tu IP actual."
  type        = list(string)
  default     = [] # Dejar vacío para que se autodetecte
}

variable "common_tags" {
  description = "Etiquetas comunes a aplicar a todos los recursos para FinOps y gobernanza."
  type        = map(string)
  default = {
    "Project"     = "TerraformFoundations"
    "Environment" = "Development"
    "Owner"       = "YourName" # Cambia esto por tu nombre o equipo
  }
}
```

**`data.tf`**
```hcl
# Propósito: Buscar información externa a Terraform. Aquí lo usamos para encontrar la AMI más reciente.
# Mejor Práctica (SRE/DevSecOps): Evita el 'hardcoding' de IDs. Esto hace tu código más robusto y seguro,
# ya que siempre usarás la última imagen parchada por AWS.

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Data source para obtener tu IP pública actual
data "http" "my_ip" {
  url = "[http://ipv4.icanhazip.com](http://ipv4.icanhazip.com)"
}
```

**`security.tf`**
```hcl
# Propósito: Centralizar la definición de recursos de seguridad.
# Mejor Práctica (DevSecOps): Define explícitamente las reglas de firewall. Principio de mínimo privilegio.

resource "aws_security_group" "instance_sg" {
  name        = "${var.instance_name}-sg"
  description = "Permitir tráfico SSH desde una IP específica."

  # Regla de entrada (Ingress): Permitir el puerto 22 (SSH).
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    # Usamos una condición: si la variable 'allowed_ssh_cidr' está vacía, usa tu IP detectada.
    # De lo contrario, usa el valor proporcionado. El '/32' indica una única IP.
    cidr_blocks = length(var.allowed_ssh_cidr) > 0 ? var.allowed_ssh_cidr : ["${chomp(data.http.my_ip.response_body)}/32"]
  }

  # Regla de salida (Egress): Permitir todo el tráfico saliente.
  # Es común para poder descargar actualizaciones, etc.
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # '-1' significa todos los protocolos.
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.common_tags
}
```

**`main.tf`**
```hcl
# Propósito: Es el punto de entrada que une todo. Aquí llamamos al módulo 'ec2-instance'.

module "ec2_instance" {
  source = "./modules/ec2-instance"

  # Pasamos las variables requeridas por el módulo.
  instance_name          = var.instance_name
  ami_id                 = data.aws_ami.amazon_linux_2.id
  key_name               = var.key_pair_name
  vpc_security_group_ids = [aws_security_group.instance_sg.id]
  tags                   = var.common_tags
}
```

**`terraform.tfvars`**
```hcl
# Propósito: Asignar valores concretos a las variables definidas en variables.tf.
# Este es el único archivo que deberías modificar para personalizar tu despliegue.
# ¡NUNCA subas este archivo a un repositorio público si contiene información sensible!

aws_region    = "us-east-1"
instance_name = "my-first-tf-instance"

# IMPORTANTE: Reemplaza "terraform-workshop-key" con el nombre EXACTO
# del Key Pair que creaste en la consola de AWS.
key_pair_name = "terraform-workshop-key"
```

**`outputs.tf`**
```hcl
# Propósito: Mostrar información útil al usuario una vez que Terraform ha terminado de aplicar los cambios.

output "instance_id" {
  description = "ID de la instancia EC2 desplegada."
  value       = module.ec2_instance.instance_id
}

output "instance_public_ip" {
  description = "IP pública de la instancia. Úsala para conectar por SSH."
  value       = module.ec2_instance.instance_public_ip
}

output "ssh_connection_command" {
  description = "Comando para conectar a la instancia vía SSH."
  value       = "ssh -i ~/.ssh/${var.key_pair_name}.pem ec2-user@${module.ec2_instance.instance_public_ip}"
}
```

---

### 🚀 **Paso 3: El Ciclo de Vida de Terraform en Acción**

Con todos los archivos en su lugar, ejecuta los siguientes comandos desde la raíz de tu proyecto.

1.  **Inicialización (`terraform init`)**
    * **Propósito:** Prepara tu directorio de trabajo. Descarga el proveedor de AWS (`hashicorp/aws`) y configura el backend de estado.
    * **Comando:**
        ```bash
        terraform init
        ```
    * **Salida Esperada:** Verás un mensaje indicando que Terraform se ha inicializado correctamente.

2.  **Validación (`terraform validate`)**
    * **Propósito:** Revisa la sintaxis de tu código en busca de errores obvios antes de intentar crear un plan.
    * **Comando:**
        ```bash
        terraform validate
        ```
    * **Salida Esperada:** `Success! The configuration is valid.`

3.  **Planificación (`terraform plan`)**
    * **Propósito:** Crea un plan de ejecución. Terraform determina qué recursos necesita crear, modificar o destruir. **Este es el paso más importante para revisar antes de aplicar cambios.**
    * **Comando:**
        ```bash
        terraform plan
        ```
    * **Salida Esperada:** Un resumen detallado de las acciones a realizar. Verás que se añadirán `+ 2 to create` (1 `aws_instance` y 1 `aws_security_group`). Revisa los detalles para asegurarte de que son correctos.

4.  **Aplicación (`terraform apply`)**
    * **Propósito:** Ejecuta el plan creado en el paso anterior. Aquí es donde Terraform se comunica con AWS y crea la infraestructura real.
    * **Comando:**
        ```bash
        terraform apply
        ```
    * **Acción:** Terraform te mostrará el plan de nuevo y te pedirá confirmación. Escribe `yes` y presiona Enter.
    * **Salida Esperada:** Verás los logs de creación de los recursos. Al final, mostrará los `outputs` que definimos, como la IP pública y el comando de conexión SSH.

---

### 🖥️ **Paso 4: Verificación y Conexión**

1.  **Consola de AWS:** Ve a la consola de EC2 en la región que usaste. Deberías ver tu nueva instancia (`my-first-tf-instance`) en estado "Running" y el nuevo grupo de seguridad.
2.  **Conexión SSH:**
    * Asegúrate de que tu archivo `.pem` (ej: `terraform-workshop-key.pem`) está en la ruta `~/.ssh/` y tiene los permisos correctos.
        ```bash
        # Mueve la clave si no lo has hecho
        # mv /mnt/c/Users/TuUsuario/Downloads/terraform-workshop-key.pem ~/.ssh/

        # Asigna permisos de solo lectura para tu usuario
        chmod 400 ~/.ssh/terraform-workshop-key.pem
        ```
    * Copia el comando del `output` `ssh_connection_command` y pégalo en tu terminal para conectarte.
        ```bash
        ssh -i ~/.ssh/terraform-workshop-key.pem ec2-user@<IP_PÚBLICA_DE_LA_INSTANCIA>
        ```
    * La primera vez te preguntará si confías en el host. Escribe `yes`. ¡Deberías estar dentro de tu instancia EC2!

---

### 🧹 **Paso 5: Limpieza de Recursos (Práctica Crítica de FinOps)**

Una vez que hayas terminado de experimentar, es **fundamental** destruir la infraestructura para no incurrir en costos.

* **Propósito:** Eliminar todos los recursos gestionados por este proyecto de Terraform.
* **Comando:**
    ```bash
    terraform destroy
    ```
* **Acción:** Terraform te mostrará todos los recursos que serán destruidos y te pedirá confirmación. Escribe `yes` y presiona Enter.
* **Verificación:** Vuelve a la consola de AWS. La instancia y el grupo de seguridad deberían haber desaparecido después de unos minutos.

---

### 📜 **Guía Resumen (How-To) de lo Aprendido**

Has completado con éxito el ciclo de vida de la Infraestructura como Código. Este es un resumen de las mejores prácticas que has validado:

| Práctica | Implementación en el Workshop | Beneficio Clave |
| :--- | :--- | :--- |
| **Infra. como Código (IaC)** | Uso de Terraform para definir todos los recursos en ficheros `.tf`. | **Automatización y Repetibilidad:** Despliegues consistentes y sin errores manuales. |
| **Modularidad (DevOps)** | Creación de un módulo `ec2-instance` separado del código raíz. | **Reutilización y Mantenimiento:** El módulo puede ser usado en otros proyectos. Cambios fáciles. |
| **Seguridad por Defecto (DevSecOps)** | Creación de un `aws_security_group` que solo permite SSH desde tu IP. | **Reducción de la Superficie de Ataque:** Se aplica el principio de mínimo privilegio. |
| **Código Robusto (SRE)** | Uso de `data "aws_ami"` para buscar la última imagen de Amazon Linux 2. | **Resiliencia y Confiabilidad:** El código no se rompe si AWS actualiza las AMIs. |
| **Trazabilidad y Costos (FinOps)** | Aplicación de `tags` comunes (`Project`, `Environment`, `Owner`) a los recursos. | **Gobernanza y Control de Gastos:** Permite filtrar costos y identificar propietarios de recursos. |
| **Gestión del Ciclo de Vida** | Ejecución de `init`, `plan`, `apply` y `destroy`. | **Control y Previsibilidad:** Siempre sabes qué cambios se van a aplicar y puedes destruir todo de forma segura. |

---

### 🚀 **Próximos Pasos y Desafíos**

¿Te sientes con confianza? Intenta expandir este proyecto con los siguientes desafíos:

1.  **Nivel Intermedio:**
    * **Añadir un Volumen EBS:** Modifica el módulo de EC2 para crear y adjuntar un volumen de almacenamiento `aws_ebs_volume` a la instancia.
    * **User Data:** Añade un script de `user_data` a la instancia para que instale automáticamente un servidor web (como Nginx) al arrancar.
    * **Variables Complejas:** Crea una variable de tipo `list(object)` para definir múltiples instancias con diferentes configuraciones y créalas usando un `for_each`.

2.  **Nivel Avanzado (DevSecOps/CI/CD):**
    * **Análisis de Seguridad Estático:** Integra una herramienta como `tfsec` o `checkov` en tu flujo de trabajo local para escanear tu código en busca de malas prácticas de seguridad antes de ejecutar `apply`.
    * **Backend Remoto:** Configura un backend de Terraform en un bucket de S3 con bloqueo de estado en DynamoDB para poder trabajar en equipo.
    * **Pipeline de CI/CD:** Crea un pipeline simple en GitHub Actions que ejecute `terraform plan` en cada Pull Request para validar los cambios automáticamente.

# AUY1105 — Infraestructura como Código II
## Evaluación Parcial 3: Gestión Avanzada de Recursos de Terraform

**Estudiante:** Carl Cuevas
**Asignatura:** Infraestructura como Código II (AUY1105)
**Modalidad:** Ejecución práctica individual
**Repositorio:** https://github.com/carlcuevas/terraform-evaluacion

---

## Tabla de Contenidos

1. [Propósito](#propósito)
2. [Infraestructura base](#infraestructura-base)
3. [Escenario 1: Recuperación del estado de Terraform](#escenario-1-recuperación-del-estado-de-terraform)
4. [Escenario 2: Actualización y reforzamiento de recursos](#escenario-2-actualización-y-reforzamiento-de-recursos)
5. [Escenario 3: Eliminación de recursos del estado](#escenario-3-eliminación-de-recursos-del-estado)
6. [Resumen de comandos utilizados](#resumen-de-comandos-utilizados)
7. [Notas y aprendizajes](#notas-y-aprendizajes)

---

## Propósito

Esta evaluación demuestra el uso de comandos avanzados de Terraform CLI para la manipulación del archivo de estado (`terraform.tfstate`) ante problemáticas comunes de gestión de infraestructura en la nube: pérdida del estado, desincronización (drift) y desasociación de recursos sin destruirlos.

Los escenarios se ejecutaron sobre una infraestructura desplegada desde cero en AWS Academy (Learner Lab), región `us-east-1`, usando una instancia EC2 (Amazon Linux) como bastión de trabajo con Terraform y AWS CLI instalados.

---

## Infraestructura base

A diferencia de un enfoque modular, esta infraestructura se definió en un único archivo `main.tf` en la raíz del proyecto, con cuatro recursos declarados directamente:

| Recurso | Dirección Terraform | ID (AWS) |
|---|---|---|
| VPC | `aws_vpc.main` | `vpc-072be6efd588bd969` |
| Subnet | `aws_subnet.main` | `subnet-009ed7895faeaf876` |
| Security Group | `aws_security_group.main` | `sg-04c4e82be191f3c16` |
| EC2 | `aws_instance.main` | `i-08eb0bf6a28a49468` (recreada luego como `i-006f61e755a505b8d`) |

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "vpc-evaluacion" }
}

resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "subnet-evaluacion" }
}

resource "aws_security_group" "main" {
  name        = "evaluacion-sg"
  description = "Security group para evaluacion"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "evaluacion-sg" }
}

resource "aws_instance" "main" {
  ami                    = "ami-0c02fb55956c7d316"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.main.id
  vpc_security_group_ids = [aws_security_group.main.id]
  tags = { Name = "ec2-evaluacion" }
}
```

Total: **4 recursos gestionados**, sin módulos ni data sources externos.

---

## Escenario 1: Recuperación del estado de Terraform

**Objetivo:** recuperar el archivo de estado tras su pérdida, sin recrear ni perder recursos existentes.

### Problema simulado
Se eliminó manualmente el archivo `terraform.tfstate` (`rm terraform.tfstate`) para simular una pérdida por error humano. Al ejecutar `terraform plan`, Terraform desconoció la infraestructura existente y planeó crear los 4 recursos desde cero.

### Solución aplicada

```bash
terraform import aws_vpc.main vpc-072be6efd588bd969
terraform import aws_subnet.main subnet-009ed7895faeaf876
terraform import aws_security_group.main sg-04c4e82be191f3c16
terraform import aws_instance.main i-08eb0bf6a28a49468
```

Cada `terraform import` devolvió `Import successful!`, asociando el recurso real de AWS con su dirección en el código.

### Verificación

```bash
terraform state list
```
Resultado — los 4 recursos quedaron registrados: *(ver Captura 1)*

```bash
terraform state show aws_vpc.main
terraform state show aws_subnet.main
terraform state show aws_security_group.main
terraform state show aws_instance.main
```

### Validación final

```bash
terraform plan
```
Resultado: **`No changes. Your infrastructure matches the configuration.`** *(ver Captura 2)*

El estado quedó completamente sincronizado con la infraestructura real.

---

## Escenario 2: Actualización y reforzamiento de recursos

**Objetivo:** detectar y sincronizar el drift entre la infraestructura real y el estado, y forzar la recreación de un recurso con `terraform taint`.

### Parte A — Cambio manual y detección del drift

Se agregó una regla de entrada al Security Group directamente en AWS (fuera de Terraform):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-04c4e82be191f3c16 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

```bash
terraform plan
```
Terraform detectó la inconsistencia: quiere revertir el cambio manual del puerto 80 (`~ update in-place`, `Plan: 0 to add, 1 to change, 0 to destroy`). *(ver Capturas 3, 4, 5)*

### Parte B — Sincronización con `terraform refresh`

```bash
terraform refresh
terraform plan
```
El `refresh` actualizó el estado con los valores reales del Security Group (incluyendo la regla del puerto 80). *(ver Capturas 6, 7, 9)*

### Parte C — Recreación de la EC2 con `terraform taint`

```bash
terraform taint aws_instance.main
```
Resultado: `Resource instance aws_instance.main has been marked as tainted.` *(ver Captura 8)*

```bash
terraform plan
```
Mostró la EC2 marcada para destrucción y recreación (`-/+ destroy and then create replacement`), junto con la actualización del Security Group para eliminar el puerto 80. `Plan: 1 to add, 1 to change, 0 to destroy.` *(ver Capturas 10–14)*

```bash
terraform apply -auto-approve
```
Resultado: la EC2 fue destruida y recreada con un nuevo ID (`i-006f61e755a505b8d`), y el Security Group fue corregido eliminando la regla manual del puerto 80.
**`Apply complete! Resources: 1 added, 1 changed, 1 destroyed.`** *(ver Captura 15)*

### Parte D — Limpieza con `terraform untaint`

```bash
terraform untaint aws_instance.main
```
Resultado: `Error: Resource instance is not tainted` — esperado, ya que la marca de taint se consume automáticamente al aplicarse la recreación.

```bash
terraform plan
```
Resultado: **`No changes. Your infrastructure matches the configuration.`** Infraestructura sincronizada.

---

## Escenario 3: Eliminación de recursos del estado

**Objetivo:** dejar de gestionar el Security Group con Terraform, sin eliminarlo de AWS. *(Pendiente — se ejecuta en clases)*

Guía preparada:

```bash
terraform state list
terraform state rm aws_security_group.main
terraform state list

# Eliminar el bloque resource "aws_security_group" "main" { ... } de main.tf
# y cambiar vpc_security_group_ids = [] en aws_instance.main

aws ec2 describe-security-groups --group-ids sg-04c4e82be191f3c16
terraform plan   # esperado: "No changes. Your infrastructure matches the configuration."
```

Como la infraestructura **no usa módulos** (a diferencia de otros compañeros que la definieron en un módulo remoto versionado), el bloque del Security Group se puede eliminar directamente del `main.tf` local. Esto permite que el `terraform plan` final dé efectivamente `No changes`, cumpliendo el resultado esperado por el enunciado sin necesidad de justificaciones adicionales sobre versionado de módulos.

---

## Resumen de comandos utilizados

| Comando | Función | Escenario |
|---|---|---|
| `terraform import` | Asocia un recurso existente con el estado | 1 |
| `terraform state list` | Lista los recursos gestionados | 1, 3 |
| `terraform state show` | Muestra los atributos de un recurso | 1 |
| `terraform refresh` | Sincroniza el estado con la infraestructura real | 2 |
| `terraform taint` | Marca un recurso para recreación | 2 |
| `terraform untaint` | Revierte la marca de recreación | 2 |
| `terraform state rm` | Elimina un recurso del estado sin destruirlo | 3 |
| `terraform plan` | Compara código vs. estado vs. realidad | 1, 2, 3 |
| `terraform apply` | Aplica los cambios a la infraestructura | 1 (base), 2 |

---

## Notas y aprendizajes

- **`terraform refresh`** está marcado como deprecado desde Terraform v0.15 en favor de `terraform apply -refresh-only`, que permite revisar y aprobar los cambios al estado antes de escribirlos. En esta evaluación se usó el comando clásico `terraform refresh` porque es el que pide explícitamente el enunciado.
- **`terraform taint`** también está deprecado desde v0.15.4; el equivalente moderno es `terraform apply -replace='<dirección>'`. Se usó `taint`/`untaint` por ser lo solicitado en el enunciado.
- Las credenciales de AWS Academy (`aws_session_token`) expiran cada vez que el lab se pausa o reinicia; deben regenerarse con `aws configure set ...` antes de cada sesión de trabajo.
- El archivo `terraform.tfstate` no se subió a GitHub (está en `.gitignore`), ya que contiene información sensible y es específico de este despliegue.
- Nombrar un Security Group con el prefijo `sg-` no está permitido por AWS (colisiona con el formato interno de IDs); se resolvió renombrando a `evaluacion-sg`.

---

*Evidencias completas (capturas de pantalla y salida estándar de cada comando) adjuntas en el informe PDF.*

# AUY1105 — Infraestructura como Código II
## Evaluación Parcial 3: Gestión Avanzada de Recursos de Terraform

**Estudiante:** Daniel Tapia Sobarzo — [@recouma](https://github.com/recouma)
**Asignatura:** Infraestructura como Código II (AUY1105)
**Modalidad:** Ejecución práctica individual

---

## Tabla de Contenidos

1. [Propósito](#propósito)
2. [Infraestructura base](#infraestructura-base)
3. [Escenario 1: Recuperación del estado de Terraform](#escenario-1-recuperación-del-estado-de-terraform)
4. [Escenario 2: Actualización y reforzamiento de recursos](#escenario-2-actualización-y-reforzamiento-de-recursos)
5. [Escenario 3: Eliminación de recursos del estado](#escenario-3-eliminación-de-recursos-del-estado)
6. [Comandos utilizados](#resumen-de-comandos-utilizados)
7. [Notas de buenas prácticas](#notas-de-buenas-prácticas)

---

## Propósito

Esta evaluación demuestra el uso de comandos avanzados de Terraform CLI para la manipulación del archivo de estado (`terraform.tfstate`) ante problemáticas comunes de gestión de infraestructura en la nube, manteniendo la integridad de la infraestructura y evitando la pérdida de datos.

Los tres escenarios se ejecutaron sobre la infraestructura modular desarrollada en las evaluaciones parciales 1 y 2, desplegada en AWS (Learner Lab) región `us-east-1`.

---

## Infraestructura base

La infraestructura está organizada en tres módulos remotos versionados:

| Módulo | Repositorio | Recursos |
|--------|-------------|----------|
| Redes | `terraform-aws-vpc-auy1105-grupo-3` | VPC, Subnet, Internet Gateway, Route Table, Route Table Association, Security Group |
| Cómputo | `terraform-aws-ec2-auy1105-grupo-3` | Instancia EC2 (`t2.micro`, Ubuntu 24.04) |
| Almacenamiento | `terraform-aws-s3-auy1105-grupo-3` | Bucket S3 + versioning, encryption, public access block |

Total: **11 recursos gestionados** (+ 1 data source `aws_ami`).

Direcciones en el state:

```
module.almacenamiento.aws_s3_bucket.this
module.almacenamiento.aws_s3_bucket_public_access_block.this
module.almacenamiento.aws_s3_bucket_server_side_encryption_configuration.this
module.almacenamiento.aws_s3_bucket_versioning.this
module.computo.data.aws_ami.this
module.computo.aws_instance.this
module.red.aws_internet_gateway.this
module.red.aws_route_table.public
module.red.aws_route_table_association.public[0]
module.red.aws_security_group.this
module.red.aws_subnet.public[0]
module.red.aws_vpc.this
```

---

## Escenario 1: Recuperación del estado de Terraform

**Objetivo:** recuperar el archivo de estado tras su pérdida, sin recrear ni perder recursos existentes.

### Problema simulado
Se eliminó el archivo `terraform.tfstate` para simular una pérdida por error humano o fallo técnico. Al ejecutar `terraform plan`, Terraform desconoce la infraestructura existente y planea recrear los 11 recursos desde cero (`Plan: 11 to add`).

### Solución aplicada
Se reconstruyó el estado importando cada recurso con `terraform import`, asociando cada dirección del módulo con el ID real del recurso en AWS.

```bash
# Redes
terraform import 'module.red.aws_vpc.this' vpc-XXXXXXXX
terraform import 'module.red.aws_subnet.public[0]' subnet-XXXXXXXX
terraform import 'module.red.aws_internet_gateway.this' igw-XXXXXXXX
terraform import 'module.red.aws_route_table.public' rtb-XXXXXXXX
terraform import 'module.red.aws_security_group.this' sg-XXXXXXXX
terraform import 'module.red.aws_route_table_association.public[0]' subnet-XXXXXXXX/rtb-XXXXXXXX

# Cómputo
terraform import 'module.computo.aws_instance.this' i-XXXXXXXX

# Almacenamiento
terraform import 'module.almacenamiento.aws_s3_bucket.this' <nombre-bucket>
terraform import 'module.almacenamiento.aws_s3_bucket_versioning.this' <nombre-bucket>
terraform import 'module.almacenamiento.aws_s3_bucket_server_side_encryption_configuration.this' <nombre-bucket>
terraform import 'module.almacenamiento.aws_s3_bucket_public_access_block.this' <nombre-bucket>
```

### Verificación
```bash
terraform state list    # confirma los 11 recursos registrados
terraform state show 'module.computo.aws_instance.this'   # valida atributos
terraform plan          # resultado esperado: "No changes"
```

**Resultado:** `No changes. Your infrastructure matches the configuration.` El estado quedó completamente sincronizado con la infraestructura real.

### Observaciones
- El data source `module.computo.data.aws_ami.this` **no se importa**; se resuelve automáticamente en cada operación.
- El orden importa: el import de la VPC falló inicialmente porque la `aws_route_table_association` depende del `count` de las subnets. Al importar primero la subnet, la VPC se importó sin problema.
- La `aws_route_table_association` usa un ID compuesto con formato `subnet_id/route_table_id`.

---

## Escenario 2: Actualización y reforzamiento de recursos

**Objetivo:** gestionar la desincronización (drift) entre la infraestructura real y el estado, usando `refresh` y `taint`.

### Parte A — Detección y sincronización de drift

**Cambio manual:** se agregó una regla de entrada (puerto 80, `0.0.0.0/0`) al Security Group desde la consola de AWS.

```bash
terraform plan               # detecta el drift: quiere eliminar la regla del puerto 80
terraform apply -refresh-only # registra el cambio real en el estado (yes)
```

**Concepto clave:** `terraform apply -refresh-only` reconcilia el **estado** con la **realidad de AWS**, pero **no modifica el código** ni la infraestructura. Por eso, un `plan` posterior sigue mostrando la diferencia entre el código (sin la regla) y el estado/realidad (con la regla). Para eliminar definitivamente el drift, se aplicó el código como fuente de verdad:

```bash
terraform apply    # elimina la regla manual, restaurando el estado deseado
terraform plan     # No changes
```

### Parte B — Recreación de recursos con taint

```bash
terraform taint 'module.computo.aws_instance.this'   # marca la EC2 para recreación
terraform plan                                        # muestra "-/+ destroy and then create replacement"
terraform apply                                       # destruye y recrea la instancia
```

**Resultado:** la EC2 fue recreada con un nuevo ID y una nueva IP pública (`instance_id` e `instance_ip` cambiaron). `Apply complete! Resources: 1 added, 0 changed, 1 destroyed.`

### Parte C — Limpieza (untaint)
Tras el `apply`, la marca de taint se consume automáticamente al recrearse el recurso. Ejecutar `terraform untaint` después confirma que ya no hay estado tainted (`Resource instance is not tainted`). El comando `untaint` es útil cuando se decide **revertir la marca antes de aplicar**.

```bash
terraform plan    # No changes — infraestructura sincronizada
```

---

## Escenario 3: Eliminación de recursos del estado

**Objetivo:** dejar de gestionar el Security Group con Terraform, **sin eliminarlo** de AWS.

```bash
terraform state list    # antes: el SG aparece
terraform state rm 'module.red.aws_security_group.this'
terraform state list    # después: el SG ya no aparece
```

**Verificación en AWS** (el recurso sigue vivo):
```bash
aws ec2 describe-security-groups --group-ids sg-XXXXXXXX \
  --query "SecurityGroups[0].GroupId" --output text
```
Devuelve el ID del SG → el recurso existe físicamente, Terraform simplemente dejó de administrarlo.

**Resultado:** `Successfully removed 1 resource instance(s).` El objetivo del escenario se cumple: el Security Group permanece operativo en AWS pero fuera de la gestión de Terraform.

### Nota sobre la arquitectura modular
El enunciado pide adicionalmente eliminar el bloque del recurso del código para que `terraform plan` devuelva `No changes`. En esta infraestructura el Security Group está definido dentro del módulo remoto `terraform-aws-vpc-auy1105-grupo-3` (versionado con `?ref=v1.0.0`), por lo que "eliminar del código" requeriría publicar una nueva versión del módulo sin ese recurso. Un `terraform plan` tras el `state rm` muestra que Terraform intentaría recrear el SG (`Plan: 1 to add`), lo que demuestra la relación entre el código y el estado.

---

## Resumen de comandos utilizados

| Comando | Función | Escenario |
|---------|---------|-----------|
| `terraform import` | Asocia un recurso existente con el estado | 1 |
| `terraform state list` | Lista los recursos gestionados | 1, 3 |
| `terraform state show` | Muestra los atributos de un recurso | 1, 2 |
| `terraform apply -refresh-only` | Sincroniza el estado con la realidad | 2 |
| `terraform taint` | Marca un recurso para recreación | 2 |
| `terraform untaint` | Revierte la marca de recreación | 2 |
| `terraform state rm` | Elimina un recurso del estado (sin destruirlo) | 3 |
| `terraform plan` | Compara código vs estado vs realidad | 1, 2, 3 |
| `terraform apply` | Aplica los cambios a la infraestructura | 2 |

---

## Notas de buenas prácticas

- **`terraform refresh` está deprecado** desde la v0.15. La práctica recomendada actual es `terraform apply -refresh-only`, que permite revisar y aprobar los cambios al estado antes de escribirlos.
- **`terraform taint` está deprecado** desde la v0.15.4. El equivalente moderno es `terraform apply -replace='<direccion>'`, que fuerza la recreación sin marcar manualmente el estado.
- Las manipulaciones de estado (`import`, `state rm`) **solo existen en la CLI de Terraform**; no tienen equivalente en AWS CLI ni en Git. AWS CLI sirve para verificar la infraestructura real, y Git para versionar el código.
- El archivo `terraform.tfstate` no debe subirse a Git (contiene información sensible y es específico de cada despliegue). Se mantiene en `.gitignore`.

---

*Evidencias completas (salida estándar de cada comando) disponibles en la carpeta `evidencias/ep3/`.*

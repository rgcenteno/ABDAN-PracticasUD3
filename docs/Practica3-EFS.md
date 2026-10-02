# Práctica AWS: Dos EFS compartidos selectivamente entre tres instancias EC2

## Situación empresarial

La empresa **DataVision Consulting** desarrolla aplicaciones para entidades financieras y necesita centralizar la información compartida entre sus servidores.

La infraestructura está compuesta por tres instancias EC2:

| Instancia    | Función                               |
| ------------ | ------------------------------------- |
| EC2-App01    | Aplicación principal                  |
| EC2-App02    | Aplicación secundaria y procesamiento |
| EC2-Backup01 | Servidor de backups y recuperación    |

La empresa tiene dos necesidades de almacenamiento claramente diferenciadas:

#### EFS-Documentos

Se utilizará para almacenar:

- Manuales técnicos.
- Procedimientos internos.
- Documentación de proyectos.
- Informes compartidos.

Este almacenamiento solo debe ser accesible desde:

- EC2-App01
- EC2-App02

El servidor de backups no necesita acceder a esta información.

---

#### EFS-Backups

Se utilizará para almacenar:

- Copias de seguridad de aplicaciones.
- Exportaciones de bases de datos.
- Ficheros de recuperación.

Este almacenamiento debe ser accesible desde:

- EC2-App01
- EC2-App02
- EC2-Backup01

---

## Objetivos

Al finalizar la práctica deberás ser capaz de:

✅ Crear dos sistemas de archivos EFS.

✅ Configurar los permisos de acceso mediante Security Groups.

✅ Montar un EFS en dos servidores y otro EFS en tres servidores.

✅ Verificar la compartición de datos.

✅ Automatizar los montajes mediante fstab.

---

## Arquitectura final

                +------------------+

                | EFS-Documentos   |

                +--------+---------+

                         |

              ---------------------

              |                   |

              |                   |

         EC2-App01          EC2-App02

                +------------------+

                |   EFS-Backups    |

                +--------+---------+

                         |

        ---------------------------------------

        |                 |                  |

        |                 |                  |

   EC2-App01        EC2-App02      EC2-Backup01

---

## Fase 1: Crear las instancias EC2

Crear tres instancias Amazon Linux 2023:

- EC2-App01

- EC2-App02

- EC2-Backup01

Con estas características

- AMI: Amazon Linux más actual que sea Apta para la capa gratuita

- Instancia t3.micro

- Utiliza el par de claves vockey

- Establece la máquina EC2 en la VPC predeterminada. Sin preferencias en AZ y Subred.

- Asignación automática de IP pública: Habilitar

- Deja el almacenamiento y el resto de parámetros por defecto y pulsa el botón Lanzar instancia.

Todas deben pertenecer a la misma VPC y tener IP Pública. Par de claves vockey.

---

## Fase 2: Configurar Security Groups

### Paso 1. Crear SG-EC2-App

Asignar a:

- EC2-App01

- EC2-App02

Permitir:

| Tipo | Puerto |
| ---- | ------ |
| SSH  | 22     |

---

### Paso 2. Crear SG-EC2-Backup

Asignar a:

- EC2-Backup01

Permitir:

| Tipo | Puerto |
| ---- | ------ |
| SSH  | 22     |

---

### Paso 3. Crear SG-EFS-Documentos

Siguiendo la siguiente [guía](https://docs.aws.amazon.com/es_es/efs/latest/ug/creating-using-create-fs.html).

Permitir:

| Tipo | Puerto |
| ---- | ------ |
| NFS  | 2049   |

Origen:

SG-EC2-App

De esta forma únicamente las instancias App01 y App02 podrán montar este EFS.

---

### Paso 4. Crear SG-EFS-Backups

Permitir:

| Tipo | Puerto |
| ---- | ------ |
| NFS  | 2049   |

Origen:

SG-EC2-App

SG-EC2-Backup

Las tres máquinas tendrán acceso.

---

## Fase 3: Crear los sistemas EFS

### Paso 5. Crear EFS-Documentos

- Nombre: EFS-Documentos
- Asociar: SG-EFS-Documentos
- Deshabilita las copias de seguridad automáticas para evitar cargos adicionales

Resto de parámetros por defecto

 ---

### Paso 6. Crear EFS-Backups

- Nombre: EFS-Backups
- Asociar: SG-EFS-Backups
- Deshabilita las copias de seguridad automáticas para evitar cargos adicionales

Resto de parámetros por defecto

---

## Fase 4: Instalar amazon-efs-utils

En las tres EC2:

```sh
sudo yum install -y amazon-efs-utils
```

---

## Fase 5: Montar EFS-Documentos

### En EC2-App01 y EC2-App02

Fíjate en el punto 3.1 de este [tutorial](https://docs.aws.amazon.com/es_es/efs/latest/ug/wt1-getting-started.html) para obtener el nombre DNS del sistema EFS. Como usamos `amazon-efs-utils` también podrías poner sólo el ID del EFS.

```sh
# Crear directorio:
sudo mkdir /documentos

#Montar:
sudo mount -t efs fs-id:/ /documentos

#Comprobar:
df -h

---
```

### Prueba de funcionamiento

    Desde App01:
    
    echo "Manual de despliegue" > /documentos/manual.txt
    
    Desde App02:
    
    cat /documentos/manual.txt
    
    Resultado esperado:
    
    Manual de despliegue
    
    ---

### Validación de seguridad

    Intentar montar EFS-Documentos desde:
    
    EC2-Backup01
    
    sudo mkdir /documentos
    
    sudo mount -t efs fs-DOCUMENTOS:/ /documentos

#### Resultado esperado

    El montaje debe fallar porque el Security Group del EFS no permite conexiones NFS desde EC2-Backup01.
    
    Este es uno de los objetivos de la práctica.
    
    ---

## Fase 6: Montar EFS-Backups

### En las tres EC2

    Crear directorio:
    
    sudo mkdir /backups
    
    Montar:
    
    sudo mount -t efs fs-BACKUPS:/ /backups
    
    ---

### Prueba de compartición

    Desde EC2-App01:
    
    echo "Backup diario" > /backups/backup_app01.sql
    
    Desde EC2-Backup01:
    
    cat /backups/backup_app01.sql
    
    Resultado esperado:
    
    Backup diario
    
    ---

## Fase 7: Automatizar los montajes

### En App01 y App02

    Editar:
    
    sudo nano /etc/fstab
    
    Añadir:
    
    fs-DOCUMENTOS:/ /documentos efs defaults,_netdev 0 0
    
    fs-BACKUPS:/ /backups efs defaults,_netdev 0 0
    
    ---

### En Backup01

    Añadir únicamente:
    
    fs-BACKUPS:/ /backups efs defaults,_netdev 0 0
    
    Observa que no debe existir ninguna entrada para EFS-Documentos.
    
    ---

## Resultado esperado

| Instancia    | /documentos | /backups |
| ------------ | ----------- | -------- |
| EC2-App01    | ✅           | ✅        |
| EC2-App02    | ✅           | ✅        |
| EC2-Backup01 | ❌           | ✅        |

    La clave de esta práctica es que **no todos los servidores tienen acceso a todos los sistemas de archivos**, reproduciendo una situación real en la que se aplica el principio de mínimo privilegio. Esto además permite trabajar la diferencia entre compartir un EFS con toda una infraestructura o únicamente con los servidores que realmente lo necesitan.

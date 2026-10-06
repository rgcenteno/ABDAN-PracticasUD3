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

- Instancia t{x}.micro de capa gratuita más actual

- Utiliza el par de claves vockey

- Establece la máquina EC2 en la VPC predeterminada. Sin preferencias en AZ y Subred.

- Asignación automática de IP pública: Habilitar

- Deja el almacenamiento y el resto de parámetros por defecto y pulsa el botón Lanzar instancia.

- Asocia a las dos primeras máquinas el rol LabInstanceProfile y a la máquina Backup el rol EMR_EC2_DefaultRole. En un **entorno real crearíamos un rol** para las máquinas con acceso a documentos y otro para la que sólo va a tener acceso a la parte de backup pero los laboratorios de AWS no nos dejan crear roles por lo que tenemos que conformarnos con los predefinidos.

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

#### Paso 1:

- Nombre: EFS-Documentos
- Deshabilita las copias de seguridad automáticas para evitar cargos adicionales

Resto de parámetros por defecto

#### Paso 2:

- Elegimos nuestra VPC y elegimos la **subred privada**. 
- Sólo usamos ipv4 
- Asociamos el grupo de seguridad SG-EFS-Documentos. Con esto **securizamos a nivel de conexión el EFS**

#### Paso 3:

Vamos a seguir las recomendaciones de seguridad para proteger un sistema EFS. Para ello vamos a marcar las siguientes opciones:

- Impedir el acceso raíz por defecto

- Evitar el acceso anónimo

- Aplicar el cifrado en tránsito para todos los clientes

Se nos creará una política JSON. Esta política va a permitir conectarse a **cualquier instancia que no sea anónima** pero nosotros sólo queremos que se puedan conectar las instancias EC2 con el Rol de IAM asociado `LabInstanceProfile`. Para conseguir esto tenemos que modificar el json generado en el statement que cuyo action es `elasticfilesystem:ClientWrite` y tiene una condición booleana llamada `elasticfilesystem:AccessedViaMountTarget` con valor `true`. En esa política veremos que el Principal (a quién aplica esta política) será algo similar a esto:

```json
"Principal": {
    "AWS": "*"
}
```

Deberemos cambiarlo por el ARN del Rol IAM que está contenido dentro de LabInstanceProfile. 

![image](imgs/instace-role-IAM-role.png)

Para conocer ese ARN, abrimos la consola EC2 y escribimos el comando:

```sh
aws sts get-caller-identity
```

Nos devolverá un resultado parecido a éste:

```json
{
    "UserId": "AROAZKKMNFFGGZLH45IBB:i-0b15f0dfd746aa0d1",
    "Account": "640647244108",
    "Arn": "arn:aws:sts::640647244108:assumed-role/LabRole/i-0b15f0dfd746aa0d1"
}
```

Ese ARN es un ARN temporal de STS. Si reiniciamos la consola o accedemos desde otro equipo, veremos que posiblemente el ARN haya variado. Revisa el documento "[ARN temporal de STS frente al ARN del rol IAM en EC2 y EFS](/anexo-sts-role)" para entender las implicaciones antes de seguir.

El nombre de un ARN temporal de STS sigue el siguiente formato:

`arn:aws:sts::<cuenta>:assumed-role/<nombre-Rol>/<Sesion-i>`

Por lo tanto de este ejemplo podemos asumir:

- Cuenta: 640647244108

- Rol: LabRole

- Sesión-i: i-0b15f0dfd746aa0d1

El **ARN permanente del rol LabRole** sería el siguiente:
`arn:aws:iam::640647244108:role/LabRole`

Ese el ARN que deberemos pegar en nuestra política quedando dicha política así (hemos modificado la línea 9):

```json
{
    "Version": "2012-10-17",
    "Id": "efs-policy-wizard-b03eb5c8-3637-4ff2-b023-8223d07bcef1",
    "Statement": [
        {
            "Sid": "efs-statement-solo-labrole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::640647244108:role/LabRole"
            },
            "Action": "elasticfilesystem:ClientWrite",
            "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-00a7abbeaf2dc7551",
            "Condition": {
                "Bool": {
                    "elasticfilesystem:AccessedViaMountTarget": "true"
                }
            }
        },
        {
            "Sid": "efs-statement-6917ef70-df8f-46db-8e1d-1c70e5e8844b",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Action": "*",
            "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-00a7abbeaf2dc7551",
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

Esta política sigue teniendo un problema, cualquier Rol podrá montar el sistema de ficheros. Para evitar esto, vamos a añadir el permiso `elasticfilesystem:ClientMount` dentro de esta política (se ha añadido en la línea 13). Así sólo las máquinas con el Rol `LabRole` (o un instance profile que lo contenga como `LabInstaceProfile`) podrán montar este volumen en una máquina EC2.

La política que quedará será parecida a esta (haz las modificaciones comentadas sobre la tuya ya que esta no funcionará si la copias y pegas directamente)

```json
{
    "Version": "2012-10-17",
    "Id": "efs-policy-wizard-b03eb5c8-3637-4ff2-b023-8223d07bcef1",
    "Statement": [
        {
            "Sid": "efs-statement-solo-labrole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::640647244108:role/LabRole"
            },
            "Action": [
                "elasticfilesystem:ClientWrite",
                "elasticfilesystem:ClientMount"
            ],
            "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-00a7abbeaf2dc7551",
            "Condition": {
                "Bool": {
                    "elasticfilesystem:AccessedViaMountTarget": "true"
                }
            }
        },
        {
            "Sid": "efs-statement-6917ef70-df8f-46db-8e1d-1c70e5e8844b",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Action": "*",
            "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-00a7abbeaf2dc7551",
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

Con estas políticas hemos conseguido los siguiente:

- El acceso debe realizarse a través de un Mount Target (`AccessedViaMountTarget`).
- El tráfico debe ir cifrado (`tls`).
- El cliente debe autenticarse mediante IAM (`iam`).
- El principal autenticado debe ser el rol `LabRole`.
- El rol tiene permisos de montaje (`ClientMount`) y escritura (`ClientWrite`).

Una cosa que tenemos que tener en cuenta. Cuando usemos una máquina EC2 que asuma el Rol `LabRole`, vamos ver que tenemos el permiso `elasticfilesystem:ClientRootAccess`. Podemos comprobarlo creando un fichero con sudo y viendo que el dueño del archivo es root:root. En un sistema sin root squashing el usuario sería nobody.

##### ¿Por qué tenemos Root Squashing pese a no dar el permiso en el recurso?

Según la [documentación de red y permisos de Amazon EFS](https://docs.aws.amazon.com/es_es/efs/latest/ug/accessing-fs-nfs-permissions.html#accessing-fs-nfs-permissions-root-user), por defecto, el "root squashing" está desactivado (`no_root_squash`).

Esto significa que Amazon EFS trata nativamente a cualquier usuario con UID 0 (root) como el usuario administrador raíz y se salta las comprobaciones de permisos. 

Para activar el root squashing (es decir, degradar al usuario root a un usuario anónimo sin privilegios), debemoss quitarle explícitamente el permiso `elasticfilesystem:ClientRootAccess`.

En realidad, al menos para un primer acceso en el que creemos estructuras de carpeta y asociemos permisos, es necesario tener acceso root ya que en el directorio / del EFS el usuario `root` es el propietario del recurso montado. Vamos a provechar esto cuando montemos por primera vez los EFS para crear las estructuras de carpetas y luego denegaremos explíticamente el root squashing. Recuerda que no en todos los sistemas de archivos en red es necesario deshabilitarlo, todo depende de la casuística que estemos implementando/trabajando.

 ---

### Paso 6. Crear EFS-Backups

Vamos a crear EFS-Backups basándonos en lo hecho con el EFS anterior pero permitiendo que se monten y escriban tanto `LabRole` como el Instance Rol asociado a la máquina `EC2-Backup01`. Además el grupo de seguridad asociado al destino de montaje será `SG-EFS-Backups` para permitir la conexión tanto de las máquinas App como de la máquina Backup.

En las políticas debemos dejar Montar y escribir en el sistema EFS tanto al rol `LabRole` como al rol `EMR_EC2_DefaultRole`.

---

## Fase 4: Instalar amazon-efs-utils

En las tres EC2:

```sh
sudo yum install -y amazon-efs-utils
```

---

## Fase 5: Montar EFS-Documentos

### ¿Qué máquinas se pueden conectar a EFS-Documentos?

- Cuando creamos el punto de acceso del EFS asociamos el grupo de seguridad SG-EFS-Documento.

- En las políticas del EFS hemos establecido:

```json
{
    "Sid": "efs-statement-dc4dd5c9-1532-47de-8772-f2220a8bfffe",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::640647244108:role/LabRole"
    },
    "Action": [
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:ClientMount"
    ],
    "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-0c4e49d543c970398",
    "Condition": {
        "Bool": {
            "elasticfilesystem:AccessedViaMountTarget": "true"
        }
    }
}
```

Por lo tanto sólo nos podremos conectar desde máquinas cuyo SG sea `SG-EC2-App` y que además tengan el Rol `LabRole`. En nuestra práctica esto lo cumplen las máquinas `EC2-App01` y `EC2-App02`.

### En EC2-App01 y EC2-App02

Fíjate en el punto 3.1 de este [tutorial](https://docs.aws.amazon.com/es_es/efs/latest/ug/wt1-getting-started.html) para obtener el nombre DNS del sistema `EFS-Documentos`. Como usamos `amazon-efs-utils` también podrías poner sólo el ID del EFS. Fíjate que en comando añadimos `-o tls,iam` para obligar a usar TLS en la conexión y enviar los datos del Rol IAM asociado a la instancia.

> Ten en cuenta que si acabas de crear el sistema EFS es posible que tengas que esperar varios minutos para que el comando `mount`funcione incluso aunque en la consola se muestre el sistema EFS como `disponible`. Eso se debe a que las resoluciones DNS tardan varios minutos en propagarse.

```sh
# Crear directorio:
sudo mkdir /documentos

#Montar:
sudo mount -t efs -o tls,iam fs-id:/ /documentos

#Comprobar: Debemos ver una entreada montada en /documentos
df -h
```

### Creación de carpetas en EFS-Documentos y asignación de permisos a las mismas.

Desde el equipo Ec2-App01, vamos a crear una carpeta `compartida` con lectura y escritura para todo el mundo y una carpeta `documentacion` con permisos de solo lectura para todos los usuarios y de lectura escritura para el usuario local documentador que crearemos y que tendrá permisos totales sobre ella. En un entorno real el usuario documentador debería ser un usuario de un AD o LDAP corporativo.

Además vamos a crear una carpeta compartida en la que cualquier usuario pueda leer y escribir.

```sh
# Creamos a documentador y le establecemos un password
sudo adduser documentador
sudo passwd documentador

# Creamos la carpeta como su (/documentos tiene como propietario a 
# documentador:documentador y sólo el propietario puede escribir en ella)
sudo mkdir /documentos/documentacion
sudo chown documentador:documentador /documentos/documentacion

# Creamos la carpeta compartida y le damos permisos 777 para que todo
# el mundo pueda leer y escribir en ella
sudo mkdir /documentos/compartida
sudo chmod 777 /documentos/compartida
```

### Prueba de funcionamiento

Con usuario `ec2-user` en máquina `Ec2-App01`

```sh
echo "Manual de despliegue > /documentos/compartida/manual.txt
# Listamos para comprobar que se ha creado el documento
ls -l /documentos/compartida

# Intentamos crear un fichero en documentacion, se espera error de permisos
echo "Manual infraestructura" /documentos/documentacion/infraestructura.txt

# Cambiamos al usuario documentador y creamos el fichero
su documentador
echo "Manual infraestructura" > /documentos/documentacion/infraestructura.txt
ls -l /documentos/documentacion
```

Con usuario `ec2-user` en máquina `Ec2-App02`

```sh
# Comprobamos que existen los ficheros de la carpeta compartida
ls -l /documentacion
```

Veremos que están la carpeta compartida con propietario root:root y la carpeta documentacion con usuario y grupo numérico. Esto es porque al ser un usuario local de la máquina Ec2-App01, no tiene la información de dicho usuario. En un entorno de Sistema en red tipo LDAP o AD veríamos correctamente el nombre.

Mostramos el contenido del fichero `/documentos/compartida/manual.txt` y que también podemos leer el fichero `/documentos/documentacion/infraestructura.txt`

```sh
cat /documentos/compartida/manual.txt
cat /documentos/documentacion/infraestructura.txt
```

Vemos que nos muestra el contenido de los ficheros sin problema.

Ahora intentaremos crear un fichero en la subcapeta documentacion y obtendremos error:

```sh
touch /documentos/documentacion/test-permisos.txt
```

Si todas las pruebas funcionaron correctamente, ya tenemos la estructura de carpetas montadas y podemos proceder a cerrar el acceso root. Para ello bastaría con **añadir** la siguiente política al EFS:

```json
{
    "Sid": "ForzarRootSquashExplicito",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "elasticfilesystem:ClientRootAccess",
    "Resource": "arn:aws:elasticfilesystem:us-east-1:640647244108:file-system/fs-0c4e49d543c970398"
}
```

Una vez añadida la política, desmontamos y montamos de nuevo el EFS y veremos que, si queremos cambiar el propietario del fichero `/documentos/documentacion/infraestructura.txt` 

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

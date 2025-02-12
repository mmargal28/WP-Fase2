# Crear y custodiar copias de seguridad

Para este proceso primero debemos de tener en cuenta cual fue nuestro método de instalación, en este caso usaremos el wordpress que preparamos e instalamos con docker compose.

# Copia de seguridad manual

## Herramientas necesarias
* Docker compose (preparado en la fase anterior)
* Zip: para poder realizar compresiones y descompresiones
* Creacioón de usuario wp-admin con su carpeta home (con el gestionamos las carpetas de backup)

### Paso 1 elaboración de scripts para la creacion de backups

Creamos 2 scripts, uno para el contenedor que contiene nuestro Wordpress y otro para MariaDB. El crontab es el de root pues con el levantamos los contenedores.

#### Script Para copia de los ficheros de la pagina

~~~bash
#!/bin/bash

# Configuración
BACKUP_DIR="/home/wp-admin/wordpress_backups" ##Directorio de copias de seguridad
WORDPRESS_CONTAINER="wordpress-app" ## Nombre del contenedor
TIMESTAMP=$(date +'%Y-%m-%d_%H-%M-%S') ## Dia y hora dde la copia
BACKUP_FILE="$BACKUP_DIR/wordpress_files_$TIMESTAMP.zip" ## Nombre de la copia

# Crear directorio de backup si no existe
mkdir -p "$BACKUP_DIR"

# Copiar los archivos de WordPress desde el contenedor en un archivo temporal
echo "Copiando archivos de WordPress..."
docker cp "$WORDPRESS_CONTAINER:/var/www/html" "$BACKUP_DIR/wordpress_files"

# Comprimir los archivos en un ZIP
echo "Comprimiendo archivos..."
zip -r "$BACKUP_FILE" "$BACKUP_DIR/wordpress_files"

# Limpiar archivos temporales
rm -rf "$BACKUP_DIR/wordpress_files"

echo "Backup de archivos completado: $BACKUP_FILE"
~~~

#### Script Para copia de la Base de datos

***Nota: necesitamos tener instalado el cliente de mariadb***
~~~bash
#!/bin/bash

# Configuración
BACKUP_DIR="/home/wp-admin/wordpress_backups"
MYSQL_HOST=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' wordpress-db)  # Si estás en Docker, intenta con "localhost" o "wordpress-db"
MYSQL_PORT="3306"
MYSQL_USER="wordpress_user"
MYSQL_PASSWORD="password123"
MYSQL_DATABASE="wordpress"
TIMESTAMP=$(date +'%Y-%m-%d_%H-%M-%S')
BACKUP_FILE="$BACKUP_DIR/wordpress_db_$TIMESTAMP.sql"

# Crear directorio de backup si no existe
mkdir -p "$BACKUP_DIR"

# Hacer el dump de la base de datos desde el host en lugar del contenedor
echo "Creando backup de la base de datos..."
mysqldump -h "$MYSQL_HOST" -P "$MYSQL_PORT" -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" > "$BACKUP_FILE"

# Comprimir el archivo SQL
echo "Comprimiendo backup de la base de datos..."
zip -r "$BACKUP_FILE.zip" "$BACKUP_FILE"

# Eliminar el archivo SQL original después de comprimirlo
rm "$BACKUP_FILE"

echo "Backup de la base de datos completado: $BACKUP_FILE.zip"
~~~

#### (Opcional) Script para borrado de copias antiguas

~~~bash
#!/bin/bash

# Configuración
BACKUP_DIR="/home/wp-admin/wordpress_backups"

# Buscar el archivo ZIP más antiguo
OLDEST_FILE=$(ls -t "$BACKUP_DIR"/*.zip | tail -n 1)

# Si hay al menos un archivo, lo elimina
if [[ -f "$OLDEST_FILE" ]]; then
    echo "Eliminando copia de seguridad más antigua: $OLDEST_FILE"
    rm "$OLDEST_FILE"
    echo "Archivo eliminado con éxito."
else
    echo "No se encontraron archivos antiguos para eliminar."
fi
~~~

Por ultimo movemos los scripts a la carpeta home de wp-admin y les damos permisos de ejecución para poder usarlos junto a crontab.

### Paso 2 configuración del crontab

Como mencionamos anteriormente nos conectamos como root para que se ejecuten los scripts desde su crontab.

Ejecutamos: `crontab -e`

Dentro del archivo de crontab realizamos las siguientes configuraciones:

~~~bash
# Backup de archivos - Lunes y Jueves a las 23:00
0 23 * * 1,4 /home/wp-admin/backup-wordpress-files.sh

# Backup de la base de datos - Lunes y Jueves a las 23:30
30 23 * * 1,4 /home/wp-admin/backup_wordpress_db.sh

# Borrar el backup más antiguo - Domingos a las 23:00 (opcional)
0 23 * * 0 /home/wp-admin/delete_oldest_backup.sh
~~~

Con esto ya tenemos una estructura básica de copias de seguridad a partir de scripts.

# Copias de seguridad a partir de plugins

## Herramientas necesarias
* Wordpress instalado

### Paso 1 Instalación del plugin

Para este ejercicio vamos a trabajar con All-in-One WP Migration and Backup, debido a su popularidad y 0 coste.

1. Nos dirigimos a Wordpress > Plugins > Añadir Plugins
2. En el buscador introducimos ***All-in-One WP Migration and Backup***

![alt text](../IMG/01-CP.png)

3. Pulsamos ***Instalar Ahora***

4. Pulsamos **Activar**

Deberá de aparecer así cuando de termine de instalar:

![alt text](../IMG/02-CP.png)

### Paso 2 Uso de la herramienta

1. Nos dirigimos a Wordpress > All-in-One WP Migration and Backup
2. Nos aparecerá esta pantalla:

![alt text](../IMG/03_CP.png)

Abrimos el desplegable y elegimos el método de exportación (en mi caso archivo):

![alt text](../IMG/04-CP.png)

3. Comenzara el proceso de extracción y después podrás descargar el archivo:

![alt text](../IMG/05-CP.png)
![alt text](../IMG/06-Cp.png)
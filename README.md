# Compose Toolchain

Stack local de herramientas DevOps compuesto por Jenkins, SonarQube Community
Build y PostgreSQL. Está pensado para desarrollo, aprendizaje y pruebas en una
red de confianza; la configuración actual no es adecuada para producción.

## Servicios

| Servicio | Imagen o construcción | Acceso desde el host | Función |
| --- | --- | --- | --- |
| Jenkins | `Dockerfile-jenkins` | <http://localhost:8080> | Automatización de pipelines |
| SonarQube | `sonarqube:community` | <http://localhost:8081> | Análisis de calidad de código |
| PostgreSQL | `postgres:15` | Solo red interna, puerto `5432` | Base de datos de SonarQube |

SonarQube puede tardar algunos minutos en quedar disponible después de iniciar
los contenedores.

## Prerrequisitos

- Docker Engine con Docker Compose v2 (`docker compose`).
- Puertos `8080` y `8081` disponibles en el host.
- Recursos suficientes para ejecutar Jenkins, SonarQube y PostgreSQL.
- Acceso al socket Unix `/var/run/docker.sock` si Jenkins ejecutará comandos de
  Docker. El host debe proporcionar este socket o un mecanismo compatible.

Antes de iniciar, revisa la configuración efectiva:

```bash
docker compose config
```

## Inicio rápido

Construye la imagen de Jenkins e inicia todo el stack:

```bash
docker compose up -d --build
docker compose ps
```

Para seguir los registros durante el primer inicio:

```bash
docker compose logs -f jenkins
docker compose logs -f sonarqube
```

## Configuración inicial

### Jenkins

La imagen desactiva deliberadamente el asistente de instalación mediante
`jenkins.install.runSetupWizard=false`. El stack tampoco crea automáticamente
un usuario administrador. Mientras no se configure autenticación y autorización
en **Administrar Jenkins > Seguridad**, Jenkins debe permanecer accesible solo
desde una máquina o red de confianza.

Cuando Jenkins necesite comunicarse con SonarQube:

- Guarda el token de SonarQube como una credencial Jenkins de tipo
  **Texto secreto**.
- Configura en SonarQube el webhook interno de la red Compose:
  `http://jenkins:8080/sonarqube-webhook/`.

### SonarQube

En una instalación nueva, ingresa a <http://localhost:8081> con las credenciales
iniciales `admin` / `admin` y cambia inmediatamente la contraseña. Crea tokens
individuales para las integraciones; no reutilices la contraseña administrativa
en Jenkins ni en pipelines.

## Plugins de Jenkins

[`plugins.txt`](plugins.txt) es la fuente de plugins instalados al construir la
imagen. Para listar desde Jenkins los plugins y versiones efectivamente cargados,
ejecuta lo siguiente en **Administrar Jenkins > Consola de scripts**:

```groovy
Jenkins.instance.pluginManager.plugins.each { plugin ->
    println("${plugin.getDisplayName()} (${plugin.getShortName()}): ${plugin.getVersion()}")
}
```

Después de modificar `plugins.txt` o `Dockerfile-jenkins`, reconstruye únicamente
Jenkins:

```bash
docker compose up -d --build jenkins
```

Docker invalida automáticamente las capas afectadas. Usa `--no-cache` solo para
forzar una reconstrucción completa o resolver problemas de caché:

```bash
docker compose build --no-cache --pull jenkins
docker compose up -d jenkins
```

Las versiones de Jenkins, SonarQube y sus plugins pueden cambiar al reconstruir
porque el proyecto utiliza etiquetas o plugins sin una versión completamente
fijada. Revisa los cambios y conserva un respaldo antes de actualizar un entorno
que contenga información importante.

## Operación habitual

```bash
# Estado de los servicios
docker compose ps

# Últimos registros de todo el stack
docker compose logs --tail=200

# Reiniciar un servicio
docker compose restart jenkins

# Detener e iniciar el stack sin eliminar contenedores
docker compose stop
docker compose start

# Eliminar contenedores y red, conservando los volúmenes
docker compose down
```

Para descargar imágenes nuevas y recrear los servicios:

```bash
docker compose pull sonarqube db_sonar
docker compose build --pull jenkins
docker compose up -d
```

## Persistencia y respaldos

El stack utiliza los siguientes volúmenes:

| Volumen | Contenido principal |
| --- | --- |
| `jenkins_home` | Configuración, credenciales, jobs y plugins de Jenkins |
| `sonarqube_data` | Datos de ejecución de SonarQube |
| `sonarqube_extensions` | Extensiones de SonarQube |
| `sonarqube_logs` | Registros de SonarQube |
| `postgresql` | Archivos auxiliares del contenedor PostgreSQL |
| `postgresql_data` | Base de datos PostgreSQL de SonarQube |

`docker compose down` conserva estos volúmenes. En cambio, el siguiente comando
elimina permanentemente toda la información de Jenkins, SonarQube y PostgreSQL:

```bash
docker compose down --volumes
```

> **Advertencia:** ejecuta `down --volumes` solamente cuando quieras reiniciar el
> entorno desde cero o ya dispongas de respaldos verificados.

Este repositorio no automatiza los respaldos. Como mínimo, respalda Jenkins con
el servicio detenido y exporta la base de datos de SonarQube antes de una
actualización:

```bash
docker compose stop jenkins
docker run --rm \
  --volume devops-infra_jenkins_home:/source:ro \
  --volume "$PWD:/backup" \
  alpine tar -czf /backup/jenkins-home.tgz -C /source .
docker compose start jenkins

docker compose exec -T db_sonar \
  pg_dump -U sonar sonar > sonarqube-database.sql
```

Prueba periódicamente la restauración en un stack separado antes de considerar
válido un respaldo.

## Seguridad

- El montaje de `/var/run/docker.sock` permite a Jenkins controlar el daemon de
  Docker del host. Trátalo como acceso administrativo y no ejecutes pipelines no
  confiables.
- La imagen actual deja el proceso de Jenkins ejecutándose como `root` después de
  instalar Docker. En combinación con el socket del host, esto confirma que el
  stack debe limitarse a un laboratorio local. Antes de usarlo en otro entorno,
  la imagen debe volver al usuario `jenkins` y alinear los permisos del socket.
- Los puertos publicados escuchan en todas las interfaces del host. Usa firewall,
  una interfaz local o un proxy con TLS antes de permitir acceso desde otra red.
- Las credenciales de PostgreSQL incluidas en Compose son solo para este entorno
  local. Para otro entorno, utiliza secretos externos y contraseñas propias.
- No expongas Jenkins mientras el asistente esté desactivado y no hayas configurado
  autenticación y autorización.

## Solución de problemas

### Un puerto ya está ocupado

Comprueba qué proceso utiliza `8080` o `8081`, libera el puerto o cambia su mapeo
en `docker-compose.yaml`.

### Jenkins no puede usar Docker

Compara los permisos y el grupo del socket del host con los grupos del usuario
`jenkins` dentro del contenedor:

```bash
ls -ln /var/run/docker.sock
docker compose exec jenkins id
docker compose logs --tail=200 jenkins
```

La imagen actual ya ejecuta Jenkins como `root`; no reproduzcas este patrón como
solución de permisos en un entorno compartido o productivo. Una imagen endurecida
debe volver al usuario `jenkins` y conceder únicamente el acceso necesario.

### SonarQube todavía no responde

`depends_on` inicia PostgreSQL antes que SonarQube, pero no garantiza que la base
de datos ya esté lista. Revisa ambos registros y espera a que termine la
inicialización:

```bash
docker compose logs --tail=200 db_sonar
docker compose logs --tail=200 sonarqube
```

Si el entorno es descartable y necesitas reiniciarlo completamente, utiliza el
procedimiento destructivo descrito en **Persistencia y respaldos**.

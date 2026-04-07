# infra-devops

## configuracion infra

Para levantar la infraestructura utilizando docker compose debemos:

    docker compose up -d

Si queremos bajar la infraestructura pero conservar los volumenes con las configuraciones de la aplicacion:

    docker compose down

En cambio, si queremos bajar la infraestructura y ademas eliminar los volumenes, lo hacemos con:

    docker compose  down --volumes

En caso de actualizar los plugins de jenkins o hacer algun cambio al jenkins file, es neceario ejecutar lo siguiente para rehacer la imagen:

    docker compose build --no-cache
    docker compose up -d

## jenkins

Revisar plugins instalados, en la consola de script de jenkins:

    Jenkins.instance.pluginManager.plugins.each{ plugin -> 
        println ("${plugin.getDisplayName()} (${plugin.getShortName()}): ${plugin.getVersion()
    }")
}

Las credenciales de sonar deben ser de tipo __*texto secreto*__ y debe contener el token creado en sonarqube

El hook de sonarqube a jenkins debe invocar la url 

    http://jenkins:8080/sonarqube-webhook/



## Sonarqube
Contraseña por defecto tras primera instalacion:

    admin/admin

# urls de acceso:

[Jenkins](http://localhost:8080)

[Sonarqube](http://localhost:8084)
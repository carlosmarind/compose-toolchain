# infra-devops

## configuracion infra

Para levantar la infraestructura utilizando docker compose debemos:

    docker compose up -d

Si queremos bajar la infraestructura pero conservar los volumenes con las configuraciones de la aplicacion:

    docker compose down

En cambio, si queremos bajar la infraestructura y ademas eliminar los volumenes, lo hacemos con:

    docker compose  down --volumes

## jenkins

Revisar plugins instalados, en la consola de script de jenkins:

    Jenkins.instance.pluginManager.plugins.each{ plugin -> 
        println ("${plugin.getDisplayName()} (${plugin.getShortName()}): ${plugin.getVersion()
    }")
}

Las credenciales de nexus deben ser de tipo __*nombre de usuario y contraseña*__

Las credenciales de sonar deben ser de tipo __*texto secreto*__ y debe contener el token creado en sonarqube

El hook de sonarqube a jenkins debe invocar la url 

    http://jenkins:8080/sonarqube-webhook/


## Sonarqube
Contraseña por defecto tras primera instalacion:

    admin/admin

## Nexus

para obtener la contraseña inicial, debes revisar el contenido del fichero _/nexus-data/admin.password del contenedor de nexus. Aca un atajo

    docker exec -it devops-infra-nexus-1 cat /nexus-data/admin.password


Recuerda habilitar "Docker Bearer Token Realm" en el panel de administracion de Nexus para poder autenticar con docker login

    Settings > Security > Realms



# urls de acceso:


[Jenkins](http://localhost:8080)

[Nexus Artifactory](http://localhost:8081)

[Sonarqube](http://localhost:8084)
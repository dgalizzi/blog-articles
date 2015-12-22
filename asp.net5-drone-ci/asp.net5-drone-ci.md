# ASP.NET 5: Integración continua con Drone y Docker

## ASP.NET 5 y Docker

Microsoft ofrece dockerfiles para crear imágenes y contenedores docker que contengan el entorno de ejecución de aplicaciones ASP.NET 5.
La imagen contiene las siguientes aplicaciones:

* dnvm: Gestor de versiones del runtime .NET. Equivalente a RVM de ruby, o virtualenv de python.
* dnx: SDK y entorno de ejecución multi-plataforma de aplicaciones .NET.
* dnu: Utilidad de desarrollo para instalar dependencias nuget. Equivalente a rubygems/bundler en ruby, o npm en nodejs.

Nota: Al momento de escribir este artículo, está en desarrollo _dotnet_, una herramienta que combina dnx y dnu.

Para que sirva como ejemplo creé un repositorio en github que contiene un sencillo test de ejemplo implementado con xunit. Pueden clonarlo de la siguiente manera:

    git clone https://github.com/dgalizzi/asp.net5-drone-ci-example.git
    cd asp.net5-drone-ci-example
    
Luego, podemos probar la aplicación usando la imagen de docker ya mencionada:

    docker run -i -t -v `pwd`:/app microsoft/aspnet:1.0.0-rc1-update1-coreclr /bin/bash
    
Los parámetros indican lo siguiente:
* -i: Modo interactivo.
* -t: Pseudo-tty
* -v: Crear un volumen, esto es, montar el pwd dentro de /app en el contenedor.
* microsoft/aspnet: Imagen sobre la cual iniciar nuestro contenedor.
* :1.0.0-rc1-update1-coreclr: Versión de la imagen a utilizar.
* /bin/bash: comando a ejecutar dentro de nuestro contenedor.

Luego dentro del contenedor ejecutamos:

    dnu restore
    dnx test

Para instalar dependencias y correr los tests xunit implementados. Deberíamos obtener una salida similar a la siguiente:

    xUnit.net DNX Runner (64-bit DNXCore 5.0)
    Discovering: app
    Discovered:  app
    Starting:    app
        MyFirstDnxUnitTests.Class1.FailingTest [FAIL]
        Assert.Equal() Failure
        Expected: 5
        Actual:   4
        Stack Trace:
            at MyFirstDnxUnitTests.Class1.FailingTest()
    Finished:    app
    === TEST EXECUTION SUMMARY ===
    app  Total: 2, Errors: 0, Failed: 1, Skipped: 0, Time: 0.417s


## Drone

Drone es una plataforma de integración continua que trabaja sobre docker. Cada build es realizado sobre un nuevo contenedor, por lo tanto si ya tenemos una imagen docker que nos provea las herramientas para buildear y testear nuestra aplicación, podremos usar drone de manera muy sencilla.

Para utilizar drone primero tenemos que definir ciertas variables de entorno, para ello podemos crear un archivo

### Webhook y github

### .drone.yml

.drone.yml es el archivo de configuración necesario para utilizar drone sobre nuestra aplicación. A continuación el .drone.yml mínimo para ejecutar los tests en nuestra aplicación.

    build:
        image: microsoft/aspnet:1.0.0-rc1-update1-coreclr
        commands:
            - dnu restore
            - dnx test
            
> Nota: dnx test ejecuta el comando test definido en project.json como dnx.runner.xunit.

En el archivo de configuración definimos los servicios que vamos a usar y configuramos dichos servicios. Aquí, **build** representa el servicio a configurar que se encargará de buildear y testear la aplicación. **image** indica la imagen docker que utilizaremos y **commands** es una lista de comandos a ejecutar dentro del contenedor docker. Todos los comandos deben retornar un exit code exitoso para que la integración continua de **success**.

### NuGet cache

Cada build que drone realice va a descargar e instalar todas las dependencias nuevamente. Para evitar esto debemos cachear las dependencias y recuperarlas en cada build. En caso de que las dependencias cambien _dnu restore_ se encargará de resolverlas.

Para definir un cache en drone utilizamos el servicio _cache_ donde definimos archivos o carpetas que serán cacheadas. Un inconveniente es que drone solo puede cachear archivos dentro de /drone (el directorio de trabajo que define drone dentro del contenedor), mientras que la imagen de aspnet que usamos instala los paquetes en /root/.dnx. Hay varias formas de solucionar esto, la que mostramos acá es definiendo la variable de entorno **DNX_PACKAGES** que define el directorio en donde se instalarán los paquetes nuget.

Entonces, sólo basta definir la variable de entorno en alguna carpeta dentro de /drone:

    build:
        image: microsoft/aspnet:1.0.0-rc1-update1-coreclr
        environment:
            - DNX_PACKAGES=/drone/nuget_cache
        commands:
            - dnu restore
            - dnx test

    cache:
        mount:
            - /drone/nuget_cache
            - project.lock.json

Lo primero es agregar en la configuración **environment** del servicio **build** la variable de entorno **DNX_PACKAGES** indicando un directorio donde guardar los paquetes nuget dentro de /drone.

Luego, en el servicio **cache** tenemos que configurar los archivos o carpetas que deseemos cachear para los siguientes builds.

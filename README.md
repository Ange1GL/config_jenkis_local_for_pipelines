# Configuración del servicio Jenkins en Windows (WinSW)

Este documento explica el archivo de definición de servicio `jenkins.xml`, usado por **WinSW** (Windows Service Wrapper) para ejecutar Jenkins como un servicio de Windows.

## Estructura general

| Elemento | Descripción |
|---|---|
| `<id>` | Identificador interno del servicio (`jenkins`). |
| `<name>` | Nombre visible del servicio en Windows. |
| `<description>` | Descripción mostrada en el panel de servicios de Windows. |
| `<env>` | Variables de entorno que se inyectan al proceso. Aquí se define `JENKINS_HOME`. |
| `<executable>` | Ruta al ejecutable Java usado para levantar Jenkins. |
| `<arguments>` | Argumentos que se pasan a la JVM y a Jenkins al iniciar el servicio. |
| `<logmode>` | Modo de rotación de logs (`rotate`). |
| `<onfailure>` | Acción a ejecutar si el proceso falla (`restart`). |
| `<extensions>` | Extensiones de WinSW, como el `RunawayProcessKiller`. |

## Detalle del elemento `<arguments>`

```xml
<arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true -jar "C:\Program Files\Jenkins\jenkins.war" --httpPort=8080 --webroot="%ProgramData%\Jenkins\war"</arguments>
```

Este es el corazón de la configuración: define **cómo se lanza la JVM** y **con qué parámetros arranca Jenkins**. A continuación, el desglose de cada flag:

### Parámetros de la JVM

| Argumento | Función |
|---|---|
| `-Xrs` | Evita que la JVM intercepte señales del sistema operativo (como `SIGHUP`), útil para que Windows pueda controlar correctamente el ciclo de vida del proceso como servicio. |
| `-Xmx256m` | Límite máximo de memoria heap para la JVM (256 MB). Se puede aumentar si Jenkins maneja muchos jobs o plugins pesados. |
| `-Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle` | Le indica a Jenkins que está corriendo como servicio de Windows, para gestionar correctamente el reinicio y apagado. |

### Parámetro clave para pipelines con repos locales

| Argumento | Función |
|---|---|
| **`-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true`** | Habilita que el plugin **Git** de Jenkins pueda hacer *checkout* de repositorios **locales** (rutas del tipo `file:///C:/ruta/al/repo` o rutas de red locales), algo que por defecto está **bloqueado por seguridad** desde versiones recientes del plugin Git, ya que Jenkins asume que los repos siempre son remotos (HTTP/SSH). |

> ⚠️ **Nota de seguridad:** habilitar esta opción permite que un pipeline lea/escriba en cualquier ruta local del sistema Jenkins. Solo se recomienda en entornos controlados (por ejemplo, servidores de desarrollo internos), no en instancias expuestas o compartidas con múltiples usuarios.

### Parámetros de arranque de Jenkins (WAR)

| Argumento | Función |
|---|---|
| `-jar "C:\Program Files\Jenkins\jenkins.war"` | Ruta al archivo `.war` de Jenkins que se ejecuta. |
| `--httpPort=8080` | Puerto HTTP donde Jenkins quedará escuchando (`http://localhost:8080`). |
| `--webroot="%ProgramData%\Jenkins\war"` | Carpeta donde Jenkins descomprime y sirve los archivos del WAR. |

## Variable de entorno `JENKINS_HOME`

```xml
<env name="JENKINS_HOME" value="%ProgramData%\Jenkins\.jenkins"/>
```

Define dónde se almacena toda la configuración de Jenkins: jobs, plugins, credenciales, historial de builds, etc.

## Extensión `RunawayProcessKiller`

```xml
<extension enabled="true" className="winsw.Plugins.RunawayProcessKiller.RunawayProcessKillerExtension" id="killOnStartup">
  <pidfile>%ProgramData%\Jenkins\jenkins.pid</pidfile>
  <stopTimeout>10000</stopTimeout>
  <stopParentFirst>false</stopParentFirst>
</extension>
```

Si WinSW se cierra de forma anómala y deja el proceso Java "huérfano" corriendo, esta extensión lo mata automáticamente al iniciar el servicio, evitando que dos instancias de Jenkins corrompan el `JENKINS_HOME` al mismo tiempo.

## Comandos útiles

```powershell
# Detener el servicio
jenkins.exe stop

# Desinstalar el servicio (requiere detenerlo primero)
jenkins.exe uninstall

# Instalar el servicio (si aplica en tu versión de WinSW)
jenkins.exe install
```

> Estos comandos no muestran salida si se ejecutan correctamente.

## Resumen rápido

Para que Jenkins pueda ejecutar **pipelines apuntando a repositorios Git locales**, el argumento indispensable es:

```
-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true
```

Sin este flag, cualquier `checkout` o `git` step de un pipeline que apunte a una ruta local (no remota) fallará con un error de seguridad del plugin Git.

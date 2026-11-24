# Config Server — Servidor de configuración centralizada

**Diplomatura en Desarrollo de Software FinTech: IA y Microservicios — UTN / Codeki**
**Student: Julita, Jesica R.**

---

## 1. Qué es y para qué sirve

Servidor de configuración centralizada basado en **Spring Cloud Config**.
Lee los archivos de configuración del repositorio git **config-repo** (un repositorio aparte, independiente de este)
y se los entrega a los microservicios cuando arrancan, de modo que la configuración queda fuera de cada aplicación.

Provee:

- La **configuración general** (`application.yml`), común a todos los servicios: la dirección de **Eureka Server**.
- La configuración específica de **product-service** y **customer-service**.

---

## 2. Qué se necesita antes de empezar

- **JDK 21** instalado.
- **IntelliJ IDEA** (Community alcanza).
- **Git** instalado, porque el servidor lee la configuración desde un repositorio git.
- Este servidor **no necesita base de datos**.

Versiones utilizadas:

| Componente | Versión |
|------------|---------|
| Spring Boot | 4.1.2 |
| Spring Cloud | 2025.1.4 |
| Java | 21 |

---

## 3. Cómo abrirlo en IntelliJ IDEA

1. Luego de clonar el repositorio `https://github.com/1251545julita/config-server` abrir IntelliJ.
2. **File > Open** → elegir la carpeta `config-server`, la que contiene el `pom.xml`.
   No abrir el `pom.xml` directamente.
3. Cuando pregunte, aceptar **Trust Project**.
4. Verificar el JDK en `File > Project Structure > Project > SDK`: tiene que apuntar a un **JDK 21**. Si no aparece ninguno, elegir `Add SDK > Download JDK` y descargar una distribución 21.
5. Si no ve el panel de Maven: `View > Tool Windows > Maven`, y tocar el botón de recargar.
6. En `src/main/resources/application.yml` completar la URL del repositorio git en `spring.cloud.config.server.git.uri` (ver punto 4).

**Si el `pom.xml` aparece en gris o no hay carpeta `External Libraries`:** clic derecho
sobre `pom.xml` → **Add as Maven Project**.

---

## 4. Configuración

### `application.yml` del config-server

```yaml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/1251545julita/config-repo   # URL del repositorio git de config-repo

server:
  port: 8888
```

- **`uri`:** repositorio git **config-repo**, donde están los archivos de configuración. Es un repositorio aparte de este.
- Si los archivos `.yml` estuvieran dentro de una subcarpeta de config-repo, habría que agregar `search-paths: <carpeta>`. Si están en la raíz, no hace falta.

### Contenido del repositorio `config-repo`

```
config-repo/                 # repositorio git aparte
├── application.yml          # configuración general: se entrega a todos los servicios
├── product-service.yml      # configuración de product-service
└── customer-service.yml     # configuración de customer-service
```

El `application.yml` general define la dirección de Eureka:

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

El nombre de cada archivo tiene que coincidir con el `spring.application.name` del microservicio.
Cuando un servicio pide su configuración, recibe la general más la propia.

> **Importante:** el servidor lee de git lo que está **commiteado y pusheado**, no los archivos sin guardar.
> Después de cambiar algo en `config-repo`, hacé commit y push antes de probar.

---

## 5. Cómo ejecutarlo

> **Orden:** primero **config-server** (puerto 8888) y **eureka-server** (puerto 8761), después
> **product-service** (puerto 8082) y por último **customer-service** (puerto 8081).

### Desde IntelliJ

Abrí `src/main/java/ar/edu/utn/diploma/configserver/ConfigServerApplication.java` y hacé clic en
la flecha verde ▶ que está al lado de `public static void main`.
La salida aparece en la consola de abajo.

### Desde la terminal, con Maven

```
mvn clean compile        # Compilar
mvn test                 # Ejecutar los tests
mvn spring-boot:run      # Ejecutar la aplicación
mvn clean package        # Generar el paquete ejecutable
java -jar target/config-server-0.0.1-SNAPSHOT.jar
```

### Desde la terminal, utilizando Maven Wrapper

El Maven Wrapper permite ejecutar Maven sin necesidad de tenerlo instalado globalmente.
En Linux, macOS o Git Bash:

```bash
./mvnw clean compile
./mvnw test
./mvnw spring-boot:run
./mvnw clean package
java -jar target/config-server-0.0.1-SNAPSHOT.jar
```

En PowerShell (Windows):

```powershell
.\mvnw.cmd clean compile
.\mvnw.cmd test
.\mvnw.cmd spring-boot:run
.\mvnw.cmd clean package
java -jar target/config-server-0.0.1-SNAPSHOT.jar
```

---

## 6. Cómo probar que funciona

Con la aplicación levantada, desde **Git Bash** (con `curl`):

**Configuración de product-service** (devuelve 200)
```bash
curl -i http://localhost:8888/product-service/default
```

**Configuración de customer-service** (devuelve 200)
```bash
curl -i http://localhost:8888/customer-service/default
```

**Solo la configuración general** (devuelve 200)
```bash
curl -i http://localhost:8888/application/default
```

Tenés que ver un JSON con la lista `propertySources`: el archivo propio del servicio y el `application.yml` general,
con `eureka.client.service-url.defaultZone` entre sus propiedades.

También se puede abrir directamente en el navegador:

http://localhost:8888/product-service/default

### Formas de consultar la configuración

| Ruta | Qué devuelve |
|------|--------------|
| `/{aplicacion}/{perfil}` | Configuración de un servicio para un perfil (`default`, `dev`, etc.) |
| `/{aplicacion}/{perfil}/{rama}` | Lo mismo, leyendo una rama específica |
| `/{aplicacion}-{perfil}.yml` | La configuración ya combinada en formato YAML |

> **Qué significa el `-i`:** muestra también los encabezados de la respuesta, incluido el código de estado.

---

## 7. Cómo lo usa un microservicio

Cada microservicio cliente necesita la dependencia `spring-cloud-starter-config` y apuntar al
config-server en su `application.yml`:

```yaml
spring:
  application:
    name: product-service      # tiene que coincidir con el nombre del archivo en config-repo
  config:
    import: "optional:configserver:http://localhost:8888"
```

Sin `optional:`, el servicio falla al arrancar si no encuentra el config-server.

---

## 8. Estructura del proyecto

```
config-server/
├── pom.xml
├── README.md
├── src/main/resources/application.yml        # Puerto del servidor y URL del repositorio config-repo
├── src/test/*                                # Tests de arranque de la aplicación
└── src/main/java/ar/edu/utn/diploma/configserver/
    └── ConfigServerApplication.java          # Clase principal, anotada con @EnableConfigServer
```

---

## 9. Arquitectura

```
config-repo (repositorio git aparte)
        │  lee archivos commiteados
        ▼
config-server (:8888)
        │  entrega configuración general + específica
        ├────────────────────────► product-service (:8082)
        └────────────────────────► customer-service (:8081)
```

---

## 10. Problemas frecuentes

**`500 Internal Server Error` al consultar `/product-service/default`.** Es la causa más común y casi siempre está en la configuración de git. La excepción real aparece en la consola del config-server (buscá `Caused by:`). Revisá:

- Que `uri` apunte al repositorio config-repo y sea accesible (clonable).- Que `default-label` sea la rama que existe (`main` o `master`).
- Que los cambios de config-repo estén commiteados y pusheados.

**Si usás una ruta local (`file:///...`) en `uri`.** Tiene que apuntar a la carpeta del repositorio que contiene `.git`, y los espacios de la ruta deben escribirse como `%20`.

**Responde 200 pero `propertySources` viene vacío.** El nombre del archivo no coincide con el `spring.application.name` del servicio, o los archivos están en una subcarpeta del repositorio y falta `search-paths`.

**Cambié un archivo de config-repo y el servicio no lo toma.** Hay que commitear y pushear el cambio, y reiniciar el microservicio, porque lee la configuración solo al arrancar.

**Los microservicios fallan con `ConfigClientFailFastException` o `Could not locate PropertySource`.** El config-server no está levantado o respondió con error. Probá `curl -i http://localhost:8888/product-service/default`.

**`Port 8888 was already in use`.** Hay otra aplicación usando ese puerto. Cerrala o cambiá `server.port` en `application.yml` (y en el `spring.config.import` de los microservicios).

**`error: invalid source release: 21`.** El JDK configurado es anterior a 21. Revisá el punto 3.

**El botón ▶ no aparece al lado del `main`.** IntelliJ todavía no terminó de indexar, o no reconoció la carpeta como proyecto Maven. Esperá a que termine la barra de progreso de abajo, o recargá el proyecto desde el panel de Maven.

**Los acentos se ven como `?` o `�` en la consola.** Pasa en `cmd` y en PowerShell por el codepage de Windows. Corré `chcp 65001` antes de ejecutar, o usá la consola de IntelliJ.

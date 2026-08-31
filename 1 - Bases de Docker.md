---

titulo: "Introducción a Docker" tema: Docker seccion: 1 estado: aprendiendo tags:

- docker
- devops
- contenedores creado: 2026-08-27

---

# Introducción a Docker

> [!abstract] Qué vas a lograr Entender qué es Docker y por qué existe, y manejar el ciclo de vida completo de un contenedor desde la consola. La interfaz gráfica la ves al final.

> [!tip] Consola primero, GUI después Docker Desktop no es otra herramienta: es un cliente del mismo motor. Si aprendes los comandos, la GUI se vuelve obvia. Al revés no funciona: terminas dando clics sin saber qué pasa por debajo.

---

## 1. Fundamentos

### 1.1 El problema que Docker resuelve

Antes de Docker, esto era el día a día:

- **"En mi máquina sí funciona"** — el código corre en tu laptop pero truena en el servidor
- **Drift entre ambientes** — dev, staging y producción se van separando poco a poco hasta que ya no se parecen
- **Onboarding eterno** — un desarrollador nuevo tarda dos días en levantar el proyecto
- **Dependencias que chocan** — un proyecto necesita Node 16 y otro Node 22, en la misma máquina

Docker empaqueta tu aplicación **junto con todo lo que necesita para correr**. Ese paquete corre igual en cualquier lado.

### 1.2 Contenedor vs Máquina Virtual

||Máquina Virtual|Contenedor|
|---|---|---|
|Aislamiento|Hipervisor|Kernel compartido|
|Arranque|Minutos|Segundos|
|Peso|Gigabytes|Megabytes|
|SO invitado|Completo|Ninguno|

La VM virtualiza **hardware completo**: le pones un sistema operativo entero encima. El contenedor virtualiza solo el **espacio de usuario** y comparte el kernel del host. Por eso pesa y arranca tan poco.

Dos piezas del kernel de Linux hacen esto posible:

- **Namespaces** → aislamiento. El contenedor tiene su propia vista de procesos, red, filesystem y usuarios. No ve lo de afuera.
- **cgroups** → límites. Cuánta CPU, memoria e I/O puede consumir.

Saber que existen estas dos piezas te quita la sensación de que Docker es magia. No lo es: son funciones del kernel que Docker orquesta por ti.

### 1.3 Los cuatro conceptos base

|Concepto|Qué es|
|---|---|
|**Imagen**|Plantilla inmutable, de solo lectura, hecha de capas|
|**Contenedor**|Instancia en ejecución de una imagen, con una capa escribible encima|
|**Registry**|Dónde viven las imágenes (Docker Hub, GHCR, ECR…)|
|**Volumen**|Dónde vive lo que debe sobrevivir a la muerte del contenedor|

> [!example] La analogía que lo aclara todo **Imagen = clase**, **contenedor = instancia.** Una imagen puede generar N contenedores, igual que una clase genera N objetos. La imagen no cambia; el contenedor sí, pero sus cambios mueren con él.

### 1.4 Cómo está armado Docker

```
Docker CLI  ──►  Docker Daemon (dockerd)  ──►  Registry
                        │
                        ├── Imágenes
                        ├── Contenedores
                        ├── Volúmenes
                        └── Redes
```

Cuando escribes `docker run`, el CLI **no hace nada por sí mismo**: le manda la orden al daemon, que es quien realmente crea el contenedor.

Ten presente esta arquitectura, porque cuando veas el error `Cannot connect to the Docker daemon` vas a saber de inmediato que el problema no es tu comando: es que el motor no está corriendo.

---

## 2. Instalación y verificación

- **Windows** → Docker Desktop sobre WSL2
- **macOS** → Docker Desktop (Apple Silicon e Intel tienen instaladores distintos)
- **Linux** → Docker Engine directamente, no necesitas Desktop

Verifica que todo quedó bien:

```bash
docker version      # versión del cliente y del servidor
docker info         # estado del daemon
docker run hello-world   # prueba de fuego
```

Si `docker version` te muestra el cliente pero no el servidor, el daemon no está arriba.

> [!warning] Arquitecturas Existen imágenes para `amd64` (Intel/AMD) y para `arm64` (Apple Silicon, Raspberry Pi). Si bajas una imagen que no tiene build para tu arquitectura vas a ver `exec format error`.
> 
> Salida de emergencia: `--platform linux/amd64` (corre emulado, más lento).

---

## 3. Trabajar con imágenes

### 3.1 Comandos

```bash
docker pull nginx:1.27       # descargar del registry
docker image ls              # ver las que tienes localmente
docker image inspect nginx   # metadata completa
docker image history nginx   # las capas que la componen
docker image rm nginx        # eliminarla
```

### 3.2 Anatomía de un nombre

```
registry  /  usuario  /  repositorio  :  tag
docker.io /  library  /  nginx        :  latest
```

Cuando escribes `docker pull nginx`, Docker completa lo que faltó: descarga `docker.io/library/nginx:latest`. Casi siempre trabajas con la forma corta, pero es útil saber qué se está asumiendo.

> [!danger] `latest` no significa "la última versión" Es un tag como cualquier otro, y por convención apunta a la versión estable más reciente… **pero puede apuntar a cualquier cosa**, y cambia sin avisarte.
> 
> Si hoy construyes con `latest` y mañana vuelves a construir, puedes obtener una imagen distinta. Es una de las causas más frecuentes de bugs en producción. Fija siempre una versión explícita: `nginx:1.27`.

### 3.3 Reproducibilidad total

```bash
docker pull nginx@sha256:abc123...
```

El _digest_ es un hash del contenido. Es lo único verdaderamente inmutable: ese hash siempre te dará exactamente los mismos bytes.

### 3.4 Por qué las capas importan

Una imagen es un stack de capas de solo lectura. Cuando corres un contenedor, Docker le pone encima una capa escribible.

Esto tiene dos consecuencias prácticas:

1. **Las descargas son rápidas.** Si dos imágenes comparten capas, solo se descarga una vez.
2. **Lo que escribes en el contenedor vive en esa capa superior**, y desaparece cuando borras el contenedor.

Corre `docker image history nginx` y vas a ver las capas listadas una por una.

---

## 4. Ciclo de vida del contenedor

Este es el corazón de la sección.

### 4.1 Crear y ejecutar

```bash
docker container run nginx
docker container run --name web -d -p 8080:80 nginx
docker container run --rm -it ubuntu bash
docker container create nginx        # lo crea pero no lo arranca
```

Flags que vas a usar todos los días:

|Flag|Qué hace|
|---|---|
|`--name`|Le pones nombre en lugar de uno aleatorio|
|`-d` / `--detach`|Corre en segundo plano y te devuelve la terminal|
|`-it`|Interactivo con terminal (para entrar a una shell)|
|`--rm`|Se autoelimina cuando termina|
|`-p` / `--publish`|Mapea `puerto_host:puerto_contenedor`|
|`-e`|Define una variable de entorno|
|`--restart`|Política de reinicio|

### 4.2 Controlar el estado

```bash
docker container stop web      # SIGTERM, apagado ordenado
docker container start web
docker container restart web
docker container kill web      # SIGKILL, muerte inmediata
```

Usa `stop` casi siempre. `kill` no le da a la aplicación la oportunidad de cerrar conexiones ni guardar nada.

### 4.3 Listar

```bash
docker container ls        # solo los que están corriendo
docker container ls -a     # TODOS, incluyendo los detenidos
```

> [!warning] Un contenedor detenido **sigue existiendo y ocupando disco**. Si solo usas `ls`, no los vas a ver y se te van a acumular sin que te des cuenta.

### 4.4 Publicar puertos

```bash
docker container run -d -p 8080:80 nginx   # host:contenedor
docker container run -d -P nginx           # puertos aleatorios
docker container port web                  # ver el mapeo real
```

El orden es `host:contenedor`. Con `-p 8080:80` entras por `localhost:8080` en tu máquina y Docker lo redirige al puerto 80 de adentro. Si te confundes de orden, no vas a poder conectarte.

### 4.5 Variables de entorno

```bash
docker container run -e POSTGRES_PASSWORD=secret postgres
docker container run --env-file .env mi-app
```

### 4.6 Logs

```bash
docker container logs web
docker container logs -f web           # se queda escuchando en vivo
docker container logs --tail 50 web    # solo las últimas 50 líneas
docker container logs -t web           # con timestamps
```

Cuando algo falle, este es tu primer comando. Siempre.

### 4.7 Diagnóstico

Este bloque es el que más te va a servir en la vida real.

```bash
docker container exec -it web sh       # entrar al contenedor vivo
docker container inspect web           # toda la configuración en JSON
docker container inspect --format '{{.NetworkSettings.IPAddress}}' web
docker container stats                 # CPU y memoria en tiempo real
docker container top web               # procesos internos
docker container cp web:/etc/nginx/nginx.conf ./nginx.conf
```

> [!important] `run`, `exec` y `attach` no son lo mismo
> 
> - **`run`** → crea un **contenedor nuevo**
> - **`exec`** → lanza un **proceso nuevo dentro** de un contenedor que ya está corriendo
> - **`attach`** → te conecta al proceso principal
> 
> Cuando quieras "meterte a ver qué pasa", el comando es `exec`, no `attach`. Con `attach` un `Ctrl+C` mal dado mata el contenedor completo.

Si `sh` no funciona, prueba `bash`. Algunas imágenes mínimas (Alpine) solo traen `sh`.

### 4.8 Eliminar y mantener limpio el disco

```bash
docker container rm web
docker container rm -f web        # forzar aunque esté corriendo
docker container prune            # borra todos los detenidos
docker system df                  # cuánto disco estás usando
docker system prune               # limpieza general
```

> [!tip] Hazlo un hábito desde ahora Si nunca corres `prune`, en un par de semanas vas a tener decenas de gigabytes en contenedores e imágenes muertas. Corre `docker system df` de vez en cuando y te vas a sorprender.

### 4.9 Políticas de reinicio

|Política|Comportamiento|
|---|---|
|`no`|Default. No reinicia nunca|
|`on-failure`|Solo si salió con código distinto de 0|
|`unless-stopped`|Reinicia siempre, salvo que tú lo hayas detenido|
|`always`|Reinicia siempre, incluso si tú lo detuviste|

Para desarrollo local, `unless-stopped` suele ser la más cómoda.

---

## 5. Dos cosas que confunden a todo el mundo al principio

### 5.1 Un contenedor vive mientras viva su proceso principal

Un contenedor **no es una máquina virtual encendida**: es un proceso. Cuando ese proceso (el PID 1) termina, el contenedor termina.

```bash
docker run ubuntu          # arranca y muere de inmediato
docker run -d nginx        # se queda corriendo
```

`ubuntu` no tiene un proceso de larga duración: ejecuta su comando por defecto, termina, y con él muere el contenedor. `nginx` levanta un servidor que se queda escuchando, así que el contenedor sigue vivo.

Cuando veas un contenedor que "no arranca", lo primero que debes preguntarte es si de verdad falló o si simplemente su proceso terminó de forma normal. Revisa el código de salida con `docker container ls -a`.

### 5.2 El contenedor no es el `localhost` de tu máquina

Cada contenedor tiene su propia red. Si tu app dentro de un contenedor intenta conectarse a `localhost:5432` buscando una base de datos que corre en **otro** contenedor, no la va a encontrar: `localhost` ahí dentro es él mismo.

```bash
docker network ls
docker network create mi-red
docker run -d --network mi-red --name db postgres
docker run -d --network mi-red --name api mi-app
```

Al estar en la misma red creada por ti, `api` puede conectarse a `db` **usando el nombre del contenedor** como hostname. La red `bridge` por defecto no te da esa resolución por nombre, por eso conviene crear una propia.

---

## 6. Variables de entorno, vistas como arquitectura

Este punto parece un simple flag, pero detrás hay un principio de diseño importante.

> [!quote] Principio III de las 12-Factor App **Configuración por entorno, código por imagen.** La _misma_ imagen debe correr en desarrollo, staging y producción cambiando únicamente variables de entorno.

De ahí se desprenden tres reglas prácticas:

1. **Nunca hornees secretos en la imagen.** Cualquiera con `docker image history` puede verlos. Los secretos entran en tiempo de ejecución.
2. **Usa `.env` y agrégalo a `.gitignore`.** Después lo pasas con `--env-file .env`.
3. **`-e` sobrescribe** lo que venga definido en el `Dockerfile`.

**Practica esto:** levanta un `postgres` cambiando solo los flags `-e` y observa cómo cambia el comportamiento sin recompilar absolutamente nada. Esa es la idea completa.

---

## 7. Docker Desktop

Ahora sí, después de los comandos.

- Dashboard de contenedores e imágenes
- Logs y terminal integrada
- Ajuste de recursos asignados (CPU/RAM), importante en Mac y Windows
- Visor de volúmenes y redes
- Extensiones

> [!example] Comprobación Levanta un contenedor desde la interfaz de Docker Desktop y luego búscalo con `docker container ls`. Ahí está. Es el mismo daemon, la misma información. La GUI solo le está mandando las mismas órdenes que tú escribirías.

**Alternativas que existen**, por si algún día trabajas en una empresa sin licencia de Docker Desktop: Rancher Desktop, Podman, Colima, OrbStack.

---

## 8. Práctica integradora

> [!success] Hazla completa, sin saltarte pasos

1. Levanta `nginx` en el puerto `8080`, con nombre propio, en modo detached y con `--restart unless-stopped`
2. Verifica en el navegador que responde
3. Entra con `docker container exec -it`
4. Modifica el `index.html` dentro del contenedor
5. Recarga y confirma que ves tu cambio
6. Revisa los logs
7. Borra el contenedor
8. Vuelve a levantarlo con el mismo comando

**Tu cambio desapareció.**

> [!question] Y esa es exactamente la pregunta correcta ¿Cómo hago para que los datos sobrevivan al contenedor? Esa es la razón de existir de los volúmenes → [[02 - Volúmenes y persistencia]]

---

## 9. Errores que vale la pena provocar a propósito

Rompe las cosas ahora, en un entorno donde no importa. Así reconoces el error cuando aparezca de verdad.

|Error|Qué te está enseñando|
|---|---|
|`port is already allocated`|Ya tienes algo usando ese puerto del host|
|`Cannot connect to the Docker daemon`|El motor no está corriendo, no es tu comando|
|El contenedor "no arranca"|Su proceso principal terminó normalmente|
|`docker rm` de uno corriendo|Ciclo de vida, y para qué sirve `-f`|
|`exec format error`|Descargaste una imagen de otra arquitectura|

---

## 10. Autoevaluación

Marca cuando puedas hacerlo sin consultar la nota:

- [ ] Explicar con mis palabras la diferencia entre imagen y contenedor
- [ ] Decir cuándo conviene un contenedor y cuándo una VM
- [ ] Levantar y eliminar contenedores publicando puertos
- [ ] Pasar configuración con variables de entorno y `--env-file`
- [ ] Diagnosticar un contenedor que falla usando `logs` e `inspect`
- [ ] Entrar a un contenedor vivo con `exec` y saber por qué no uso `attach`
- [ ] Explicar por qué mi cambio se perdió al borrar el contenedor
- [ ] Mantener limpio mi disco con `prune`

---

## Chuleta rápida

```bash
# Imágenes
docker pull <img>:<tag>
docker image ls | inspect | history | rm

# Ciclo de vida
docker container run [-d] [-it] [--rm] [--name X] [-p H:C] [-e K=V] <img>
docker container ls [-a]
docker container stop | start | restart | kill <x>
docker container rm [-f] <x>

# Diagnóstico
docker container logs [-f] [--tail N] <x>
docker container exec -it <x> sh
docker container inspect <x>
docker container stats

# Limpieza
docker container prune
docker system df
docker system prune
```

---

## Siguientes notas

- [[02 - Volúmenes y persistencia]]
- [[03 - Dockerfile e imágenes propias]]
- [[04 - Redes en Docker]]
- [[05 - Docker Compose]]
- [Documentación oficial](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
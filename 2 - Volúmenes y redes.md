---
titulo: "Volúmenes y redes en Docker"
tema: Docker
seccion: 2
estado: aprendiendo
tags:
  - docker
  - devops
  - contenedores
  - volumenes
  - redes
creado: 2026-10-12
---
```cmd
docker run \
--name world-db \
-dp 3306:3306 \
-v world-db-volume:/var/lib/mysql \
-e MARIADB_USER=example-user \
-e MARIADB_PASSWORD=user-password \
-e MARIADB_ROOT_PASSWORD=root-secret-password \
-e MARIADB_DATABASE=world-db \
mariadb:jammy
```
# Volúmenes y redes en Docker

> [!abstract] Qué vas a lograr
> Dejar de ver los contenedores como piezas sueltas. Al terminar vas a poder meterte a un contenedor con una terminal interactiva, hacer que tus datos sobrevivan al contenedor, conectar varios contenedores entre sí por nombre y levantar una aplicación real de tres piezas: **MariaDB + phpMyAdmin (sobre Apache)**.

> [!tip] La pregunta que te trajo aquí
> En la práctica de [[1 - Bases de Docker]] borraste el contenedor y **tu cambio desapareció**. Esta sección responde dos preguntas:
> 1. ¿Cómo hago que los datos sobrevivan? → **volúmenes**
> 2. ¿Cómo hago que dos contenedores hablen entre sí? → **redes**

---

## 1. Terminal interactiva dentro del contenedor

### 1.1 Qué significa realmente `-it`

Son dos flags distintos que casi siempre van juntos:

|Flag|Qué hace|Sin él…|
|---|---|---|
|`-i` / `--interactive`|Mantiene abierto el STDIN: puedes **escribirle** al contenedor|El contenedor no recibe lo que tecleas|
|`-t` / `--tty`|Asigna una pseudo-terminal: prompt, colores, `Ctrl+C`|Ves texto plano, sin prompt, y las teclas especiales no funcionan bien|

```bash
docker container run -it --rm alpine sh       # contenedor nuevo con shell
docker container run -it --rm node:22 bash    # node trae bash
docker container exec -it web sh              # shell dentro de uno que YA corre
```

### 1.2 `run -it` vs `exec -it`

> [!important] La diferencia que importa
> - **`run -it imagen sh`** → la shell **es** el proceso principal (PID 1). Si escribes `exit`, el contenedor muere.
> - **`exec -it contenedor sh`** → la shell es un **proceso extra**. Si escribes `exit`, solo se cierra tu shell; el contenedor sigue vivo.

Esto conecta directo con la regla de la sección anterior: _un contenedor vive mientras viva su proceso principal_.

### 1.3 Salir sin matar el contenedor

Si entraste con `run -it` y quieres dejarlo corriendo:

```
Ctrl + P   y luego   Ctrl + Q     # te desconectas, el contenedor sigue
```

Y para volver a entrar a uno detenido que fue creado con `-it`:

```bash
docker container start -ai mi-contenedor   # -a attach, -i interactivo
```

---

## 2. Revisar el filesystem de Alpine y Node

Antes de montar volúmenes conviene saber **qué hay adentro** de una imagen. Es la mejor forma de perderle el miedo.

### 2.1 Alpine

```bash
docker container run -it --rm alpine sh
```

Ya adentro:

```sh
cat /etc/os-release     # qué distribución es
ls /                    # bin etc home lib media mnt opt proc root run sbin srv sys tmp usr var
ls -l /bin              # casi todo apunta a /bin/busybox
which bash              # no existe → por eso usamos sh
apk add curl            # el gestor de paquetes de Alpine
ps                      # vas a ver que sh es el PID 1
```

Lo que debes notar:

- **Pesa ~8 MB.** No trae casi nada: ni `bash`, ni `curl`, ni editores.
- Usa **BusyBox**: un solo binario que se hace pasar por `ls`, `cp`, `grep`, etc.
- Usa **musl** en vez de **glibc**. Algunos binarios compilados para Debian/Ubuntu no corren aquí.
- El gestor de paquetes es `apk`, no `apt`.

### 2.2 Node

```bash
docker container run -it --rm node:22 bash
```

```bash
cat /etc/os-release          # Debian
which node npm               # /usr/local/bin/...
node -v
ls /usr/local/lib/node_modules   # npm, corepack...
id node                      # usuario 'node' con uid 1000
ls /home                     # existe /home/node
echo $PATH
```

Y si no le pasas comando:

```bash
docker container run -it --rm node:22    # te abre el REPL de node directamente
```

Eso pasa porque el `CMD` por defecto de la imagen es `node`. Puedes verlo con:

```bash
docker image inspect node:22 --format '{{.Config.Cmd}}'
```

### 2.3 Compara tamaños

```bash
docker pull node:22
docker pull node:22-alpine
docker image ls node
```

|Imagen|Base|Tamaño aprox.|
|---|---|---|
|`alpine`|Alpine|~8 MB|
|`node:22-alpine`|Alpine|~150 MB|
|`node:22-slim`|Debian recortado|~200 MB|
|`node:22`|Debian completo|~1 GB|

> [!tip] ¿Cuál uso?
> Para aprender y depurar, la completa: trae todo. Para producción, `-alpine` o `-slim`. Si algo raro falla en Alpine (módulos nativos, binarios), sospecha de **musl** antes que de tu código.

---

## 3. Volúmenes: que los datos sobrevivan

### 3.1 El problema

Recuerda las capas: la imagen es de solo lectura y el contenedor escribe en una **capa delgada encima**. Esa capa vive y muere con el contenedor.

```
┌─────────────────────────┐
│  Capa escribible        │  ← se borra con `docker rm`
├─────────────────────────┤
│  Capas de la imagen     │  ← solo lectura
└─────────────────────────┘
```

Un volumen es un directorio que vive **fuera** de ese stack de capas y que Docker "conecta" en una ruta del contenedor.

### 3.2 Los tres tipos de montaje

|Tipo|Dónde viven los datos|Quién lo administra|Uso típico|
|---|---|---|---|
|**Volumen nombrado**|`/var/lib/docker/volumes/...`|Docker|Bases de datos, datos persistentes|
|**Bind mount**|Una carpeta tuya del host|Tú|Código fuente en desarrollo, archivos de config|
|**tmpfs**|Memoria RAM|Nadie, es efímero|Datos temporales o sensibles que no deben tocar disco|

```
          Host                              Contenedor
┌──────────────────────────┐        ┌──────────────────────┐
│ Docker volumes/db-data ──┼───────►│ /var/lib/mysql       │  volumen nombrado
│ ~/proyecto/src ──────────┼───────►│ /app                 │  bind mount
│ RAM ─────────────────────┼───────►│ /tmp/cache           │  tmpfs
└──────────────────────────┘        └──────────────────────┘
```

### 3.3 Volúmenes nombrados

```bash
docker volume create db-data
docker volume ls
docker volume inspect db-data      # ve el "Mountpoint" real en el host
docker volume rm db-data
docker volume prune                # borra los que no usa ningún contenedor
```

Usarlo:

```bash
docker container run -d --name db \
  -v db-data:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD=secret \
  mariadb:11
```

Si `db-data` no existe, Docker lo crea al vuelo.

> [!example] Prueba que de verdad persiste
> 1. Levanta `db` con el volumen y crea una base de datos
> 2. `docker container rm -f db`
> 3. Levanta otro contenedor con **el mismo** `-v db-data:/var/lib/mysql`
> 4. La base de datos sigue ahí
>
> El contenedor es desechable. El volumen no.

### 3.4 Volúmenes anónimos

```bash
docker container run -d -v /var/lib/mysql mariadb:11
```

Sin nombre antes de los `:`, Docker crea un volumen con un hash como nombre. Funciona, pero es casi imposible saber a qué pertenece después. Muchas imágenes (MariaDB, Postgres) los crean solas porque lo declaran con `VOLUME` en su Dockerfile.

> [!warning] Aquí se te va el disco
> Cada vez que levantas una base de datos sin `-v`, se crea un volumen anónimo nuevo que **no se borra** con `docker rm`. Revísalos con `docker volume ls` y usa `docker container rm -v` o `docker volume prune`.

---

## 4. Mapeo de directorios (bind mounts)

### 4.1 La sintaxis

```bash
docker container run -d --name web -p 8080:80 \
  -v "$PWD/html":/usr/share/nginx/html \
  nginx:1.27
```

El orden es el mismo que con puertos: **`host:contenedor`**.

Ahora edita `html/index.html` en tu editor favorito, recarga el navegador y el cambio aparece **al instante**. Borra el contenedor, vuélvelo a crear: el archivo sigue ahí, porque siempre fue tuyo.

> [!success] Esto resuelve la práctica de la sección 1
> El `index.html` ya no vive dentro del contenedor. Vive en tu disco y el contenedor solo lo está "viendo".

### 4.2 Solo lectura

```bash
-v "$PWD/html":/usr/share/nginx/html:ro
```

`:ro` (read-only) evita que el contenedor modifique tus archivos. Úsalo siempre que el contenedor solo necesite **leer** (configs, sitios estáticos).

### 4.3 `-v` vs `--mount`

```bash
# Corta
-v db-data:/var/lib/mysql
-v "$PWD/html":/usr/share/nginx/html:ro

# Explícita
--mount type=volume,source=db-data,target=/var/lib/mysql
--mount type=bind,source="$PWD/html",target=/usr/share/nginx/html,readonly
```

|Diferencia|`-v`|`--mount`|
|---|---|---|
|Legibilidad|Corta pero críptica|Larga pero clara|
|Si la ruta del host no existe|**La crea** como carpeta vacía (de root)|**Falla** con error|
|Cómo distingue volumen de bind|Si empieza con `/` o `.` es bind|`type=` lo dice explícito|

> [!tip] `--mount` te protege de typos
> Con `-v "$PWD/htlm":/...` (mal escrito) Docker crea una carpeta vacía y nginx sirve un 403. Con `--mount` te avisa del error de inmediato.

### 4.4 Volumen nombrado vs bind mount: el detalle que confunde

|Situación|Volumen nombrado **vacío**|Bind mount|
|---|---|---|
|La imagen ya tenía archivos en esa ruta|Docker **copia** esos archivos al volumen|Tu carpeta **tapa** lo que había|

Por eso si haces bind mount de una carpeta vacía sobre `/usr/share/nginx/html`, el `index.html` de bienvenida de nginx "desaparece". No se borró: quedó oculto debajo de tu carpeta.

### 4.5 Bind mount para desarrollo con Node

No necesitas instalar Node en tu máquina:

```bash
docker container run -it --rm \
  -v "$PWD":/app \
  -w /app \
  -p 3000:3000 \
  node:22-alpine sh
```

- `-v "$PWD":/app` → tu proyecto aparece en `/app`
- `-w /app` → el directorio de trabajo inicial
- Adentro corres `npm install`, `npm run dev`, etc.

> [!warning] Permisos en Linux
> Los archivos que el contenedor crea (por ejemplo `node_modules`) van a quedar con dueño **root** en tu máquina. Para evitarlo, corre como tu propio usuario:
> ```bash
> docker container run -it --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app node:22-alpine sh
> ```

### 4.6 Respaldar un volumen

Truco clásico: un contenedor temporal que monta **el volumen y una carpeta tuya** al mismo tiempo.

```bash
# Respaldar
docker container run --rm \
  -v db-data:/data:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/db-data.tar.gz -C /data .

# Restaurar
docker container run --rm \
  -v db-data:/data \
  -v "$PWD":/backup \
  alpine tar xzf /backup/db-data.tar.gz -C /data
```

Alpine aquí es solo una "navaja suiza" desechable.

---

## 5. Redes: que los contenedores se hablen

### 5.1 Recordatorio

En [[1 - Bases de Docker]] viste que **`localhost` dentro de un contenedor es el propio contenedor**. Para que `api` encuentre a `db` necesitas que estén en la misma red.

### 5.2 Los drivers de red

```bash
docker network ls
```

|Driver|Qué hace|Cuándo usarlo|
|---|---|---|
|`bridge`|Red privada virtual en tu host. **Default**|Casi siempre|
|`host`|El contenedor usa directamente la red del host, sin aislamiento|Rendimiento extremo, solo Linux|
|`none`|Sin red|Procesos que no deben tener red|
|`overlay`|Red entre varios hosts|Docker Swarm|
|`macvlan`|El contenedor obtiene su propia MAC/IP en tu LAN|Casos especiales|

### 5.3 Bridge por defecto vs bridge propio

|Característica|`bridge` por defecto|Red creada por ti|
|---|---|---|
|DNS por nombre de contenedor|❌ No|✅ Sí|
|Aislamiento entre proyectos|❌ Todos juntos|✅ Cada red separada|
|Conectar/desconectar en caliente|❌|✅|

> [!important] Regla práctica
> **Nunca** uses la red `bridge` por defecto para aplicaciones de varios contenedores. Crea siempre una red propia por proyecto.

### 5.4 Comandos

```bash
docker network create app-net
docker network inspect app-net               # ve los contenedores conectados y sus IPs
docker network connect app-net mi-contenedor # conectar uno que ya existe
docker network disconnect app-net mi-contenedor
docker network rm app-net
docker network prune
```

### 5.5 Comprobar el DNS interno

```bash
docker network create test-net
docker container run -d --name servidor --network test-net nginx:1.27
docker container run -it --rm --network test-net alpine sh
```

Desde la shell de Alpine:

```sh
ping servidor               # resuelve por nombre
wget -qO- http://servidor   # te devuelve el HTML de nginx
```

Ahora repite **sin** `--network test-net` y verás que `servidor` no resuelve.

> [!tip] Alias
> `--network-alias db` le da al contenedor un nombre extra en esa red. Útil si tu app espera conectarse a `db` pero el contenedor se llama `mariadb-prod`.

### 5.6 Puertos publicados vs red interna

Esto es lo que más se confunde:

- **`-p 8080:80`** → sirve para que **tú (el host)** entres al contenedor
- **La red interna** → los contenedores se hablan entre sí por **el puerto interno**, sin necesidad de `-p`

```
Navegador ──► localhost:8080 ──(-p)──► phpmyadmin:80
                                            │
                                     red app-net
                                            │
                                            ▼
                                       mariadb:3306   ← sin -p, nadie de afuera la ve
```

> [!danger] No publiques la base de datos si no lo necesitas
> Si `mariadb` solo la usa otro contenedor, **no le pongas `-p 3306:3306`**. Así no queda expuesta en tu máquina ni en tu red.

---

## 6. Aplicaciones con múltiples contenedores

Una aplicación real casi nunca es un solo contenedor. Lo normal es:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │────►│   Backend    │────►│ Base de datos│
└──────────────┘     └──────────────┘     └──────────────┘
```

**Un proceso por contenedor.** No metas Apache, PHP y MariaDB en un mismo contenedor: pierdes la capacidad de escalarlos, actualizarlos y reiniciarlos por separado.

Las relaciones entre ellos se arman con lo que ya viste:

|Relación|Herramienta|
|---|---|
|Quién habla con quién|**Redes**|
|Qué datos sobreviven|**Volúmenes**|
|Qué se expone al exterior|**Puertos publicados**|
|Cómo se configura cada uno|**Variables de entorno**|

---

## 7. Práctica: MariaDB + phpMyAdmin (Apache)

La imagen oficial `phpmyadmin` ya trae **Apache + PHP** adentro. Tú solo tienes que conectarla con MariaDB.

### 7.1 Red y volumen

```bash
docker network create mariadb-net
docker volume create mariadb-data
```

### 7.2 MariaDB

```bash
docker container run -d \
  --name mariadb \
  --network mariadb-net \
  -v mariadb-data:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD=root123 \
  -e MARIADB_DATABASE=world-db \
  -e MARIADB_USER=usuario \
  -e MARIADB_PASSWORD=usuario123 \
  --restart unless-stopped \
  mariadb:11
```

Nota que **no** tiene `-p`. Solo la va a ver phpMyAdmin a través de la red.

Revisa que arrancó bien:

```bash
docker container logs -f mariadb    # espera "ready for connections"
```

### 7.3 phpMyAdmin

```bash
docker container run -d \
  --name phpmyadmin \
  --network mariadb-net \
  -e PMA_HOST=mariadb \
  -p 8080:80 \
  --restart unless-stopped \
  phpmyadmin:5
```

- `PMA_HOST=mariadb` → **el nombre del contenedor** funciona como hostname gracias a la red propia
- `-p 8080:80` → Apache escucha en el 80 dentro, tú entras por el 8080

### 7.4 Comprobar

1. Abre `http://localhost:8080`
2. Entra con `root` / `root123` (o `usuario` / `usuario123`)
3. Crea una tabla dentro de `world-db` e inserta un par de filas

### 7.5 La prueba de fuego

```bash
docker container rm -f mariadb phpmyadmin
```

Vuelve a correr los comandos de 7.2 y 7.3 tal cual. Entra a phpMyAdmin.

**Tu tabla sigue ahí.** Lo que se borró fue el contenedor; los datos vivían en `mariadb-data`.

> [!question] Y ahora la pregunta obvia
> Escribir estos `docker run` larguísimos, en orden, cada vez, con la red y el volumen creados antes… es tedioso y fácil de equivocarse. ¿No hay una forma de describir todo esto en un archivo y levantarlo con un solo comando?
> Sí: **Docker Compose** → [[3 - Docker Compose]]

### 7.6 Limpieza

```bash
docker container rm -f mariadb phpmyadmin
docker network rm mariadb-net
docker volume rm mariadb-data      # ⚠️ aquí sí se pierden los datos
```

---

## 8. Errores que vale la pena provocar a propósito

|Error|Qué te está enseñando|
|---|---|
|`ping db` falla desde otro contenedor|Están en la red `bridge` por defecto o en redes distintas|
|phpMyAdmin: `mysqli::real_connect(): (HY000/2002)`|`PMA_HOST` mal escrito, o MariaDB todavía no termina de arrancar|
|Nginx da 403 después de un bind mount|La carpeta del host está vacía o la ruta tiene un typo|
|`node_modules` con dueño root en tu máquina|El contenedor corre como root; usa `-u $(id -u):$(id -g)`|
|`bash: not found` en Alpine|Alpine solo trae `sh`|
|`docker volume rm` → `volume is in use`|Algún contenedor (aunque esté detenido) lo sigue usando|
|`network has active endpoints`|Desconecta o borra los contenedores antes de borrar la red|
|Escribes `exit` y el contenedor muere|Entraste con `run -it`: la shell era el PID 1|

---

## 9. Autoevaluación

Marca cuando puedas hacerlo sin consultar la nota:

- [ ] Explicar qué hace `-i` y qué hace `-t` por separado
- [ ] Salir de un contenedor sin matarlo
- [ ] Explorar el filesystem de una imagen y decir qué base usa
- [ ] Explicar por qué Alpine es tan pequeña y qué problemas puede traer
- [ ] Decir cuándo uso volumen nombrado, bind mount o tmpfs
- [ ] Explicar por qué un bind mount "esconde" archivos de la imagen
- [ ] Respaldar y restaurar un volumen con un contenedor temporal
- [ ] Explicar por qué no uso la red `bridge` por defecto
- [ ] Conectar dos contenedores por nombre
- [ ] Explicar la diferencia entre publicar un puerto y comunicarse por la red interna
- [ ] Levantar MariaDB + phpMyAdmin y comprobar que los datos sobreviven

---

## Chuleta rápida

```bash
# Terminal interactiva
docker container run -it --rm <img> sh
docker container exec -it <x> sh
docker container start -ai <x>
# Ctrl+P Ctrl+Q → salir sin detener

# Volúmenes
docker volume create | ls | inspect | rm | prune <v>
docker container run -v <volumen>:/ruta <img>
docker container run -v "$PWD/dir":/ruta[:ro] <img>
docker container run --mount type=bind,source="$PWD",target=/app <img>
docker container run --tmpfs /tmp <img>

# Redes
docker network create | ls | inspect | rm | prune <red>
docker network connect | disconnect <red> <x>
docker container run --network <red> [--network-alias <alias>] <img>

# Combo multi-contenedor
docker network create app-net
docker volume create app-data
docker run -d --name db  --network app-net -v app-data:/var/lib/mysql -e ... mariadb:11
docker run -d --name web --network app-net -p 8080:80 -e PMA_HOST=db phpmyadmin:5
```

---

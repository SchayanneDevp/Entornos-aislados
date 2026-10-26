# Guía de Despliegue y Aislamiento para Modelamiento Económico, Ciencia de Datos y Software a la Medida

> **Versión 1.0 · Octubre 2026**
> **Stack:** R · Python · Stata · Julia · Frontend/Backend web
> **Entornos de despliegue:** Linux Server · Windows Server · Hostinger VPS

---

## Índice

0. [Alcance y cómo usar esta guía](#0-alcance-y-cómo-usar-esta-guía)
1. [Fundamentos: hardware, kernel y virtualización](#1-fundamentos-hardware-kernel-y-virtualización)
2. [Matriz de decisión: qué nivel de aislamiento usar](#2-matriz-de-decisión-qué-nivel-de-aislamiento-usar)
3. [Receta A: modelamiento económico y ciencia de datos reproducibles](#3-receta-a-modelamiento-económico-y-ciencia-de-datos-reproducibles)
4. [Receta B: software a la medida (frontend + backend)](#4-receta-b-software-a-la-medida-frontend--backend)
5. [Recetas por plataforma](#5-recetas-por-plataforma)
6. [CI/CD y estrategias de despliegue](#6-cicd-y-estrategias-de-despliegue)
7. [Seguridad y cadena de suministro](#7-seguridad-y-cadena-de-suministro)
8. [Checklists](#8-checklists)
9. [Bibliografía anotada](#9-bibliografía-anotada)
10. [Ruta de aprendizaje](#10-ruta-de-aprendizaje)
11. [Glosario](#11-glosario)

---

## 0. Alcance y cómo usar esta guía

### 0.1 Qué problema resuelve

Dos tipos de proyecto, dos tipos de aislamiento:

| Tipo de proyecto | Qué se busca aislar | Riesgo principal |
|---|---|---|
| **Modelamiento económico / ciencia de datos** | Versiones de lenguajes, paquetes, librerías del sistema, semillas aleatorias | "Hoy corre, en seis meses da otro resultado o no corre" |
| **Software a la medida para clientes** | Un cliente de otro, un ambiente de otro (dev/staging/prod), secretos, recursos | Fuga de datos entre clientes, caídas en cascada, despliegues irreversibles |

### 0.2 Supuestos

- Despliegas en **servidor local** (Linux Server y Windows Server) y en **Hostinger**.
- Hostinger: esta guía asume un **VPS** (virtualización KVM, acceso root, soporte de Docker). El **hosting compartido** normalmente no permite instalar Docker ni controlar el sistema operativo; allí solo caben sitios estáticos o PHP/WordPress. Verifica el plan concreto en el panel de Hostinger.
- Trabajas en solitario o con un equipo pequeño. Por eso la guía prefiere herramientas simples (Docker Compose, Make, GitHub Actions) antes que Kubernetes.

### 0.3 Convenciones

- Las versiones de imágenes y herramientas en los ejemplos (`python:3.12-slim`, `rocker/r-ver:4.4.2`, `julia:1.10`) son **ilustrativas**. Fija siempre las versiones que realmente uses y revisa las vigentes antes de desplegar.
- Los bloques de código son plantillas: adáptalos, no los copies a ciegas.
- Dónde aparece `<ALGO>`, debes reemplazarlo.

---

## 1. Fundamentos: hardware, kernel y virtualización

Esta sección es breve pero sólida. Todo el aislamiento que verás después (entornos, contenedores, máquinas virtuales) se apoya en estas piezas.

### 1.1 El hardware: las piezas que importan

| Pieza | Qué hace | Por qué te importa en despliegues |
|---|---|---|
| **CPU** | Ejecuta instrucciones. Tiene núcleos, hilos, cachés y **modos de privilegio** | Define cuántos procesos corren en paralelo y qué instrucciones (SIMD, extensiones de virtualización) existen |
| **RAM** | Memoria volátil rápida | Un modelo en R, Stata o Julia que excede la RAM muere o se vuelve lentísimo (swap) |
| **MMU** (unidad de gestión de memoria, dentro de la CPU) | Traduce direcciones virtuales a físicas y hace cumplir permisos | Es lo que impide que un proceso lea la memoria de otro: base de todo aislamiento |
| **Almacenamiento** (NVMe/SSD/HDD) | Datos persistentes | Los datos de clientes y los volúmenes de contenedores viven aquí; define los backups |
| **Tarjeta de red (NIC)** | Entrada y salida de paquetes | Puertos, firewalls y redes virtuales dependen de ella |
| **GPU** (opcional) | Cálculo masivamente paralelo | Solo si tus modelos la usan; complica el aislamiento (drivers, passthrough) |
| **Firmware (UEFI/BIOS)** | Primer código que corre al encender | Inicializa el hardware y entrega el control al *bootloader* |

**Modos de privilegio de la CPU.** La CPU distingue al menos dos niveles de privilegio:

- **Modo kernel** (en x86, "anillo 0"): puede ejecutar cualquier instrucción y tocar cualquier dirección de memoria o dispositivo.
- **Modo usuario** (en x86, "anillo 3"): tus programas (R, Python, Julia, Stata, Nginx). No pueden tocar hardware directamente.

Un programa en modo usuario que necesita algo privilegiado (leer un archivo, abrir un socket, pedir memoria) debe pedírselo al kernel mediante una **llamada al sistema** (*syscall*). Ese es el único punto de entrada, y por eso es el sitio donde se aplican muchos controles de seguridad.

### 1.2 El kernel: el puente entre el hardware y tus programas

El **kernel** es el núcleo del sistema operativo. Corre en modo privilegiado y cumple cinco funciones:

1. **Planificar procesos**: decide qué proceso usa qué núcleo de CPU y por cuánto tiempo (*scheduler*).
2. **Gestionar la memoria**: da a cada proceso su propio espacio de direcciones virtual usando la MMU, y decide cuándo mover páginas a disco (swap) o matar procesos si falta memoria (en Linux, el *OOM killer*).
3. **Abstraer el almacenamiento**: ofrece el sistema de archivos (ext4, XFS, NTFS) sobre discos físicos.
4. **Manejar la red**: implementa TCP/IP, firewall (netfilter en Linux) y entrega paquetes a los procesos correctos.
5. **Controlar el acceso al hardware**: mediante **drivers**, que traducen las órdenes genéricas del kernel a las particularidades de cada dispositivo.

```
┌──────────────────────────────────────────────────────────┐
│  MODO USUARIO     R · Python · Julia · Stata · Nginx · DB │
│                   (cada proceso cree tener la máquina    │
│                    entera para sí: memoria virtual)      │
├────────────────────── syscalls ──────────────────────────┤
│  MODO KERNEL      Planificador │ Memoria │ Archivos      │
│                   Red │ Seguridad │ Drivers              │
├──────────────────────────────────────────────────────────┤
│  HARDWARE         CPU · RAM · Disco · NIC · GPU          │
└──────────────────────────────────────────────────────────┘
```

#### Cómo está construido

| | **Linux** | **Windows (NT)** |
|---|---|---|
| Lenguajes | Principalmente **C** y ensamblador para cada arquitectura; desde la serie 6.x se admite **Rust** para algunos componentes | Principalmente **C** y **C++**, con ensamblador |
| Diseño | **Monolítico modular**: casi todo corre en modo kernel, pero los drivers pueden cargarse como **módulos** (`.ko`) sin reiniciar | **Híbrido**: kernel y ejecutivo en modo kernel, con una capa **HAL** que lo separa del hardware concreto |
| Piezas típicas | Planificador, gestor de memoria, VFS (capa de archivos), pila de red, subsistema de drivers, LSM (AppArmor/SELinux) | Gestor de objetos, gestor de memoria, gestor de E/S, monitor de seguridad, HAL |
| Código | Abierto, con ramas mantenidas por la comunidad y distribuciones (Ubuntu, Debian, RHEL) | Cerrado; se actualiza vía Windows Update |

#### Del botón de encendido a tu aplicación (arranque)

```
Encendido → Firmware UEFI → Bootloader (GRUB / Windows Boot Manager)
         → Kernel (inicializa CPU, memoria, drivers)
         → Linux: initramfs → init/systemd (PID 1)    Windows: servicios del sistema
         → Servicios: SSH, Docker, Nginx, tus aplicaciones
```

> **Por qué entender el kernel es clave aquí.** Un contenedor *no es* una máquina: es un conjunto de procesos que el kernel aísla con reglas. Si no sabes qué reglas son, no sabes qué tan fuerte es tu aislamiento. Lo ves en la siguiente sección.

### 1.3 Las primitivas de aislamiento que ofrece el kernel

Antes de los hipervisores, hay que entender lo que el propio kernel puede hacer. Estas son las piezas con las que se construyen los contenedores.

| Primitiva (Linux) | Qué aísla | Analogía |
|---|---|---|
| **Usuarios y permisos** | Quién puede leer/escribir/ejecutar | Llaves personales |
| **Namespaces** | Lo que un proceso **ve**: procesos (PID), red, sistema de archivos (mount), hostname, usuarios | Gafas que muestran solo parte del mundo |
| **cgroups** | Cuánto **consume** un grupo de procesos: CPU, memoria, E/S | Un medidor con tope |
| **Capabilities** | Divide el poder de "root" en permisos pequeños | Llaves parciales en vez de llave maestra |
| **seccomp** | Qué syscalls puede invocar un proceso | Lista de llamadas permitidas |
| **LSM: AppArmor / SELinux** | Políticas obligatorias de acceso | Reglamento que ni el dueño puede saltarse |

En Windows existen equivalentes: *job objects* (límites de recursos), *silos* (base de los contenedores de Windows), niveles de integridad y AppContainer.

**Idea central:** un contenedor es un proceso normal con *namespaces* + *cgroups* + restricciones (*capabilities*, *seccomp*, LSM). **Comparte el kernel del host.** Si hay un fallo en ese kernel, el aislamiento puede romperse.

### 1.4 Virtualización por hardware e hipervisores

Un **hipervisor** (o VMM, *Virtual Machine Monitor*) permite que varios sistemas operativos completos, cada uno con **su propio kernel**, compartan el mismo hardware.

**Cómo es posible:** las CPUs modernas incluyen extensiones de virtualización (Intel **VT-x**, AMD **AMD-V**). Añaden un modo en el que el hipervisor controla la máquina y los sistemas invitados corren en otro modo, creyendo que están en modo kernel. Además, la traducción de memoria anidada (Intel **EPT**, AMD **NPT**) y la **IOMMU** (aislamiento de dispositivos) hacen que la virtualización sea eficiente y segura.

#### Tipo 1 y Tipo 2

```
   TIPO 1 (bare-metal)                    TIPO 2 (hospedado)
┌────────┐ ┌────────┐ ┌────────┐       ┌────────┐ ┌────────┐
│  VM 1  │ │  VM 2  │ │  VM 3  │       │  VM 1  │ │  VM 2  │
│ kernel │ │ kernel │ │ kernel │       │ kernel │ │ kernel │
│ propio │ │ propio │ │ propio │       │ propio │ │ propio │
├────────┴─┴────────┴─┴────────┤       ├────────┴─┴────────┤
│        HIPERVISOR            │       │  Hipervisor (app) │
├──────────────────────────────┤       ├───────────────────┤
│         HARDWARE             │       │ SO anfitrión +    │
└──────────────────────────────┘       │ su kernel         │
                                       ├───────────────────┤
                                       │     HARDWARE      │
                                       └───────────────────┘
```

| | **Tipo 1 (bare-metal)** | **Tipo 2 (hospedado)** |
|---|---|---|
| Dónde corre | **Directamente sobre el hardware** | Como una **aplicación** dentro de un sistema operativo anfitrión |
| Ejemplos | VMware ESXi, Microsoft Hyper-V, Xen, **KVM** (integrado en Linux), Proxmox VE (Debian + KVM) | VirtualBox, VMware Workstation/Fusion, Parallels |
| Rendimiento | Alto: el camino hacia el hardware es corto | Algo menor, sobre todo en entrada/salida, porque cada operación pasa también por el SO anfitrión |
| Aislamiento | Fuerte (cada VM con su kernel, la superficie de ataque es el hipervisor) | Fuerte entre VMs, pero depende además de la seguridad del SO anfitrión |
| Uso típico | Servidores de producción, nubes, VPS | Laboratorio en tu portátil, pruebas, desarrollo |

**Dos precisiones importantes**

1. **Un hipervisor no "se conecta al kernel".** El tipo 1 *es* la capa que administra el hardware y reparte CPU y memoria entre las VMs; cada VM arranca **su propio kernel** encima. El tipo 2 delega en el kernel del SO anfitrión para planificar y entregar recursos, y por eso paga más sobrecarga.
2. **Casos que mezclan ambos mundos.**
   - **KVM** es un módulo del kernel de Linux que convierte a Linux en hipervisor; QEMU emula los dispositivos y los drivers *virtio* aceleran la entrada/salida. Por eso se clasifica de forma habitual como tipo 1, aunque técnicamente vive dentro de un kernel completo.
   - **Hyper-V** en Windows Server: al activarlo, el Windows "de siempre" pasa a ejecutarse como una VM privilegiada (la *partición raíz*) sobre el hipervisor.

> **Tu Hostinger VPS** usa **KVM**: tu "servidor" es una VM con su propio kernel, no un contenedor. Eso te da aislamiento fuerte respecto a otros clientes de Hostinger y te permite instalar Docker dentro.

### 1.5 VM, contenedor y microVM: la comparación que decide todo

```
        VM                       CONTENEDOR                 microVM / sandbox
┌─────┐ ┌─────┐            ┌─────┐ ┌─────┐ ┌─────┐         ┌─────┐ ┌─────┐
│App A│ │App B│            │App A│ │App B│ │App C│         │App A│ │App B│
│Libs │ │Libs │            │Libs │ │Libs │ │Libs │         │Libs │ │Libs │
│Kern.│ │Kern.│            └──┬──┘ └──┬──┘ └──┬──┘         │Kern.│ │Kern.│
└──┬──┘ └──┬──┘            ┌───┴───────┴───────┴──┐         │míni.│ │míni.│
┌──┴───────┴──┐            │  Motor (Docker) +    │         └──┬──┘ └──┬──┘
│ Hipervisor  │            │  KERNEL COMPARTIDO   │         ┌──┴───────┴──┐
└─────────────┘            └──────────────────────┘         │ Hipervisor  │
                                                            │ ligero      │
                                                            └─────────────┘
```

| | **Contenedor** (Docker/Podman) | **Sandbox** (gVisor) | **microVM** (Firecracker, Kata) | **VM completa** (KVM, Hyper-V) |
|---|---|---|---|---|
| Kernel | **Compartido con el host** | Un "kernel en espacio de usuario" intercepta las syscalls | Kernel invitado mínimo propio | Kernel invitado completo propio |
| Arranque | Milisegundos | Milisegundos | Fracciones de segundo | Segundos a minutos |
| Sobrecarga | Mínima | Baja a media | Baja | Media a alta |
| Aislamiento | Medio | Medio-alto | Alto | Alto |
| Úsalo para | Empaquetar tus apps y modelos | Ejecutar código **no confiable** con más seguridad | Multi-inquilino serio, ejecución de código de terceros | Separar clientes, sistemas operativos distintos |

### 1.6 Qué implica todo esto para modelamiento económico y ciencia de datos

- **Memoria:** un contenedor ve la RAM del host salvo que le pongas límite con cgroups. Si le pones un límite menor a lo que necesita tu modelo, el kernel lo mata. Dimensiona con datos reales.
- **Hilos y librerías numéricas:** un contenedor "ve" todos los núcleos del host. Librerías como OpenBLAS o MKL pueden lanzar demasiados hilos y competir entre sí. Controla `OMP_NUM_THREADS`, `OPENBLAS_NUM_THREADS` o `JULIA_NUM_THREADS`.
- **Reproducibilidad numérica:** un contenedor reproduce *software*, no *hardware*. En CPUs distintas, un mismo contenedor puede dar diferencias minúsculas en aritmética de punto flotante por instrucciones SIMD o librerías BLAS distintas. Documenta la CPU usada en resultados críticos y usa tolerancias en las pruebas, no igualdad exacta.
- **Entrada/salida:** montar carpetas del host en Windows (por ejemplo `/mnt/c` bajo WSL2) es mucho más lento que usar el sistema de archivos nativo de Linux. Para datos grandes, usa volúmenes dentro de Linux.
- **Seguridad:** como los contenedores comparten kernel, **parchear el kernel del host** es parte de tu seguridad, no solo las imágenes.

---

## 2. Matriz de decisión: qué nivel de aislamiento usar

### 2.1 La escalera de aislamiento

Cada nivel **añade** protección y costo. Se pueden combinar (por ejemplo, entorno de lenguaje + contenedor + VM).

| Nivel | Tecnología | Aísla | **No** aísla | Costo operativo |
|---|---|---|---|---|
| **L0** | Entornos del lenguaje: `renv`, `venv`/`uv`/`conda`, `Project.toml` de Julia | Paquetes y versiones del lenguaje | Librerías del sistema, el intérprete en sí, el SO | Muy bajo |
| **L1** | **Nix / Guix** | Todo el árbol de dependencias, incluidas librerías del sistema | El kernel; curva de aprendizaje alta | Medio |
| **L2** | **Docker / Podman** | Procesos, sistema de archivos, red, recursos | El kernel (compartido) | Bajo-medio |
| **L3** | **gVisor** (sandbox), **Kata**, **Firecracker** (microVM) | Interposición o separación del kernel | Nada serio, pero exige ajuste y tiene incompatibilidades | Alto |
| **L4** | **VM completa**: KVM/libvirt, Proxmox, Hyper-V, **VPS de Hostinger** | Kernel completo propio | Hardware físico compartido | Medio |
| **L5** | Servidor físico o cuenta cloud por cliente | Hardware y facturación | — | Alto |

### 2.2 Reglas prácticas para tu caso

| Situación | Recomendación |
|---|---|
| **Modelo económico que debe poder reproducirse en años** | L0 (lockfiles) **+** L2 (Docker). Opcional: L1 (Nix) si necesitas reproducibilidad más estricta |
| **Pipeline con Stata** | Stata corre en el host o en una VM (licencia); R/Python/Julia en contenedores; orquesta todo con Make o un script (ver 3.5) |
| **Aplicación web para un cliente normal** | L2 dentro de L4: Docker Compose sobre un VPS KVM o una VM |
| **Cliente con datos sensibles (financieros, de salud, personales)** | **Un VPS/VM dedicado por cliente** (L4). No compartas host entre clientes |
| **Varios clientes pequeños, bajo presupuesto** | Un servidor propio con **un stack de Compose por cliente**, redes separadas, límites de recursos, secretos separados (ver 4.3, modelo "puente") |
| **Ejecutar notebooks o código de terceros no confiable** | L3 (gVisor o microVM) como mínimo; mejor VM desechable |
| **Equipo que no domina Linux** | Priorizar L0 + L2 con Docker Compose; evita Kubernetes |

### 2.3 Cuándo NO usar contenedores

- **Datos y servicios con estado crítico sin experiencia en backups** (bases de datos): funcionan bien en contenedores, pero exigen volúmenes persistentes y copias probadas. Si no puedes garantizarlo, usa una base gestionada o instalada en la VM.
- **Software que requiere GUI de escritorio o drivers especiales** (algunas versiones de Stata en Windows con interfaz gráfica).
- **Cuando el único objetivo es "que funcione en mi máquina"**: bastan L0 y un buen README.

---

## 3. Receta A: modelamiento económico y ciencia de datos reproducibles

### 3.1 Principio rector

La reproducibilidad exige **código + datos + entorno** y no solo código y datos. Esto lo defienden de forma explícita publicaciones del Banco Mundial y la literatura de ciencias sociales computacionales sobre "compendios de investigación reproducibles".

### 3.2 Estructura de repositorio recomendada

```
proyecto-modelo/
├── README.md                 # Cómo reproducir en 3 comandos
├── LICENSE
├── Makefile                  # Orquesta el pipeline (o justfile / Snakemake / targets)
├── docker-compose.yml        # Servicios de análisis (si aplica)
├── .gitignore                # Excluye datos crudos, .env, caches
├── data/
│   ├── raw/                  # Inmutable. Nunca editar a mano. Fuera de Git si es pesado/sensible
│   ├── interim/
│   └── processed/
├── code/
│   ├── python/               # pyproject.toml + uv.lock
│   ├── r/                    # renv.lock
│   ├── julia/                # Project.toml + Manifest.toml
│   └── stata/                # .do files + ado/ vendorizado
├── docker/
│   ├── python.Dockerfile
│   ├── r.Dockerfile
│   └── julia.Dockerfile
├── results/                  # Salidas regenerables (tablas, figuras, logs)
├── docs/                     # Metodología, diccionario de datos, decisiones
└── .github/workflows/ci.yml
```

### 3.3 Capas de reproducibilidad (en orden)

1. **Control de versiones** (Git) para código y configuración.
2. **Lockfiles** por lenguaje (sección 3.4).
3. **Contenedor** con versión del SO y librerías del sistema fijadas.
4. **Datos versionados o verificables**: guarda hashes (SHA-256) de los datos crudos; para datos grandes usa DVC o un almacenamiento de objetos con versiones.
5. **Semillas aleatorias explícitas** en cada script que use aleatoriedad.
6. **Pipeline automatizado** de un solo comando (`make all`).
7. **CI** que reconstruye y corre el pipeline, al menos con una muestra de datos.
8. **Registro del entorno de ejecución**: versiones, CPU, fecha, hash del commit, guardado junto a cada resultado.

### 3.4 Gestión de entornos por lenguaje

#### Python

- Herramientas: `uv` (rápido, con lockfile), `pip-tools`, Poetry o conda/mamba si dependes de librerías científicas binarias.
- Archivos que se versionan: `pyproject.toml` y el lockfile (`uv.lock`, `poetry.lock` o `requirements.txt` con hashes).
- Fija la versión de Python (`.python-version` o en la imagen base).

```dockerfile
# docker/python.Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    OMP_NUM_THREADS=2 OPENBLAS_NUM_THREADS=2
RUN pip install --no-cache-dir uv==<VERSION_FIJADA_DE_UV>
WORKDIR /app
COPY code/python/pyproject.toml code/python/uv.lock ./
RUN uv sync --frozen --no-dev
ENV PATH="/app/.venv/bin:$PATH"
COPY code/python/src ./src
RUN useradd -r -u 10001 runner
USER runner
CMD ["python", "-m", "src.main"]
```

#### R

- Herramientas: **`renv`** para el lockfile de paquetes; la imagen `rocker/r-ver` fija la versión de R y apunta a una instantánea fechada de CRAN, lo que ayuda a la reproducibilidad.
- Se versiona `renv.lock`.

```dockerfile
# docker/r.Dockerfile
FROM rocker/r-ver:4.4.2
RUN apt-get update && apt-get install -y --no-install-recommends \
      libcurl4-openssl-dev libssl-dev libxml2-dev \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /project
COPY code/r/renv.lock renv.lock
RUN R -q -e "install.packages('renv'); renv::restore(lockfile='renv.lock', prompt=FALSE)"
COPY code/r/ .
CMD ["Rscript", "run_all.R"]
```

#### Julia

- Herramientas: `Project.toml` y **`Manifest.toml`** (este último es el lockfile; **debe versionarse** en proyectos de aplicación).
- Para reducir el tiempo de arranque en producción: `PackageCompiler.jl` (sysimage).

```dockerfile
# docker/julia.Dockerfile
FROM julia:1.10
WORKDIR /project
COPY code/julia/Project.toml code/julia/Manifest.toml ./
RUN julia --project=. -e 'using Pkg; Pkg.instantiate(); Pkg.precompile()'
COPY code/julia/ .
ENV JULIA_NUM_THREADS=2
CMD ["julia", "--project=.", "src/main.jl"]
```

#### Stata

Stata es software **propietario con licencia**, y no tiene un gestor de paquetes con lockfile. Qué hacer:

- Pon `version <N>` al inicio de cada `.do` para fijar el comportamiento del lenguaje.
- **Vendoriza los `.ado`** que uses (cópialos al repo en `code/stata/ado/`) y apunta Stata a esa carpeta: `adopath ++ "ado"`. No dependas de `ssc install` en caliente, porque instala la última versión del paquete.
- Registra la versión exacta de Stata y de cada ado en el log de resultados.
- **Licencia y contenedores:** revisa los términos de tu licencia antes de meter Stata en una imagen. Nunca publiques una imagen que incluya Stata ni subas la licencia a Git. Si decides contenerizarlo, construye la imagen tú mismo con tu instalador y monta la licencia en tiempo de ejecución.
- **Opción más simple y robusta:** ejecuta Stata **en lote** en el host o en una VM dedicada (`stata-mp -b do archivo.do` en Linux) y deja a Python/R/Julia en contenedores.

#### Nix (opcional, avanzado)

Nix describe **todas** las dependencias, incluidas las del sistema, de forma declarativa. Es más estricto que Docker en reproducibilidad de *construcción*, pero con curva de aprendizaje alta. Úsalo si necesitas reconstruir entornos idénticos dentro de varios años. Existen herramientas orientadas a ciencia de datos (por ejemplo, `rix` para R).

### 3.5 Pipeline multi-lenguaje (con Stata en el host)

```makefile
# Makefile
.PHONY: all clean data stata python r julia

all: data stata python r julia

data:
	docker compose run --rm python python -m src.etl

stata: data
	cd code/stata && stata-mp -b do 01_modelo.do

python: stata
	docker compose run --rm python python -m src.analisis

r: python
	docker compose run --rm r Rscript 02_tablas.R

julia: r
	docker compose run --rm julia julia --project=. src/simulacion.jl

clean:
	rm -rf results/* data/interim/* data/processed/*
```

```yaml
# docker-compose.yml (servicios de análisis, no de producción)
services:
  python:
    build: { context: ., dockerfile: docker/python.Dockerfile }
    volumes: [ "./data:/data", "./results:/results" ]
    mem_limit: 8g
    cpus: 4
  r:
    build: { context: ., dockerfile: docker/r.Dockerfile }
    volumes: [ "./data:/data", "./results:/results" ]
    mem_limit: 8g
  julia:
    build: { context: ., dockerfile: docker/julia.Dockerfile }
    volumes: [ "./data:/data", "./results:/results" ]
    mem_limit: 8g
```

> **Nota de rendimiento:** con volúmenes montados sobre carpetas del host, la entrada/salida puede ser lenta en Windows. En Linux es nativa.

### 3.6 Semillas y registro del entorno

```python
# Python
import random, numpy as np
SEED = 20261026
random.seed(SEED); np.random.seed(SEED)
```
```r
# R
set.seed(20261026)
```
```julia
# Julia
using Random; Random.seed!(20261026)
```
```stata
* Stata
set seed 20261026
version 18
```

Guarda junto a cada resultado un `run_info.json` con: hash del commit, fecha, versiones de lenguajes y librerías, modelo de CPU, número de hilos y semillas.

### 3.7 Datos sensibles y de clientes

- **Nunca** metas datos dentro de la imagen del contenedor. Móntalos en tiempo de ejecución.
- Cifra el disco del servidor o el volumen de datos si contiene información personal o confidencial.
- Excluye los datos del repositorio (`.gitignore`) y documenta su origen y procedimiento de obtención.
- Registra quién accede y cuándo cuando el cliente lo exija.

---

## 4. Receta B: software a la medida (frontend + backend)

### 4.1 Arquitectura de referencia

```
Internet
   │  HTTPS (443)
   ▼
┌──────────────┐   red "frontend"   ┌────────────────┐
│ Proxy inverso│───────────────────▶│  Backend / API │
│ Caddy/Nginx  │                    │ FastAPI, Django│
│ + TLS        │──▶ archivos        │ Plumber, Oxygen│
│              │    estáticos del   └───────┬────────┘
└──────────────┘    frontend                │ red "backend" (interna, sin Internet)
                    (React/Vue build)       ├──────────────▶ PostgreSQL
                                            └──────────────▶ Worker de modelos
                                                             (Python / R / Julia)
```

**Piezas y opciones**

| Capa | Opciones habituales | Nota |
|---|---|---|
| **Frontend** | React, Vue, Svelte (compilado a estáticos con Vite), o Streamlit/Dash/Shiny para paneles | Sirve los estáticos desde el proxy; no necesitas un contenedor Node en producción |
| **Backend/API** | Python (FastAPI, Django), R (Plumber), Julia (Oxygen.jl, Genie.jl), Node.js | Para modelos económicos pesados, no los ejecutes dentro de la API: usa un worker separado |
| **Base de datos** | PostgreSQL | Con volumen persistente y backups probados |
| **Cola/Workers** | Redis + RQ/Celery (Python), o simplemente una tabla de tareas en PostgreSQL | Aísla los cálculos largos para que no tumben la API |
| **Proxy inverso** | Caddy (TLS automático), Nginx, Traefik | Un único punto de entrada |

### 4.2 Plantilla de Docker Compose con endurecimiento básico

```yaml
# docker-compose.yml (producción por cliente)
name: cliente-acme

services:
  proxy:
    image: caddy:2
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./frontend/dist:/srv/frontend:ro
      - caddy_data:/data
    networks: [frontend]

  api:
    build: ./backend
    restart: unless-stopped
    env_file: .env                 # NO versionar. Solo variables no secretas
    secrets: [db_password]
    user: "10001:10001"            # no root
    read_only: true                # sistema de archivos de solo lectura
    tmpfs: [/tmp]
    cap_drop: [ALL]                # sin capabilities extra
    security_opt: ["no-new-privileges:true"]
    mem_limit: 1g
    cpus: 1.0
    depends_on:
      db: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks: [frontend, backend]

  worker:
    build: ./modelos
    restart: unless-stopped
    user: "10001:10001"
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 4g
    cpus: 2.0
    networks: [backend]            # sin salida a Internet

  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    networks: [backend]

networks:
  frontend: {}
  backend:
    internal: true                 # sin acceso a Internet

volumes:
  pgdata:
  caddy_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt   # fuera de Git, permisos 600
```

```
# Caddyfile
cliente.ejemplo.com {
    encode gzip
    handle /api/* { reverse_proxy api:8000 }
    handle {
        root * /srv/frontend
        try_files {path} /index.html
        file_server
    }
}
```

> Esta plantilla incluye usuario sin privilegios, sistema de archivos de solo lectura, capabilities eliminadas, límites de recursos y una red interna para base de datos y workers. Con eso reduces mucho el daño posible de una vulnerabilidad en una de las piezas.

### 4.3 Aislamiento entre clientes: tres modelos

| Modelo | Cómo es | Aislamiento | Costo | Cuándo |
|---|---|---|---|---|
| **Silo** | **Un VPS/VM por cliente** | Alto (kernel propio) | Mayor (un servidor por cliente) | Datos sensibles, clientes grandes, exigencias contractuales |
| **Puente** | Un servidor, **un stack de Compose por cliente** (redes, volúmenes, secretos y usuario propios) | Medio (kernel compartido) | Medio | Varios clientes pequeños con datos no críticos |
| **Pool** | Una sola aplicación multi-inquilino (todos los clientes comparten app y base) | Bajo, depende totalmente de tu código | Menor | Productos SaaS propios, con ingeniería de seguridad madura |

**Recomendación para tu caso:** empieza con **Silo** para clientes con información sensible y **Puente** para el resto. Evita **Pool** hasta tener pruebas automatizadas de aislamiento de datos y experiencia con ese patrón.

**Reglas del modelo Puente** (un host, varios clientes):

- Un directorio, un proyecto de Compose (`name:`), una red y un conjunto de volúmenes por cliente.
- Contraseñas y secretos distintos por cliente; nunca compartas una base de datos.
- Límites de CPU y memoria por cliente para que uno no ahogue a los demás.
- Un único proxy inverso en el host que enruta por dominio, con cada stack en su propia red.
- Backups separados por cliente.

### 4.4 Ambientes: desarrollo, staging, producción

| Ambiente | Propósito | Datos | Cómo se despliega |
|---|---|---|---|
| **Dev** | Programar | Sintéticos o anonimizados | Compose local |
| **Staging** | Probar como en producción | Copia anonimizada | Automático desde la rama de integración |
| **Prod** | Cliente real | Reales | Manual aprobado o automático desde `main`/tags |

- Misma imagen en staging y producción; solo cambia la configuración.
- Configuración por variables de entorno (principio de la *Twelve-Factor App*); nada de credenciales en el código.

### 4.5 Gestión de secretos

| Nivel | Técnica |
|---|---|
| Mínimo aceptable | Archivos de secretos con permisos 600 fuera de Git; Docker secrets en Compose |
| Mejor | **SOPS + age** para cifrar secretos y poder versionarlos cifrados |
| Avanzado | Gestor de secretos (Vault, o el de tu proveedor cloud) |

Rota las credenciales cuando alguien deje el proyecto y tras cualquier sospecha de fuga.

---

## 5. Recetas por plataforma

### 5.1 Linux Server (local)

**Base recomendada:** Ubuntu LTS o Debian estable. Rocky/Alma Linux si el cliente exige familia RHEL.

1. **Acceso:** SSH solo con llaves, sin contraseña ni root remoto (`PasswordAuthentication no`, `PermitRootLogin no`).
2. **Firewall:** UFW o nftables; abre solo 22 (idealmente restringido por IP), 80 y 443.
3. **Actualizaciones:** parches automáticos de seguridad (`unattended-upgrades`) y ventana de reinicio para actualizar el kernel.
4. **Fail2ban** o equivalente contra fuerza bruta.
5. **Docker Engine** (o **Podman** si prefieres contenedores sin demonio y sin root), con Compose.
6. **Servicios al arranque:** `restart: unless-stopped` en Compose o unidades `systemd`.
7. **Si quieres VMs en el mismo servidor:** **KVM + libvirt**, o **Proxmox VE** (basado en Debian + KVM) con interfaz web. Es la manera de implementar el modelo Silo en tu propio hardware.
8. **Backups:** regla **3-2-1** (3 copias, 2 medios, 1 fuera del sitio), con restauración probada periódicamente.
9. **Monitoreo:** espacio en disco, RAM, CPU, salud de contenedores (ver sección 6.6).

### 5.2 Windows Server (local)

- **Hyper-V** es el hipervisor tipo 1 de Windows Server. Es la base para el modelo Silo en tu propio hardware.
- **Lo más robusto para aplicaciones Linux:** crea una **VM Linux** en Hyper-V y ejecuta Docker dentro de ella. Así tratas esa VM como un servidor Linux normal (sección 5.1).
- **Contenedores de Windows** (aislamiento por proceso o por Hyper-V) solo si tu aplicación **requiere** Windows (por ejemplo, .NET Framework clásico). Las imágenes son grandes y el ecosistema es más reducido.
- **Linux directamente en Windows Server (WSL2, Docker Desktop):** pensado para escritorio de desarrollo. No lo uses como base de producción sin verificar soporte y licencias vigentes de Microsoft y Docker para tu caso.
- **IIS** sirve bien para aplicaciones ASP.NET o como proxy inverso (con ARR) delante de otros servicios.
- **Stata en Windows:** puede correr en la propia VM Windows o servidor y ser invocado por lotes; deja los demás lenguajes en contenedores Linux.
- Endurece: Windows Update gestionado, firewall de Windows con reglas mínimas, RDP restringido (VPN o IP), cuentas sin privilegios para los servicios, y copias de seguridad de las VMs.

### 5.3 Hostinger

| Producto | ¿Sirve para esta guía? | Notas |
|---|---|---|
| **VPS (KVM)** | **Sí** | Virtualización KVM con acceso root y plantillas de Docker. Puedes montar la plantilla de la sección 4.2 casi tal cual |
| **Hosting compartido** | Solo para sitios estáticos o PHP | Sin control del sistema operativo; no sirve para tus modelos ni para stacks propios |

Pasos para un VPS:

1. Crea el VPS (elige Ubuntu LTS; hay plantillas con Docker si quieres ahorrar tiempo).
2. Crea un usuario sin privilegios, desactiva el acceso root por SSH y usa llaves.
3. Activa el firewall del panel y/o UFW (22, 80, 443).
4. Instala Docker y Compose (o usa la plantilla), despliega con la receta 4.2.
5. Apunta el dominio al VPS y deja que Caddy emita el certificado TLS.
6. Configura backups: usa los de Hostinger **y** una copia externa propia; verifica que sabes restaurar.
7. **Un VPS por cliente sensible** (modelo Silo). Revisa los límites de CPU, RAM y disco de cada plan frente a tus modelos.

---

## 6. CI/CD y estrategias de despliegue

### 6.1 Flujo recomendado

```
Commit → Pull Request → CI (build + tests + escaneo) → merge a main
      → imagen versionada (tag + digest) en un registro (GHCR/Docker Hub)
      → despliegue a staging → verificación → despliegue a producción
      → monitoreo → (si falla) rollback a la imagen anterior
```

### 6.2 Ejemplo mínimo con GitHub Actions

```yaml
# .github/workflows/ci.yml
name: ci
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # fija por versión o por SHA
      - name: Construir imagen
        run: docker build -t app:test ./backend
      - name: Pruebas
        run: docker run --rm app:test pytest -q
      - name: Escaneo de vulnerabilidades
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:<VERSION> image --exit-code 1 --severity HIGH,CRITICAL app:test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Desplegar por SSH
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          HOST: ${{ secrets.DEPLOY_HOST }}
        run: |
          install -m 600 /dev/null key && echo "$SSH_KEY" > key
          ssh -i key -o StrictHostKeyChecking=accept-new deploy@"$HOST" \
            "cd /opt/cliente-acme && git pull --ff-only && docker compose pull && docker compose up -d"
```

> Mejora siguiente: construir la imagen **en CI**, subirla a un registro con una etiqueta inmutable (hash del commit) y que el servidor solo haga `pull`. Así lo que probaste es exactamente lo que corre.

### 6.3 Estrategias de despliegue

| Estrategia | Cómo funciona | Ventaja | Riesgo |
|---|---|---|---|
| **Recreate** | Apaga la versión vieja y arranca la nueva | La más simple | Hay corte breve |
| **Rolling** | Reemplaza instancias poco a poco | Sin corte si hay varias réplicas | Conviven dos versiones un rato |
| **Blue-Green** | Dos entornos completos; se cambia el tráfico de uno a otro | Rollback instantáneo | Duplica recursos |
| **Canary** | Una fracción pequeña del tráfico va a la nueva versión | Detecta fallos con poco impacto | Requiere métricas y enrutado |

Para tu escala, empieza con **Recreate** o un **Blue-Green sencillo** (dos stacks de Compose y el proxy que cambia de destino).

### 6.4 Migraciones de base de datos

- Versiona las migraciones (Alembic, Django migrations, Flyway, etc.).
- Aplica el patrón **expandir y contraer**: primero añade columnas o tablas compatibles hacia atrás, despliega el código, y solo después elimina lo antiguo. Así el rollback del código no se rompe.
- Haz **backup antes de cada migración** en producción.

### 6.5 Rollback

- Conserva las últimas N imágenes etiquetadas.
- El rollback debe ser **un comando** y estar probado: `docker compose up -d` con la etiqueta anterior.
- Si una migración no es reversible, el plan de rollback es restaurar el backup. Pruébalo.

### 6.6 Observabilidad y operación

| Qué | Herramientas simples |
|---|---|
| **Logs** | `docker logs`, journald; centralizar con Loki o similares si crece |
| **Métricas** | Prometheus + Grafana, o `node_exporter` + `cAdvisor` |
| **Disponibilidad** | Uptime Kuma o un servicio externo de *uptime* |
| **Alertas** | Disco > 80 %, memoria alta, contenedores reiniciándose, certificados por vencer, backups fallidos |
| **Salud** | `healthcheck` en cada servicio y endpoint `/health` en la API |

### 6.7 Backups y recuperación ante desastres

- Regla **3-2-1** y copias **cifradas**.
- Define **RPO** (cuántos datos aceptas perder) y **RTO** (cuánto tardas en volver). Pídeselos al cliente.
- Una copia que nunca has restaurado **no cuenta como copia**: programa pruebas de restauración.
- Para PostgreSQL: `pg_dump` periódico más archivado de WAL si necesitas recuperación a un punto en el tiempo.

---

## 7. Seguridad y cadena de suministro

### 7.1 Modelo de amenazas en una línea

Piensa en **quién podría hacer qué**: un atacante externo contra tu API, una dependencia comprometida, un cliente que accede a datos de otro, un empleado con demasiados permisos, una credencial filtrada.

### 7.2 Controles esenciales

| Área | Control |
|---|---|
| **Imágenes** | Bases mínimas y oficiales (`-slim`, `distroless`), fijadas por versión (idealmente por *digest*), reconstruidas con regularidad |
| **Dependencias** | Lockfiles, escaneo (Trivy, `pip-audit`, `npm audit`), actualizaciones controladas (Dependabot/Renovate) |
| **Privilegios** | Contenedores sin root, `cap_drop: ALL`, `no-new-privileges`, sistema de archivos de solo lectura |
| **Red** | Redes internas para BD y workers, puertos mínimos publicados, TLS en todo lo público |
| **Secretos** | Fuera del código y de las imágenes (sección 4.5) |
| **Host** | Parches de kernel y SO (en contenedores, el kernel del host es tu superficie de ataque) |
| **Acceso** | SSH con llaves, MFA en GitHub, Hostinger y registros, principio de mínimo privilegio |
| **Cadena de suministro** | Fija las versiones de las *actions* de CI, genera un **SBOM** (lista de componentes) y firma las imágenes si el cliente lo exige |
| **Registros** | Logs de acceso y de cambios; no registres datos sensibles |

### 7.3 Cuándo subir de contenedor estándar a sandbox o microVM

Considera **gVisor**, **Kata Containers** o **Firecracker** cuando:

- Ejecutes **código que no controlas** (notebooks de usuarios, plugins, scripts subidos).
- Alojes **varios clientes en el mismo host** con datos que no deben mezclarse bajo ningún escenario.
- Un contrato o norma exija separación más fuerte que la de un contenedor estándar.

Si no es tu caso, un contenedor endurecido dentro de una VM por cliente suele ser suficiente y más sencillo de operar.

### 7.4 Errores frecuentes

- Montar `/var/run/docker.sock` dentro de un contenedor (equivale a dar root en el host).
- Usar `--privileged` "para que funcione".
- Publicar el puerto de la base de datos a Internet.
- Usar la etiqueta `latest` en producción.
- Guardar `.env` o claves en Git.
- Compartir una misma base de datos entre clientes.
- No probar nunca la restauración de backups.

---

## 8. Checklists

### 8.1 Antes de desplegar (por cliente)

- [ ] Modelo de aislamiento decidido y documentado (Silo / Puente / Pool).
- [ ] Imágenes con versiones fijadas y escaneadas sin hallazgos críticos.
- [ ] Contenedores sin root, con límites de recursos y `cap_drop: ALL`.
- [ ] BD y workers en red interna; solo el proxy publicado.
- [ ] Secretos fuera de Git; credenciales únicas por cliente.
- [ ] TLS activo y renovación automática verificada.
- [ ] Backups automáticos configurados **y restauración probada**.
- [ ] `healthcheck` y monitoreo con alertas activos.
- [ ] Plan de rollback escrito y ensayado.
- [ ] SSH solo con llaves, firewall mínimo, parches automáticos de seguridad.

### 8.2 Reproducibilidad de un modelo económico

- [ ] Lockfiles versionados (`uv.lock`/`renv.lock`/`Manifest.toml`) y `version N` en los `.do`.
- [ ] `.ado` de Stata vendorizados.
- [ ] Imagen base fijada (y, si es crítico, por *digest*).
- [ ] Semillas explícitas en todos los scripts.
- [ ] Hashes de los datos crudos registrados.
- [ ] `make all` reproduce resultados desde cero.
- [ ] `run_info.json` guardado con cada ejecución.
- [ ] Tolerancias numéricas definidas en las pruebas.
- [ ] README con los 3 comandos para reproducir.
- [ ] Una máquina limpia reprodujo los resultados al menos una vez.

### 8.3 Entrega a un cliente

- [ ] Documentación de arquitectura, accesos y procedimientos (despliegue, rollback, restauración).
- [ ] Inventario de secretos y quién los administra.
- [ ] Acuerdo explícito sobre actualizaciones, parches y mantenimiento.
- [ ] Transferencia de repositorios, registros y dominios, según el contrato.
- [ ] Datos de prueba eliminados y credenciales de desarrollo revocadas.

---

## 9. Bibliografía anotada

> **Cómo leer esta lista.** Las entradas marcadas con ✅ las **verifiqué en fuentes en línea** al preparar esta guía. Las marcadas con 📚 provienen de mi conocimiento general y **conviene confirmar edición y disponibilidad** antes de comprarlas.

### 9.1 Universidades y cursos

| | Recurso | Para qué |
|---|---|---|
| ✅ | **MIT 6.566 (Seguridad de Sistemas), lección sobre aislamiento** (apuntes en `css.csail.mit.edu`) | Compara procesos, contenedores (namespaces + cgroups), gVisor, Firecracker y VMs. Base conceptual de las secciones 1 y 2 |
| ✅ | **MIT, *The Missing Semester of Your CS Education*** (`missing.csail.mit.edu`) | Shell, scripting, Git, depuración, CI. Prerrequisito práctico |
| ✅ | **University of Denver, ACS 3220 Docker & DevOps Deployments** (repositorio público) | Modelo de currículo práctico: contenedores, orquestación y despliegue |
| ✅ | **University of Oregon, EC607 *Data Science for Economists*, lección de Docker** | Contenedores explicados para economistas, con ejemplos en R |

### 9.2 Aislamiento, contenedores y kernel

| | Recurso | Para qué |
|---|---|---|
| ✅ | **Liz Rice, *Container Security*** (O'Reilly, 2020) | La referencia más citada: namespaces, cgroups, capabilities, seccomp, AppArmor/SELinux, gVisor, Kata, Firecracker y cómo se rompe el aislamiento |
| 📚 | **Michael Kerrisk, *The Linux Programming Interface*** | Referencia profunda de la interfaz de Linux (procesos, memoria, señales, sockets) |
| 📚 | **Tanenbaum y Bos, *Modern Operating Systems*** / **Silberschatz et al., *Operating System Concepts*** | Fundamentos de SO que se usan en cursos universitarios |
| 📚 | **Nigel Poulton, *Docker Deep Dive*** | Docker práctico, de cero a nivel intermedio |
| 📚 | **Brendan Burns et al., *Kubernetes: Up & Running*** | Cuando llegues a orquestación |
| 📚 | **Documentación oficial**: Docker, Podman, KVM/libvirt, Hyper-V, Proxmox | Fuente primaria y actualizada |

### 9.3 Entrega, operación y arquitectura

| | Recurso | Para qué |
|---|---|---|
| 📚 | **Humble y Farley, *Continuous Delivery*** | La base de pipelines, ambientes y despliegue |
| 📚 | **Forsgren, Humble y Kim, *Accelerate*** | Métricas y prácticas que predicen buen desempeño de entrega |
| 📚 | **Google, *Site Reliability Engineering*** (gratis en línea) | Confiabilidad, monitoreo, respuesta a incidentes |
| 📚 | **Michael Nygard, *Release It!*** | Patrones de estabilidad y fallos en producción |
| 📚 | **Adam Wiggins, *The Twelve-Factor App*** (gratis en línea) | Configuración, procesos, logs y despliegue de apps modernas |
| 📚 | **Skelton y Pais, *Team Topologies*** | Organización de equipos y responsabilidades |
| 📚 | **Martin Kleppmann, *Designing Data-Intensive Applications*** | Bases de datos, consistencia, sistemas distribuidos |

### 9.4 Reproducibilidad en economía y ciencia de datos

| | Recurso | Para qué |
|---|---|---|
| ✅ | **Banco Mundial, *Save your environment: the (often) overlooked problem of research reproducibility in economics*** (blog *Let's Talk Development*) | Argumenta que se requiere "código, datos y entorno" y discute el papel de los contenedores y sus límites |
| ✅ | ***Computational Reproducibility in Computational Social Science*** (arXiv 2307.01918) | Reproducibilidad proactiva, compendios de investigación, Docker/Apptainer |
| ✅ | **SSRC, *Social Science Replication in the Age of Software*** | Contexto del problema; cita el estudio de la Fed (Chang y Li, 2015) sobre replicación en economía |
| ✅ | **Bruno Rodrigues, *Rix: reproducible data science environments with Nix*** (R Consortium) | Nix aplicado a ciencia de datos |
| 📚 | **The Turing Way** (libro abierto en línea) | Guía comunitaria de investigación reproducible |
| 📚 | **Software Carpentry / Data Carpentry** | Habilidades computacionales básicas para investigación |
| 📚 | **QuantEcon** (lecturas y código en Python/Julia) | Economía computacional con buenas prácticas de código |
| 📚 | **Social Science Data Editors** (plantilla de README para paquetes de replicación) y políticas de datos/código de revistas como las de la AEA | Estándares que piden las revistas de economía |

### 9.5 Infraestructura del proveedor

| | Recurso | Para qué |
|---|---|---|
| ✅ | **Hostinger VPS**: documentación y páginas de producto | Confirman virtualización KVM, acceso root y soporte para Docker en los planes VPS |

---

## 10. Ruta de aprendizaje

| Fase | Objetivo | Entregable |
|---|---|---|
| **1** (semanas 1–2) | Línea de comandos, Git, Linux básico (*Missing Semester*) | Repositorio propio con scripts y dotfiles |
| **2** (semanas 3–4) | Kernel y aislamiento: namespaces, cgroups, VMs vs contenedores (MIT 6.566 + Liz Rice, caps. 1–4) | Notas propias y un contenedor "a mano" con `unshare` |
| **3** (semanas 5–6) | Docker y Compose; empaquetar un modelo real en R/Python/Julia | Imagen reproducible de uno de tus modelos |
| **4** (semanas 7–8) | Pipeline completo con Make, semillas y `run_info.json` | `make all` reproduce resultados en una máquina limpia |
| **5** (semanas 9–10) | Backend + frontend + BD con la plantilla 4.2 | Stack de ejemplo desplegado en un VPS de pruebas |
| **6** (semanas 11–12) | CI/CD, backups, monitoreo, endurecimiento | Pipeline con despliegue y rollback probado |
| **Después** | KVM/Proxmox o Hyper-V, gVisor/Firecracker, Nix, Kubernetes si el negocio lo pide | — |

---

## 11. Glosario

| Término | Definición breve |
|---|---|
| **Kernel** | Núcleo del SO que gestiona CPU, memoria, archivos, red y hardware en modo privilegiado |
| **Syscall** | Llamada de un programa al kernel para pedir un servicio |
| **MMU** | Componente de la CPU que traduce direcciones virtuales y aplica permisos de memoria |
| **Namespace** | Mecanismo de Linux que limita lo que un proceso ve |
| **cgroup** | Mecanismo de Linux que limita lo que un grupo de procesos consume |
| **Contenedor** | Proceso(s) aislados con namespaces y cgroups que comparten el kernel del host |
| **Imagen** | Plantilla inmutable a partir de la cual se crea un contenedor |
| **Hipervisor** | Software que permite ejecutar varias máquinas virtuales sobre el mismo hardware |
| **Tipo 1 / Tipo 2** | Hipervisor sobre el hardware / hipervisor como aplicación de un SO anfitrión |
| **KVM** | Módulo de Linux que lo convierte en hipervisor |
| **VT-x / AMD-V** | Extensiones de la CPU que aceleran la virtualización |
| **microVM** | VM muy ligera con kernel mínimo (Firecracker, Kata) |
| **gVisor** | Sandbox que reimplementa parte del kernel en espacio de usuario |
| **Lockfile** | Archivo que fija las versiones exactas de todas las dependencias |
| **Reproducibilidad** | Obtener los mismos resultados con el mismo código, datos y entorno |
| **Multi-inquilino (multi-tenant)** | Una misma aplicación o infraestructura sirve a varios clientes |
| **RPO / RTO** | Pérdida de datos tolerable / tiempo máximo de recuperación |
| **SBOM** | Lista de los componentes de software de una aplicación o imagen |
| **CI/CD** | Integración y entrega/despliegue continuos, automatizados |

---

*Esta guía es un documento vivo: revisa versiones, licencias (especialmente Stata, Docker Desktop y Windows Server) y las políticas del proveedor antes de cada despliegue importante.*

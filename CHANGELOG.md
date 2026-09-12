# Changelog

Todos los cambios relevantes de este proyecto se documentan en este archivo.

El formato sigue [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y el
versionado es [semántico](https://semver.org/lang/es/).

---

## [No publicado]

### Corregido

- **Deteccion de capacidades subestimaba lo instalado en usuarios sin
  privilegios.** `aa-status`, `dmidecode`, `ufw`, `nft`, `iptables` y
  `getenforce` viven por convención en `/usr/sbin`, directorio ausente del
  `PATH` de un usuario sin privilegios en Debian/Ubuntu (confirmado en vivo:
  `/usr/local/bin:/usr/bin:/bin:/usr/games`). La detección usaba `command -v`
  a secas, así que sin root el script reportaba "No se detectó SELinux ni
  AppArmor instalado" cuando AppArmor sí estaba instalado — una afirmación
  falsa sobre datos nunca buscados donde correspondía. Ya existía este mismo
  parche, pero solo para `sshd`; se generalizó a una función
  (`herramienta_disponible`) que también revisa `/usr/sbin`, `/sbin` y
  `/usr/local/sbin`, aplicada a los siete binarios administrativos que el
  script detecta. Encontrado probando por primera vez contra una VM real de
  Debian 12.
- **1.14 Firewall nunca leía reglas de `firewalld`.** La función solo probaba
  `ufw`, `nft` e `iptables`; en un servidor RHEL/CentOS/Fedora real, con
  firewalld activo y corriendo como root, el reporte decía "no se pudieron
  leer sin privilegios de root" — una afirmación falsa sobre la causa,
  detectada al probar por primera vez contra una VM real de CentOS Stream 9.
  Se agregó la rama `firewall-cmd --list-all-zones` y se distingue
  explícitamente el motivo real por el que faltan reglas (sin privilegios /
  sin herramienta instalada / herramienta detectada pero sin salida) en vez
  de un mensaje fijo que no correspondía a los tres casos.

- **1.2.3 Actualizaciones Pendientes con `zypper` reportaba falsos negativos.**
  Dos bugs encontrados probando en vivo contra openSUSE Leap real: (1) sin
  refresco previo de repositorios, a diferencia de la rama `apt`, la primera
  consulta en un equipo recién aprovisionado podía necesitar construir el
  cache de un repositorio y el `timeout` la mataba antes de imprimir nada,
  reportando "sin actualizaciones pendientes" cuando había cuatro; (2) el
  `awk` no excluía la fila de encabezado de la tabla de `zypper
  list-updates`, que se colaba como si fuera un paquete llamado "Name".
  Corregido con un `zypper refresh` explícito (mismo patrón que `apt-get
  update`) y excluyendo el encabezado por contenido exacto.

### Validado

- **Familia RHEL probada en una VM real** (CentOS Stream 9, imagen cloud
  oficial sobre KVM/libvirt), no solo contra documentación: las ramas de
  `dnf`, `update-crypto-policies` y SELinux (vía `getenforce`) funcionan
  como estaba escrito. Detalle en el README, sección "Compatibilidad
  probada en RHEL".
- **Familia SUSE probada en una VM real** (openSUSE Leap 15.6, imagen cloud
  oficial sobre KVM/libvirt), incluido el piso `bash 4.0+` del proyecto
  contra bash 4.4.23 real (la versión más antigua de las tres distros
  probadas): `rpm -qa`, `iptables -L -n` y AppArmor vía `aa-status`
  funcionan como estaba escrito. Detalle en el README, sección
  "Compatibilidad probada en SUSE".
- **Debian probado en una VM real** (Debian 12 bookworm, imagen cloud
  oficial sobre KVM/libvirt): `apt`/`dpkg-query`, actualizaciones pendientes
  sin falsos negativos y AppArmor vía `aa-status` funcionan como estaba
  escrito. Detalle en el README, sección "Compatibilidad probada en
  Debian".

---

## [0.1.0-slice] — 2026-09-12

Primera versión funcional. Es una **prueba de concepto vertical**: corre de punta a
punta y genera reportes completos contra equipos reales, pero no cubre todavía el
alcance planificado ni está validada fuera de Debian/Ubuntu.

El objetivo de esta versión era demostrar que la arquitectura funciona de verdad
—registro de secciones y hallazgos en TSV, render HTML reutilizando el lenguaje
visual de la versión Windows— antes de invertir en el resto de las secciones.

### Agregado

- Script único `cat-serverfullreport` (bash 4.0+, ~4.700 líneas, sin dependencias
  fuera del sistema base).
- **41 secciones de recolección** cubriendo sistema, almacenamiento, red, seguridad,
  TLS, tareas programadas, salud, plataforma y agentes de protección.
- **Motor de hallazgos con 14 tópicos**, que clasifica en `CRIT` / `WARN` / `INFO` /
  `OK` con evidencia concreta, recomendación y referencia a controles CIS e ISO 27001.
- Puntaje de salud (parte de 100; descuenta 12 por `CRIT` y 4 por `WARN`).
- Reporte HTML autocontenido con índice lateral filtrable, búsqueda global, tablas
  ordenables, tema claro/oscuro y hoja de estilos para impresión.
- Opciones de línea de comandos: `-o/--output`, `-q/--quiet`, `--format`, `--redact`,
  `--sections`, `--skip-sections`, `--event-log-days`, `--scan-git-repos`,
  `--include-missing-updates`, `-h/--help`.
- **`--format HTML,JSON,MD`** para elegir los formatos de salida. Los archivos de una
  misma ejecución comparten el timestamp del nombre. El JSON replica el esquema de la
  versión Windows (`Metadata`, `Capabilities`, `Findings`, `FindingsSummary`,
  `Sections`), en UTF-8 sin BOM y con fechas ISO 8601, para que una herramienta de
  ingesta sirva para ambas plataformas. `--redact` se aplica a los tres formatos.
- **`--redact`** para generar una versión compartible del reporte: enmascara
  direcciones IPv4 e IPv6, cuentas, rutas de directorios personales, números de serie,
  direcciones MAC y hashes largos. Deja intactas las rutas de sistema, las direcciones
  comodín y de loopback, y las versiones de paquetes y kernel, porque no identifican a
  nadie y son el contenido mismo de la auditoría.
- Degradación explícita sin privilegios: cada sección afectada indica qué le faltó y
  por qué, en vez de omitirse en silencio o inventar un valor.
- Soporte de listado de paquetes para `apt`, `dnf`, `yum`, `zypper`, `apk` y `pacman`;
  de actualizaciones pendientes para `apt`, `dnf`, `yum` y `zypper`.

### Decisiones de diseño que conviene conocer

- **Bash en vez de Python.** Hay servidores sin Python instalado, a veces por política
  de seguridad. Un script de bash sin dependencias corre en cualquier servidor tal como
  viene.
- **TSV como formato interno en vez de JSON**, para no depender de `jq`, que no está
  instalado por defecto en la mayoría de las distribuciones de servidor.
- **`LC_ALL=C` forzado** en un único punto por donde pasan todos los comandos externos.
  Sin esto, un servidor con `LANG=es_ES.UTF-8` devolvía `221,1M` donde los parsers
  esperaban `221.1M`, y fallaban en silencio.
- **Sin timeout no hay auditoría confiable**: todo comando externo pasa por una capa
  con `timeout`. Un montaje NFS con el servidor caído o un LDAP inalcanzable en
  `nsswitch.conf` colgaban el reporte completo.
- **Aislamiento por colector.** No se usa `set -e`: cada colector se ejecuta envuelto,
  y su falla se registra sin abortar el reporte.
- **En JSON, todos los valores de datos son strings.** Emitir números desnudos desde
  bash arriesga producir JSON inválido cuando el valor real es `N/D`, `221.1M` o está
  vacío. Solo los contadores son números.
- **El JSON se sanea contra bytes no válidos en UTF-8; el HTML no.** La especificación
  de JSON exige UTF-8 válido, así que un byte inválido hace que un parser estricto
  rechace el archivo completo; un navegador lo tolera. El saneado se hace en una sola
  pasada con `iconv -c`, y **sin evaluar su código de salida**: con `-c`, iconv sale
  con código 1 justamente cuando descartó bytes, o sea en el caso de éxito. La primera
  versión de ese bloque conservaba el archivo inválido por creerle al código de salida.
- **El enmascarado exige dos condiciones, no una.** Una celda se enmascara solo si la
  columna es de un tipo enmascarable **y** el valor tiene esa forma. Enmascarar solo
  por patrón habría destruido la versión del kernel (`6.6.87.2` tiene cuatro octetos
  válidos); enmascarar solo por nombre de columna habría roto la columna `Origen`, que
  es una red en la tabla de rutas y el nombre de un repositorio en la de paquetes.
- **Solo lectura, estricto.** Esto descartó sondear los protocolos TLS negociables,
  que habría exigido levantar un listener o abrir conexiones. La sección de TLS reporta
  únicamente lo que se puede leer con certeza y **no emite veredicto de cumplimiento**,
  a propósito.

### Limitaciones de esta versión

- Sin modo flota (auditoría de varios equipos en paralelo).
- Sin exportación a CSV, y sin comparación contra baseline (drift).
- Sin modo remoto nativo por SSH: para auditar un servidor remoto se copia el script
  y se ejecuta allá.
- Actualizaciones pendientes no implementadas para `apk` ni `pacman`.
- Cobertura de RHEL/CentOS y SUSE escrita contra el formato documentado pero **no
  validada en un equipo real**.

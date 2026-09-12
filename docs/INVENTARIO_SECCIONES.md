# Inventario de secciones

Fuente de verdad de las columnas que expone cada seccion de `cat-serverfullreport`,
generada leyendo el resultado de una ejecucion real del script como root. Las
reglas del motor de hallazgos leen estas columnas por nombre para decidir si
hay un dato utilizable, y el enmascarado de `--redact` decide que celda ocultar
tambien por nombre de columna (ver `tipo_columna()` en el script). Si este
archivo diverge de las columnas reales que emite el codigo, el resultado son
falsos positivos en las reglas o datos identificatorios que `--redact` deja
pasar en claro sin que nadie lo note.

> Una seccion cuyo rol o dato no esta presente en el equipo auditado se
> registra igual en el reporte, pero como un mensaje explicativo en vez de una
> tabla (por ejemplo, "SELinux/AppArmor no detectado" o "journalctl no esta
> disponible en este equipo"). En ese caso la seccion no tiene las columnas
> listadas abajo. Esto es deliberado: el as-built documenta explicitamente que
> ese rol o dato no existe en vez de omitir la seccion del reporte.

## Secciones

| Id | Titulo | Columnas | Enmascarado con --redact |
|---|---|---|---|
| 1.1 | Sistema Operativo y Kernel | Hostname, SistemaOperativo, Kernel, Arquitectura, Uptime | - |
| 1.2 | Servicios (systemd) | Unidad, Carga, Activo, Subestado, Descripcion | Descripcion (texto) |
| 1.2.3 | Actualizaciones Pendientes | Paquete, Origen, VersionNueva, EsDeSeguridad | Origen (ip) |
| 1.3 | Uso de Disco (Sistemas de Archivos) | PuntoMontaje, SistemaArchivos, TotalKB, UsadoKB, LibreKB, UsoPct | - |
| 1.4.1 | Interfaces de Red (IPv4) | Interfaz, DireccionIP, Prefijo, Broadcast, Scope | DireccionIP (ip), Broadcast (ip) |
| 1.4.2 | Tabla de Rutas | Destino, Gateway, Interfaz, Origen, Metrica | Destino (ip), Gateway (ip), Origen (ip) |
| 1.4.3 | Puertos en Escucha | Protocolo, DireccionLocal, Puerto, Proceso | DireccionLocal (ip) |
| 1.5.1 | Discos y Particiones (lsblk) | Nombre, Tamano, Tipo, SistemaArchivos, PuntoMontaje | - |
| 1.5.2 | Cifrado de Disco (LUKS) | Dispositivo, PuntoMontaje | - |
| 1.6 | Paquetes Instalados | Paquete, Version | - |
| 1.7 | Servidor SSH (Configuracion Efectiva) | Directiva, Valor | - |
| 1.8.1 | Cuentas con Privilegios de Root (UID 0) | Usuario, UID, GID, Shell, DirectorioHome | Usuario (cuenta), DirectorioHome (ruta) |
| 1.8.2 | Grupos con Privilegios de Sudo | Grupo, Miembros, ArchivosEnSudoersD | Miembros (cuenta) |
| 1.8.3 | Politica de Contrasenas (login.defs) | Parametro, Valor | - |
| 1.8.4 | Politica de Auditoria (auditd) | Campo, Valor | - |
| 1.8.5 | Binarios con SUID | Ruta, Propietario | Ruta (ruta), Propietario (cuenta) |
| 1.8.6 | Capabilities de Linux Asignadas | Ruta, Capabilities | Ruta (ruta) |
| 1.8.7 | Usuarios Locales | Usuario, UltimoCambioPassword, VenceEn, InactivaEn | Usuario (cuenta) |
| 1.9 | Control de Acceso Obligatorio (SELinux / AppArmor) | Mecanismo, Estado, Detalle | Detalle (texto) |
| 1.10.1 | Tareas Programadas (cron) | Origen, Detalle | Origen (ip), Detalle (texto) |
| 1.10.2 | Timers de systemd | Timer, Servicio | - |
| 1.11 | Hardening de Kernel (sysctl) | Parametro, Valor | - |
| 1.11.2 | Configuracion TLS de OpenSSL | Campo, Valor | - |
| 1.12 | Reinicio Pendiente | ReinicioPendiente, Motivo | - |
| 1.13 | Certificados TLS Instalados | Archivo, CN, FechaExpiracion, DiasParaExpirar | - |
| 1.14 | Firewall | Campo, Valor | - |
| 1.15 | Roles de Servidor (nginx / Apache / Samba / NFS) | Rol, Estado, Detalle | Detalle (texto) |
| 1.16 | Hardware del Host | Hostname, Fabricante, Modelo, TipoSistema, MaquinaVirtual, CPU, NucleosFisicos, NucleosLogicos, MemoriaTotalGB, NumeroSerie | NumeroSerie (serial) |
| 1.17.1 | Errores Recientes del Journal | FechaHoraUnidad | - |
| 1.17.2 | Intentos de Login Fallidos | Detalle | Detalle (texto) |
| 1.17.3 | Top 15 Procesos por Memoria | Usuario, PID, CPUPct, MemPct, Comando | Usuario (cuenta), Comando (texto) |
| 1.17.4 | Salud SMART de Discos Fisicos | Disco, EstadoSMART | - |
| 1.17.5 | Apagados Inesperados y Reinicios | Detalle | Detalle (texto) |
| 1.18 | Sincronizacion de Hora (NTP) | ZonaHoraria, RelojSincronizado, ServicioNTP | - |
| 1.19.1 | Interprete de Python | Version, Ruta | Ruta (ruta) |
| 1.19.2 | Paquetes de Python (pip) | Paquete, Version | - |
| 1.19.3 | Entornos Virtuales de Python (venv) | Ruta, VersionPython | Ruta (ruta) |
| 1.19.4 | Repositorios Git Detectados | Ruta, RamaActual, Remoto | Ruta (ruta), Remoto (texto) |
| 1.19.5 | Contenedores Docker en Ejecucion | Nombre, Imagen, Estado | - |
| 1.20.1 | Antivirus / EDR Detectado | AgenteDetectado | - |
| 1.20.2 | Agentes de Backup Detectados | HerramientaDetectada | - |

## Como mantener este archivo

Al agregar una seccion nueva al script, agrega tambien su fila en la tabla de
arriba con los nombres exactos de columna que emite el colector (los que van
en el `printf` del encabezado del TSV, no una version resumida o traducida).
Un nombre de columna que no coincide caracter por caracter con el codigo hace
que las reglas del motor de hallazgos que lo buscan no encuentren nada y
queden calladas sin que se note el error.

Si alguna columna nueva contiene datos identificatorios — direcciones IP,
nombres de cuenta, rutas de directorios personales, numeros de serie, o texto
libre que puede contener cualquiera de esos datos en medio de la cadena — hay
que mapearla ademas en `tipo_columna()` dentro de `cat-serverfullreport`. Si no
se mapea ahi, `--redact` la deja pasar en claro aunque el usuario haya pedido
explicitamente que el reporte quede anonimizado.

Tres columnas de esta tabla (`Detalle` en 1.17.2 y 1.17.5, `AgenteDetectado`
en 1.20.1) corresponden a secciones de una sola columna que, cuando el dato o
la herramienta no esta disponible en el equipo auditado, salen como mensaje en
vez de tabla — por eso conviene verificar el nombre exacto contra el codigo en
vez de contra una corrida cualquiera, que puede no haber tenido filas.

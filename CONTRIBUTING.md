# Cómo contribuir

Gracias por el interés. Este documento describe las reglas del proyecto y cómo
proponer cambios.

Casi todas las reglas duras de más abajo vienen de un bug real que costó tiempo
encontrar. No son preferencias de estilo: romperlas rompe el script en producción,
y lo peor es que varias fallan **en silencio**, produciendo un reporte que se ve
perfecto y es incorrecto.

---

## Antes de empezar

- **Un cambio por pull request.** Explica el **por qué**, no el qué.
- Actualiza [CHANGELOG.md](CHANGELOG.md) siguiendo el formato Keep a Changelog.
- Sube `SCRIPT_VERSION` solo si el cambio cierra una versión.
- **Nunca subas una corrida real.** Los reportes contienen cuentas locales, IPs
  internas, puertos en escucha y números de serie. El `.gitignore` ya cubre
  `AsBuilt_*` y la carpeta `.draft/`, pero revisa igual antes de hacer commit.

---

## Reglas duras

### 1. Bash 4.0+, un solo archivo

No partir el script. El proyecto entero existe para que se pueda copiar **un** archivo
a un servidor y ejecutarlo.

Bash 4.0 es el piso: se usan arreglos asociativos (`declare -A`), `${var,,}` y
expansión de parámetros con clases de caracteres. No se persigue compatibilidad con
`sh` POSIX estricto. Tampoco agregar dependencias: nada de `jq`, `python`, `curl` ni
nada que no venga en el sistema base de una distribución de servidor.

### 2. Solo lectura, sin excepciones

El script nunca modifica el equipo auditado. No inicia servicios, no abre puertos, no
escribe fuera de sus archivos temporales.

Esto tiene consecuencias reales de diseño. Por ejemplo: determinar con certeza qué
protocolos TLS acepta un sistema exige abrir una conexión TLS de verdad. Como eso
violaría la regla, la sección de TLS reporta solo lo que puede leer y **no emite
veredicto de cumplimiento**. Preferimos no responder una pregunta antes que
responderla mal.

### 3. Todo comando externo pasa por `ejecutar_comando`

```bash
salida="$(ejecutar_comando 10 lsblk -b -o NAME,SIZE)"
```

Dos razones, las dos ya cobraron bugs:

- **Locale.** `ejecutar_comando` fuerza `LC_ALL=C`. En un servidor con
  `LANG=es_ES.UTF-8`, `lsblk` devolvía `221,1M` donde el parser esperaba `221.1M`, y
  `awk` imprimía `0,02` en vez de `0.02`. Forzarlo en el único punto por donde pasan
  todos los comandos evita tener que acordarse en cada colector nuevo.
- **Timeout.** Sin él, un montaje NFS cuyo servidor no responde deja a `findmnt`
  colgado para siempre, y `getent` con un LDAP inalcanzable en `nsswitch.conf` espera
  el timeout de la librería. Cualquiera de los dos cuelga el reporte completo.

**Cuidado con los builtins de bash.** `timeout` solo puede ejecutar binarios reales,
así que `ejecutar_comando 5 command -v foo` **siempre falla en silencio**. Este error
se cometió tres veces en este proyecto. Para detectar si un binario existe:

```bash
# correcto dentro de un colector
if ejecutar_comando 5 which foo >/dev/null 2>&1; then ...

# correcto en detectar_capacidades (corre local, y 'command -v' no forkea)
command -v foo >/dev/null 2>&1 && CAPS[foo]=1
```

### 4. Nunca afirmar sobre un dato que no se pudo leer

El principio central del proyecto:

> Ninguna regla emite un hallazgo — ni negativo ni positivo — sobre un dato que no
> pudo leer. Un `OK` falso es peor que el silencio, porque el administrador confía
> en él.

En la práctica, toda regla verifica antes de evaluar:

```bash
case "$valor" in
  ''|N/D|*[!0-9]*) continue ;;   # no se pudo determinar: no se afirma nada
esac
```

El caso que originó la regla: los montajes de solo lectura de `snap` reportan siempre
100 % de uso **por diseño**, porque nunca se puede escribir en ellos. Evaluarlos como
un disco normal producía ocho falsos `CRIT` en cualquier Ubuntu con snapd.

### 5. Aislamiento por colector

**No usar `set -e`.** Está desactivado a propósito: cada colector corre envuelto en
`medir_seccion`, que registra la falla sin abortar el reporte. Un rol ausente, un
permiso denegado o un binario inexistente jamás deben tumbar la ejecución completa.

Por la misma razón, toda lectura de `CAPS` lleva valor por defecto:

```bash
if [ "${CAPS[es_root]:-0}" = "1" ]; then ...
```

Con `set -u` activo, leer una clave no escrita aborta el script entero con "unbound
variable" — justo lo contrario de lo que promete este proyecto.

### 6. TSV: escribir con `tsv_escape`, leer con `dividir_tsv`

**Nunca leer una fila TSV con `IFS=$'\t' read -r a b c`.** El tabulador es carácter de
espacio en blanco para `IFS`, y bash **colapsa las secuencias** de esos caracteres. Una
fila legítima con un campo vacío en el medio se lee con un campo menos y **todos los
valores siguientes se corren un lugar a la izquierda**:

```bash
$ printf 'A\t\tC\tD' | { IFS=$'\t' read -r f1 f2 f3 f4; echo "[$f1][$f2][$f3][$f4]"; }
[A][C][D][]          # mal: el vacio desaparecio y todo se corrio

$ dividir_tsv $'A\t\tC\tD'; printf '[%s]' "${CAMPOS[@]}"
[A][][C][D]          # bien
```

Esto se detectó en un reporte real: **8 de 12 hallazgos** mostraban la recomendación
bajo la columna "Detalle", el control bajo "Recomendación" y se quedaban sin enlace.
En las tablas de datos es peor todavía, porque el corrimiento no se nota: un valor
aparece bajo el encabezado equivocado y se ve plausible.

Al escribir, pasar **cada celda** por `tsv_escape`, que elimina tabuladores y saltos
de línea del contenido.

### 7. Escapado de HTML: cuidado con el `&`

En `${var//patron/reemplazo}` de bash, un `&` **sin escapar** dentro del reemplazo se
sustituye por el texto que coincidió con el patrón, igual que en `sed`. No es un `&`
literal. Por eso cada reemplazo de `html_escapar_var` lleva `\&`.

El escapado vive en **un solo lugar** (`html_escapar_var`); `html_escapar` es solo un
envoltorio que imprime. Tener dos copias de esas cinco líneas es exactamente cómo se
reintroduce esta trampa arreglándola en una sola.

En los bucles calientes del renderizador usar `html_escapar_var` (escribe en la global
`HTML_ESC`, sin subshell) en vez de `$(html_escapar ...)`: cada `$(...)` forkea un
proceso, y el renderizador se ejecuta una vez por celda del reporte.

### 8. Si tu sección expone datos identificatorios, mapea sus columnas en `--redact`

El enmascarado se decide en `tipo_columna()`, por **nombre de columna**. Si agregas
una sección con una columna que contenga IPs, cuentas, rutas personales o números de
serie, agrégala ahí; si no, `--redact` la dejará pasar en claro.

El enmascarado exige **dos condiciones**: que la columna sea de un tipo enmascarable
**y** que el valor tenga esa forma. Las dos son necesarias, y cada una evita un fallo
concreto:

- Solo por patrón destruiría datos legítimos: la versión del kernel `6.6.87.2` tiene
  cuatro octetos válidos y es indistinguible de una IPv4.
- Solo por nombre de columna rompería los nombres repetidos: `Origen` es una red en la
  tabla de rutas y el nombre de un repositorio en la de paquetes.

Para columnas de texto libre donde el dato sensible puede estar **en medio** de una
cadena más larga (una línea de comando, una descripción), usa el tipo `texto`, que
enmascara por patrón dentro del valor. El caso que lo obligó: la columna `Comando` de
"Top procesos" mostraba `/home/usuario/.npm-global/.../index.js` y filtraba el nombre
de la cuenta con `--redact` activo.

Al probar `--redact`, **audita el resultado buscando fugas**, no confíes en que
funcionó. Un reporte "compartible" que filtra datos es peor que no tener la opción,
porque se comparte creyendo que está limpio. La primera versión de esta función
filtraba 9 direcciones IP reales porque el patrón estaba anclado al final del valor y
las direcciones venían con sufijos (`10.0.0.5%eth0`, `10.0.0.5/24`, `10.0.0.5:22`).

### 9. Si tu sección agrega columnas, los tres formatos las heredan gratis

Los exportadores (`generar_html`, `exportar_json`, `exportar_markdown`) leen los
mismos TSV y los mismos arreglos `SEC_*`. Una sección nueva aparece en HTML, JSON y
Markdown sin tocar ningún exportador — eso es lo que compra la decisión de usar TSV
como formato interno.

Lo que sí hay que respetar al tocar un exportador:

- **Leer con `dividir_tsv`** (regla 6). Aplica igual a los tres.
- **Aplicar `--redact`** (regla 8). Un formato que no enmascara convierte el flag en
  una mentira: el usuario comparte el archivo creyendo que está limpio.
- **En JSON, los valores de datos van como strings.** Solo `RowCount`, los contadores
  de `FindingsSummary` y `TiempoTotalSegundos` son números. Un `N/D` o un `221.1M`
  emitido como número desnudo produce JSON inválido.
- **Cuidado con las comas finales** al armar JSON a mano: el último elemento de cada
  array y objeto no lleva coma. Es el error más fácil de cometer sin un serializador.
- **En Markdown, escapar el pipe** `|` en cada celda con `markdown_escapar_celda`, o
  la tabla se rompe.

Y un comando externo que conviene recordar: **`iconv -c` sale con código 1 cuando
descartó bytes**, o sea en su caso de éxito. El saneado UTF-8 del JSON valida la forma
del resultado (que termine en `}`) en vez de creerle al código de salida. Es el mismo
patrón que `systemd-detect-virt`, que imprime `none` y sale con 1 — ver regla 3.

### 10. Cero caracteres no-ASCII en el script

Ni en comentarios ni en cadenas. Escribir "Configuracion", "Numero", "espiritu". El
HTML generado sí puede usar entidades (`&oacute;`, `&mdash;`).

Verificable:

```bash
LC_ALL=C grep -n '[^ -~	]' cat-serverfullreport
```

Hoy: cero coincidencias. Mantenerlo así. Los archivos `.md` del repositorio sí llevan
acentos normales.

### 11. Español neutro

Todo el texto visible: mensajes del script, notas de cada sección, hallazgos y
documentación.

**Prohibido el voseo** en todos lados: nunca "descargá", "ejecutá", "usá", "podés",
"tenés". Sin modismos regionales de ningún país. Ante la duda, la forma que entiende
cualquier administrador hispanohablante.

Cada superficie tiene su registro, y son distintos a propósito:

- **Documentación (`.md`): tuteo.** "descarga", "ejecuta", "usa".
- **Mensajes del script: usted.** "Verifique que...", "Vuelva a ejecutarlo". Es el
  registro de una herramienta corporativa de infraestructura. **No corregirlo a
  tuteo**: no es un descuido, es la convención.

Los anglicismos técnicos ya asentados (baseline, as-built, drift, host) se dejan como
están: traducirlos hace el texto menos claro, no más.

---

## Agregar una sección

```bash
recolectar_mi_seccion() {
  local id="1.X" archivo="$TMPDIR_TRABAJO/sec_1_X.tsv"

  if ! ejecutar_comando 5 which miherramienta >/dev/null 2>&1; then
    agregar_seccion_mensaje "$id" "Titulo Legible" \
      "Que es esto y por que le importa a un administrador." \
      "miherramienta no esta instalada en este equipo."
    return 0
  fi

  local salida
  salida="$(ejecutar_comando 10 miherramienta --listar 2>/dev/null)"
  if [ -z "$salida" ]; then
    agregar_seccion_mensaje "$id" "Titulo Legible" \
      "Que es esto y por que le importa." \
      "No se pudo obtener la informacion."
    return 0
  fi

  {
    printf 'Columna1\tColumna2\n'
    printf '%s\n' "$salida" | awk 'BEGIN{OFS="\t"} NF>=2 {print $1, $2}'
  } > "$archivo"

  agregar_seccion_tabla "$id" "Titulo Legible" \
    "Una frase explicando que es y por que le importa a un administrador." \
    "$archivo"
}
```

Después hay que registrar la llamada con `medir_seccion` dentro de `main`.

La nota de la sección es valor real, no relleno. "Aqui se muestran los servicios" no
sirve; "Los servicios en modo automatico que no estan corriendo suelen indicar una
falla de arranque" sí.

Cuando el rol no está, **registrar la sección igual** con el mensaje explicativo: el
índice queda completo y el as-built documenta explícitamente que ese rol no existe.

---

## Agregar una regla de hallazgos

```bash
motor_regla_mi_topico() {
  local archivo="${SEC_DATAFILE[1.X]:-}"
  [ -n "$archivo" ] && [ -s "$archivo" ] || return 0   # sin datos: no se afirma nada

  local valor
  valor="$(awk -F'\t' 'NR==2{print $2}' "$archivo")"
  case "$valor" in
    ''|N/D|*[!0-9]*) return 0 ;;                        # no evaluable
  esac

  if [ "$valor" -gt 100 ]; then
    agregar_hallazgo "WARN" "Categoria" "Titulo corto del hallazgo" \
      "Evidencia concreta: valor $valor" \
      "Que hacer al respecto." \
      "CIS X.Y / ISO 27001 A.Z" "1.X"
  else
    agregar_hallazgo "OK" "Categoria" "Descripcion afirmativa de lo verificado" "" \
      "Por que esta bien." "CIS X.Y" "1.X"
  fi
}
```

El detalle debe llevar **evidencia concreta** (`PASS_MAX_DAYS=99999 en
/etc/login.defs`), no una afirmación genérica. Es lo que permite verificar el hallazgo
sin volver al servidor.

---

## Cómo probar

No hay framework de tests. Cuatro verificaciones, en orden:

**1. Sintaxis**

```bash
bash -n cat-serverfullreport
```

**2. Sin caracteres no-ASCII**

```bash
LC_ALL=C grep -n '[^ -~	]' cat-serverfullreport
```

**3. Ejecutarlo entero, y revisar stderr**

```bash
./cat-serverfullreport -o /tmp
```

Una corrida limpia termina con código 0 y **stderr vacío**. Cualquier cosa en stderr
es un bug, aunque el reporte se haya generado.

**4. La que de verdad importa: contra un servidor real, como root**

```bash
scp cat-serverfullreport servidor:/tmp/
ssh servidor 'sudo /tmp/cat-serverfullreport -o /tmp'
```

**No des por bueno un cambio sin correr el script completo contra datos reales.** Los
bugs más difíciles de este proyecto —el colapso de campos TSV, el corrimiento de
columnas, los tamaños con coma decimal, el `SIGPIPE` en un pipeline con `head`— eran
invisibles en pruebas aisladas y solo aparecieron con datos reales a escala real.
Probar en dos entornos distintos (distribuciones o idiomas diferentes) atrapa bastante
más que probar dos veces en el mismo.

---

## Ideas pendientes

Sin fecha ni compromiso, en orden aproximado de utilidad:

- Validar la cobertura de RHEL / CentOS en un equipo real.
- Actualizaciones pendientes para `apk` y `pacman`.
- Exportación a JSON / CSV / Markdown.
- Comparación contra baseline para detectar drift.
- Modo flota: auditar varios equipos en paralelo.
- Modo remoto nativo por SSH multiplexado. `ejecutar_comando` es el único punto por
  donde pasan todos los comandos externos, así que es ahí donde bifurcaría. Hoy ese
  modo **no existe** y no hay ninguna rama a medio construir.

---

## Licencia

Al contribuir, aceptas que tu aporte se publique bajo la licencia
[MIT](LICENSE) del proyecto.

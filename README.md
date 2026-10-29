# pipeline-template-security-scan

Template reutilizable de GitHub Actions para **SAST** (análisis estático: revisa el código sin ejecutarlo). Soporta **JavaScript/TypeScript y Java** (vanilla, Spring Boot y Quarkus). Cada proyecto lo consume con un solo workflow de unas 20 líneas.

## Uso en un proyecto

1. Copiá [`examples/security.yml`](examples/security.yml) a `.github/workflows/security.yml` del proyecto.
2. (Opcional) En branch protection, marcá como requerido **un solo check**: `security / Security gate`.
3. (Opcional) Agregá overrides propios del proyecto:

| Archivo en el proyecto | Efecto |
|---|---|
| `.semgrep/` | Reglas Semgrep propias (se suman a `p/default`) |
| `.gitleaks.toml` / `.gitleaksignore` | Reglas y falsos positivos de secretos |
| `.trivyignore` | CVEs aceptados / falsos positivos |
| `spotbugs-exclude.xml` | Falsos positivos de SpotBugs ([formato](https://spotbugs.readthedocs.io/en/latest/filter.html)) |

## Estrategia: capas con detección automática

```
security-scan.yml (orquestador, lo único que llaman los proyectos)
│
├─ detect ───────────── git ls-files → ["java","javascript"]  (o override manual)
│
├─ CAPA 1 · común (siempre)          sast-common.yml
│     Semgrep  → patrones inseguros en cualquier lenguaje
│     Gitleaks → secretos commiteados (todo el historial)
│     Trivy    → dependencias con CVEs + misconfig de Docker/Terraform/K8s
│
├─ CAPA 2 · por lenguaje (condicional)
│     javascript → sast-javascript.yml  Semgrep JS/TS (p/javascript p/typescript p/nodejsscan p/react)
│     java       → sast-java.yml        Semgrep Java + SpotBugs/FindSecBugs
│
└─ security-gate ─────── único check requerido; agrega todos los resultados
```

## Java: vanilla, Spring Boot y Quarkus

`sast-java.yml` corre dos jobs:

| Job | Requiere compilar | Qué encuentra |
|---|---|---|
| `semgrep-java` | No | `p/java` + `p/findsecbugs`: inyecciones, deserialización, XXE, crypto débil. Corre aunque el build falle |
| `spotbugs` | Sí | SpotBugs + FindSecBugs sobre bytecode: análisis de flujo más profundo, detectores de Spring MVC/Security y JAX-RS (Quarkus) |

- **Build autodetectado:** con `pom.xml` usa `./mvnw` (o `mvn`) con `-DskipTests compile`. Con `build.gradle(.kts)` usa `./gradlew` (o `gradle`) con `classes -x test`. Funciona en proyectos multi-módulo.
- **Framework detectado:** Spring Boot o Quarkus aparece en el resumen del job. No requiere configuración especial.
- **Si la compilación necesita algo distinto** (perfiles, un submódulo, repos privados), usá `java-build-command`. Por ejemplo: `./mvnw -B -Pci -pl api -am -DskipTests compile`.
- **Si no podés compilar en CI**, usá `enable-spotbugs: false` y queda sólo Semgrep.
- **Severidad de SpotBugs:** se reportan sólo bugs de la categoría `SECURITY`. La prioridad se mapea así: alta → critical, normal → high y baja → low. Los detectores informativos como `SPRING_ENDPOINT` caen en low.

## Inputs

| Input | Default | Descripción |
|---|---|---|
| `languages` | `auto` | `auto` o lista separada por comas: `javascript,java` |
| `fail-on-severity` | `high` | `none` · `critical` · `high` · `medium` · `low` |
| `exclude-paths` | `node_modules,vendor,dist,build,target,...` | Directorios excluidos del análisis de código fuente |
| `upload-sarif` | `true` | Sube a *Security → Code scanning*. Ponelo en `false` en repos privados sin GHAS |
| `semgrep-config` | `p/default` | Rulesets de la capa común, separados por espacio |
| `java-version` | `21` | JDK (Temurin) para compilar |
| `java-build-command` | — | Compilación custom; vacío = autodetectar |
| `enable-spotbugs` | `true` | `false` = Java sólo con Semgrep, sin compilar |

### Cómo se calcula la severidad

Se toma `security-severity` de la regla (escala CVSS): critical ≥ 9.0, high ≥ 7.0, medium ≥ 4.0. Si la regla no la trae, se usa el nivel SARIF: `error` = 8 (high), `warning` = 5 (medium), `note` = 2 (low).
Excepción: **Gitleaks** bloquea ante cualquier secreto, con cualquier umbral salvo `none`.

## Cómo contribuir

### Flujo

1. Creá una rama desde `main` (`feat/sast-kotlin`, `fix/trivy-skip-dirs`, etc.).
2. Hacé los cambios y validalos en local (ver abajo).
3. Abrí un PR contra `main`. El workflow [`ci.yml`](.github/workflows/ci.yml) corre `actionlint` y tiene que pasar.
4. Cuando se mergea, publico la versión (ver [Publicar y versionar](#publicar-y-versionar-el-template)).

### Validar en local

```bash
# Sintaxis de workflows, expresiones ${{ }} y shellcheck de cada run:
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12 .github/workflows/*.yml examples/security.yml
```

### Probar un cambio desde un proyecto real

Los workflows se llaman entre sí con `@v1`. Por eso, si un repo de prueba apunta a tu rama (`security-scan.yml@feat/mi-rama`), el orquestador sale de tu rama, pero los workflows `sast-*.yml` y la action `sarif-report` se siguen tomando de `v1`. Para probar el cambio completo:

1. En tu rama, reemplazá temporalmente `@v1` por `@feat/mi-rama` en las referencias internas:
   `grep -rl '@v1' .github | xargs sed -i 's|pipeline-template-security-scan/\(.*\)@v1|pipeline-template-security-scan/\1@feat/mi-rama|'`
   (en macOS: `sed -i ''`)
2. En el repo de prueba, apuntá a `security-scan.yml@feat/mi-rama` y corré el workflow.
3. **Antes de mergear, volvé las referencias a `@v1`.** El PR no se aprueba si quedan referencias a ramas.

### Agregar un lenguaje nuevo

1. Crear `.github/workflows/sast-<lenguaje>.yml` copiando `sast-javascript.yml`; la herramienta debe generar SARIF y terminar con `sarif-report`.
2. En `security-scan.yml`, agregar el regex del lenguaje a `PATTERNS` en `detect`, agregar el job `sast-<lenguaje>` con `if: contains(fromJSON(needs.detect.outputs.languages), '<lenguaje>')` y sumarlo a `needs` de `security-gate`.
3. Documentar el lenguaje en este README (diagrama de capas, inputs nuevos y archivos de override).

### Actualizar la versión de una herramienta

Las versiones están en el bloque `env:` de cada workflow. Para binarios descargados (Gitleaks, SpotBugs, FindSecBugs), actualizá también el `*_SHA256`:

```bash
curl -sSfL -o tool.tgz <url-de-la-release> && shasum -a 256 tool.tgz
```

Usá versiones con al menos dos semanas de publicadas y revisá el changelog por cambios en flags o en el formato SARIF.

### Checklist del PR

- [ ] `actionlint` pasa sin errores.
- [ ] Las referencias internas apuntan a `@v1`, no a una rama.
- [ ] Toda herramienta nueva genera SARIF y pasa por `sarif-report`.
- [ ] Versiones fijadas (nada de `latest`), con checksum si se descarga un binario.
- [ ] Los valores de `inputs` llegan a los scripts por `env:`, nunca con `${{ }}` dentro de `run:`, para evitar inyección.
- [ ] El README está actualizado si cambian inputs, herramientas o comportamiento.
- [ ] Si hay un breaking change (se renombra o se quita un input, o cambia un default de forma que rompe a los consumidores), se indica en el PR porque requiere `v2`.

## Publicar y versionar el template

```bash
git tag v1.0.0 && git tag -f v1 && git push origin v1.0.0 && git push -f origin v1
```

- Los workflows internos se referencian entre sí como `codeaseguro/pipeline-template-security-scan/...@v1`. **Si forkeás o renombrás el repo, reemplazá ese prefijo** (`grep -rl codeaseguro/pipeline-template-security-scan .github | xargs sed -i 's|codeaseguro/pipeline-template-security-scan|ORG/REPO|g'`).
- Un breaking change implica `v2`: actualizá también las referencias internas `@v1` → `@v2`.
- Si el repo es **privado/interno**: *Settings → Actions → General → Access* → "Accessible from repositories in the organization".
- Las versiones de las herramientas están fijadas, con checksum cuando se descargan binarios, en el bloque `env:` de cada workflow. Dependabot actualiza las actions.

## Licencia

[MIT](LICENSE) © 2026 Codea Seguro. Las herramientas que ejecuta el template (Semgrep, Gitleaks, Trivy, SpotBugs, FindSecBugs) tienen sus propias licencias.

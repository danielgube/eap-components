# Manual para crear componentes EAP

Esta guía permite añadir un componente sin conocer el código fuente de EAP.
Describe el contrato público de los manifiestos de componentes para el esquema
`schemaVersion: 3`, sus límites actuales y el proceso completo de prueba y
publicación.

## 1. Qué es un componente

Un componente es software que EAP puede resolver, descargar, verificar,
instalar y activar dentro de uno o varios profiles. Puede publicar ejecutables
en `PATH`, variables de entorno, comandos y launchers de aplicaciones.

Ejemplos: Java, Maven, Git, Node.js, Bruno, Eclipse o IntelliJ IDEA.

Use un componente cuando se cumpla alguna de estas condiciones:

- el software tiene versiones o líneas que EAP debe instalar y actualizar;
- debe aportar ejecutables o variables al entorno completo del profile;
- es una aplicación portable con configuración o workspace propios;
- representa una aplicación ya instalada en el equipo que EAP sólo debe
  vincular.

Para una utilidad pequeña formada por scripts y uno o varios comandos suele ser
mejor crear una Pocketool. Consulte el manual `CREAR_POCKETOOLS.md` del
repositorio `eap-pocketools`.

## 2. Límites actuales

Antes de empezar, compruebe que el componente cabe en el contrato actual:

- EAP sólo instala automáticamente archivos ZIP.
- Todas las fuentes remotas deben usar HTTPS.
- El ZIP debe proporcionar SHA-256 o SHA-512 mediante uno de los resolvers
  soportados.
- El destino es Windows; los manifiestos actuales usan Windows x64.
- Un repositorio de componentes sólo contiene JSON. EAP todavía no descarga ni
  ejecuta adaptadores Python externos.
- Si el fabricante publica sus versiones mediante una API o estructura no
  soportada, no basta con inventar un nuevo valor en `resolver.type`: será
  necesario incorporar primero un resolver compatible en EAP o esperar a la
  futura Adapter API.
- MSI, EXE instalables, scripts de instalación, enlaces simbólicos y archivos
  que deban escribirse fuera de EAP no están soportados como instalaciones
  administradas.

La excepción es un componente `external`: EAP no instala el programa, sino que
el usuario selecciona un `.exe` ya existente en el equipo.

## 3. Estructura del repositorio

Un repositorio de componentes tiene esta forma:

```text
catalog.json
components/
  mi-componente.json
```

El nombre del archivo debe coincidir exactamente con el ID:

```text
components/<id>.json
```

No se admiten subdirectorios alternativos, rutas absolutas ni `..`.

El componente también debe aparecer en `catalog.json`:

```json
{
  "schemaVersion": 1,
  "catalogVersion": "1.1.0",
  "components": [
    {
      "id": "mi-componente",
      "manifest": "components/mi-componente.json"
    }
  ]
}
```

Reglas del catálogo:

- `schemaVersion` debe ser `1`.
- `catalogVersion` identifica la versión del catálogo. Se recomienda SemVer y
  aumentarla cuando cambie cualquier manifiesto.
- `components` debe contener entre 1 y 1000 entradas.
- Los IDs no pueden repetirse.
- El catálogo y cada manifiesto tienen un límite de 1 MiB durante la descarga.
- Dos repositorios externos no pueden publicar el mismo ID. EAP rechazará la
  composición en vez de escoger uno silenciosamente.
- Un componente externo sí puede sustituir al snapshot integrado en EAP con el
  mismo ID. Así funciona el catálogo oficial.

## 4. Ejemplo mínimo completo

Este ejemplo representa una aplicación gráfica publicada como ZIP en GitHub.
Es el punto de partida recomendado para la mayoría de aplicaciones portables.

```json
{
  "schemaVersion": 3,
  "id": "mi-cliente",
  "displayName": "Mi Cliente",
  "description": "Cliente portable de ejemplo",
  "category": "applications",
  "kind": "application",
  "info": {
    "description": "Cliente portable cuyos ajustes se conservan dentro del profile activo.",
    "paths": [
      {
        "displayName": "Configuración de Mi Cliente",
        "base": "profile",
        "relativePath": "components/mi-cliente/user-data"
      },
      {
        "displayName": "Workspace activo",
        "base": "workspace",
        "relativePath": "."
      }
    ]
  },
  "launchers": [
    {
      "id": "mi-cliente",
      "displayName": "Mi Cliente",
      "type": "application",
      "workspaceMode": "environment",
      "executable": "{{component.root}}/MiCliente.exe",
      "arguments": [],
      "environment": {
        "EAP_COMPONENT_DATA": "{{data.component}}"
      },
      "dataDirectories": [
        "{{data.component}}/user-data"
      ],
      "startMode": "detached"
    }
  ],
  "capability": {
    "id": "app.mi-cliente",
    "exclusive": false
  },
  "platform": {
    "os": "windows",
    "architecture": "x64",
    "archiveType": "zip"
  },
  "tracks": [
    {
      "id": 2,
      "displayName": "Mi Cliente 2.x estable"
    }
  ],
  "defaultProvider": "community",
  "defaultTrack": 2,
  "updatePolicy": "same-track",
  "providers": [
    {
      "id": "community",
      "componentId": "mi-cliente-community",
      "displayName": "Mi Cliente Community",
      "vendor": "Ejemplo",
      "license": "MIT",
      "homepage": "https://example.org/",
      "resolver": {
        "type": "github-release-asset",
        "apiUrl": "https://api.github.com/repos/example/mi-cliente/releases?per_page=100",
        "assetPattern": "^mi-cliente-(?P<version>\\d+\\.\\d+\\.\\d+)-windows-x64\\.zip$"
      },
      "verification": {
        "checksumAlgorithm": "sha256",
        "source": "GitHub release asset digest"
      }
    }
  ],
  "install": {
    "directoryTemplate": "mi-cliente/{provider}/{version}",
    "stripSingleRoot": false,
    "maxExtractBytes": 1073741824,
    "requiredFiles": [
      "MiCliente.exe",
      "resources/app.asar"
    ],
    "validation": {
      "type": "files-only"
    }
  },
  "data": {
    "directories": [
      {
        "path": "{{data.component}}/user-data",
        "displayName": "Configuración de Mi Cliente",
        "role": "configuration",
        "showInDashboard": true
      }
    ],
    "files": []
  },
  "environment": {
    "variables": {
      "MI_CLIENTE_HOME": "{{component.root}}"
    },
    "path": []
  }
}
```

Para adaptar el ejemplo hay que cambiar, como mínimo:

1. ID, nombres y metadatos.
2. Línea o líneas soportadas.
3. URL de la API de releases.
4. Expresión regular del asset.
5. Ejecutable, archivos obligatorios y rutas de datos.
6. Entrada correspondiente de `catalog.json`.

## 5. Contrato del manifiesto

### 5.1 Campos de primer nivel

| Campo | Obligatorio | Contrato |
|---|---:|---|
| `schemaVersion` | Sí | Debe valer `3`; EAP conserva lectura compatible de manifiestos `1` y `2`. |
| `id` | Sí | Identificador estable; debe coincidir con catálogo y archivo. |
| `displayName` | Sí | Nombre visible para el usuario. |
| `description` | Recomendado | Descripción breve. |
| `category` | Opcional | Agrupación informativa, por ejemplo `applications`. |
| `kind` | Sí | `application`, `external`, `runtime`, `server`, `service` o `tool`. |
| `info` | Sí | Descripción breve y rutas importantes relativas. |
| `launchers` | Sí | Lista; puede estar vacía salvo para `external`. |
| `capability` | Recomendado | Capacidad que otras herramientas pueden requerir. |
| `platform` | Recomendado | Metadatos de SO, arquitectura y archivo. |
| `tracks` | Sí | Una o más líneas instalables. |
| `defaultProvider` | Sí | ID de un proveedor declarado. |
| `defaultTrack` | Sí | ID de una línea declarada. |
| `updatePolicy` | Sí | `same-track` o `manual`. |
| `versioning` | No | Comparación de versiones; `scheme` puede ser `numeric` (por defecto) o `java`. |
| `majorUpdates` | No | `confirm-component-name` para ofrecer líneas mayores con confirmación reforzada. |
| `providers` | Sí | Uno o más proveedores. |
| `install` | Sí | Contrato de instalación o vinculación. |
| `data` | Opcional | Directorios y archivos mutables del profile. |
| `environment` | Sí | Variables, PATH y comandos publicados. |
| `requires` | Opcional | Dependencias informativas de componentes. |

`server` es una clasificación informativa. EAP lo instala, activa y actualiza
como un runtime; sólo tendrá una acción Run si el manifiesto declara launchers.
La gestión de instancias, puertos o procesos debe permanecer en un launcher o
una Pocketool explícitos.

### 5.2 Información y rutas importantes

`info` explica qué aporta el componente y dónde guarda sus datos relevantes.
EAP muestra este bloque antes de instalar y desde el acceso `[Ni]` de la
pantalla principal.

```json
"info": {
  "description": "Node.js aísla la configuración, cachés y paquetes globales de npm dentro del profile activo.",
  "paths": [
    {
      "displayName": "Caché npm",
      "base": "profile",
      "relativePath": "home/.npm"
    },
    {
      "displayName": "Configuración npm (.npmrc)",
      "base": "profile",
      "relativePath": "home/.npmrc"
    }
  ]
}
```

Reglas:

- `description` es obligatoria, no puede superar 400 caracteres y debe tener
  como máximo tres frases breves.
- `paths` debe contener al menos una entrada.
- `base` sólo admite `profile` o `workspace`.
- `relativePath` usa `/`, nunca es absoluta y no admite segmentos `..`.
- Use `.` para señalar la raíz de la base elegida.
- EAP concatena cada ruta relativa con el profile de datos o el workspace
  activos y muestra el resultado como ruta absoluta.

### 5.3 Identificadores y versiones

Los IDs de componente, proveedor y launcher deben cumplir:

```text
^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$
```

No use nombres reservados de Windows como `CON`, `NUL`, `COM1` o `LPT1`.
Mantenga el ID para siempre: forma parte de rutas, profiles y lockfiles.

Las versiones resueltas deben comenzar por una letra o número, medir como
máximo 128 caracteres y usar únicamente letras, números, `.`, `_`, `+` y `-`.
EAP ordena normalmente las versiones usando todos sus grupos numéricos.

### 5.4 Líneas o tracks

```json
"tracks": [
  {"id": 1, "displayName": "Producto 1.x"},
  {"id": "2026.2", "displayName": "Producto 2026.2"}
]
```

El ID puede ser entero o texto. Para resolvers automáticos, los grupos numéricos
de la versión deben comenzar por los grupos de la línea:

- línea `4` acepta `4.1.2`;
- línea `2026.2` acepta `2026.2.1`;
- línea `3.14` acepta `3.14.8`.

`same-track` permite buscar actualizaciones dentro de la misma línea. Use
`manual` para componentes vinculados o sin actualización automática.

La comparación normal extrae y compara numéricamente los grupos de la versión.
Las distribuciones Java pueden declarar `"versioning": {"scheme": "java"}`
para aplicar su orden de release y build sin que EAP dependa del ID `java` ni de
un nombre de componente concreto.

Los componentes que puedan avanzar entre versiones mayores pueden declarar:

```json
"majorUpdates": "confirm-component-name"
```

En ese caso cada track debe ser un entero positivo que represente exactamente
la versión mayor (`26`, `27`, etc.). EAP ofrece las actualizaciones patch y
minor de la línea activa de forma normal. Cuando el catálogo publique una
línea mayor nueva, también la ofrece, pero muestra un aviso de compatibilidad y
exige escribir el ID del componente para instalarla. Añada el nuevo track sólo
después de validar la release y sus recomendaciones de migración.

No active esta política para runtimes cuyo major forma parte del contrato del
proyecto, como Java, Go, PHP, Node.js o Python, ni para herramientas que deban
permanecer fijadas a una línea de compatibilidad.

## 6. Proveedores y resolución de versiones

Cada proveedor contiene como mínimo:

```json
{
  "id": "community",
  "componentId": "producto-community",
  "displayName": "Producto Community",
  "resolver": {},
  "verification": {}
}
```

`vendor`, `license` y `homepage` son metadatos recomendados. `componentId`
identifica la distribución concreta y debe ser estable.

### 6.1 Resolver recomendado: GitHub Releases

```json
"resolver": {
  "type": "github-release-asset",
  "apiUrl": "https://api.github.com/repos/owner/repository/releases?per_page=100",
  "assetPattern": "^producto-(?P<version>\\d+\\.\\d+\\.\\d+)-windows-x64\\.zip$"
}
```

Comportamiento:

- `apiUrl` puede devolver una release o una lista.
- Se ignoran drafts y prereleases.
- `assetPattern` es una expresión regular y debe coincidir con el nombre
  completo del ZIP.
- La expresión debe incluir el grupo nombrado `(?P<version>...)`.
- La versión capturada debe pertenecer al track seleccionado.
- La release debe publicar `digest` para el asset, normalmente
  `sha256:<hash>`. Si GitHub no publica digest, este resolver no puede verificar
  el archivo.
- El asset debe ser un `.zip` descargable mediante HTTPS.

### 6.2 Resolvers disponibles

| `resolver.type` | Uso | Campos |
|---|---|---|
| `github-release-asset` | ZIP estable en GitHub Releases | `apiUrl`, `assetPattern` |
| `jetbrains-product-release` | API oficial de productos JetBrains | `apiUrl`, `productCode`, `downloadKey` opcional (`windowsZip`) |
| `eclipse-epp-release` | Paquetes Eclipse EPP | `downloadBaseUrl`, `packageName`, `releaseBuild` opcional (`R`) |
| `adoptium-v3` | JDK de Adoptium | `baseUrl`; opcionales `vendor`, `jvmImpl` |
| `corretto-index` | JDK de Amazon Corretto | `indexUrl`, `resourceBaseUrl` |
| `html-directory` | Directorio HTML con ZIP y checksum separado | `indexUrl`, `releasePattern`, plantillas de artefacto y checksum |
| `apache-directory` | Resolver antiguo de Maven; sólo compatibilidad | `indexUrl`, `downloadBaseUrl` |
| `nodejs-index` | Índice oficial de Node.js | `indexUrl`, `downloadBaseUrl` |
| `json-index` | API JSON declarativa con selección de release, artefacto y checksum | `indexUrl`, `releases`, `artifacts` |
| `python-install-manager-index` | Índice Windows de Python Install Manager | `indexUrl`; opcionales `company`, `architectureTag` |
| `vscode-update-api` | API de actualización de VS Code | `updateUrl` |
| `external-executable` | Programa ya instalado en el host | Sin campos adicionales; sólo para `kind: external` |

Los resolvers específicos están ligados a la estructura concreta indicada. Use
`json-index` para una API JSON y `html-directory` para un índice HTML con
checksum separado; copie otro bloque sólo si el producto usa exactamente la
misma API.

### 6.3 Resolver JSON declarativo

`json-index` permite integrar una API JSON nueva sin modificar EAP ni ejecutar
código procedente del catálogo. Sólo admite selección de datos, filtros escalares,
una expresión regular para normalizar la versión y plantillas HTTPS limitadas.

```json
"resolver": {
  "type": "json-index",
  "indexUrl": "https://example.org/releases.json?track={track}",
  "releases": {
    "path": "/releases",
    "versionPath": "/name",
    "versionPattern": "^v(?P<version>\\d+\\.\\d+\\.\\d+)$",
    "filters": {
      "/stable": true
    }
  },
  "artifacts": {
    "path": "/files",
    "filters": {
      "/os": "windows",
      "/arch": "x64",
      "/type": "archive"
    },
    "fileNamePath": "/name",
    "urlPath": "/url",
    "sha256Path": "/sha256",
    "sizePath": "/size"
  }
}
```

Las rutas usan un subconjunto seguro de JSON Pointer:

- `/` representa el documento u objeto actual.
- Cada segmento navega por una propiedad; `~1` representa `/` y `~0`, `~`.
- `*` permite seleccionar elementos o claves coincidentes. Si quedan varios
  artefactos, `selection` debe ser `first` o `last`; el valor predeterminado
  `only` rechaza una selección ambigua.
- `{track}` está disponible en `indexUrl`, rutas y filtros. Las plantillas de URL
  también admiten `{version}` y `{fileName}`.
- `versionPattern`, cuando se usa, debe incluir `(?P<version>...)`.
- `artifacts` debe declarar exactamente un origen de URL (`urlPath` o
  `urlTemplate`) y un checksum (`sha256Path` o `sha512Path`). El resultado debe
  ser un ZIP HTTPS con un checksum válido.

Los antiguos tipos específicos de Go y PHP siguen siendo aceptados por EAP para
compatibilidad con revisiones ya publicadas, pero los componentes nuevos deben
usar `json-index`.

Esto permite que un repositorio externo publique componentes como Go o PHP sin
un cambio previo en el código de EAP. La frontera sigue siendo deliberadamente
segura: la fuente debe ser una API JSON HTTPS, el artefacto un ZIP y el checksum
debe estar disponible en esa respuesta.

### 6.4 Directorio HTML declarativo

`html-directory` cubre repositorios que publican versiones mediante enlaces
HTML y el checksum en un documento independiente:

```json
"resolver": {
  "type": "html-directory",
  "indexUrl": "https://downloads.example.org/product-{track}/",
  "releasePattern": "href=[\"']v?(?P<version>\\d+\\.\\d+\\.\\d+)/[\"']",
  "artifactUrlTemplate": "https://downloads.example.org/product-{track}/v{version}/bin/product-{version}.zip",
  "checksumUrlTemplate": "{artifactUrl}.sha512",
  "checksumAlgorithm": "sha512"
}
```

- `releasePattern` debe capturar `(?P<version>...)`.
- Las plantillas sólo aceptan HTTPS y los tokens documentados.
- `checksumAlgorithm` admite `sha256` o `sha512`.
- El artefacto final debe ser un ZIP y pertenecer al track solicitado.
- `{track}` y `{version}` están disponibles en las URL; la plantilla del
  checksum también admite `{artifactUrl}` y `{fileName}`.

Maven y Tomcat usan este contrato. `apache-directory` permanece disponible
para no romper manifiestos de Maven publicados con versiones anteriores.

Formatos distintos, instaladores, autenticación especial o páginas que no
expongan una versión estable todavía necesitan ampliar un resolver genérico o
incorporar un adaptador auditado en EAP.

`verification` es obligatorio y documenta el origen de la verificación. Los
resolvers obtienen el checksum desde su fuente oficial. `java-release` también
puede usar:

```json
"verification": {
  "implementorContains": "Eclipse Adoptium"
}
```

## 7. Instalación

### 7.1 Componente ZIP administrado

```json
"install": {
  "directoryTemplate": "producto/{provider}/{version}",
  "stripSingleRoot": true,
  "maxExtractBytes": 2147483648,
  "requiredFiles": [
    "bin/producto.exe"
  ],
  "validation": {
    "type": "files-only"
  }
}
```

| Campo | Contrato |
|---|---|
| `directoryTemplate` | Ruta relativa bajo `components`; admite `{provider}` y `{version}`. |
| `stripSingleRoot` | Opcional. Si es `true`, el ZIP debe contener exactamente una carpeta raíz y EAP instala su contenido. |
| `maxExtractBytes` | Opcional. Máximo total descomprimido, en bytes. Debe ser positivo. |
| `requiredFiles` | Lista de archivos que deben existir tras extraer. Use `/` como separador. |
| `validation` | Estrategia adicional de validación. |

La extracción rechaza rutas absolutas, `..`, rutas duplicadas, nombres de
dispositivo Windows, enlaces simbólicos, exceso de tamaño y ratios de compresión
peligrosos.

### 7.2 Validaciones disponibles

#### `files-only`

Comprueba únicamente `requiredFiles`. Es la opción recomendada para aplicaciones
gráficas, porque no las inicia durante la instalación.

```json
"validation": {"type": "files-only"}
```

#### `command`

Ejecuta un smoke test dentro del payload:

```json
"validation": {
  "type": "command",
  "command": ["bin/producto.exe", "--version"],
  "expectContains": "Producto",
  "timeoutSeconds": 30
}
```

- El primer elemento debe ser una ruta relativa a un archivo del payload.
- `.cmd` y `.bat` se ejecutan mediante `cmd.exe`.
- El directorio actual es la raíz extraída.
- Código de salida distinto de cero, timeout o ausencia de `expectContains`
  provocan el rechazo de la instalación.
- Este tipo ejecuta código descargado automáticamente. Úselo sólo con
  distribuciones confiables y comandos no interactivos.

#### `java-release`

Es específico de un JDK. Comprueba el archivo `release`, el track,
`IMPLEMENTOR` y ejecuta `bin/java.exe -version`.

#### `eclipse-package`

Es específico de Eclipse. Comprueba que `eclipse.ini` declare mediante `-vm` un
JRE incluido y confinado dentro del payload.

## 8. Launchers

Un launcher publica una aplicación o comando arrancable desde EAP:

```json
{
  "id": "producto",
  "displayName": "Producto",
  "type": "application",
  "workspaceMode": "environment",
  "executable": "{{component.root}}/producto.exe",
  "arguments": ["{{workspace.selected}}"],
  "environment": {},
  "unset": [],
  "dataDirectories": [],
  "dataCopies": [],
  "startMode": "detached"
}
```

| Campo | Valores y comportamiento |
|---|---|
| `type` | `application` o `command`. |
| `workspaceMode` | `environment` usa el proyecto del profile; `component-data` usa un workspace privado del componente. |
| `executable` | Debe quedar dentro del payload. En un componente externo debe ser exactamente `{{external.executable}}`. |
| `arguments` | Lista de argumentos con plantillas. |
| `startMode` | `detached` devuelve el control; `wait` espera y devuelve el código de salida. |
| `environment` | Variables adicionales sólo para el launcher. |
| `unset` | Variables heredadas que se eliminan para el launcher. |
| `dataDirectories` | Directorios que EAP crea; deben quedar dentro de `{{data.component}}`. |
| `dataCopies` | Copias iniciales de directorios del payload hacia datos mutables. |

Ejemplo de copia mutable:

```json
"dataCopies": [
  {
    "source": "{{component.root}}/configuration",
    "target": "{{data.component}}/configuration",
    "mode": "if-missing"
  }
]
```

`source` debe ser un directorio del payload, `target` debe quedar bajo los datos
del componente y el único modo admitido es `if-missing`. EAP nunca sobrescribe
el destino existente.

Los launchers no pueden redefinir ni eliminar las variables que forman la
identidad portable del profile, como `HOME`, `USERPROFILE`, `APPDATA`,
`LOCALAPPDATA`, temporales, XDG o `JAVA_TOOL_OPTIONS`.

## 9. Entorno publicado

```json
"environment": {
  "variables": {
    "PRODUCTO_HOME": "{{component.root}}",
    "PRODUCTO_OPTIONS": "--profile={{profile.id}}"
  },
  "appendable": ["PRODUCTO_OPTIONS"],
  "unset": [],
  "path": [
    "{{component.root}}/bin"
  ],
  "dataPath": [],
  "commands": []
}
```

- `variables` y `path` son obligatorios, aunque estén vacíos.
- `appendable` permite que un valor `env.NOMBRE` del usuario se anteponga,
  separado por un espacio, al valor gestionado por el componente. La parte de
  EAP queda al final y tiene precedencia cuando una opción se repite.
- Cada entrada de `path` debe resolverse dentro de la instalación.
- `dataPath` crea rutas mutables dentro del profile y las añade a `PATH`.
- `unset` elimina primero cualquier variante mayúscula/minúscula de la variable.
- No publique rutas de `core`.

Un comando de entorno crea un wrapper `.cmd` que llama a un ejecutable del
payload y reenvía los argumentos del usuario:

```json
"commands": [
  {
    "name": "producto",
    "executable": "{{component.root}}/bin/producto.exe",
    "arguments": ["--portable"]
  }
]
```

El nombre sigue las reglas de ID. Los argumentos declarados son literales y
sólo admiten letras, números, `.`, `_`, `+` y `-`; no pueden contener espacios.

## 10. Tokens de plantilla

Los tokens desconocidos provocan un error. No todos están disponibles en todos
los bloques.

### Entorno (`environment.*`)

- `{{component.root}}`
- `{{component.provider}}`
- `{{external.executable}}`
- `{{profile.root}}`, `{{profile.home}}`, `{{profile.temp}}`
- `{{data.component}}`
- `{{workspace.root}}`, `{{workspace.selected}}`
- `{{eap.root}}`
- `{{profile.id}}`, `{{environment.id}}`

### Launchers

Todos los anteriores y además:

- `{{component.version}}`
- `{{data.component.uri}}`, útil para argumentos que requieren `file:///...`

### `data.directories`, `data.files` y contenido de archivos

Todos los tokens del entorno y además:

- `{{component.version}}`
- `{{data.component.posix}}`, ruta con separadores `/`

`directoryTemplate` no usa estos tokens: sólo `{provider}` y `{version}`.

## 11. Datos mutables

Los payloads de `components/` se consideran inmutables. Configuración, caché,
plugins y workspaces deben ir al profile mediante `data` o mediante opciones del
launcher.

```json
"data": {
  "directories": [
    {
      "path": "{{data.component}}/config",
      "displayName": "Configuración",
      "role": "configuration",
      "showInDashboard": true
    }
  ],
  "files": [
    {
      "path": "{{data.component}}/producto.properties",
      "displayName": "Configuración portable",
      "role": "configuration",
      "showInDashboard": false,
      "mode": "if-missing",
      "content": "home={{data.component.posix}}/config\n"
    }
  ]
}
```

Roles admitidos:

- `cache`
- `commands`
- `configuration`
- `data`
- `extensions`
- `repository`
- `workspace`

Todas las rutas deben quedar dentro del profile. Los archivos sólo admiten
`mode: if-missing`, su contenido máximo es 1 MiB y nunca se sobrescriben después
de que el usuario o la aplicación los modifiquen.

## 12. Capacidades y dependencias

Una capacidad permite que una Pocketool exija un componente activo:

```json
"capability": {
  "id": "runtime.producto",
  "exclusive": true
}
```

Use nombres estables y cualificados, por ejemplo `runtime.java` o
`app.api-client`.

Los componentes también pueden declarar requisitos informativos:

```json
"requires": [
  {
    "capability": "runtime.java",
    "minimumTrack": 17
  }
]
```

Actualmente EAP no instala ni bloquea automáticamente componentes por estos
requisitos. Sirven como metadatos. `minimumTrack` debe ser entero en un
manifiesto de componente.

## 13. Componentes externos

Un componente externo representa una aplicación que EAP no puede redistribuir
o instalar:

```json
{
  "schemaVersion": 3,
  "id": "producto-externo",
  "displayName": "Producto Externo",
  "kind": "external",
  "launchers": [
    {
      "id": "producto-externo",
      "displayName": "Producto Externo",
      "type": "application",
      "workspaceMode": "environment",
      "executable": "{{external.executable}}",
      "arguments": ["{{workspace.selected}}"],
      "startMode": "detached"
    }
  ],
  "tracks": [
    {"id": "local", "displayName": "Instalación local"}
  ],
  "defaultProvider": "external",
  "defaultTrack": "local",
  "updatePolicy": "manual",
  "providers": [
    {
      "id": "external",
      "componentId": "producto-externo-local",
      "displayName": "Instalación externa",
      "resolver": {"type": "external-executable"},
      "verification": {"type": "local-executable"}
    }
  ],
  "install": {
    "type": "external-executable",
    "executableNames": ["producto.exe"],
    "prompt": "Ruta completa a producto.exe"
  },
  "data": {"directories": [], "files": []},
  "environment": {"variables": {}, "path": []}
}
```

Debe tener al menos un launcher. Sus nombres permitidos deben ser archivos `.exe`
simples, sin rutas. El usuario lo vincula mediante:

```bat
eap.cmd component install producto-externo --executable "C:\Program Files\Producto\producto.exe" --profile pruebas --yes
```

## 14. Desarrollo y prueba de extremo a extremo

### Paso 1: elegir el ejemplo más próximo

- Aplicación ZIP en GitHub: `components/bruno.json` o `vscodium.json`.
- Aplicación con estado portable complejo: `dbeaver.json` o
  `intellij-idea.json`.
- CLI con smoke test: `git.json`, `maven.json` o `nodejs.json`.
- Aplicación ya instalada: `kiro.json`.

No copie un resolver específico de Java, Eclipse, JetBrains, Node o Python para
un producto que no use exactamente esa fuente.

### Paso 2: validar el JSON localmente

PowerShell puede detectar errores de sintaxis:

```powershell
Get-Content .\catalog.json -Raw | ConvertFrom-Json | Out-Null
Get-Content .\components\mi-componente.json -Raw | ConvertFrom-Json | Out-Null
```

Esto sólo valida JSON. La validación autoritativa la realiza EAP al refrescar el
repositorio.

### Paso 3: publicar en una rama `main` de prueba

EAP consulta actualmente la rama `main`. Haga commit y push del catálogo a un
repositorio GitHub de prueba. Para un repositorio que sólo contenga componentes
nuevos y con IDs únicos:

```bat
eap.cmd component repository add pruebas https://github.com/usuario/eap-components-pruebas --yes
eap.cmd component refresh pruebas
eap.cmd catalog
```

Si prueba un fork completo del catálogo oficial, use temporalmente el mismo ID
de fuente para evitar colisiones con `danielgube`:

```bat
eap.cmd component repository add danielgube https://github.com/usuario/eap-components --yes
eap.cmd component refresh danielgube
```

Al terminar, restaure la URL oficial:

```bat
eap.cmd component repository add danielgube https://github.com/danielgube/eap-components --yes
eap.cmd component refresh danielgube
```

### Paso 4: probar en un profile desechable

```bat
eap.cmd profile create pruebas-componente
eap.cmd component resolve mi-componente --provider community --track 2 --json
eap.cmd component install mi-componente --provider community --track 2 --profile pruebas-componente --yes
eap.cmd component list --profile pruebas-componente
eap.cmd doctor
```

Si tiene launcher:

```bat
eap.cmd launch mi-componente --profile pruebas-componente --dry-run
eap.cmd launch mi-componente --profile pruebas-componente
```

Compruebe:

- que la versión seleccionada es la esperada;
- que el checksum procede de la fuente oficial;
- que `requiredFiles` describe realmente el ZIP;
- que variables y PATH apuntan al payload;
- que configuración, caché y workspace no se escriben en el payload;
- que una actualización dentro del mismo track funciona;
- que desactivar y reactivar no requiere red;
- que `doctor` no informa de errores.

## 15. Publicación y actualización

Para publicar un componente nuevo:

1. Añada `components/<id>.json`.
2. Añada su entrada a `catalog.json`.
3. Incremente `catalogVersion`.
4. Valide y pruebe desde EAP.
5. Haga commit y push a `main`.

Para actualizar uno existente:

- no cambie su ID ni los IDs de proveedor sin una migración;
- añada un track cuando aparezca una nueva línea compatible;
- para `majorUpdates`, publique el track mayor sólo después de revisar la
  release y sus posibles incompatibilidades;
- cambie `defaultTrack` sólo después de probarla;
- no fije una versión concreta si el resolver ya obtiene la última corrección;
- aumente `maxExtractBytes` únicamente cuando el tamaño real lo justifique;
- conserve las rutas de datos existentes para no perder preferencias.

EAP fija en el lock el repositorio, commit, ruta y hash del manifiesto usado.

## 16. Errores frecuentes

### `Resolver no soportado`

El valor de `resolver.type` no existe en EAP. Use uno de la tabla o solicite un
nuevo adapter/resolver.

### `no captura 'version'`

`assetPattern` no contiene el grupo nombrado `(?P<version>...)`.

### `no publicó el checksum`

El asset de GitHub no tiene `digest`, o la fuente no publica SHA-256/SHA-512 en
el formato esperado. No omita la verificación.

### `no pertenece a la línea`

La versión capturada no empieza por los grupos numéricos del track.

### `Falta el archivo requerido`

Revise la estructura real del ZIP y `stripSingleRoot`.

### `sale del payload` o `sale de su perfil`

Una ruta resuelta intenta escapar de su frontera. Payloads, datos y workspaces
deben mantenerse separados.

### `publicado por dos repositorios`

Dos fuentes externas tienen el mismo ID. Quite una, cambie el ID del componente
nuevo o sustituya temporalmente la URL de la misma fuente durante las pruebas.

## 17. Checklist antes de una pull request

- [ ] El ID coincide en carpeta, archivo, manifiesto y catálogo.
- [ ] `schemaVersion` es `3`.
- [ ] `catalogVersion` se ha incrementado.
- [ ] Proveedor y track predeterminados existen.
- [ ] El resolver es compatible y todas las URLs son HTTPS.
- [ ] La versión resuelta pertenece al track.
- [ ] El checksum procede de una fuente oficial.
- [ ] El ZIP es Windows x64 y no necesita instalador.
- [ ] `stripSingleRoot` coincide con la forma real del ZIP.
- [ ] `requiredFiles` incluye ejecutables y archivos estructurales importantes.
- [ ] La validación no abre accidentalmente una GUI.
- [ ] Variables, PATH y ejecutables permanecen dentro del payload.
- [ ] Configuración, caché, plugins y workspaces permanecen en el profile.
- [ ] Se ha probado `resolve`, `install`, `launch --dry-run` y `doctor`.
- [ ] No se han incluido secretos, tokens ni credenciales en el manifiesto.

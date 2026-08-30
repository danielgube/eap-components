# Manual para crear componentes EAP

Esta guía permite añadir un componente sin conocer el código fuente de EAP.
Describe el contrato público de los manifiestos de componentes para el esquema
`schemaVersion: 1`, sus límites actuales y el proceso completo de prueba y
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
  "schemaVersion": 1,
  "id": "mi-cliente",
  "displayName": "Mi Cliente",
  "description": "Cliente portable de ejemplo",
  "category": "applications",
  "kind": "application",
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
| `schemaVersion` | Sí | Debe valer `1`. |
| `id` | Sí | Identificador estable; debe coincidir con catálogo y archivo. |
| `displayName` | Sí | Nombre visible para el usuario. |
| `description` | Recomendado | Descripción breve. |
| `category` | Opcional | Agrupación informativa, por ejemplo `applications`. |
| `kind` | Sí | `application`, `external`, `runtime`, `service` o `tool`. |
| `launchers` | Sí | Lista; puede estar vacía salvo para `external`. |
| `capability` | Recomendado | Capacidad que otras herramientas pueden requerir. |
| `platform` | Recomendado | Metadatos de SO, arquitectura y archivo. |
| `tracks` | Sí | Una o más líneas instalables. |
| `defaultProvider` | Sí | ID de un proveedor declarado. |
| `defaultTrack` | Sí | ID de una línea declarada. |
| `updatePolicy` | Sí | `same-track` o `manual`. |
| `providers` | Sí | Uno o más proveedores. |
| `install` | Sí | Contrato de instalación o vinculación. |
| `data` | Opcional | Directorios y archivos mutables del profile. |
| `environment` | Sí | Variables, PATH y comandos publicados. |
| `requires` | Opcional | Dependencias informativas de componentes. |

### 5.2 Identificadores y versiones

Los IDs de componente, proveedor y launcher deben cumplir:

```text
^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$
```

No use nombres reservados de Windows como `CON`, `NUL`, `COM1` o `LPT1`.
Mantenga el ID para siempre: forma parte de rutas, profiles y lockfiles.

Las versiones resueltas deben comenzar por una letra o número, medir como
máximo 128 caracteres y usar únicamente letras, números, `.`, `_`, `+` y `-`.
EAP ordena normalmente las versiones usando todos sus grupos numéricos.

### 5.3 Líneas o tracks

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
| `apache-directory` | Estructura de Maven en Apache Downloads | `indexUrl`, `downloadBaseUrl` |
| `nodejs-index` | Índice oficial de Node.js | `indexUrl`, `downloadBaseUrl` |
| `python-install-manager-index` | Índice Windows de Python Install Manager | `indexUrl`; opcionales `company`, `architectureTag` |
| `vscode-update-api` | API de actualización de VS Code | `updateUrl` |
| `external-executable` | Programa ya instalado en el host | Sin campos adicionales; sólo para `kind: external` |

Los resolvers salvo GitHub están ligados a la estructura concreta indicada. No
son plantillas HTTP genéricas. Copie el bloque de un componente existente sólo
si el nuevo producto usa exactamente la misma API.

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
    "PRODUCTO_HOME": "{{component.root}}"
  },
  "unset": [],
  "path": [
    "{{component.root}}/bin"
  ],
  "dataPath": [],
  "commands": []
}
```

- `variables` y `path` son obligatorios, aunque estén vacíos.
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
  "schemaVersion": 1,
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
- [ ] `schemaVersion` es `1`.
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

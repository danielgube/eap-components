# Crear componentes EAP — guía operativa

Esta guía es una especificación de trabajo para personas y agentes de IA. Su
objetivo es añadir o actualizar un componente con el mínimo de interpretación.

## Resultado obligatorio

Un componente nuevo no está publicado hasta que existen **los dos cambios**:

```text
components/<id>.json                 # definición del componente
catalog.json                         # entrada que apunta a la definición
```

EAP sólo descarga los manifiestos enumerados en `catalog.json`. Crear
`components/<id>.json` sin registrarlo en el catálogo no produce un error: el
archivo simplemente no existe para EAP y no aparecerá como instalable.

La entrada mínima es:

```json
{
  "id": "mi-componente",
  "manifest": "components/mi-componente.json"
}
```

El mismo ID debe aparecer, con idéntica capitalización, en:

1. `catalog.json` → `components[].id`;
2. el nombre `components/<id>.json`;
3. el campo raíz `id` del manifiesto.

Al cambiar cualquier manifiesto, incremente también `catalogVersion` y publique
los cambios en `main`.

## Restricciones que se comprueban antes de empezar

Un componente administrado es viable sólo si el fabricante ofrece:

- un ZIP para Windows con todos los archivos necesarios;
- descarga por HTTP o HTTPS; use HTTPS siempre que esté disponible;
- SHA-256/SHA-512 oficial o, como último recurso, una página HTML que enlace el
  ZIP y pueda resolverse con `html-links`;
- una fuente de versiones compatible con uno de los resolvers de esta guía.

EAP no instala MSI o EXE, no ejecuta scripts de instalación, no crea enlaces
simbólicos y no escribe fuera de sus directorios administrados. Si sólo se debe
usar un `.exe` ya instalado, cree un componente `external`.

No invente un `resolver.type` ni omita `verification`. Sólo `html-links` admite
`"verification": {"type": "none"}`. No coloque configuración, cachés o
workspaces dentro del payload instalado.

## Flujo de trabajo para un agente

Siga este orden. No dé el componente por terminado antes del paso 7.

1. Busque el manifiesto existente más parecido por distribución y resolver; no
   se guíe sólo por la categoría del producto.
2. Consulte la fuente oficial y confirme versión estable, nombre del ZIP, URL,
   licencia y checksum, si existe.
3. Inspeccione el ZIP real. Determine si contiene una única carpeta raíz y
   anote ejecutables y archivos estructurales que irán en `requiredFiles`.
4. Cree o actualice `components/<id>.json` con `schemaVersion: 3`.
5. Registre el componente en `catalog.json` e incremente `catalogVersion`.
6. Valide la sintaxis de ambos JSON y revise la checklist final.
7. Publique en una rama `main`, refresque la fuente en EAP y pruebe `resolve`,
   `install`, `launch --dry-run` cuando corresponda y `doctor`.

Si la fuente o el artefacto no encajan en un resolver soportado, deténgase y
comunique esa limitación. La ausencia de checksum no bloquea `html-links`, pero
sí debe declararse con `verification.type: "none"`.

## Plantilla: componente ZIP administrado

Use esta base para herramientas, runtimes, servidores y aplicaciones portables
publicadas en GitHub Releases. Quite los bloques opcionales que no necesite.

```json
{
  "schemaVersion": 3,
  "id": "mi-componente",
  "displayName": "Mi Componente",
  "description": "Descripción breve",
  "category": "tools",
  "kind": "tool",
  "info": {
    "description": "Explica en una frase qué aporta y dónde conserva su estado.",
    "paths": [
      {
        "displayName": "Configuración",
        "base": "profile",
        "relativePath": "components/mi-componente/config",
        "type": "directory"
      }
    ]
  },
  "launchers": [],
  "capability": {
    "id": "tool.mi-componente",
    "exclusive": true
  },
  "platform": {
    "os": "windows",
    "architecture": "x64",
    "archiveType": "zip"
  },
  "tracks": [
    {
      "id": 1,
      "displayName": "Mi Componente 1.x"
    }
  ],
  "defaultProvider": "official",
  "defaultTrack": 1,
  "updatePolicy": "same-track",
  "providers": [
    {
      "id": "official",
      "componentId": "mi-componente-official",
      "displayName": "Distribución oficial",
      "vendor": "Fabricante",
      "license": "SPDX-o-Proprietaria",
      "homepage": "https://example.org/",
      "resolver": {
        "type": "github-release-asset",
        "apiUrl": "https://api.github.com/repos/owner/repository/releases?per_page=100",
        "assetPattern": "^producto-(?P<version>\\d+\\.\\d+\\.\\d+)-windows-x64\\.zip$"
      },
      "verification": {
        "checksumAlgorithm": "sha256",
        "source": "GitHub release asset digest"
      }
    }
  ],
  "install": {
    "directoryTemplate": "mi-componente/{provider}/{version}",
    "stripSingleRoot": false,
    "maxExtractBytes": 1073741824,
    "requiredFiles": [
      "bin/mi-componente.exe"
    ],
    "validation": {
      "type": "files-only"
    }
  },
  "data": {
    "directories": [
      {
        "path": "{{data.component}}/config",
        "displayName": "Configuración",
        "role": "configuration",
        "showInDashboard": true
      }
    ],
    "files": []
  },
  "environment": {
    "variables": {
      "MI_COMPONENTE_HOME": "{{component.root}}"
    },
    "path": [
      "{{component.root}}/bin"
    ]
  }
}
```

Referencias recomendadas del repositorio:

- CLI con smoke test: `git.json`, `maven.json`, `nodejs.json`;
- aplicación ZIP: `bruno.json`, `vscodium.json`;
- estado portable complejo: `dbeaver.json`, `intellij-idea.json`;
- aplicación instalada externamente: `kiro.json`.

## Contrato mínimo del manifiesto

Campos raíz obligatorios para componentes nuevos:

| Campo | Regla |
|---|---|
| `schemaVersion` | Use `3`. |
| `id` | ID estable; coincide con archivo y catálogo. |
| `displayName` | Nombre mostrado al usuario. |
| `kind` | `application`, `external`, `runtime`, `server`, `service` o `tool`. |
| `info` | Descripción y al menos una ruta importante. |
| `launchers` | Lista; puede ser `[]`, salvo en componentes `external`. |
| `tracks` | Lista no vacía de `{id, displayName}`. |
| `providers` | Lista no vacía de proveedores. |
| `defaultProvider` | ID de un proveedor declarado. |
| `defaultTrack` | ID de un track declarado. |
| `updatePolicy` | `same-track` o `manual`. |
| `install` | Instalación ZIP o vinculación externa. |
| `environment` | Debe incluir `variables` y `path`, aunque estén vacíos. |

Campos habituales opcionales: `description`, `category`, `capability`,
`platform`, `data`, `requires`, `versioning` y `majorUpdates`.

Los IDs de componente, proveedor, launcher y comando deben cumplir:

```text
^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$
```

No use nombres reservados de Windows (`CON`, `NUL`, `COM1`, `LPT1`, etc.). No
cambie un ID publicado: forma parte de rutas, profiles y lockfiles.

### `info`

```json
"info": {
  "description": "Entre una y tres frases breves; máximo 400 caracteres.",
  "paths": [
    {
      "displayName": "Caché",
      "base": "profile",
      "relativePath": "components/producto/cache",
      "type": "directory"
    }
  ]
}
```

- `base`: `profile` o `workspace`.
- `relativePath`: relativa, con `/`, sin `..`; use `.` para la raíz.
- `type`: obligatorio en schema 3; `directory` o `file`.

### Tracks y actualizaciones

El ID puede ser entero o texto. La versión resuelta debe comenzar por los
grupos numéricos del track: el track `3` acepta `3.2.1`; `2026.2` acepta
`2026.2.4`.

- `same-track`: busca actualizaciones dentro del track activo.
- `manual`: no busca actualizaciones automáticas; úselo para `external`.
- `versioning.scheme`: `numeric` por defecto o `java` para distribuciones Java.
- `majorUpdates: "confirm-component-name"`: permite ofrecer un track mayor con
  confirmación reforzada. Sólo con `same-track`, no con `external`, y todos los
  tracks deben ser enteros positivos que representen el major.

No active actualizaciones mayores para runtimes cuyo major sea parte del
contrato del proyecto, como Java, Node.js, Go, PHP o Python.

## Elegir resolver

Para un componente nuevo, prefiera uno de estos:

| Resolver | Cuándo usarlo | Requisito decisivo |
|---|---|---|
| `github-release-asset` | GitHub Releases | El asset ZIP tiene `digest` SHA-256 en la API. |
| `json-index` | API JSON genérica | La respuesta expone versión, ZIP HTTPS y SHA-256/SHA-512. |
| `html-directory` | Índice HTML | Hay patrón de versión, URL de ZIP y checksum separado. |
| `html-links` | Web de releases, GitLab, Gitea o autoindex de Apache | Un enlace identifica el ZIP y contiene la versión; el checksum puede no existir. |
| `external-executable` | Programa ya instalado | Sólo para `kind: external`. |

EAP también conserva resolvers específicos para catálogos existentes:
`adoptium-v3`, `corretto-index`, `apache-directory`, `dbeaver-download-page`,
`nodejs-index`, `golang-downloads-index`, `php-windows-releases`,
`python-install-manager-index`, `vscode-update-api`,
`jetbrains-product-release` y `eclipse-epp-release`. No los reutilice para otro
producto salvo que consuma exactamente la misma API.

`github-release` **no** es un resolver soportado. Para GitHub use
`github-release-asset` si el asset publica digest. Si no lo publica, use
`html-links` sobre la página de releases cuando el ZIP aparezca como enlace.

### GitHub Releases

```json
"resolver": {
  "type": "github-release-asset",
  "apiUrl": "https://api.github.com/repos/owner/repository/releases?per_page=100",
  "assetPattern": "^producto-(?P<version>\\d+\\.\\d+\\.\\d+)-windows-x64\\.zip$"
}
```

- Se ignoran drafts y prereleases.
- `assetPattern` debe coincidir con el nombre completo del ZIP.
- Debe contener el grupo nombrado `(?P<version>...)`.
- La versión capturada debe pertenecer al track.
- La API de GitHub debe devolver `digest: sha256:<hash>` para el asset.

### Enlaces HTML genéricos sin checksum

Use `html-links` cuando una página web enlaza los ZIP pero no ofrece una API o
un checksum fácil de consumir. Cubre páginas de releases renderizadas en el
servidor, GitLab, Gitea y listados de directorio de Apache (`autoindex`) por
HTTP o HTTPS.

Ejemplo real para Windows Terminal estable x64:

```json
{
  "resolver": {
    "type": "html-links",
    "indexUrl": "https://github.com/microsoft/terminal/releases/",
    "linkPattern": "^Microsoft\\.WindowsTerminal.*?_(?P<version>\\d+(?:\\.\\d+){3})_x64\\.zip$",
    "excludePatterns": [
      "Preview"
    ]
  },
  "verification": {
    "type": "none"
  }
}
```

Contrato exacto:

- `indexUrl` es una URL absoluta HTTP o HTTPS. Sólo admite el token opcional
  `{track}`.
- `linkPattern` es una expresión regular y debe contener
  `(?P<version>...)`. Se aplica como coincidencia completa, primero al texto
  visible normalizado del enlace y después al nombre de archivo decodificado
  de su URL.
- `excludePatterns` es opcional. Si cualquiera aparece en el texto, nombre de
  archivo o URL absoluta, ese enlace se descarta antes de aplicar
  `linkPattern`.
- La comparación no depende del orden del HTML: EAP extrae todas las versiones,
  conserva sólo las que pertenecen al track solicitado y elige la mayor según
  `versioning`. Si dos enlaces publican la misma versión, conserva el primero.
- Las expresiones no distinguen mayúsculas de forma predeterminada. Añada
  `"caseSensitive": true` si deben distinguirlas.
- Se admiten enlaces relativos, absolutos y `<base href>`. EAP sigue
  automáticamente `<include-fragment src>`; esto permite resolver las páginas
  de GitHub que cargan sus assets como fragmentos.
- Para páginas que enlazan primero a un detalle de release, añada
  `followPattern`: se busca en la URL absoluta de cada enlace. `maxDepth` es
  opcional, vale `1` por defecto y sólo admite valores de `1` a `3`. No declare
  `maxDepth` sin `followPattern`.
- Se consultan como máximo 32 páginas y 5 MiB por página. Un fragmento o página
  secundaria inaccesible se ignora; la `indexUrl` inicial sí es obligatoria.
- El enlace seleccionado debe terminar en `.zip` y producir un nombre de
  archivo seguro.

Ejemplo de navegación a páginas de detalle:

```json
{
  "resolver": {
    "type": "html-links",
    "indexUrl": "http://intranet.example.org/producto/releases/",
    "followPattern": "/producto/releases/v\\d+\\.\\d+\\.\\d+$",
    "maxDepth": 1,
    "linkPattern": "^producto-(?P<version>\\d+\\.\\d+\\.\\d+)-windows-x64\\.zip$",
    "excludePatterns": [
      "preview",
      "arm64"
    ]
  },
  "verification": {
    "type": "none"
  }
}
```

`html-links` no ejecuta JavaScript, no inicia sesión y no descubre paginación
por sí solo. La página o sus fragmentos deben contener enlaces HTML reales.

Este resolver no verifica autenticidad contra un checksum publicado. Tras la
descarga, EAP calcula un SHA-256 local y lo guarda en la instalación y el lock
para reutilización y reproducibilidad. Ese hash describe lo descargado; no
demuestra que el servidor haya entregado el archivo legítimo. En HTTP tampoco
protege la descarga en tránsito.

### API JSON genérica

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
      "/arch": "x64"
    },
    "fileNamePath": "/name",
    "urlPath": "/url",
    "sha256Path": "/sha256",
    "sizePath": "/size"
  }
}
```

Las rutas usan un subconjunto de JSON Pointer; `*` selecciona elementos. Si
quedan varios artefactos, defina `selection` como `first` o `last`; el valor
predeterminado `only` exige uno. `artifacts` declara exactamente un origen de
URL (`urlPath` o `urlTemplate`) y un checksum (`sha256Path` o `sha512Path`).

Tokens permitidos: `{track}` en índice, rutas y filtros; además `{version}` y
`{fileName}` en plantillas de artefacto.

### Índice HTML genérico

```json
"resolver": {
  "type": "html-directory",
  "indexUrl": "https://downloads.example.org/producto-{track}/",
  "releasePattern": "href=[\"']v?(?P<version>\\d+\\.\\d+\\.\\d+)/[\"']",
  "artifactUrlTemplate": "https://downloads.example.org/producto-{track}/v{version}/producto-{version}.zip",
  "checksumUrlTemplate": "{artifactUrl}.sha512",
  "checksumAlgorithm": "sha512"
}
```

`releasePattern` captura `version`; el algoritmo es `sha256` o `sha512`.
`checksumUrlTemplate` admite `{artifactUrl}` y `{fileName}` además de track y
versión.

## Instalación y validación

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

- `directoryTemplate` es relativa a `components` y sólo admite `{provider}` y
  `{version}`.
- `stripSingleRoot: true` exige que el ZIP tenga exactamente una carpeta raíz.
- `requiredFiles` usa `/` y describe la estructura después de extraer y aplicar
  `stripSingleRoot`.
- `maxExtractBytes` es opcional, positivo y debe basarse en el tamaño real.

Validaciones:

- `files-only`: comprueba `requiredFiles`; recomendada para GUI y servidores.
- `command`: ejecuta un smoke test no interactivo del payload.
- `java-release`: validación específica de JDK.
- `eclipse-package`: validación específica de Eclipse.

Ejemplo de `command`:

```json
"validation": {
  "type": "command",
  "command": ["bin/producto.exe", "--version"],
  "expectContains": "Producto",
  "timeoutSeconds": 30
}
```

No use `command` si abre una GUI, inicia un servidor persistente o modifica el
sistema.

## Entorno, datos y launchers

El payload instalado es inmutable. Todo dato modificable se declara bajo
`data` y debe resolverse dentro del profile.

```json
"data": {
  "directories": [
    {
      "path": "{{data.component}}/cache",
      "displayName": "Caché",
      "role": "cache",
      "showInDashboard": false
    }
  ],
  "files": [
    {
      "path": "{{data.component}}/producto.properties",
      "displayName": "Configuración",
      "role": "configuration",
      "showInDashboard": true,
      "mode": "if-missing",
      "content": "home={{data.component.posix}}/config\n"
    }
  ]
}
```

Roles: `cache`, `commands`, `configuration`, `data`, `extensions`, `repository`
y `workspace`. Los modos de archivo admitidos son `if-missing` y
`merge-properties`.

El entorno mínimo es:

```json
"environment": {
  "variables": {},
  "path": []
}
```

Campos opcionales: `appendable`, `unset`, `dataPath` y `commands`.
`environment.path` apunta al payload; `dataPath`, a directorios mutables del
profile. No publique rutas de `core`.

Un launcher típico:

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
  "dataDirectories": ["{{data.component}}/config"],
  "startMode": "detached"
}
```

- `type`: `application` o `command`.
- `workspaceMode`: `environment` o `component-data`.
- `startMode`: `detached` o `wait`.
- `dataCopies` puede copiar un directorio del payload a datos mutables sólo con
  `mode: "if-missing"`.
- No redefina ni elimine variables de identidad del profile (`HOME`,
  `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, temporales, XDG, etc.).

Tokens comunes:

- payload: `{{component.root}}`, `{{component.provider}}`;
- profile: `{{profile.id}}`, `{{profile.root}}`, `{{profile.home}}`,
  `{{profile.temp}}`;
- datos: `{{data.component}}`, `{{data.component.posix}}`,
  `{{data.component.uri}}`;
- workspace: `{{workspace.root}}`, `{{workspace.selected}}`;
- EAP: `{{eap.root}}`, `{{environment.id}}`;
- launcher/datos: `{{component.version}}`;
- sólo externo: `{{external.executable}}`.

No todos los tokens son válidos en todos los bloques. Copie el patrón de un
manifiesto existente y no invente tokens.

## Plantilla: componente externo

```json
{
  "schemaVersion": 3,
  "id": "producto-externo",
  "displayName": "Producto Externo",
  "kind": "external",
  "info": {
    "description": "Aplicación local vinculada a EAP.",
    "paths": [
      {
        "displayName": "Workspace activo",
        "base": "workspace",
        "relativePath": ".",
        "type": "directory"
      }
    ]
  },
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
    {
      "id": "local",
      "displayName": "Instalación local"
    }
  ],
  "defaultProvider": "external",
  "defaultTrack": "local",
  "updatePolicy": "manual",
  "providers": [
    {
      "id": "external",
      "componentId": "producto-externo-local",
      "displayName": "Instalación externa",
      "resolver": {
        "type": "external-executable"
      },
      "verification": {
        "type": "local-executable"
      }
    }
  ],
  "install": {
    "type": "external-executable",
    "executableNames": ["producto.exe"],
    "prompt": "Ruta completa a producto.exe"
  },
  "data": {
    "directories": [],
    "files": []
  },
  "environment": {
    "variables": {},
    "path": []
  }
}
```

Un componente externo exige al menos un launcher. Cada `executableNames` debe
ser un nombre simple terminado en `.exe`, sin ruta.

## Registro y publicación

Edite `catalog.json` en la misma entrega:

```json
{
  "schemaVersion": 1,
  "catalogVersion": "1.7.0",
  "components": [
    {
      "id": "mi-componente",
      "manifest": "components/mi-componente.json"
    }
  ]
}
```

Reglas:

- el catálogo usa `schemaVersion: 1`;
- `manifest` debe ser exactamente `components/<id>.json`;
- no puede haber IDs duplicados;
- catálogo y manifiesto tienen un límite de 1 MiB cada uno;
- dos repositorios externos no pueden publicar el mismo ID;
- EAP consulta la rama `main` de repositorios GitHub;
- si cualquier manifiesto declarado es inválido, el refresh rechaza el snapshot
  completo y conserva la revisión cacheada anterior.

## Validación de extremo a extremo

### 1. Sintaxis local

```powershell
Get-Content .\catalog.json -Raw | ConvertFrom-Json | Out-Null
Get-Content .\components\mi-componente.json -Raw | ConvertFrom-Json | Out-Null
```

Esto sólo valida JSON, no el contrato EAP ni las URLs.

### 2. Fuente publicada

Para un repositorio de prueba con IDs nuevos:

```bat
eap.cmd component repository add pruebas https://github.com/usuario/eap-components-pruebas --yes
eap.cmd component refresh pruebas
eap.cmd catalog
```

Para probar un fork completo del catálogo oficial, sustituya temporalmente la
URL de la misma fuente; añadirlo con otro ID provocaría colisiones:

```bat
eap.cmd component repository add danielgube https://github.com/usuario/eap-components --yes
eap.cmd component refresh danielgube
```

### 3. Resolución e instalación

```bat
eap.cmd profile create pruebas-componente
eap.cmd component resolve mi-componente --provider official --track 1 --json
eap.cmd component install mi-componente --provider official --track 1 --profile pruebas-componente --yes
eap.cmd component list --profile pruebas-componente
eap.cmd doctor
```

Si hay launcher:

```bat
eap.cmd launch mi-componente --profile pruebas-componente --dry-run
```

Confirme la versión, el origen del checksum, la estructura instalada, las rutas
de entorno y que el estado mutable queda fuera del payload. En `html-links`, el
resolve indica que no hay checksum publicado y la instalación registra un
SHA-256 local.

## Definition of Done

- [ ] El ID coincide en catálogo, nombre de archivo y manifiesto.
- [ ] El manifiesto usa `schemaVersion: 3`.
- [ ] `catalog.json` contiene la entrada y `catalogVersion` se incrementó.
- [ ] El resolver figura en esta guía y la fuente oficial responde.
- [ ] La versión resuelta pertenece al track solicitado.
- [ ] El ZIP es para Windows y usa HTTPS, salvo que HTTP sea imprescindible.
- [ ] Hay SHA-256/SHA-512 oficial o se usa `html-links` con
      `verification.type: "none"` de forma explícita.
- [ ] `stripSingleRoot` y `requiredFiles` coinciden con el ZIP real.
- [ ] La validación no abre una GUI ni deja un proceso persistente.
- [ ] Configuración, caché y workspace quedan en el profile, no en el payload.
- [ ] `component refresh` muestra el ID en `eap.cmd catalog`.
- [ ] `resolve`, `install` y `doctor` terminan correctamente.
- [ ] `launch --dry-run` es correcto cuando existe launcher.
- [ ] No hay secretos, tokens ni credenciales en los JSON.

## Diagnóstico rápido

| Síntoma | Causa probable |
|---|---|
| No aparece tras refrescar | Falta la entrada en `catalog.json`, no se publicó en `main` o el refresh conservó la caché anterior. |
| `Resolver no soportado` | `resolver.type` no está implementado en EAP. |
| `no captura 'version'` | Falta el grupo `(?P<version>...)` en el patrón. |
| `no publicó el checksum` | La fuente no expone SHA-256/SHA-512 en el lugar esperado. |
| `no publicó un ZIP` con `html-links` | `linkPattern`, exclusiones, track o navegación no coinciden con los enlaces HTML reales. |
| `no pertenece a la línea` | La versión capturada no comienza por los grupos del track. |
| `Falta el archivo requerido` | `requiredFiles` o `stripSingleRoot` no reflejan el ZIP. |
| `sale del payload` / `sale de su perfil` | Una ruta cruza la frontera permitida. |
| `publicado por dos repositorios` | Dos fuentes externas declaran el mismo ID. |

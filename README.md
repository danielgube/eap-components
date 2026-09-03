# EAP Components

Catálogo oficial de componentes para [EAP](https://github.com/danielgube/eap).

Cada componente se declara mediante un manifiesto JSON dentro de
`components/`. El archivo `catalog.json` publica los manifiestos disponibles.
EAP obtiene siempre una revisión inmutable del repositorio, valida todos los
manifiestos y conserva una copia local antes de activarla.

Los manifiestos actuales usan `schemaVersion: 3` y declaran obligatoriamente un
bloque `info` con una descripción breve y las rutas importantes relativas al
profile o al workspace. EAP resuelve esas rutas de forma absoluta tanto en la
pantalla principal como antes de instalar el componente.

## Estructura

```text
catalog.json
components/
  java.json
  tomcat.json
  golang.json
  php.json
  maven.json
  ...
```

En esta primera versión los manifiestos sólo pueden utilizar resolvers y
validadores incluidos en EAP. Los adaptadores ejecutables externos se añadirán
posteriormente mediante una API versionada y aislada; el catálogo no admite
código Python remoto.

## Uso

EAP incluye este repositorio como fuente predeterminada:

```properties
components.repository.danielgube=https://github.com/danielgube/eap-components
```

Para actualizar la caché local:

```bat
eap.cmd component refresh
```

El catálogo incluido dentro de EAP se mantiene como snapshot de bootstrap y
respaldo offline.

## Crear un componente

El contrato completo, los resolvers admitidos, ejemplos, límites y el proceso
de prueba y publicación están documentados en
[`CREAR_COMPONENTES.md`](CREAR_COMPONENTES.md).

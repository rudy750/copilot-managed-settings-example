[English](README.md) | **[Español](README.es.md)**

# Ejemplo de configuración administrada empresarial de GitHub Copilot

Este repositorio es un ejemplo público, no productivo, de la configuración
administrada empresarial de GitHub Copilot. Demuestra:

- Una política base de `managed-settings.json`.
- Listas de permitidos y denegados de servidores MCP.
- Diferentes políticas de MCP para equipos empresariales.
- Configuraciones adicionales para permisos, modelos, plugins, control remoto,
  telemetría y el sandbox de Copilot CLI.

Los valores utilizan organizaciones, dominios, repositorios y slugs de equipo
ficticios. Reemplaza cada valor `contoso` antes de usar esta configuración.

> [!IMPORTANT]
> Una configuración de producción administrada por el servidor pertenece al
> directorio `copilot/` de un repositorio `.github-private` propiedad de la
> organización, seleccionado como la fuente empresarial de gobernanza de
> clientes. Este repositorio público es solo una implementación de referencia
> y no aplica configuraciones a una empresa.

## Estructura del repositorio

```text
copilot/
├── managed-settings.json
├── team-mappings.json
└── teams/
    ├── developers.json
    └── security-engineering.json
```

| Archivo | Propósito |
| --- | --- |
| `copilot/managed-settings.json` | Define los valores predeterminados de toda la empresa y marca las claves seleccionadas como sobrescribibles por los equipos empresariales. |
| `copilot/team-mappings.json` | Asigna archivos de configuración de equipo a slugs de equipos empresariales. |
| `copilot/teams/developers.json` | Otorga a los equipos de desarrolladores acceso a servidores MCP remotos y locales aprobados. |
| `copilot/teams/security-engineering.json` | Otorga a los equipos de seguridad acceso a servidores MCP de seguridad aprobados mientras bloquea los procesos MCP locales. |

## Cómo funcionan las configuraciones específicas de equipo

Solo las implementaciones administradas por el servidor admiten
sobrescrituras de equipos empresariales.

El archivo base envuelve los valores sobrescribibles por equipo en un objeto
`overridable`:

```json
{
  "allowedMcpServers": {
    "overridable": [
      { "serverUrl": "https://mcp.contoso.example/shared/*" }
    ]
  }
}
```

Un archivo de equipo asignado utiliza la sintaxis de valor normal:

```json
{
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.contoso.example/shared/*" },
    { "serverUrl": "https://mcp.contoso.example/development/*" }
  ]
}
```

`team-mappings.json` asigna cada nombre de archivo de configuración a uno o
más slugs de equipos empresariales:

```json
{
  "developers.json": [
    "application-developers",
    "platform-engineering"
  ],
  "security-engineering.json": [
    "security-engineering"
  ]
}
```

Si un usuario pertenece a varios equipos empresariales asignados, GitHub
combina los archivos de equipo usando el valor menos restrictivo para cada
clave. Las decisiones a nivel de plataforma siguen teniendo precedencia.

## Listas de permitidos y denegados de MCP

### `allowedMcpServers`

Cuando esta clave está presente, solo se permiten los servidores MCP que
coincidan con una entrada. Omitir la clave permite todos los servidores,
sujeto a `deniedMcpServers`. Un array vacío bloquea todos los servidores
excepto los servidores predeterminados integrados.

Cada entrada debe contener exactamente un comparador:

| Comparador | Uso |
| --- | --- |
| `serverName` | Coincide exactamente con el nombre de servidor asignado por el usuario. Es conveniente, pero no identifica fuertemente a un servidor. |
| `serverUrl` | Coincide con una URL remota de MCP HTTP o SSE. Admite comodines para subdominios o prefijos de ruta. |
| `serverCommand` | Coincide con un comando de servidor stdio local y cada argumento exactamente. No se admiten comodines ni expansión de shell. |

Prefiere `serverUrl` o `serverCommand` cuando la política deba identificar el
servidor real en lugar de una etiqueta seleccionada por el usuario.

### `deniedMcpServers`

Una entrada de denegación coincidente bloquea un servidor incluso cuando
también coincide con la lista de permitidos. Las reglas de denegación tienen
precedencia sobre las reglas de permiso. Los servidores propios integrados de
Copilot, incluido el servidor MCP de GitHub integrado, no se pueden bloquear.

Los archivos de equipo en este repositorio repiten las entradas de denegación
base porque sobrescriben `deniedMcpServers`. Esto mantiene visibles e
intactas las restricciones base mientras se agregan restricciones específicas
del equipo.

### Detalles de coincidencia

- `serverName` es una coincidencia de etiqueta exacta y distingue mayúsculas
  de minúsculas, sin comodines.
- `serverUrl` se aplica solo a servidores remotos HTTP o SSE.
- `serverCommand` se aplica solo a servidores stdio locales y debe coincidir
  exactamente con todo el array de comandos.
- La coincidencia de URL se canonicaliza mediante el cliente, incluyendo
  convertir a minúsculas el esquema y el host y eliminar los puertos
  predeterminados.
- Cuando varias fuentes de configuración administrada definen una lista de
  permitidos, la lista efectiva es su intersección.
- Cuando varias fuentes de configuración administrada definen una lista de
  denegados, la lista efectiva es su unión.

## Configuraciones administradas adicionales

El ejemplo base demuestra varias configuraciones más allá de la gobernanza de
MCP.

| Configuración | Qué hace |
| --- | --- |
| `model` | Establece el modelo predeterminado para nuevas conversaciones. `"auto"` habilita la selección automática de modelo. Los usuarios aún pueden elegir otro modelo permitido para una conversación individual. |
| `permissions.disableBypassPermissionsMode` | Establecer esto en `"disable"` evita que los usuarios habiliten el modo de omisión de permisos tipo permitir-todo o YOLO. |
| `permissions.deny` | Bloquea comandos de shell, operaciones de archivo o dominios coincidentes. La denegación tiene la mayor precedencia de reglas de permiso. |
| `permissions.ask` | Requiere una aprobación humana nueva cada vez que se solicita una operación coincidente. |
| `permissions.allow` | Permite operaciones coincidentes sin una solicitud. Si existen reglas administradas o una lista de permitidos, las operaciones admitidas no coincidentes requieren aprobación. |
| `enabledPlugins` | Requiere que un plugin esté habilitado con `true`, o deshabilitado con `false`, usando claves `PLUGIN@MARKETPLACE`. |
| `extraKnownMarketplaces` | Agrega mercados de plugins aprobados por la empresa. `autoUpdate` controla si los clientes deben actualizar y refrescar los plugins de ese mercado. |
| `strictKnownMarketplaces` | Restringe la instalación de plugins a los mercados listados. Un array vacío bloquea por completo la instalación de plugins. |
| `telemetry` | Configura la exportación de OpenTelemetry. Mantén `captureContent` deshabilitado a menos que la empresa haya aprobado explícitamente la recopilación de prompts y respuestas. |
| `remoteControl` | Controla si las sesiones alojadas en un dispositivo pueden controlarse de forma remota. `requireSSO` limita el control a clientes autorizados para las organizaciones de GitHub listadas. |
| `sandbox` | Aplica restricciones mínimas del sandbox de Copilot CLI para la ejecución de comandos, servidores MCP locales, servidores de lenguaje, archivos, acceso a la red y credenciales. |

Los selectores de permisos en el ejemplo tienen estos significados:

| Selector | Coincide con |
| --- | --- |
| `Shell(...)` | Comandos de shell. Un comando seguido de ` *` coincide con ese prefijo de comando. |
| `Read(...)` | Lecturas de archivos. `/` significa la raíz del espacio de trabajo, `~/` el directorio de inicio y `//` la raíz del sistema de archivos. |
| `Edit(...)` | Escrituras de archivos, usando las mismas raíces de ruta y comportamiento de comodín que `Read`. |
| `Domain(...)` | Orígenes de red. `*.example.com` coincide con el dominio y sus subdominios. |

La precedencia de permisos es `deny` > `ask` > `allow`.

## Configuraciones intencionalmente no habilitadas

El ejemplo define un mercado de plugins aprobado pero no habilita
automáticamente ningún plugin. Agrega entradas a `enabledPlugins` solo
después de validar el nombre del plugin y el mercado.

La exportación de telemetría está deshabilitada. Para habilitarla,
proporciona un endpoint OTLP real y elige `http/json` o `http/protobuf`. No
confirmes credenciales de recolector en un repositorio; distribuye
encabezados sensibles mediante un mecanismo de configuración aprobado.

La política de sandbox no incluye rutas de concesión de sistema de archivos
personalizadas. Las listas de rutas administradas comparan cadenas exactas
con las concesiones configuradas por el usuario, por lo que copiar rutas de
marcador de posición puede romper involuntariamente las herramientas de
desarrollo.

## Lista de verificación de implementación

1. Crea un repositorio `.github-private` propiedad de la organización y
   selecciónalo como la fuente empresarial de gobernanza de clientes.
2. Copia el directorio `copilot/` en ese repositorio.
3. Reemplaza todos los dominios ficticios, nombres de organización,
   repositorios, comandos y slugs de equipos empresariales.
4. Verifica que cada configuración sea compatible con todos los clientes de
   Copilot utilizados en la empresa.
5. Revisa los comandos MCP locales como arrays exactos; las versiones de
   paquete y los argumentos opcionales son parte de la identidad.
6. Protege los cambios en `copilot/managed-settings.json`,
   `copilot/team-mappings.json` y `copilot/teams/` con protección de rama y
   revisión de administrador o gestor de IA.
7. Prueba con un equipo empresarial pequeño antes de un despliegue amplio.
8. Permite hasta una hora para que los clientes compatibles se actualicen, o
   reinicia el cliente o vuelve a iniciar sesión para activar una
   actualización inmediata.

## Referencias

- [Referencia de configuración administrada empresarial](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings)
- [Introducción a la configuración administrada empresarial](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/get-started)
- [Sobrescribir la configuración administrada empresarial para equipos](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/override-settings-for-teams)

GitHub puede agregar configuraciones o cambiar el soporte de clientes con el
tiempo. Consulta la referencia actual antes de implementar este ejemplo.

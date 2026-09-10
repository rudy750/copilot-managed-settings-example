[English](README.md) | **[Español](README.es.md)**

# Ejemplo de configuración administrada empresarial de GitHub Copilot

Este repositorio es un ejemplo público, no productivo, de la configuración
administrada empresarial de GitHub Copilot. Demuestra:

- Una política base de `managed-settings.json` que configura únicamente las
  listas de permitidos y denegados de servidores MCP.
- Diferentes políticas de MCP para equipos empresariales.

Los ejemplos de MCP utilizan servidores listados en el
[Registro de MCP de GitHub](https://github.com/mcp). Otros valores de ejemplo
utilizan organizaciones, dominios, repositorios y slugs de equipo ficticios.
Reemplaza cada valor `contoso` antes de usar esta configuración.

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
| `copilot/managed-settings.json` | Define las listas de permitidos y denegados de MCP de toda la empresa y las marca como sobrescribibles por los equipos empresariales. |
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
      { "serverUrl": "https://mcp.context7.com/mcp" },
      { "serverUrl": "https://mcp.deepwiki.com/mcp" },
      { "serverCommand": ["uvx", "markitdown-mcp"] }
    ]
  }
}
```

Cualquier entrada en la lista base `overridable` es un **valor
predeterminado**: se permite para todos los miembros de la empresa a menos que
un archivo de equipo asignado defina su propio `allowedMcpServers` y omita esa
entrada. Como `developers.json` y `security-engineering.json` definen ambos
`allowedMcpServers`, reemplazan por completo la lista base para sus miembros.
Los miembros de cualquier otro equipo empresarial, o los que no pertenecen a
un equipo asignado, reciben los valores predeterminados base, incluidos
servidores que no están en la lista de permitidos de ningún equipo, como
DeepWiki y MarkItDown.

Un archivo de equipo asignado utiliza la sintaxis de valor normal:

```json
{
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.context7.com/mcp" },
    { "serverUrl": "https://learn.microsoft.com/api/mcp" }
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

### Servidores del registro utilizados en este ejemplo

| Servidor | Uso en la política |
| --- | --- |
| [Context7](https://github.com/mcp/upstash/context7) | Servidor remoto de documentación compartido, permitido por la base y ambos equipos. |
| [Microsoft Learn](https://github.com/mcp/microsoftdocs/mcp) | Servidor remoto de documentación permitido para los equipos de desarrolladores. |
| [Playwright](https://github.com/mcp/microsoft/playwright-mcp) | Servidor local de automatización de navegador permitido para los equipos de desarrolladores y denegado para los equipos de seguridad. |
| [Sentry](https://github.com/mcp/getsentry/sentry-mcp) | Servidor remoto de diagnóstico de aplicaciones permitido para los equipos de seguridad. |
| [DeepWiki](https://github.com/mcp/cognitionai/deepwiki) | Servidor remoto de documentación y preguntas y respuestas de repositorios, permitido como valor predeterminado base y ausente en la lista de permitidos de ambos equipos. |
| [MarkItDown](https://github.com/mcp/microsoft/markitdown) | Servidor local de conversión de documentos a Markdown, permitido como valor predeterminado base y ausente en la lista de permitidos de ambos equipos. |

Las políticas identifican los servidores, pero no configuran credenciales.
Context7 y Sentry pueden requerir que los usuarios se autentiquen al
conectarse.

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
- [Registro de MCP de GitHub](https://github.com/mcp)

GitHub puede agregar configuraciones o cambiar el soporte de clientes con el
tiempo. Consulta la referencia actual antes de implementar este ejemplo.

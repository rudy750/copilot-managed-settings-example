[English](README.md) | **[Español](README.es.md)**

# Ejemplo de configuración administrada empresarial de GitHub Copilot

Este repositorio es un ejemplo público, no productivo, de la configuración
administrada empresarial de GitHub Copilot. Demuestra:

- Una política base de `managed-settings.json` que configura únicamente las
  listas de permitidos y denegados de servidores MCP.
- Diferentes políticas de MCP para equipos empresariales.

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

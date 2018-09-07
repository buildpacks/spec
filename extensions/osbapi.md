# Open Service Broker API

The following additional environment variables extend the [Environment - Provided by the Platform](../buildpack.md#provided-by-the-platform) section in the Buildpack Interface Specification.

| Env Variable    | Description                            | Detect | Build | Launch
|-----------------|----------------------------------------|--------|-------|--------
| `PACK_SERVICES` | OSBAPI service bindings                |        |       | [x]
| `PACK_SERVICES` | OSBAPI service bindings (restricted)   | [x]    | [x]   |

Where:
- Each additional environment variable value MUST be provided in JSON format.
- When restricted, `PACK_SERVICES` SHOULD NOT contain credentials that are only necessary for launch.

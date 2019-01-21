# Open Service Broker API

The following additional environment variables extend the [Environment - Provided by the Platform](../buildpack.md#provided-by-the-platform) section in the Buildpack Interface Specification.

| Env Variable    | Description                            | Detect | Build | Launch
|----------------|----------------------------------------|--------|-------|--------
| `CNB_SERVICES` | OSBAPI service bindings                |        |       | [x]
| `CNB_SERVICES` | OSBAPI service bindings (restricted)   | [x]    | [x]   |

Where:
- Each additional environment variable value MUST be provided in JSON format.
- When restricted, `CNB_SERVICES` SHOULD NOT contain credentials that are only necessary for launch.

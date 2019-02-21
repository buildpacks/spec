# Application Runtime Extension

This document specifies a contract between a platform and an application process running in an OCI image.

This extension only applies to application process types.

## Runtime Environment Variables

The following environment variables extend the [Environment - Provided by the Platform](../buildpack.md#provided-by-the-platform) section in the Buildpack Interface Specification.
The are exclusively available during the launch phase.

| Env Variable         | Example Value            | Description                   | Mandatory
|----------------------|--------------------------|------------------------------------------
| `PORT`               | `8080`                   | port for primary process type |
| `CNB_APP_ROUTES`     | `{"http":{"port":8080}}` | routes for each process type  | [x]
| `CNB_APP_INDEX`      | `0`                      | app instance index (logical)  | [x]
| `CNB_APP_NAME`       | `example-app`            | app name                      | [x]

### Format: `PORT`

`PORT` MUST be an unsigned 16-bit integer.

### Format: `CNB_APP_ROUTES`

The `CNB_APP_ROUTES` environment variable MUST be JSON-formatted and follow this schema:

```
{
    "<route name>": {
        "port": <internal port>,
        "uri": "<external uri>"
    }
}
```

Where `port` MUST be an unsigned 16-bit integer if present.


Example:
```
{
    "http": {
        "port": 8080,
        "uri": "https://app.example.com/"
    },
    "debug": {
        "port": 12034,
    }
}
```

### Format: `CNB_APP_INDEX`

`CNB_APP_INDEX` MUST be a logical identifier that uniquely differentiates a single instance of an application from other instances.

Examples: `1`, `01`, `web.1`

### Format: `CNB_APP_NAME`

`CNB_APP_NAME` MUST be a string that identifies the app. It SHOULD be unique among apps that it communicates with.
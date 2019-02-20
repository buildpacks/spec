# Application Runtime Extensions

This document specifies a contract between a platform and processes running in an application OCI image.  

## Runtime Environment Variables

The following environment variables extend the [Environment - Provided by the Platform](../buildpack.md#provided-by-the-platform) section in the Buildpack Interface Specification.
The are exclusively available during the launch phase.

| Env Variable         | Example Value                    | Description                       | Mandatory
|----------------------|----------------------------------|----------------------------------------------
| `PORT`               | `8080`                           | port for primary web process type | [x]
| `CNB_APP_ROUTES`     | `{"web":{"http":{"port":8080}}}` | routes for each process type      | [x]
| `CNB_APP_INDEX`      | `0`                              | app instance index (logical)      | [x] 
| `CNB_APP_NAME`       | `example-app`                    | app name                          |

### Format: `CNB_APP_ROUTES`

The `CNB_APP_ROUTES` enviroment variable MUST be JSON-formatted and follow this schema:

```
{
    "<process type>": {
        "<route name>": {
            "port": <internal port>,
            "uri": "<external uri>"
        }
    }
}
```

Example:
```
{
    "web": {
        "http": {
            "port": 8080,
            "uri": "app.example.com"
        },
        "debug": {
            "port": 12034,
            "uri: "203.0.113.254:4444"
        }
    },
    "worker": {
        "debug": {
            "port": 12035
        }
    }
}
```
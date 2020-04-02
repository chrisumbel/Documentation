# Debug Logging

Sometimes, it is desirable to turn on debug logging in the Discovery client. To do so, you can modify the `appsettings.json` file and turn on Debug level logging for the Steeltoe components, as shown in the following example:

Here is an example `appsettings.json` file:

```json
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Warning",
      "Steeltoe": "Debug"
    }
  },
  ...
}
```
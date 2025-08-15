
## Change default settings
Default settings are managed by the static `SettingsManager` class which loads a single instance of the `Settings` class. A few common defaults you may want or need to change include:

| Setting | Default value |
| --- | --- |
| BaseCurrency | USD |
| FXDirectory  | C:\Data\Forex |

To temporarily override a default, set the appropriate property in `SettingsManager` before running your code. For example, to use a different money management template, you could set the `MoneyManagementTemplate` property:
```csharp
SettingsManager.Default.MoneyManagerOptions.Template = @"Templates\ComparisonTemplate.xltx";
```

To make these changes permanent, call `SettingsManager.Default.Save()`. Alternatively, create an instance of the `Settings` class directly, set the properties of interest, and call `Save()`.
```csharp
new Settings
{
    FXDirectory = @"C:\FX",
    MoneyManagerOptions = new MoneyManagerOptions
    {
        Template = @"Templates\CustomTemplate.xltx"
    }
}.Save();
```
  This will create a json configuration file named settings.json in your output directory that looks like this:
```json
{
  "FXDirectory": "C:\\FX",
  "MoneyManagerOptions": {
    "Template": "Templates\\CustomTemplate.xltx"
  }
}
```
Be sure to add this file to your Visual Studio project and set the 'Copy to Output Directory' property to 'Copy always'. This ensures that settings.json is copied to the output directory. When settings are first accessed, if a settings.json exists in the output directory, it will be automatically loaded, overriding the built-in defaults.
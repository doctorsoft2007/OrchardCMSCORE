### Creating a host

When running Orchard 2, you need a client. The default implementation is to have a client talk to a host.

The client can be any project that creates the host.

To create the host in a web project you would do:

```c#
public class Startup {
    public IServiceProvider ConfigureServices(IServiceCollection services) {
        return services
            // AddHostSample is where the magic is done. This extension method lives in the Host (Orchard.Hosting.Web)
            .AddHostSample()
            .BuildServiceProvider();
    }
}
```

The host has a small wrapper:

```c#
public static IServiceCollection AddHostSample(this IServiceCollection services) {
    // This will setup all your core services for a host
    return services.AddHost(internalServices => {
        // The core of the host
        internalServices.AddHostCore();
        //... All extra things you want registered so that you don't have to touch the core host.
    });
```

### Additional module locations

Additional locations for module discovery can be added in your client setup:

```c#
public class Startup {
    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
        services.AddWebHost();

        // Add folders the easy way
        services.AddModuleFolder("Core/Orchard.Core");
        services.AddModuleFolder("Modules");
        services.AddThemeFolder("Themes");

        // Add folders the more configurable way
        services.Configure<ExtensionHarvestingOptions>(options => {
            var expander = new ModuleLocationExpander(
                DefaultExtensionTypes.Module,
                new[] { "Core/Orchard.Core", "Modules" },
                "Module.txt"
                );

            options.ModuleLocationExpanders.Add(expander);
        });
    });
}
```

### Tenant Configuration

All tenant configuration lives in `src\Orchard.Web\App_Data\Sites\Default` within settings files, e.g. `Settings.txt`:

```yaml
State: Running
Name: Default
RequestUrlHost: localhost:5000
RequestUrlPrefix:
```

However, you can override these values within a .json or .xml file. The order of precendence is:
Settings.txt -> Settings.xml -> Settings.json

You can also override the 'Sites' folder in your client setup

```c#
public class Startup {
    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
        services.AddWebHost();

        // Change the folder name here
        services.ConfigureShell("Sites");
    });
}
```

### Orchard file System

Orchard now has a build in file system that is scoped to the running site. To use this, you just need to inject in IOrchardFileSystem.

You use non virtual paths for access, so for example, lets say you have this folder.

> D:\Orchard2\src\Orchard.Web\Modules\Orchard.Lists\Module.txt

and in you code you want to read that file,

```c#
public void GetMeThatFile()
{
  var fileText = _fileSystem.ReadFile("Modules\Orchard.Lists\Module.txt");
  // The physical path will be D:\Orchard2\src\Orchard.Web\Modules\Orchard.Lists\Module.txt
}
```

The file system is scoped to the Orchard.Web folder by default. If however you want another filesystem, you can create a new one elsewhere.

```c#
public void CreateMeAFileSystem()
{
  var root = "C:\MyFileSystemRootPath";
  var fileSystem = new OrchardFileSystem(
    root,
    new PhysicalFileProvider(root),
    _logger);

  // now if I get my module file..
  var fileText = fileSystem.ReadFile("Modules\Orchard.Lists\Module.txt");
  // The physical path will be C:\MyFileSystemRootPath\Modules\Orchard.Lists\Module.txt
}
```

If you would like to deal with files within a particular Extension Folder you can do this:

```c#
public void GetMePlacement()
{
  // First get the extension.
  ExtensionDescriptor extensionDescriptor = _extensionManager.GetExtension("Orchard.Lists");

  // Second use the extension to get the placement info file
  IFileInfo placementInfoFile = _fileSystem
    .GetExtensionFileProvider(extensionDescriptor, _logger)
    .GetFileInfo("Placement.info");
}
```
##The needs (in construction)

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, these modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load their assemblies.

- So, **Dynamic Loading** is necessary, we need to load all modules assemblies (and their dependencies) at runtime. By doing this we can also pass to the razor view engine all needed metadata references for views compilation (not described here). We also need to store specific assemblies in some probing folders, this to retrieve them in different contexts.

- **Dynamic Compilation** is useful in a development context, you can update a module source file, a package or a core project and just hit F5, all dependent modules will be dynamically re-compiled. When a module has a dependency on a core project which is not part of `Orchard.Web`, if needed this non ambient core project is also re-compiled at runtime.

- **In production**, most of the time it will be better to use pre-compiled modules, but maybe useful sometimes to only have to publish a source file and let things go. It has already been tested in some environments where core projects are not there and / or there is no packages storage, but in production we have to think about some possible limitations, e.g related to files writing permissions.

###Assemblies

- **Compilation assemblies**: Used to reference all dependencies when compiling. Most of the time a project / package library has only one compilation assembly which is the same as the runtime assembly (some packages have only one of them).

- **Runtime assemblies**: Used for a concrete implementation, need to be loaded for all dependencies. Most of the time a project / package library has only one runtime assembly which is the same as the compilation assembly.

- **Specific runtime assemblies**: Some runtime libraries provides differents implementations for different runtimes. Only the right assemblies need to be loaded according to the current runtime environment.

- **Native assemblies**: Some libraries need to provide native implementations for different runtimes. Because Modules are portable libraries not intended to be published, we don't care about native outputs.

- **Resource assemblies**: Embedded resources in `{project}.resources.dll` files used by the Resource Manager.

##Dotnet commands

- **dotnet compile** (used by dotnet build): The runtime assembly (and its .pdb file) is outputed.

        // {project}/bin/{config}/{framework}/{project}.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.dll

    When a project is built to have an entry point and to preserve its compilation context (`emitEntryPoint` and `preserveCompilationContext` in project.json), a bigger assembly is generated and also a dependencies file.

        // {project}/bin/{config}/{framework}/{project}.deps.json
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.deps.json

- **dotnet build**: It uses dotnet compile and all referenced projects are also built if needed.

    runtime assemblies of all referenced projects are also outputed.

        // {project}/bin/{config}/{framework}
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Data.dll

    Runtime assemblies of referenced packages are not outputed.

    Resources assemblies are ouputed.

        // {project}/bin/{config}/{framework}/{locale}/{project}.resources.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/fr/Orchard.Web.resources.dll

- **dotnet publish**: All above assemblies are outputed at the root of the published target, plus the following.

    Runtime assemblies of referenced packages which are not part of the targeted framework (see in `dotnet/shared/Microsoft.NETCore.App/{version}`) are also outputed.

        root/System.Data.Common.dll
        root/Microsoft.Extensions.DependencyModel.dll

    Those which have specific implementations for different runtimes are outputed in:

        // runtimes/{rid}/lib/{tfm}
        root/runtimes/unix/lib/netstandard1.3/System.IO.Pipes.dll

    All compilation assemblies which differ from their runtime assembly counterpart are outputed.

        root/refs/System.IO.dll

    Native assemblies for different runtime environments are ouputed.

        // runtimes/{rid}/native
        root/runtimes/osx/native/lmdb.dylib

    As with dotnet build, resources assemblies are ouputed and therefore here.

        // {locale}/{project}.resources.dll
        root/fr/Orchard.Web.resources.dll

- **dotnet restore**: The `project.json` file is parsed to retrieve all dependencies, packages are updated, and, if needed, the `project.lock.json` file is written with updated dependencies metadata.

##Project Model APIs

Dotnet cli use these APIs which allow to e.g create a project context and retrieve all its dependencies (referenced project / package libraries). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g by providing full paths). But to use the Project Model on a project, its `project.json` and `project.lock.json` files need to be there.

##Dependency Model APIs

When a project is built to have an entry point and to preserve its compilation context, as needed by the main application to be executed and e.g to compile razor views, a bigger assembly is generated and also a `{project}.deps.json` file. Then we can use the Dependency Model APIs to retrieve dependencies infos through this deps json file. It is less powerful than the Project Model because assets are not fully resolved (e.g by providing only relative paths). It doesn't rely on the presence of the 2 project json files but it needs the related deps json file and a bigger assembly has been generated.


##Dynamic Compilation
##Dynamic Loading
##Probing Folders
##Contexts



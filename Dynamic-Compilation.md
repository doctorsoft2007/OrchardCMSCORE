##The needs (in construction)

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, these modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load their assemblies.

- So, **Dynamic Loading** is necessary, we need to load all modules assemblies (and their dependencies) at runtime. By doing this we can also pass to the razor view engine all needed metadata references for views compilation (not described here). We also need to store specific assemblies in some probing folders, this to retrieve them in different contexts.

- **Dynamic Compilation** is useful in a development context, you can update a module source file, a package or a core project and just hit F5, all dependent modules will be dynamically re-compiled. When a module has a dependency on a core project which is not part of `Orchard.Web`, if needed this non ambient core project is also re-compiled at runtime.

- **In production**, most of the time it will be better to use pre-compiled modules, but maybe useful sometimes to only have to publish a source file and let things go. It has already been tested in some environments where core projects are not there and / or there is no packages storage, but in production we have to think about some possible limitations, e.g related to files IO operations.

###Assemblies

- **Compilation assemblies**: When compiling a module, we parse all its projects / packages dependencies, and reference them through their compilation assemblies. Most of the time a project / package has only one compilation assembly which is the same as the runtime assembly (see below) used for implementation. Some packages have only compilation assembly(ies), others only runtime assembly(ies).

- **Runtime assemblies**: When loading a module, we parse all its projects / packages dependencies, and load their runtime assemblies for a concrete implementation. Most of the time a project / package has only one runtime assembly which is the same as the compilation assembly (see above) used for compilation. Some packages have only compilation assembly(ies), others only runtime assembly(ies).

- **Specific runtime assemblies**: Some runtime libraries provides differents implementations for different runtimes. Only the right assemblies need to be loaded according to the current runtime environment.

- **Native assemblies**: Native implementations for different runtimes. They are outputed when publishing a standalone or a portable application. But Modules are portable libraries so we don't care about native outputs.

- **Resource assemblies**: Todo.


##Dotnet commands

- **dotnet compile** (used indirectly by `dotnet build`): The runtime assembly (and eventually its .pdb file) is outputed to the `{project}/bin/{config}/{framework}` folder. For an assembly with an entry point (`"emitEntryPoint": true` in project.json), a `{project}.deps.json` file is also generated and outputed. Then, if the compilation context is preserved (`"preserveCompilationContext": true`), more infos are added in this file, but this option produce bigger assemblies.

- **dotnet build**: All referenced projects (through the dependency graph) are also built if needed. Then, a dotnet compile is done and, generally, runtime assemblies of all referenced projects are also outputed to the `{project}/bin/{config}/{fm}` folder, but not runtime assemblies of referenced packages.

- **dotnet publish**: All above assemblies are outputed at the root of the published target, plus the following. Runtime assemblies of referenced packages which are not part of the targeted framework e.g `netcoreapp1.0` (see in `dotnet/shared/Microsoft.NETCore.App/{version}`) are also outputed. Those which have specific implementations for different runtimes are outputed in:

        runtimes/{rid}/lib/{tfm}
        e.g `runtimes/unix/lib/netstandard1.3/System.IO.Pipes.dll
All compilation assemblies which differ from their runtime assembly counterpart are outputed in the `refs` subfolder. Native assemblies for different runtime environments are ouputed in:

        `runtimes/{rid}/native`
        e.g `runtimes/osx/native/lmdb.dylib`

- **dotnet restore**: The `project.json` file is parsed to retrieve all dependencies, packages are updated, and, if needed, the `project.lock.json` file is written with updated dependencies metadata.

##Project Model APIs

Dotnet cli use these APIs which allow to e.g create a project context and retrieve all its dependencies (referenced libraries of type Project / Package). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g by providing full paths). But to use the Project Model on a project, its `project.json` and `project.lock.json` files need to be there.

##Dependency Model APIs

When a project is built to have an entry point and to preserve its compilation context, as needed by the main application to be executed and e.g to compile razor views, a bigger assembly is generated and also a `{project}.deps.json`. Then we can use the Dependency Model APIs to retrieve dependencies infos through this deps json file. It is less powerful than the Project Model because assets are not fully resolved (e.g by providing only relative paths). It doesn't rely on the presence of the 2 project json files but it needs the related deps json file and a bigger assembly has been generated.


##Dynamic Compilation
##Dynamic Loading
##Probing Folders
##Contexts



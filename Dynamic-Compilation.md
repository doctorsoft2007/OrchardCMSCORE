##The needs (in construction)

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, these modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load their assemblies.

- **Dynamic Loading** is therefore necessary, we need to load all modules assemblies (and their dependencies) at runtime. By doing this we can also pass to the razor view engine all needed metadata references for views compilation (not described here). We also need to store specific assemblies in some probing folders, this to retrieve them in different contexts.

- **Dynamic Compilation** is useful in a development context, you can update a module source file, a package or a core project and just hit F5, all dependent modules will be dynamically re-compiled. When a module has a dependency on a core project which is not part of `Orchard.Web`, if needed this non ambient core project is also re-compiled at runtime.

- **In production**, most of the time it will be better to use pre-compiled modules, but maybe useful sometimes to only have to publish a source file and let things go. Dynamic compilation and loading has already been tested in some environments where core projects are not there and / or there is no packages storage.

###Library assemblies

- **Compilation assemblies**: Used to referenced all project dependencies when compiling.

- **Default runtime assemblies**: Used for implementation at runtime, need to be loaded for all dependencies.

- **Compile only assemblies**: Most of the time a project / package library has one compilation assembly which is also used as the default runtime assembly. If not, it's a compile only assembly.

- **Specific runtime assemblies**: Some libraries provides Different implementations for different runtime environments.

- **Native assemblies**: Some libraries need to provide native implementations for different runtimes environments. Because Modules are not intended to be published, we don't care about native outputs.

- **Resource assemblies**: Embedded resources in `{project}.resources.dll` files used by the Resource Manager.

##Dotnet commands

- **dotnet compile** (used by dotnet build): The project assembly (and its .pdb file) are outputed.

        // {project}/bin/{config}/{framework}/{project}.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.dll

    When a project is compiled to have an entry point and to preserve its compilation context (`emitEntryPoint` and `preserveCompilationContext` in project.json), it has a bigger assembly and a dependencies file is also generated.

        // {project}/obj/{config}/{framework}/{project}dotnet-compile.deps.json
        Orchard.Web/obj/Debug/netcoreapp1.0/Orchard.Webdotnet-compile.deps.json

- **dotnet build**: It uses dotnet compile and all referenced projects are also built if needed.

    When a project is compiled to have an entry point and to preserve its compilation context, another dependencies file is generated and outputed.

        // {project}/bin/{config}/{framework}/{project}.deps.json
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.deps.json

    Runtime assemblies of all referenced projects are also outputed.

        // {project}/bin/{config}/{framework}/{assembly}.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Data.dll

    Runtime assemblies of referenced packages are not outputed.

    Resources assemblies are ouputed.

        // {project}/bin/{config}/{framework}/{locale}/{project}.resources.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/fr/Orchard.Web.resources.dll

- **dotnet publish**: All above assemblies are outputed at the root of the published target, plus the following.

    Default runtime assemblies of referenced packages which are not part of the targeted framework (see in `dotnet/shared/Microsoft.NETCore.App/{version}`) are also outputed.

        root/System.Data.Common.dll
        root/Microsoft.Extensions.DependencyModel.dll

    Specific runtime assemblies of referenced packages which are not part of the targeted framework are also outputed.

        // runtimes/{rid}/lib/{tfm}/{assembly}.dll
        root/runtimes/unix/lib/netstandard1.3/System.IO.Pipes.dll

    All compilation assemblies which are not also default runtime assemblies (because part of the framework and / or specific runtime assemblies are used) are outputed.

        root/refs/System.IO.dll

    Native assemblies for different targeted runtimes are ouputed.

        // runtimes/{rid}/native{assembly}.{lib-suffix}
        root/runtimes/osx/native/lmdb.dylib

    As with dotnet build, resources assemblies are ouputed.

        // {locale}/{project}.resources.dll
        root/fr/Orchard.Web.resources.dll

- **dotnet restore**: The `project.json` file is parsed to retrieve all dependencies, packages are updated, and, if needed, the `project.lock.json` file is written with e.g updated dependencies metadata.

##Project Model API

Dotnet cli use this API which allow to create a project context and e.g retrieve all its dependencies (referenced project / package libraries). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g by providing full paths). But to use the Project Model on a project, its `project.json` and `project.lock.json` files need to be there.

##Dependency Model API

When a project is built to have an entry point and to preserve its compilation context, as needed by the main application to be executed and e.g to compile razor views, a bigger assembly is generated and also a `{project}.deps.json` file. Then we can use the Dependency Model API to retrieve dependencies through this `.deps.json` file. It is less powerful than the Project Model because assets are not fully resolved (e.g by providing only relative paths). It doesn't rely on the presence of the 2 project json files but it needs the related `.deps.json` file and a bigger assembly has been generated.

##Library contexts

- **Resolved Project**: The 2 project json files are there.

- **Resolved Package**: There is a package storage which contains this referenced package.

- **Unresolved Project**: There is no project json files, e.g a core project not there in a published context.

- **Unresolved Package**: There is no package storage (e.g in production) or it doesn't contains this package.

- **Pre-compiled Project**: A Resolved Project with the 2 project json files and its precompiled binaries but without any source files.

- **Pre-compiled Module**: A Module project without project json files but with all necessary binaries (not yet implemented).

- **Ambient Project**: The main project itself or a project belonging to the main project dependencies, so a core project (not all core projects are ambient).

- **Ambient Package**: A package belonging to the main project dependencies, so including those of the targeted framework.

- **Ambient Assembly**: An assembly related to an ambient project or an ambient package.

##Dynamic compilation

- **Roslyn compiler**: We use the Roslyn csharp compiler `csc.exe` by referencing the `Microsoft.Net.Compilers.netcore` package. Then, at runtime we copy it in an executable `csc.dll` and create automatically a `csc.runtimeconfig.json` runtime config file, as `dotnet core setup` do to generate its outputs. Then, as `dotnet compile` do, we can execute `csc.dll`.

- **Compilation**: The implementation is mainly inspired of the `dotnet compile` source code which uses the Project Model API to resolve compilation options, output paths, source files, and all compilation assemblies of the project dependencies (other projects or packages libraries). Then, options and references are formatted and stored in the `dotnet-compile-csc.rsp` response file. Then, this response file is passed as an argument to the `csc.dll` compiler.

- **Configuration**: We need to pass to the Project Model API the curent configuration `Debug` or `Release`. Here we use the configuration under which the main project has been built. To do this, we use the Dependency Model API on the main project (we don't rely on its project.json file) to grab its compilation options and then retrieve the configuration.

- **Conditions**: Only Resolved Projects (see above) which are not Ambient Projects can be compiled (so all modules and some core projects). We also check if a project is not already compiled on the same startup, then we check (only once per project) all compilation IO (see below) to see if it needs to be compiled. By doing this we do a part of the `dotnet build` job.

- **Compilation IO**: Compilation inputs are the 2 project json files, source files, eventually resource input files, and all compilation assemblies (often the same as runtime ones) of the project dependencies. Compilation outputs are the generated project assembly (and its .pdb file), and eventually a resource output assemblies. Notice that a project compilation output can be a compilation input when referenced as a dependency in another project.

- **Compilation IO checking**: We check the presence and the last write time of all resolved compilation IO files. A project need to be compiled if there is a missing IO or if an input is newer than the earliest output. But IO are not resolved in the same way, a source file resolution is already based on its presence. So, we can't only rely on this, e.g when a source file is moved or removed, or an old one is added. That's why the previous compilation context is stored in the `dotnet-compile-csc.rsp` response file, so we can check if anything else has been changed.

- **Dependencies**: Before compiling a project itself we parse all its dependencies. If a dependency is a non Ambient and Resolved Project, the compilation is called on it, and so on. By recursively compiling projects through the dependency graph, we are doing a part of the `dotnet build` job. Then we resolve all compilation references from each dependency.

    Note: Because a dynamic compilation is first called on a Module project, here a non Ambient core Project can only be compiled as a dependency.

    Note: For each dependency which is a Project or a Package library, we don't need to parse their own dependencies. They already belong to those of the project being compiled and whose all the dependency graph has been resolved. So, most of the time, when referencing a dependency, its compilation and runtime collections have only one and the same assembly. This is always true for a Project (not a Package) which never have a compile only assembly. So, here we will not always use the terms of compilation, runtime and collections.

    If a dependency is an Unresolved Project, we have to resolve ourselves the assembly path by searching in probing folders where the assembly may have been stored (see Dynamic storing). First from the runtime directory where ambient assemblies are, then fallback to the project (being compiled) bin folder, then to the shared probing folder. Here, we search at the top level of these folders, so we only need the file name which is the same as the library identity name, this to resolve the assembly path and add it to the compilation references.

    If a dependency is an Unresolved Package (not there), we have to resolve ourselves the assembly path by searching in probing folders. First from the runtime directory, then fallback to the project (being compiled) bin folder, then to the shared probing folder. An Unresolved Package, from other collections, still provides relative paths from which we can still extract the assembly name. Then, if the compile time assembly is not also used as the default runtime one (so compile only), we also combine the `refs` sublfolder to the file name, this to resolve the assembly path and add it to the compilation references.

    If a dependency is a Precompiled Project, so with no source files but still the project json files and its binaries. In this case the compilation assemblies collection is empty, we have to resolve ourselves the assembly path by searching in probing folders. But here, we first search in this precompiled project bin folder where the assembly is intended to be, then we fallback to the project (being compiled) bin folder, then to the shared probing folder. Here also, we search at the top level of these folders and the file name is the same as the library identity name, so we can resolve the assembly path and add it to the compilation references.

    If a dependency is a Precompiled Module (not yet implemented), so with all needed binaries but without project json files. Here, the module bin folder is not related to this pattern `{project}/bin/{config}/{framework}`, all binaries are intended to be directly in the `{project}/bin` folder. So here we will have to do a mix between a Precompiled Project and an Unresolved Project, and we will also have to resolve ourselves the full path of this precompiled module bin folder (its pseudo project output path).

    If a dependency is an Ambient and Resolved Project, the assembly path resolved by the Project Model could be added as it is to the compilation references. But, because e.g VS may output binaries in another folder (e.g artifacts), we first check if the assembly file exists, if not we fallback to the runtime directory (because here it's an ambient assembly).

    If a dependency is a non Ambient and Resolved Project or is a Resolved Package, all the paths of its compilation assemblies, as resolved by the Project Model, are added to the compilation references.

- **Parallel compilation**: Extensions loading is done in parallel, therefore Module projects are also compiled in parallel. We use a dictionary of lock objects based on project names, this to prevent simultaneous compilations of the same project. We also use simple locks to prevent simultaneous writting of the same file.

##Dynamic loading

##Dynamic storing

- **Modules binaries folders**: When dynamically compiling a Module, its assembly is naturally outputed in its binary folder `{project}/bin/{config}/{framework}/{project}.dll`. After loading a module, we also store here all non ambient assemblies (not part of the main project). Can be related to a package or a core project only used by the module, a dependency on another module ... All assets which are compile only assemblies or specific runtime assemblies are stored in a structured way, as `dotnet publish` do (see below).

- **Shared dependencies folder**: `App_Data/dependencies`

- **Structured probing folders**: In a probing folder, as it is done in a runtime directory when publishing, default runtime assemblies are stored at the top, compile only assemblies in a `refs` subfolder, resources assemblies in their related `{locale}` subfolder, and specific runtime assemblies in a `runtimes` subfolder. In the `runtimes` subfolder we use the same pattern `runtimes/{rid}/lib/{tfm}/{assembly}.dll` as dotnet publish uses.

    The nuget package storage uses the same kind of patterns but not exactly the same, e.g compile only assemblies differ based on the targeted framework but not based on the runtime environment. So, here they are all stored in the `refs` subfolder in a flattened way.

- **Runtime directory**: We don't load resources assemblies in memory but we also store them in the runtime directory to be found by the Resource Manager. This by using this relative path `{locale}/{project}.resources.dll`, as `dotnet build` or `dotnet publish` do.



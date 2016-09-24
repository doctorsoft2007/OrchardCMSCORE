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

##Project Model API

Dotnet cli use this API which allow to e.g create a project context and retrieve all its dependencies (referenced project / package libraries). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g by providing full paths). But to use the Project Model on a project, its `project.json` and `project.lock.json` files need to be there.

##Dependency Model API

When a project is built to have an entry point and to preserve its compilation context, as needed by the main application to be executed and e.g to compile razor views, a bigger assembly is generated and also a `{project}.deps.json` file. Then we can use the Dependency Model API to retrieve dependencies infos through this deps json file. It is less powerful than the Project Model because assets are not fully resolved (e.g by providing only relative paths). It doesn't rely on the presence of the 2 project json files but it needs the related deps json file and a bigger assembly has been generated.

##Library contexts

- **Resolved Project**: The 2 project json files are there.

- **Resolved Package**: There is a package storage which contains this referenced package.

- **Unresolved Project**: There is no project json files, e.g a core project not there in a published context.

- **Unresolved Package**: There is no package storage (e.g in production) or it doesn't contains this package.

- **Pre-compiled Project**: A Resolved Project with the 2 project json files but without any source files.

- **Pre-compiled Module**: A Module project without project json files but with all necessary binaries (not implemented).

-**Ambient Project**: The main project itself or a project belonging to the main project dependencies, so a core project (not all core projects are ambient).

-**Ambient Package**: A package belonging to the main project dependencies, including those of the targeted framework.

-**Ambient Assembly**: An assembly related to an ambient project or an ambient package.

##Dynamic Compilation

- **Roslyn compiler**: We use the Roslyn csharp compiler `csc.exe` by referencing the `Microsoft.Net.Compilers.netcore` package. Then, at runtime we copy it in an executable `csc.dll` and create automatically a `csc.runtimeconfig.json` runtime config file, as `dotnet core setup` do to generate its outputs. Then, as `dotnet compile` do, we can execute `csc.dll`.

- **Compilation**: The implementation is mainly inspired of the `dotnet compile` source code which uses the Project Model API to resolve compilation options, output paths, source files, and all compilation assemblies of the project dependencies (other projects or packages libraries). Then, options and references are formatted and stored in the `dotnet-compile-csc.rsp` response file. Then, this response file is passed as an argument to the `csc.dll` compiler.

- **Conditions**: Only a Resolved Project which is not an Ambient Project (see above) can be compiled. We also check if it is not already compiled during the same startup, then we check (only once per project) all compilation IO (see below) to see if it needs to be compiled. By doing this we begin to do a part of the `dotnet build` job. 

- **Compilation IO**: Compilation inputs are the 2 project json files, source files, eventually resource input files, and all compilation assemblies (often the same as runtime ones) of the project dependencies. Compilation outputs are the generated project runtime assembly (and its .pdb file), and eventually a project resource output assembly. Notice that a compilation output can be a compilation input when referenced as a dependency in another project.

- **Compilation IO checking**: We check the presence and the last write time of all resolved compilation IO files. A project need to be compiled if there is a missing IO or if an input is newer than the earliest output. But IO are not resolved in the same way, when an assembly path is resolved you can check if it exists, but a source file resolution is already based on its presence. So, we can't only rely on this, e.g when you move or remove a source file, or add an old one. That's why the previous compilation context is stored in the `dotnet-compile-csc.rsp` response file, so we can check if anything else has been changed.

- **Dependencies**: Before compiling a project itself, if a dependency is a Resolved Project, the compilation is called on it, and so on. By recursively compiling projects through the dependency graph, we are doing a part of the `dotnet build` job.

If a dependency is an Unresolved Package (not there), we have to resolve ourselves assembly paths by searching in the runtime directory or probing folders (see below). Here, we can still use other assembly collections which provide relative paths from which we extract assembly file names. Then, if a compile time assembly is also in the runtime assemblies collection, we only use its file name to search. Otherwise, we combine the `refs` sublfolder to the file name before searching in the runtime directory then fallback to probing folders.

If a dependency is an Ambient and Resolved Project (so a core project), normally all the resolved paths of its compilation assemblies (normally only one), as they have been resolved by the Project Model (this core project output path), are added to the references list used by the compiler. But, because e.g VS may output binaries in another folder (e.g artifacts), we first check for a resolved path if the file exists, then fallback to the runtime directory (because it's an ambient project).

If a dependency is a Resolved Project but not Ambient (so a module) or is a Resolved Package, all the resolved paths of its compilation assemblies, as they have been resolved by the Project Model, are added to the references list used by the compiler.

- **Parallel compilation**: Extension loading is done in parallel, therefore Module projects are also compiled in parallel. We use a dictionary of lock objects based on project names, this to prevent simultaneous compilations of the same project. We also use simple locks to prevent simultaneous writting of the same file.

##Dynamic Loading

##Probing Folders

- **Modules binaries folders**:

- **Dependencies folder**:



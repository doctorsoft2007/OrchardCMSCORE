##The needs (in construction)

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, Modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load their assemblies.

- **Dynamic loading** is therefore necessary, we need to load assemblies of all modules dependencies at runtime. By doing this we can also pass to the razor view engine all needed metadata references for views compilation (not described here). We also need to store specific assemblies in some probing folders, this to retrieve them in different contexts.

- **Dynamic compilation** is useful in a development context, you can update a module source file, a package or a core project and just hit F5, all dependent modules will be dynamically re-compiled. When a module has a dependency on a core project which is not part of `Orchard.Web`, if needed this non ambient core project is also re-compiled at runtime.

- **In production**, most of the time it will be better to use pre-compiled modules, but maybe useful sometimes to only have to publish a source file and let things go. Dynamic compilation and loading has already been tested in some environments where core projects are not there and / or there is no packages storage.

###Library assemblies

- **Compilation assemblies**: Referenced for all dependencies when compiling a project.

- **Default runtime assemblies**: Used for implementation at runtime, need to be loaded for all dependencies.

- **Compile only assemblies**: Most of the time a library has only one compilation assembly which is the same as the default runtime assembly. If not it's a compile only assembly because another assembly is used at runtime (see below).

- **Specific runtime assemblies**: Some libraries provides different implementations for different runtime environments.

- **Native assemblies**: Some libraries need to provide native implementations for different runtimes environments. Because Modules are not intended to be published themselves, we don't care about native outputs.

- **Resource assemblies**: Embedded resources in `{project}.resources.dll` files used by the resource manager.

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

    Compile only assemblies are outputed.

        root/refs/System.IO.dll

    Native assemblies for different targeted runtimes are ouputed.

        // runtimes/{rid}/native{assembly}.{lib-suffix}
        root/runtimes/osx/native/lmdb.dylib

    As with dotnet build, resources assemblies are ouputed.

        // {locale}/{project}.resources.dll
        root/fr/Orchard.Web.resources.dll

- **dotnet restore**: The `project.json` file is parsed to retrieve all dependencies, packages are updated, and, if needed, the `project.lock.json` file is written with updated dependencies metadata.

##Project model API

Dotnet cli use this API which allow to create a project context and e.g retrieve all its dependencies (referenced project / package libraries). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g by providing full paths). But here, the 2 `project.json` and `project.lock.json` files need to be there.

##Dependency model API

When a project is built to have an entry point and to preserve its compilation context, as needed by the main application to be executed and to compile razor views, a bigger assembly is generated and also a `{project}.deps.json` file. Then we can use the Dependency Model API to retrieve dependencies through this `.deps.json` file. It is less powerful than the Project Model because assets are not fully resolved (e.g by providing only relative paths). It doesn't rely on the 2 project json files but it needs the related `.deps.json`.

##Library contexts

- **Resolved project**: The 2 project json files are there.

- **Resolved package**: There is a package storage which contains this referenced package.

- **Unresolved project**: There is no project json files, e.g a core project not there in a published context.

- **Unresolved package**: There is no package storage (e.g in production) or it doesn't contains this package.

- **Precompiled project**: A resolved project but without any source files, but with all binaries.

- **Precompiled module**: An unresolved project but with all necessary binaries directly in its bin folder.

- **Ambient project**: The main project or a project belonging to its dependencies (not all core projects are ambient).

- **Ambient package**: A package belonging to the main project dependencies, so including those of the targeted framework.

- **Ambient assembly**: An assembly related to an ambient project or an ambient package.

##Dynamic compilation

- **Roslyn compiler**: We use the Roslyn csharp compiler `csc.exe` by referencing the `Microsoft.Net.Compilers.netcore` package. Then, at runtime we copy it in an executable `csc.dll` and create automatically a `csc.runtimeconfig.json` runtime config file, as `dotnet core setup` do to generate its outputs. Then, as `dotnet compile` do, we can execute `csc.dll`.

- **Compilation**: The implementation is mainly inspired of the `dotnet compile` source code which uses the project model API to resolve compilation options, output paths, source files, and all compilation assemblies of the project dependencies. Then, options and references are stored in a `dotnet-compile-csc.rsp` response file which is passed as an argument to the `csc.dll` compiler.

- **Configuration**: We need to pass to the project model API the current configuration `Debug` or `Release`. Here we use the configuration under which the main project has been built. To do this, we use the dependency model API on the main project (whithout relying on its project.json files) to grab its compilation options and then retrieve the configuration.

- **Conditions**: Only resolved projects which are not ambient can be compiled, and only once per startup because we check if it is not already compiled (e.g as a dependency). Then we check all compilation IO to see if it needs to be compiled. Here we do a part of the `dotnet build` job.

- **Compilation IO**: Compilation inputs are the 2 project json files, source files, eventually resources input files, and all compilation assemblies of the project dependencies. Compilation outputs are the generated project assembly (and its .pdb file), and eventually resources output assemblies. Notice that a project compilation output can be a compilation input when referenced as a dependency in another project.

- **Compilation IO checking**: We check the presence and the last write time of all resolved compilation IO files. A project need to be compiled if there is a missing IO or if an input is newer than the earliest output. But IO are not resolved in the same way, a source file resolution is already based on its presence. So, we can't only rely on this, e.g when a source file is moved, removed, or an old one is added. That's why, to see if anything else has changed, we use the previous compilation context stored in the `dotnet-compile-csc.rsp` response file.

- **Building**: Before compiling a project itself we parse all its dependencies. If a dependency is a non ambient and resolved project (here can be a core project), we recursively call Dynamic Compilation on it, and so on through the dependency graph, as `dotnet build` do.

    Then we resolve all compilation assembly paths from each dependency and add them to the references list needed for compilation. So, here we can do a complete compilation IO checking and, if needed, the project is compiled. If compilation succeeds and if there are resources input files, resources assemblies are also generated.

- **Dependencies**: For each dependency which is a project or a package library, we don't need to parse their own dependencies. They already belong to those of the project being compiled and whose all the dependency graph has been resolved. So, most of the time, a dependency has, in its related collections, only one compilation assembly which is the same as the default runtime assembly. So, here we don't always use the terms of compilation, runtime and collections.

    If a dependency is an **unresolved project**, we have to resolve ourselves the assembly path by searching in probing folders where the assembly may have been stored. First from the runtime directory where ambient assemblies are, then fallback to the project (being compiled) binary folder and the shared probing folder where we try to resolve the most recent assembly file. Here, we only need the file name which is the same as the library identity name.

    If a dependency is a **precompiled module**, because it has no project files, it is also checked as an unresolved project, so we first do the above. Then, if we can't resolve the assembly path, we fallback to the bin folder of this (possible) precompiled module. Here we use the project path of the dependency itself, and, in place of using the regular output path `bin/{config}/{framework}/{project}.dll`, we search directly in the bin folder `bin/{project}.dll`.

    If a dependency is an **unresolved package**, we have to resolve ourselves the assembly path as above from the same probing folders. From other collections, an unresolved package still provides relative paths from which we can extract the assembly name. Then, if the compile time assembly is a compile only assembly, we also combine the `refs` sublfolder to the file name.

    If a dependency is an **unresolved package** which is indirectly referenced through a **precompiled module**, we first do the above. Then, if we can't resolve the assembly path, we try to find a (possible) parent precompiled module which contains the package assembly in its bin folder. Here, we use the dependency parents collection of libraries which also have parents ... So, we can lookup for all parent projects, then search in their bin folder directly.

    If a dependency is a **precompiled project** with no source files but still the project json files, it is resolved but the compilation assemblies collection is empty. So, we need to resolve ourselves the assembly path from probing folders as above. But here, we don't search in the runtime directory because an ambient and resolved project is intended to have all source files. Then, if not resolved, we fallback to the regular output path of this precompiled project.

    If a dependency is an **ambient and resolved project**, the assembly path resolved by the project model could be added as it is to the compilation references. But, because e.g VS may output binaries in another folder (e.g artifacts), we first check if the assembly file exists, if not we fallback to the runtime directory (because it's an ambient assembly).

    If a dependency is a **non ambient and resolved Project** or is a **resolved package**, all the paths of its compilation assemblies, as resolved by the Project Model, are added to the compilation references.

- **Parallel compilation**: Extensions loading is done in parallel, therefore module projects are also compiled in parallel. We use a dictionary of lock objects based on project names, this to prevent simultaneous compilations of the same project. We also use simple locks to prevent simultaneous writting of the same file.

##Dynamic loading

- **Loading**: If a module is not already loaded, we parse all its dependencies and use the default `AssemblyLoadContext` to load in memory all the related runtime assemblies which are not ambient. We don't check for each individual assembly if it is already loaded, seems to be done internally, but here trying to load an ambient assembly would fail.

- **Dependencies**: If a module is a **resolved project**, assemblies paths of all its dependencies are resolved in the same way as when compiling. But here, we use runtime assemblies collections and, if a dependency is not resolved, we never fallback to the runtime directory because it only contains ambient assemblies.

    If a package provide different **specific runtime assemblies**, each one provides a non empty rid (runtime identifier). Then we look up for the first rid compatible to the current runtime environment, and use it to resolve the right runtime assembly path. This by using a relative path with this pattern `runtimes/{rid}/lib/{tfm}/{assembly}.dll`.

    If a module is a **precompiled module** without project json files, we can't use the project model API to resolve its dependencies. But all needed assemblies are intended to be in the module bin folder directly, and in the same structured way we are using to store assemblies. So here, we simply parse the module bin folder where default runtime assemblies are intended to be at the top level, specific runtime ones under the `runtimes` subfolder, and compile only ones in the `refs` subfolder.

##Dynamic storing

- **Probing folders**: Specific folders where binaries are stored and then can be retrieved in different contexts.

- **Structured probing folders**: In each probing folder, as it is done in the runtime directory by `dotnet publish`, default runtime assemblies are stored at the top level, compile only assemblies in a `refs` subfolder, resources assemblies in their related `{locale}` subfolders, and specific runtime assemblies by using this pattern `runtimes/{rid}/lib/{tfm}/{assembly}.dll`.

    The nuget package storage uses the same kind of patterns but not exactly, e.g compile only assemblies differ based on the targeted framework but not on the runtime environment. So, here they are all flattened in the `refs` subfolder.

- **Module binary folder**: When compiling a module at runtime, its assembly is naturally outputed in its binary folder `{project}/bin/{config}/{framework}/{project}.dll`. While loading a module, we also store here all non ambient assemblies of all its dependencies. For a **precompiled module**, all binaries are intended to be already stored in its bin folder directly and in the same structured way.

- **Shared dependencies folder**: Idem as above but in the `App_Data/dependencies` location and shared by all modules.

- **Runtime directory**: Here, we only use the runtime directory to store non ambient **resource assemblies**, this to be found by the resource manager.

- **Resolving**: Before loading a non ambient assembly of a module dependency, we first try to resolve it from its regular location (e.g in the package storage), then we fallback to the probing folders, the module binary folder and the shared probing folder, from where we try to find the most recent one.

- **Storing**: So, each non ambient assembly, of all dependencies of a given module, is first resolved and loaded, then, without knowing from which location it has been resolved, the assembly is stored in probing folders, the module binary folder and the shared probing folder. In each probing folder, we first check that the assembly is not already there or has an older date.

- **Updating**: The result of the above implementation is that we also update each probing folder with the last resolved assembly or with a more recent one found in another probing folder. This means e.g that an assembly in a module bin folder (even a precompiled module) can be updated, through the shared probing folder, with a more recent one coming from another module.

##Concerns

- **Files IO permissions**: E.g in production and particularly when writing in the runtime directory.

- **Files writting concurrency**: E.g in a multi instances context with a shared file system.

##Todo

- **Packaging**: To be able to share modules as packages, maybe we would have to implement some custom packaging processes, e.g inspired from what `dotnet pack` and `dotnet restore` do.

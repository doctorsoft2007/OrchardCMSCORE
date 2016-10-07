##Summary

- [The needs](#the-needs)
- [Library assemblies](#library-assemblies)
- [Dotnet commands](#dotnet-commands)
- [Project model API](#project-model-api)
- [Dependency model API](#dependency-model-api)
- [Library contexts](#library-contexts)
- [Dynamic compilation](#dynamic-compilation)
- [Dynamic loading](#dynamic-loading)
- [Dynamic storing](#dynamic-storing)
- [Views compilation](#views-compilation)
- [Concerns](#concerns)
- [Todo](#todo)

##The needs

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, these modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load their assemblies.

- [**Dynamic loading**](#dynamic-loading) is therefore necessary, we need to load [non ambient assemblies](#library-assemblies) of all modules dependencies at runtime. By doing this we can also pass to the razor view engine all needed metadata references for [views compilation](#views-compilation). We also need to [store at runtime](#dynamic-storing) these assemblies in some probing folders, this to retrieve them in different contexts.

- [**Dynamic compilation**](#dynamic-compilation) is useful in a development context, you can update a module source file, a package or a core project and just hit F5, all dependent modules will be dynamically re-compiled. When a module has a dependency on a core project which is not part of `Orchard.Web`, if needed this [non ambient project](#library-contexts) is also re-compiled at runtime.

- In a published context, most of the time it will be better to use [precompiled modules](#library-contexts), but maybe useful in some scenarios to only have to push e.g an updated source file. Dynamic compilation and loading have been tested in different contexts where core projects are not there and / or there is no packages storage.

###Library assemblies

A module project is a library which have dependencies on other libraries which are other projects and / or packages. When resolved through one of the [**dotnet model API**](#project-model-api), each library provides different collections related to different kind of assemblies.

- **Compilation assemblies**: Referenced for all dependencies when compiling a project.

- **Default runtime assemblies**: Used for implementation, need to be loaded for all dependencies.

- **Compile only assemblies**: Most of the time a library has only one compilation assembly which is the same as the default runtime one. Otherwise it's a compile only assembly and another one is used for implementation.

- **Specific runtime assemblies**: Some libraries provides different implementations for different runtime environments.

- **Native assemblies**: Some libraries need to provide native implementations for different runtimes environments. Because modules are not intended to be published themselves, we don't care about native outputs.

- **Resources assemblies**: `{project}.resources.dll` files used by the resource manager.

- **Ambient assembly**: An assembly related to an [**ambient project**](#library-contexts) or an [**ambient package**](#library-contexts).

##Dotnet commands

Here we describe how dotnet commands output binaries, this because we use the same structure to [**store at runtime**](#dynamic-storing) all [non ambient assemblies](#library-assemblies) of modules dependencies in probing folders, and then retrieve them in different contexts for compilation and loading.

- **dotnet compile** (used by dotnet build): The project assembly (and its .pdb file) is outputed.

        // {project}/bin/{config}/{framework}/{project}.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.dll

    When compiled as an executable and with its compilation context, a dependencies file is also generated and used for compilation, the project assembly is then bigger.

        // {project}/obj/{config}/{framework}/{project}dotnet-compile.deps.json
        Orchard.Web/obj/Debug/netcoreapp1.0/Orchard.Webdotnet-compile.deps.json

- **dotnet build**: It uses dotnet compile and all referenced projects are also built if needed.

    When compiled as an executable and with its compilation context, another dependencies file is outputed.

        // {project}/bin/{config}/{framework}/{project}.deps.json
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Web.deps.json

    [Runtime assemblies](#library-assemblies) of all referenced projects are also outputed.

        // {project}/bin/{config}/{framework}/{assembly}.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/Orchard.Data.dll

    [Runtime assemblies](#library-assemblies) of referenced packages are not outputed.

    [Resources assemblies](#library-assemblies) are ouputed.

        // {project}/bin/{config}/{framework}/{locale}/{project}.resources.dll
        Orchard.Web/bin/Debug/netcoreapp1.0/fr/Orchard.Web.resources.dll

- **dotnet publish**: All above assemblies are outputed at the root of the published target, plus the following.

    [Default runtime assemblies](#library-assemblies) of referenced packages which are not part of the targeted framework (see in `dotnet/shared/Microsoft.NETCore.App/{version}`) are also outputed.

        root/System.Data.Common.dll
        root/Microsoft.Extensions.DependencyModel.dll

    [Specific runtime assemblies](#library-assemblies) of referenced packages which are not part of the targeted framework are also outputed.

        // runtimes/{rid}/lib/{tfm}/{assembly}.dll
        root/runtimes/unix/lib/netstandard1.3/System.IO.Pipes.dll

    [Compile only assemblies](#library-assemblies) are outputed.

        root/refs/System.IO.dll

    [Native assemblies](#library-assemblies) for different targeted runtimes are ouputed.

        // runtimes/{rid}/native{assembly}.{lib-suffix}
        root/runtimes/osx/native/lmdb.dylib

    As with dotnet build, [resources assemblies](#library-assemblies) are ouputed.

        // {locale}/{project}.resources.dll
        root/fr/Orchard.Web.resources.dll

- **dotnet restore**: The `project.json` file is parsed to retrieve all dependencies, packages are updated, and, if needed, the `project.lock.json` file is written with updated dependencies metadata.

##Project model API

Dotnet cli use this API which allows to create a project context and e.g retrieve all its dependencies (referenced project / package libraries). It is powerful because, when a library is marked as resolved, all the related compilation, runtime and native collections are populated with fully resolved assets (e.g with full paths). But here, the 2 `project.json` and `project.lock.json` files need to be there.

##Dependency model API

When a project is built as an executable and with its compilation context, as needed by the main application to be executed and to compile razor views, a bigger assembly is generated and also a `{project}.deps.json` file. Then we can use the Dependency Model API to retrieve dependencies through this file. It is less powerful than the project model because assets are not fully resolved (e.g only relative paths). It doesn't rely on the 2 project json files but it needs the related `.deps.json` file.

##Library contexts

We use the [**project model API**](#project-model-api) to resolve a module and its dependencies. The module itself and its dependencies are all libraries which can be in different contexts. A core project / package may not be there in a published context, a module may be precompiled in production ...

- **Resolved project**: The 2 project json files are there.

- **Resolved package**: There is a packages storage which contains this referenced package.

- **Unresolved project**: There is no project json files, e.g a core project not there in a published context.

- **Unresolved package**: There is no packages storage (e.g in production) or it doesn't contains this package.

- **Precompiled project**: A resolved project but without any source files, but with all binaries.

- **Precompiled module**: An unresolved project but with all necessary binaries directly in its bin folder.

- **Ambient project**: The main project or a project belonging to its dependencies (not all core projects are ambient).

- **Ambient package**: A package belonging to the main project dependencies, so including those of the targeted framework.

##Dynamic compilation

- **Roslyn compiler**: We use the Roslyn csharp compiler `csc.exe` by referencing the `Microsoft.Net.Compilers.netcore` package. Then, at runtime we copy it in an executable `csc.dll` and automatically create a `csc.runtimeconfig.json` runtime config file, as `dotnet core setup` do to generate its outputs. Then, as `dotnet compile` do, we can execute `csc.dll`.

- **Native pdb writers**: According to the debug type option, the compiler outputs a `portable` or a `full` `.pdb` debug infos file. For the `full` option it needs native pdb writers, so we reference the `Microsoft.DiaSymReader.Native` package. When publishing the native assets are outputed to the runtime directory, and during development we do it at runtime from the package storage. We also create a fake `csc.deps.json` file only to reference these windows native pdb writers, this to allow the compiler to find them.


- **Compilation**: The implementation is mainly inspired of the `dotnet compile` source code which uses the [**project model API**](#project-model-api) to resolve compilation options, output paths, source files, and all [compilation assemblies](#library-assemblies) of the project dependencies. Then, options and references are stored in a `dotnet-compile-csc.rsp` response file which is passed as an argument to the `csc.dll` compiler.

- **Configuration**: We need to pass to the [**project model API**](#project-model-api) the current configuration `Debug` or `Release`. Here we use the configuration under which the main project has been built. To do this, we use the [**dependency model API**](#dependency-model-api) on the main project (so without relying on its project.json files) to grab its compilation options and then retrieve the configuration.

- **Conditions**: Only [**resolved projects**](#library-contexts) which are not ambient can be compiled, and only once per startup because we check if it is not already compiled (e.g as a dependency). Then we check all compilation IO to see if it needs to be compiled. Here we do a part of the dotnet build job.

- **Compilation IO**: Compilation inputs are the 2 project json files, source files, eventually resources input files, and all [compilation assemblies](#library-assemblies) of the project dependencies. Compilation outputs are the generated project assembly (and its .pdb file), and eventually [resources output assemblies](#library-assemblies). Notice that a project compilation output can be a compilation input when referenced as a dependency in another project.

- **Compilation IO checking**: We check the presence and the last write time of all resolved compilation IO files. A project need to be compiled if there is a missing IO or if an input is newer than the earliest output. But IO are not all resolved in the same way, a source file resolution is already based on its presence. So, we can't only rely on this, e.g when a source file is moved, removed, or an old one is added. That's why, to see if anything else has changed, we use the previous compilation context stored in the `dotnet-compile-csc.rsp` response file.

- **Building**: Before compiling a project itself we parse all its dependencies. If a dependency is a [**non ambient and resolved project**](#library-contexts) (here can be a core project), we recursively call dynamic compilation on it, and so on through the dependency graph, as `dotnet build` do.

    Then we resolve all [compilation assembly](#library-assemblies) paths from each dependency and add them to the references list needed for compilation. So, here we can do a complete compilation IO checking and, if needed, the project is compiled. If compilation succeeds and if there are resources input files, [resources assemblies](#library-assemblies) are also generated.

- **Dependencies**: For each dependency which is a project or a package library, we don't need to parse their own dependencies. They already belong to those of the project being compiled and whose all the dependency graph has been resolved. And, most of the time, a dependency has, in its related collections, only one [compilation assembly](#library-assemblies) which is the same as the [default runtime assembly](#library-assemblies). So, here when we only use the term of assembly, we are talking about this unique one.

    If a dependency is an [**unresolved project**](#library-contexts), we have to resolve ourselves the assembly path by searching in probing folders where the assembly may have been stored. First from the runtime directory where [ambient assemblies](#library-assemblies) are, then fallback to the project (being compiled) binary folder and the shared probing folder where we try to resolve the most recent assembly file. Here, we only need the file name which is the same as the library identity name.

    If a dependency is a [**precompiled module**](#library-contexts), because it has no project files, it is also checked as an unresolved project, so we first do the above. Then, if we can't resolve the assembly path, we fallback to the bin folder of this (possible) precompiled module. Here we use the project path of the dependency itself, and, in place of using the regular output path, we search in the bin folder directly.

    If a dependency is an [**unresolved package**](#library-contexts), we have to resolve ourselves the assembly path as above from the same probing folders. An unresolved package still provides relative paths from which we can extract the assembly name. Then, if the compile time assembly is a [compile only assembly](#library-assemblies), we also combine the `refs` sublfolder to the file name.

    If a dependency is an [**unresolved package**](#library-contexts) which is indirectly referenced through a [**precompiled module**](#library-contexts), we first do the above. Then, if we can't resolve the assembly path, we try to find a (possible) parent precompiled module which contains the package assembly. Here, we use the dependency parents collection which are libraries which also have parents ... So, we can look up for all parent projects and search in their bin folder directly.

    If a dependency is a [**precompiled project**](#library-contexts) with no source files but still the project json files, it is resolved but the [compilation assemblies](#library-assemblies) collection is empty. So, we need to resolve ourselves the assembly path from probing folders as above. But here, we don't search in the runtime directory because an [**ambient and resolved project**](#library-contexts) is intended to have all source files. Then, if not resolved, we fallback to the regular output path of this precompiled project.

    If a dependency is an [**ambient and resolved project**](#library-contexts), the assembly path resolved by the project model could be added as it is to the compilation references. But, because e.g VS may output binaries in another folder (e.g artifacts), we first check if the assembly file exists, if not we fallback to the runtime directory (because it's an ambient assembly).

    If a dependency is a [**non ambient and resolved project**](#library-contexts) or is a [**resolved package**](#library-contexts), all the paths of its [compilation assemblies](#library-assemblies), as resolved by the project model, are added to the compilation references.

- **Parallel compilation**: Extensions loading is done in parallel, therefore module projects are also compiled in parallel. We use a dictionary of lock objects based on project names, this to prevent simultaneous compilations of the same project. We also use simple locks to prevent simultaneous writings of the same file.

##Dynamic loading

- **Loading**: If a module is not already loaded, we parse all its dependencies and use the default `AssemblyLoadContext` to load in memory all the related [non ambient runtime assemblies](#library-assemblies). We don't check for each individual assembly if it is already loaded this way, seems to be done internally, but here trying to load an [ambient assembly](#library-assemblies) would fail.

- **Dependencies**: If a module is a [**resolved project**](#library-contexts), assemblies paths of all its dependencies are resolved in the same way as when [**compiling**](#dynamic-compilation). But here, we use [runtime assemblies](#library-assemblies) collections and, if a dependency is not resolved, we never fallback to the runtime directory because it only contains ambient assemblies.

    If a package provide different [specific runtime assemblies](#library-assemblies), each one provides a non empty rid (runtime identifier). Then we look up for the first rid compatible to the current runtime environment, and use it to resolve the right runtime assembly path.

    If a module is a [**precompiled module**](#library-contexts) without project json files, we can't use the [**project model API**](#project-model-api) to resolve its dependencies. But all needed assemblies are intended to be in the module bin folder directly, and through the same structure which is used to [**store assemblies**](#dynamic-storing). So here, we simply parse the module bin folder where [default runtime assemblies](#library-assemblies) are intended to be at the top level, [specific runtime ones](#library-assemblies) under the `runtimes` subfolder, and [compile only ones](#library-assemblies) in the `refs` subfolder.

##Dynamic storing

- **Probing folders**: Specific folders where binaries are stored and therefore can be retrieved in different contexts.

- **Structured probing folders**: In each probing folder we store assemblies in a structured way, as [**dotnet publish**](#dotnet-commands) outputs its binaries in the runtime directory, [default runtime assemblies](#library-assemblies) are stored at the top level, [compile only assemblies](#library-assemblies) in the `refs` subfolder, [resources assemblies](#library-assemblies) in their related `{locale}` subfolders, and specific [runtime assemblies](#library-assemblies) under the `runtimes` subfolder.

    The nuget packages storage uses the same kind of patterns but not exactly, e.g [compile only assemblies](#library-assemblies) differ based on the targeted framework but not on the runtime environment. So, here they are all flattened in the `refs` subfolder.

- **Module binary folder**: When compiling a module project at runtime, its main assembly is naturally outputed in its binary folder. While loading it, we also store here all [**non ambient assemblies**](#library-assemblies) of all its dependencies.

        {project}/bin/{config}/{framework}/{project}.dll
        {project}/bin/{config}/{framework}/{a-default-runtime}.dll
        {project}/bin/{config}/{framework}/refs/{a-compile-only}.dll
        {project}/bin/{config}/{framework}/{locale}/{a}.resources.dll
        {project}/bin/{config}/{framework}/runtimes/{rid}/lib/{tfm}/{a-specific-runtime}.dll

    For a [**precompiled module**](#library-contexts), all binaries are already stored in its bin folder directly.

        {module}/bin/{module}.dll
        {module}/bin/{a-default-runtime}.dll
        {module}/bin/refs/{a-compile-only}.dll
        {module}/bin/{locale}/{a}.resources.dll
        {module}/bin/runtimes/{rid}/lib/{tfm}/{a-specific-runtime}.dll

- **Shared dependencies folder**: Idem as above but in the `App_Data/dependencies` location and shared by all modules.

        root/App_Data/dependencies/{a-module}.dll
        root/App_Data/dependencies/{a-default-runtime}.dll
        root/App_Data/dependencies/refs/{a-compile-only}.dll
        root/App_Data/dependencies/{locale}/{a}.resources.dll
        root/App_Data/dependencies/runtimes/{rid}/lib/{tfm}/{a-specific-runtime}.dll

- **Runtime directory**: Here, we only store ourselves [non ambient resources assemblies](#library-assemblies), this to be found by the resource manager.

        root/{locale}/{a}.resources.dll

- **Resolving**: Before loading a module dependency assembly, we first try to resolve it from its regular location (e.g from the packages storage), then we fallback to the probing folders (module binary and shared probing folders) from where we try to find the most recent one.

- **Storing**: After loading a module dependency assembly, without knowing from which location it has been resolved, the assembly is stored in probing folders (module binary and shared probing folders) where we first check that the assembly is not already there or has an older date.

- **Updating**: The result of the above implementation is that we also update each probing folder with the last fully resolved assembly or with a more recent one found in another probing folder. This means e.g that an assembly in a module binary folder may be updated, through the shared probing folder, with a more recent one coming from another module.

##Views compilation

- When compiling a view, the razor view engine can reference all [ambient assemblies](#library-assemblies) which are part of the running application. But, modules are not [ambient projects](#library-contexts), so we first need to [load at runtime](#dynamic-loading) all the non ambient [runtime assemblies](#library-assemblies) of the module and its dependencies. Then we also have to pass all the related [compilation assemblies](#library-assemblies) as metadata references to the razor view engine.

- The razor view engine allows us on startup to pass these additional compilation references through a configuration option.

- [Dynamic loading](#dynamic-loading) uses, as roslyn, the same default `AssemblyLoadContext`, so we don't have any model / view data type mismatches.

##Concerns

- **Files IO permissions**: E.g in production and particularly when writing in the runtime directory.

- **Files writing concurrency**: E.g in a multi-instances context with a shared file system.

##Todo

- **Packaging**: To be able to share modules as packages, maybe we would have to implement some custom packaging processes, e.g inspired from what `dotnet pack` and `dotnet restore` do.

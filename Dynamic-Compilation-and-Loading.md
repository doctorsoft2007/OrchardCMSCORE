#The needs (in construction)

To be a modular application, `Orchard.Web` doesn't specify, in its `project.json` file, any dependencies on Modules and Themes which are individual portable library projects. So, when you build `Orchard.Web`, these modules are not compiled. You can manually build them (e.g from VS tools) but when you launch `Orchard.Web`, we still need to load modules assemblies.

- So, **Dynamic Loading** is necessary, we need to load all modules assemblies (and their dependencies) at runtime. By doing this we can also pass to the razor view engine all needed metadata references for modules views compilation (not described here). We also need to store modules specific assemblies in some probing folders, this to retrieve them in different contexts that will be described later.

- **Dynamic Compilation** is useful in a development context, e.g you can update a module source file and just hit F5, the module (and all dependent modules) will be dynamically re-compiled. Idem when you update a non ambient core project which is not part of `Orchard.Web`, it will be re-compiled with all dependent modules. Idem when you update some packages through `dotnet restore`.

- **In production**, most of the time it will be better to use pre-compiled modules, but maybe useful sometimes to only have to publish a source file and let things go, we will discuss later about the relevance to use dynamic compilation in a published context. Notice that it has been tested in some environments where core projects are not there and / or there is no packages storage.
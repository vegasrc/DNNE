# Native Exports for .NET

Prototype for a .NET managed assembly to expose a native export.

This work is inspired by work in the [Xamarin][xamarin_embed_link], [CoreRT][corert_feature_link], and [DllExport][dllexport_link] projects.

## Requirements

### Minimum

* [.NET 5.0](https://dotnet.microsoft.com/download/dotnet/5.0) or greater.
* [C99](https://en.cppreference.com/w/c/language/history) compatible compiler.

### DNNE NuPkg Requirements

**Windows:**
* [Visual Studio 2015](https://visualstudio.microsoft.com/) or greater.
    - The x86_64 version of the .NET runtime is the default install.
    - In order to target x86, the x86 .NET runtime must be explicitly installed.
* Windows 10 SDK - Installed with Visual Studio.
* x86, x86_64, ARM64 compilation supported.
    - The Visual Studio package containing the desired compiler architecture must have been installed.

**macOS:**
* [clang](https://clang.llvm.org/) compiler on the path.
* Current platform and environment paths dictate native compilation support.

**Linux:**
* [clang](https://clang.llvm.org/) compiler on the path.
* Current platform and environment paths dictate native compilation support.

## Exporting details

- The exported function must be marked `static` and `public`.

- The type exporting the function cannot be a nested type.

- Mark functions to export with [`UnmanagedCallersOnlyAttribute`][unmanagedcallersonly_link].

    ```CSharp
    public class Exports
    {
        [UnmanagedCallersOnlyAttribute(EntryPoint = "FancyName")]
        public static int MyExport(int a)
        {
            return a;
        }
    }
    ```

- The manner in which native exports are exposed is largely a function of the compiler being used. On the Windows platform an option exists to provide a [`.def`](https://docs.microsoft.com/cpp/build/reference/exports) file that permits customization of native exports. Users can provide a path to a `.def` file using the [`DnneWindowsExportsDef`](./src/msbuild/DNNE.props) MSBuild property. Note that if a `.def` file is provided no user functions will be exported by default.

<a name="nativeapi"></a>

## Native API

The native API is defined in [`src/platform/dnne.h`](./src/platform/dnne.h).

The `DNNE_ASSEMBLY_NAME` must be set during compilation to indicate the name of the managed assembly to load. The assembly name should not include the extension. For example, if the managed assembly on disk is called `ClassLib.dll`, the expected assembly name is `ClassLib`.

The generated source will need to be linked against the [`nethost`](https://docs.microsoft.com/dotnet/core/tutorials/netcore-hosting#create-a-host-using-nethosth-and-hostfxrh) library as either a static lib (`libnethost.[lib|a]`) or dynamic/shared library (`nethost.lib`). If the latter linking is performed, the `nethost.[dll|so|dylib]` will need to be deployed with the export binary or be on the path at run time.

The `set_failure_callback()` function can be used prior to calling an export to set a callback in the event runtime load or export discovery fails.

Failure to load the runtime or find an export results in the native library calling [`abort()`](https://en.cppreference.com/w/c/program/abort).

The `preload_runtime()` function can be used to preload the runtime. This may be desirable prior to calling an export to avoid the cost of loading the runtime during the first export dispatch.

## Exporting a managed function

1) Adorn the desired managed function with `UnmanagedCallersOnlyAttribute`.
    - Optionally set the `EntryPoint` property to indicate the name of the native export. See below for discussion of influence by calling convention.
    - If the `EntryPoint` property is `null`, the name of the mananged function is used. This default name will not include the namespace or class containing the function.
    - User supplied values in `EntryPoint` will not be modified or validated in any manner. This string will be consume by a C compiler and should therefore adhere to the [C language's restrictions on function names](https://en.cppreference.com/w/c/language/functions).
    - On the x86 platform only, multiple calling conventions exist and these often influence exported symbols. For example, see MSVC [C export decoration documentation](https://docs.microsoft.com/cpp/build/reference/decorated-names#FormatC). DNNE does not attempt to mitigate symbol decoration - even through `EntryPoint`. If the consuming application requires a specific export symbol _and_ calling convention that is decorated in a customized way, it is recommended to manually compile the generated source - see [`DnneBuildExports`](./src/msbuild/DNNE.props) - or if on Windows supply a `.def` file - see [`DnneWindowsExportsDef`](./src/msbuild/DNNE.props). Typically, setting the calling convention to `cdecl` for the export will address issues on any x86 platform.
        ```CSharp
        [UnmanagedCallersOnly(CallConvs = new []{typeof(System.Runtime.CompilerServices.CallConvCdecl)})]
        ```

1) Set the `<EnableDynamicLoading>true</EnableDynamicLoading>` property in the managed project containing the methods to export. This will produce a `*.runtimeconfig.json` that is needed to activate the runtime during export dispatch.

### Native code customization

The mapping of .NET types to their native representation is addressed by the concept of [blittability](https://docs.microsoft.com/dotnet/framework/interop/blittable-and-non-blittable-types). This approach however limits what can be expressed by the managed type signature when being called from an unmanaged context. For example, there is no way for DNNE to know how it should describe the following C struct in C# without being enriched with knowledge of how to construct marshallable types.

```C
struct some_data
{
    char* str;
    union
    {
        short s;
        double d;
    } data;
};
```

The following attributes can be used to enable the above scenario. They must be defined by the project in order to be used - DNNE provides no assembly to reference. Refer to [`ExportingAssembly`](./test/ExportingAssembly/Dnne.Attributes.cs) for an example.

```CSharp
namespace DNNE
{
    /// <summary>
    /// Provide C code to be defined early in the generated C header file.
    /// </summary>
    /// <remarks>
    /// This attribute is respected on an exported method declaration or on a parameter for the method.
    /// The following header files will be included prior to the code being defined.
    ///   - stddef.h
    ///   - stdint.h
    ///   - dnne.h
    /// </remarks>
    internal class C99DeclCodeAttribute : System.Attribute
    {
        public C99DeclCodeAttribute(string code) { }
    }

    /// <summary>
    /// Define the C type to be used.
    /// </summary>
    /// <remarks>
    /// The level of indirection should be included in the supplied string.
    /// </remarks>
    internal class C99TypeAttribute : System.Attribute
    {
        public C99TypeAttribute(string code) { }
    }
}
```

The above attributes can be used to manually define the native type mapping to be used in the export definition. For example:

```CSharp
public unsafe static class NativeExports
{
    public struct Data
    {
        public int a;
        public int b;
        public int c;
    }

    [UnmanagedCallersOnly]
    [DNNE.C99DeclCode("struct T{int a; int b; int c;};")]
    public static int ReturnDataCMember([DNNE.C99Type("struct T")] Data d)
    {
        return d.c;
    }

    [UnmanagedCallersOnly]
    public static int ReturnRefDataCMember([DNNE.C99Type("struct T*")] Data* d)
    {
        return d->c;
    }
}
```

In addition to providing declaration code directly, users can also supply `#include` directives for application specific headers. The [`DnneAdditionalIncludeDirectories`](./src/msbuild/DNNE.props) MSBuild property can be used to supply search paths in these cases. Consider the following use of the `DNNE.C99DeclCode` attribute.

```CSharp
[DNNE.C99DeclCode("#include <fancyapp.h>")]
```

## Generating a native binary using the DNNE NuPkg

1) The DNNE NuPkg is published on [NuGet.org](https://www.nuget.org/packages/DNNE), but can also be built locally.

    * Build the DNNE NuPkg locally by building [`create_package.proj`](./src/create_package.proj).

        `> dotnet build create_package.proj`

1) Add the NuPkg to the target managed project.

    * See [`DNNE.props`](./src/msbuild/DNNE.props) for the MSBuild properties used to configure the build process.

    * If NuPkg was built locally, remember to update the project's `nuget.config` to point at the local location of the recently built DNNE NuPkg.

    ```xml
    <ItemGroup>
      <PackageReference Include="DNNE" Version="1.*" />
    </ItemGroup>
    ```

1) Build the managed project to generate the native binary. The native binary will have a `NE` suffix and the system extension for dynamic/shared native libraries (i.e., `.dll`, `.so`, `.dylib`).
    * The [Runtime Identifier (RID)](https://docs.microsoft.com/dotnet/core/rid-catalog) is used to target a specific SDK.
    * For example, on Windows the `--runtime` flag or MSBuild [`RuntimeIdentifier`](https://docs.microsoft.com/dotnet/core/project-sdk/msbuild-props#runtimeidentifier) property can be used to target `win-x86` or `win-x64`.
    * The name of the native binary can be supplied by setting the MSBuild property `DnneNativeBinaryName`. It is incumbent on the setter of this property that it doesn't collide with the name of the managed assembly. Practially, this only impacts the Windows platform because managed and native binaries share the same extension (i.e., `.dll`).
    * A header file containing the exports will be placed in the output directory. The [`dnne.h`](./src/platform/dnne.h) will also be placed in the output directory.
    * On Windows an [import library (`.lib`)](https://docs.microsoft.com/windows/win32/dlls/dynamic-link-library-creation#using-an-import-library) will be placed in the output directory.

1) Deploy the native binary, managed assembly and associated `*.json` files for consumption from a native process.
    * Although not technically needed, the exports header and import library (Windows only) can be deployed with the native binary to make consumption easier.
    * Set the `DnneAddGeneratedBinaryToProject` MSBuild property to `true` in the project if it is desired to have the generated native binary flow with project references. Recall that the generated binary is bitness specific.

### Generate manually

1) Run the [dnne-gen](./src/dnne-gen) tool on the managed assembly.

1) Take the generated source from `dnne-gen` and the DNNE [platform](./src/platform) source to compile a native binary with the desired native exports. See the [Native API](#nativeapi) section for build details.

1) Deploy the native binary, managed assembly and associated `*.json` files for consumption from a native process.

### Experimental attribute

There are scenarios where updating `UnmanagedCallersOnlyAttribute` may take time. In order to enable independent development and experimentation, the `DNNE.ExportAttribute` is also respected. This type can be modified to suit one's needs and `dnne-gen` updated to respect those changes at code gen time. The user should define the following in their assembly. They can then modify the attribute and `dnne-gen` as needed.

``` CSharp
namespace DNNE
{
    internal class ExportAttribute : Attribute
    {
        public ExportAttribute() { }
        public string EntryPoint { get; set; }
    }
}
```

The calling convention of the export will be the default for the .NET runtime on that platform. See the description of [`CallingConvention.Winapi`](https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.callingconvention).

Using `DNNE.ExportAttribute` to export a method requires a `Delegate` of the appropriate type and name to be at the same scope as the export. The naming convention is `<METHODNAME>Delegate`. For example:

```CSharp
public class Exports
{
    public delegate int MyExportDelegate(int a);

    [DNNE.Export(EntryPoint = "FancyName")]
    public static int MyExport(int a)
    {
        return a;
    }
}
```

# Additional References

[dotnet repo](https://github.com/dotnet/runtime)

[`nethost` example](https://github.com/dotnet/samples/tree/master/core/hosting/HostWithHostFxr)

<!-- Links -->
[xamarin_embed_link]: https://docs.microsoft.com/xamarin/tools/dotnet-embedding/release-notes/preview/0.4
[corert_feature_link]: https://github.com/dotnet/corert/tree/master/samples/NativeLibrary
[dllexport_link]: https://github.com/3F/DllExport
[csharp_funcptr_link]: https://github.com/dotnet/csharplang/blob/master/proposals/function-pointers.md
[unmanagedcallersonly_link]: https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.unmanagedcallersonlyattribute

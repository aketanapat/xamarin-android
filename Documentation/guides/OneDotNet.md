# .NET 6 and Xamarin.Android

_NOTE: this document is very likely to change, as the requirements for
.NET 6 are better understood._

A .NET 6 project for a Xamarin.Android application will look something
like:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0-android</TargetFramework>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
</Project>
```

For a "library" project, you would omit the `$(OutputType)` property
completely or specify `Library`.

See the [Target Framework Names in .NET 5][net5spec] spec for details.

[net5spec]: https://github.com/dotnet/designs/blob/5e921a9dc8ecce33b3195dcdb6f10ef56ef8b9d7/accepted/2020/net5/net5.md

## Consolidation of binding projects

In .NET 6, there will no longer be a concept of a [binding
project][binding] as a separate project type. Any of the MSBuild item
groups or build actions that currently work in binding projects will
be supported through a .NET 6 Android application or library.

For example, a binding library could look like:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net5.0-android</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <TransformFile Include="Transforms\Metadata.xml" />
    <EmbeddedJar Include="Jars\foo.jar" />
  </ItemGroup>
</Project>
```

This will bind C# types for the Java types found in `foo.jar` using
the metadata fixups from `Metadata.xml`.

[binding]: https://docs.microsoft.com/xamarin/android/platform/binding-java-library/

## .NET Configuration Files

No support for [configuration files][config] such as `Foo.dll.config`
or `Foo.exe.config` is available in Xamarin.Android projects targeting
.NET 6. [`<dllmap>`][dllmap] configuration elements are not supported
in .NET Core at all, and other element types for compatibility
packages like [System.Configuration.ConfigurationManager][nuget] have
never been supported in Xamarin.Android projects.

[config]: https://docs.microsoft.com/dotnet/framework/configure-apps/
[nuget]: https://www.nuget.org/packages/System.Configuration.ConfigurationManager/

## Changes to MSBuild tasks

In .NET 6 the behavior of the following MSBuild tasks will change, but
"legacy" projects will stay the same:

* `<ValidateJavaVersion/>` - used to require Java 1.6, 1.7, or 1.8
  based on the version of the Android Build Tools or
  `$(TargetFrameworkVersion)`. .NET 6 will require Java 1.8.

* `<ResolveAndroidTooling/>` - used to support the
  `$(AndroidUseLatestPlatformSdk)` setting or multiple
  `$(TargetFrameworkVersion)`. .NET 6 will always target the latest
  Android APIs for `Mono.Android.dll`.

## Changes to MSBuild properties

`$(AndroidSupportedAbis)` should not be used. Instead of:

```xml
<PropertyGroup>
  <!-- Used in legacy Xamarin.Android projects -->
  <AndroidSupportedAbis>armeabi-v7a;arm64-v8a;x86;x86_64</AndroidSupportedAbis>
</PropertyGroup>
```

Instead use .NET's concept of [runtime identifiers][rids]:

```xml
<PropertyGroup>
  <!-- Used going forward in .NET -->
  <RuntimeIdentifiers>android.21-arm;android.21-arm64;android.21-x86;android.21-x64</RuntimeIdentifiers>
</PropertyGroup>
```

`$(AndroidUseIntermediateDesignerFile)` will be `True` by default.

`$(AndroidBoundExceptionType)` will be `System` by default.  This will
[alter the types of exceptions thrown from various methods][abet-sys] to
better align with existing .NET 6 semantics, at the cost of compatibility with
previous Xamarin.Android releases.

`$(AndroidClassParser)` will be `class-parse` by default. `jar2xml`
will not be supported.

`$(AndroidDexTool)` will be `d8` by default. `dx` will not be
supported.

`$(AndroidCodegenTarget)` will be `XAJavaInterop1` by default.
`XamarinAndroid` will not be supported.

`$(DebugType)` will be `portable` by default. `full` and `pdbonly`
will not be supported.

`$(MonoSymbolArchive)` will be `False`, since `mono-symbolicate` is
not yet supported.

If Java binding is enabled with `@(InputJar)`, `@(EmbeddedJar)`,
`@(LibraryProjectZip)`, etc. then `$(AllowUnsafeBlocks)` will default
to `True`.

Referencing an Android Wear project from an Android application will
not be supported:

```xml
<ProjectReference Include="..\Foo.Wear\Foo.Wear.csproj">
  <IsAppExtension>True</IsAppExtension>
</ProjectReference>
```

[rids]: https://docs.microsoft.com/dotnet/core/rid-catalog
[abet-sys]: https://github.com/xamarin/xamarin-android/issues/4127

## Default file inclusion

Default Android related file globbing behavior is defined in [`AutoImport.props`][autoimport].
This behavior can be disabled for Android items by setting `$(EnableDefaultAndroidItems)` to `false`, or
all default item inclusion behavior can be disabled by setting `$(EnableDefaultItems)` to `false`.

[autoimport]: https://github.com/dotnet/designs/blob/4703666296f5e59964961464c25807c727282cae/accepted/2020/workloads/workload-resolvers.md#workload-props-files

## Linker (ILLink)

.NET 5 and higher has new [settings for the linker][linker]:

* `<PublishTrimmed>true</PublishTrimmed>`
* `<TrimMode>copyused</TrimMode>` - Enable assembly-level trimming.
* `<TrimMode>link</TrimMode>` - Enable member-level trimming.

In Android application projects by default, `Debug` builds will not
use the linker and `Release` builds will set `PublishTrimmed=true` and
`TrimMode=link`. `TrimMode=copyused` is the default the dotnet/sdk,
but it doesn't not seem to be appropriate for mobile applications.
Developers can still opt into `TrimMode=copyused` if needed.

If the legacy `AndroidLinkMode` setting is used, both `SdkOnly` and
`Full` will default to equivalent linker settings:

* `<PublishTrimmed>true</PublishTrimmed>`
* `<TrimMode>link</TrimMode>`

With `AndroidLinkMode=SdkOnly` only BCL and SDK assemblies marked with
`%(Trimmable)` will be linked at the member level.
`AndroidLinkMode=Full` will set `%(TrimMode)=link` on all .NET
assemblies similar to the example in the [trimming
documentation][linker-full].

It is recommended to migrate to the new linker settings, as
`AndroidLinkMode` will eventually be deprecated.

[linker]: https://docs.microsoft.com/dotnet/core/deploying/trimming-options
[linker-full]: https://docs.microsoft.com/dotnet/core/deploying/trimming-options#trimmed-assemblies

## dotnet cli

There are currently a few "verbs" we are aiming to get working in
Xamarin.Android:

    dotnet build
    dotnet publish
    dotnet run

Currently in .NET 5 console apps, `dotnet publish` is where all the
work to produce a self-contained "app" happens:

* The linker via the `<IlLink/>` MSBuild task
* .NET Core's version of AOT, named "ReadyToRun"

https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-core-3-0#readytorun-images

However, for Xamarin.Android, `dotnet build` should produce something
that is runnable. This basically means we need to create an `.apk` or
`.aab` file during `build`. We will need to reorder any MSBuild tasks
from the dotnet SDK, so that they will run during `build`.

This means Xamarin.Android would run:

* Run `aapt` to generate `Resource.designer.cs` and potentially emit
  build errors for issues in `@(AndroidResource)` files.
* Compile C# code
* Run the new [ILLink][illink] MSBuild target for linking.
* Generate java stubs, `AndroidManifest.xml`, etc. This must happen
  after the linker.
* Compile java code via `javac`
* Convert java code to `.dex` via d8/r8
* Create an `.apk` or `.aab` and sign it

`dotnet publish` will be reserved for publishing an app for Google
Play, ad-hoc distribution, etc. It could be able to sign the `.apk` or
`.aab` with different keys. As a starting point, this will currently
copy the output to a `publish` directory on disk.

`dotnet run` can be used to launch applications on a
device or emulator via the `--project` switch:

    dotnet run --project HelloAndroid.csproj

[illink]: https://github.com/mono/linker/blob/master/src/linker/README.md

### Preview testing

The following instructions can be used for early preview testing.

  1) Install the [latest .NET 5 preview][0]. Preview 4 or later is required.

  2) Create a `nuget.config` file that has a package source pointing to
     local packages or `xamarin-impl` feed, as well as the .NET 5 feed:

```xml
<configuration>
  <packageSources>
    <add key="xamarin-impl" value="https://pkgs.dev.azure.com/azure-public/vside/_packaging/xamarin-impl/nuget/v3/index.json" />
    <add key="dotnet5" value="https://dnceng.pkgs.visualstudio.com/public/_packaging/dotnet5/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

  3) Open an existing Android project (ideally something minimal) and
    tweak it as shown below. The version should match the version of the
    packages you want to use:

```xml
<Project Sdk="Microsoft.Android.Sdk/10.0.100">
  <PropertyGroup>
    <TargetFramework>net5.0-android</TargetFramework>
    <RuntimeIdentifier>android.21-arm64</RuntimeIdentifier>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
</Project>
```

  4) Build (and optionally run) the project:

    dotnet build *.csproj -t:Run

## Package Versioning Scheme

This is the package version scheme: `OS-Major.OS-Minor.InternalRelease[-prereleaseX]+sha.1b2c3d4`.

* Major: The major OS version.
* Minor: The minor OS version.
* Patch: Our internal release version based on `100` as a starting point.
    * Service releases will bump the last two digits of the patch version
    * Feature releases will round the patch version up to the nearest 100
      (this is the same as bumping the first digit of the patch version, and
      resetting the last two digits to 0).
    * This follows [how the dotnet SDK does it][1].
* Pre-release: Optional (e.g.: Android previews, CI, etc.)
    * For CI we use a `ci` prefix + the branch name (cleaned up to only be
      alphanumeric) + the commit distance (number of commits since any of the
      major.minor.patch versions changed).
        * Example: `10.0.100-ci.master.1234`
        * Alphanumeric means `a-zA-Z0-9-`: any character not in this range
          will be replaced with a `-`.
    * Pull requests have `pr` prefix, followed by `gh`+ PR number + commit
      distance.
        * Example: `10.1.200-ci.pr.gh3333.1234`
    * If we have a particular feature we want people to subscribe to (such as
      an Android preview release), we publish previews with a custom pre-release
      identifier:
        * Example: `10.1.100-android-r.beta.1`
        * This way people can sign up for only official previews, by
          referencing `*-android-r.beta.*`
        * It's still possible to sign up for all `android-r` builds, by
          referencing `*-ci.android-r.*`
* Build metadata: Required Hash
    * This is `sha.` + the short commit hash.
        * Use the short hash because the long hash is quite long and
          cumbersome. This leaves the complete version open for duplication,
          but this is extremely unlikely.
    * Example: `10.0.100+sha.1a2b3c`
    * Example (CI build): `10.0.100-ci.master.1234+sha.1a2b3c`
    * Since the build metadata is required for all builds, we're able to
      recognize incomplete version numbers and determine if a particular
      version string refers to a stable version or not.
        * Example: `10.0.100`: incomplete version
        * Example: `10.0.100+sha.1a2b3c`: stable
        * Example: `10.0.100-ci.d17-0.1234+sha.1a2b3c`: CI build
        * Example: `10.0.100-android-r.beta.1+sha.1a2b3c`: official
          preview
            * Technically it's possible to remove the prerelease part, but
              we’d still be able to figure out it’s not a stable version by
              using the commit hash.


[0]: https://github.com/dotnet/installer#installers-and-binaries
[1]: https://github.com/dotnet/designs/blob/master/accepted/2018/sdk-version-scheme.md

---
title: dotnet nuget push command
description: The dotnet nuget push command pushes a package to the server and publishes it.
author: karann-msft
ms.date: 02/14/2020
---
# dotnet nuget push

**This article applies to:** ✔️ .NET Core 3.1 SDK and later versions

## Name

`dotnet nuget push` - Pushes a package to the server and publishes it.

## Synopsis

```dotnetcli
dotnet nuget push [<ROOT>] [-d|--disable-buffering] [--force-english-output]
    [--interactive] [-k|--api-key <API_KEY>] [-n|--no-symbols]
    [--no-service-endpoint] [-s|--source <SOURCE>] [--skip-duplicate]
    [-sk|--symbol-api-key <API_KEY>] [-ss|--symbol-source <SOURCE>]
    [-t|--timeout <TIMEOUT>]
    [--configfile <FILE>]

dotnet nuget push -h|--help
```

## Description

The `dotnet nuget push` command pushes a package to the server and publishes it. The push command uses server and credential details found in the system's NuGet config file or chain of config files. For more information on config files, see [Configuring NuGet Behavior](/nuget/consume-packages/configuring-nuget-behavior). NuGet's default configuration is obtained by loading *%AppData%\NuGet\NuGet.config* (Windows) or *$HOME/.nuget/NuGet/NuGet.Config* (Linux/macOS), then loading any *nuget.config* or *.nuget\nuget.config* starting from the root of drive and ending in the current directory.

The command pushes an existing package. It doesn't create a package. To create a package, use [`dotnet pack`](dotnet-pack.md).

### Hierarchical folder structure

This command can store packages in a hierarchical folder structure, which is recommended to optimize performance. It stores packages in a hierarchical folder structure when publishing to a local folder (feed), like `nuget add` does, if there already is at least one package in the feed that is in a hierarchical folder structure. If the feed has a hierarchical folder structured package already in it, `dotnet nuget push` respects that structure. So, if you want to publish to a local feed using the .NET CLI instead of the NuGet CLI:

* Before you publish the first package, go to your global packages folder, at *%userprofile%.nuget\packages*, and select the root folder of a package id. It can be any package that is not part of a framework, like .NET standard or ASP.NET.
* Copy the selected package folder into the root folder of the local feed.
* Use `dotnet nuget push` to publish your package to the local feed.
* You can now delete the folder you previously copied in, and you can freely use `dotnet nuget push` to publish to your local feed.

Alternatively, use the NuGet CLI for the first package, then you can use `dotnet nuget push` for the rest. For more information, see [Local feeds](/nuget/hosting-packages/local-feeds).  

## Arguments

- **`ROOT`**

  Specifies the file path to the package to be pushed.

## Options

- **`-d|--disable-buffering`**

  Disables buffering when pushing to an HTTP(S) server to reduce memory usage.

- **`--force-english-output`**

  Forces the application to run using an invariant, English-based culture.

[!INCLUDE [help](../../../includes/cli-help.md)]

[!INCLUDE [interactive](../../../includes/cli-interactive-3-0.md)]

- **`-k|--api-key <API_KEY>`**

  The API key for the server.

- **`-n|--no-symbols`**

  Doesn't push symbols (even if present).

- **`--no-service-endpoint`**

  Doesn't append "api/v2/package" to the source URL.
- **`-s|--source <SOURCE>`**

  Specifies the server URL. NuGet identifies a UNC or local folder source and simply copies the file there instead of pushing it using HTTP.
  > [!IMPORTANT]
  > Starting with NuGet 3.4.2, this is a mandatory parameter unless the NuGet config file specifies a `DefaultPushSource` value. For more information, see [Configuring NuGet behavior](/nuget/consume-packages/configuring-nuget-behavior).

- **`--skip-duplicate`**

  When pushing multiple packages to an HTTP(S) server, treats any 409 Conflict response as a warning so that other pushes can continue.

- **`-sk|--symbol-api-key <API_KEY>`**

  The API key for the symbol server.

- **`-ss|--symbol-source <SOURCE>`**

  Specifies the symbol server URL.

- **`-t|--timeout <TIMEOUT>`**

  Specifies the timeout for pushing to a server in seconds. Defaults to 300 seconds (5 minutes). Specifying 0 applies the default value.

- **`--configfile`**

  The NuGet configuration file (*nuget.config*) to use. If specified, only the settings from this file will be used. If not specified, the hierarchy of configuration files from the current directory will be used. For more information, see [Common NuGet Configurations](/nuget/consume-packages/configuring-nuget-behavior).

## Examples

- Push *foo.nupkg* to the default push source specified in the NuGet config file, using an API key:

  ```dotnetcli
  dotnet nuget push foo.nupkg -k 4003d786-cc37-4004-bfdf-c4f3e8ef9b3a
  ```

- Push *foo.nupkg* to the official NuGet server, specifying an API key:

  ```dotnetcli
  dotnet nuget push foo.nupkg -k 4003d786-cc37-4004-bfdf-c4f3e8ef9b3a -s https://api.nuget.org/v3/index.json
  ```
  
- Push *foo.nupkg* to the custom push source `https://customsource`, specifying an API key:

  ```dotnetcli
  dotnet nuget push foo.nupkg -k 4003d786-cc37-4004-bfdf-c4f3e8ef9b3a -s https://customsource/
  ```

- Push *foo.nupkg* to the default push source specified in the NuGet config file:

  ```dotnetcli
  dotnet nuget push foo.nupkg
  ```

- Push *foo.symbols.nupkg* to the default symbols source:

  ```dotnetcli
  dotnet nuget push foo.symbols.nupkg
  ```

- Push *foo.nupkg* to the default push source specified in the NuGet config file, with a 360-second timeout:

  ```dotnetcli
  dotnet nuget push foo.nupkg --timeout 360
  ```

- Push all *.nupkg* files in the current directory to the default push source specified in the NuGet config file:

  ```dotnetcli
  dotnet nuget push "*.nupkg"
  ```

  > [!NOTE]
  > If this command doesn't work, it might be due to a bug that existed in older versions of the SDK (.NET Core 2.1 SDK and earlier versions).
  > To fix this, upgrade your SDK version or run the following command instead:
  > `dotnet nuget push "**/*.nupkg"`
  
  > [!NOTE]
  > The enclosing quotes are required for shells such as bash that perform file globbing. For more information, see [NuGet/Home#4393](https://github.com/NuGet/Home/issues/4393#issuecomment-667618120).

- Push all *.nupkg* files to the default push source specified in the NuGet config file, even if a 409 Conflict response is returned by an HTTP(S) server:

  ```dotnetcli
  dotnet nuget push "*.nupkg" --skip-duplicate
  ```

- Push all *.nupkg* files in the current directory to a local feed directory:

  ```dotnetcli
  dotnet nuget push "*.nupkg" -s c:\mydir
  ```
  
- For pushing to Azure Artifacts, [see Azure Artifacts' push documentation](/azure/devops/artifacts/nuget/dotnet-exe#publish-packages).

---
layout: post
title: Configuring your C# project for semver
description: Node.js command line tool which configures your C# project to allow for easy versioning
tags: c# .NET semantic versioning semver
---

A while ago I got tired of manually configuring my C# projects to allow for easy [semantic versioning](<http://semver.org/>) and decided to automate the steps involved.

I created a small Node.js command line tool which does following things:

1. Creates a text file (default: `version.txt`) in the project folder which contains the version number (default: `0.1.0`).
2. Copies a `BuildCommon.targets` file to `{project-root}/build/BuildCommon.targets` which is used to extend the build/clean process.
3. Adds a reference to the `BuildCommon.targets` file in the project file.
4. Comments out existing version related attributes in `AssemblyInfo.cs`.

I can then change the version number in `version.txt` to `1.0.0-rc1`, `1.0.0` etc., build the project, and the assembly will have the correct version set. 

# Extending the build process

The build process is extended by overriding the `CompileDependsOn` property and injecting the custom task before the original property.

The `UpdateVersion` task creates `AssemblyVersion.cs` in the intermediate output path (`$(IntermediateOutputPath)`) and adds the version related attributes to it. The values are read from the text file created in step 1 above.

```xml
<PropertyGroup>
  <CompileDependsOn>
    UpdateVersion;
    $(CompileDependsOn);
  </CompileDependsOn>
</PropertyGroup>
``` 

The created file is also added to the build process.

```xml
<Compile Include="$(IntermediateOutputPath)AssemblyVersion.cs" />
```

# Extending the clean process

MSBuild maintains a list of the files that should be removed when cleaning the project, in the item list `FileWrites`.

```xml
<FileWrites Include="$(IntermediateOutputPath)AssemblyVersion.cs"/>
```

# Source code and npm package
[GitHub](<https://github.com/trydis/csversion>) 
[npm package](<https://www.npmjs.com/package/csversion>) 
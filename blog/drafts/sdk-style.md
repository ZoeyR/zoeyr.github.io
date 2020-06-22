With the the release of [Visual Studio 2019](https://devblogs.microsoft.com/visualstudio/visual-studio-2019-release-candidate-rc-now-available/) coming up, I thought it was a good idea to refactor my extenions. One of the changes I wanted to make was to transition the projects from the traditional .csproj format to the newer "[SDK-style](https://www.hanselman.com/blog/UpgradingAnExistingNETProjectFilesToTheLeanNewCSPROJFormatFromNETCore.aspx)" .csproj. For those who don't know, the "SDK-style" .csproj offers a much simpler project file layout and performs much better when running inside of VS.

**Disclaimer**: Before proceeding any further, I would like to note that using SDK-style projects for your VSIX projects is _completely unsupported_. There are many features missing and it is up to you to decide if the trade-offs are worth it.

# The Setup

To get started, we will be converting an [example](https://github.com/ZoeyR/SdkDemoVsix) project from a traditional .csproj to Sdk-style .csproj. Clone the repo and switch to the "before" branch if you want to follow along. You should have a folder structure similar to this:

```
SdkDemoVsix
├── SdkDemoVsix.sln
└── SdkDemoVsix
    ├── Properties
    │   └── AssemblyInfo.cs
    ├── Resources
    │   ├── BasicToolWindowCommand.png
    │   └── BasicToolWindowPackage.ico
    ├── BasicToolWindow.cs
    ├── BasicToolWindowCommand.cs
    ├── BasicToolWindowControl.xaml
    ├── BasicToolWindowControl.xaml.cs
    ├── BasicToolWindowPackage.cs
    ├── BasicToolWindowPackage.vsct
    ├── SdkDemoVsix.csproj
    ├── VSPackage.resx
    ├── packages.config
    └── source.extension.vsixmanifest
```

# Deleting redundant files

Sdk-style projects take care of many things for us. So the first thing to do is delete any files that will be made redundant by the migration. In our case we only have two files to delete, `packages.config` and `AssemblyInfo.cs`.

# Writing a new .csproj

The next step will be to create a new .csproj file. Since this is the most complicated, and error-prone, part of the process I will present the entire file and then go through each section in detail.

```xml
<Project>

  <PropertyGroup>
    <VSToolsPath Condition="'$(VSToolsPath)' == ''">$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v$(VisualStudioVersion)</VSToolsPath>
  </PropertyGroup>

  <Import Sdk="Microsoft.NET.Sdk" Project="Sdk.props" />

  <PropertyGroup>
    <TargetFramework>net46</TargetFramework>
    <OutputType>Library</OutputType>
    <AssemblyName>SdkDemoVsix</AssemblyName>
    <UseCodebase>true</UseCodebase>
    <GeneratePkgDefFile>true</GeneratePkgDefFile>
    <IncludeAssemblyInVSIXContainer>true</IncludeAssemblyInVSIXContainer>
    <IncludeDebugSymbolsInVSIXContainer>true</IncludeDebugSymbolsInVSIXContainer>
    <IncludeDebugSymbolsInLocalVSIXDeployment>true</IncludeDebugSymbolsInLocalVSIXDeployment>
    <CopyBuildOutputToOutputDirectory>true</CopyBuildOutputToOutputDirectory>
    <CopyOutputSymbolsToOutputDirectory>false</CopyOutputSymbolsToOutputDirectory>
  </PropertyGroup>

  <ItemGroup>
    <Page Include="**\*.xaml" Exclude="App.xaml">
      <SubType>Designer</SubType>
      <Generator>MSBuild:Compile</Generator>
    </Page>
    <Compile Update="**\*.xaml.cs">
      <DependentUpon>%(Filename)</DependentUpon>
      <SubType>Code</SubType>
    </Compile>
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Update="VSPackage.resx">
      <MergeWithCTO>true</MergeWithCTO>
      <ManifestResourceName>VSPackage</ManifestResourceName>
    </EmbeddedResource>
  </ItemGroup>

  <ItemGroup>
    <VSCTCompile Include="BasicToolWindowPackage.vsct">
      <ResourceName>Menus.ctmenu</ResourceName>
    </VSCTCompile>
    <Content Include="Resources\BasicToolWindowCommand.png" />
    <Content Include="Resources\BasicToolWindowPackage.ico" />
  </ItemGroup>

  <Import Sdk="Microsoft.NET.Sdk" Project="Sdk.targets" />
  <Import Project="$(VSToolsPath)\VSSDK\Microsoft.VsSDK.targets" />
</Project>
```

That's it! Our .csproj file went from 207 lines to a very meager 55\. You'll notice that the new .csproj file contains no C# files and the reference section is missing. This is because the SDK-style project system takes care of all this for us! Now lets cover each section in detail to explain what's going on.

```xml
<Import Sdk="Microsoft.NET.Sdk" Project="Sdk.props" />
```

This line imports the standard .NET Sdk properties, normally this would end up getting imported in the first line of the .csproj like this: `<Project Sdk="Microsoft.NET.Sdk">`. The reason that's not done here is because VSIX projects are very sensitive the order in which their props and targets files are imported, and using the Project syntax takes that control away.

```xml
<PropertyGroup>
  <TargetFramework>net46</TargetFramework>
  <OutputType>Library</OutputType>
  <AssemblyName>SdkDemoVsix</AssemblyName>
  <UseCodebase>true</UseCodebase>
  <GeneratePkgDefFile>true</GeneratePkgDefFile>
  <IncludeAssemblyInVSIXContainer>true</IncludeAssemblyInVSIXContainer>
  <IncludeDebugSymbolsInVSIXContainer>true</IncludeDebugSymbolsInVSIXContainer>
  <IncludeDebugSymbolsInLocalVSIXDeployment>true</IncludeDebugSymbolsInLocalVSIXDeployment>
  <CopyBuildOutputToOutputDirectory>true</CopyBuildOutputToOutputDirectory>
  <CopyOutputSymbolsToOutputDirectory>false</CopyOutputSymbolsToOutputDirectory>
</PropertyGroup>
```

This segment is copied almost verbatim from a standard VSIX project file, the biggest different is changing `<TargetFrameworks>` to `<TargetFramework>`, the rest of the properties are identical to their original counterparts and serve the same purpose.

```xml
<ItemGroup>
  <Page Include="**\*.xaml" Exclude="App.xaml">
    <SubType>Designer</SubType>
    <Generator>MSBuild:Compile</Generator>
  </Page>
  <Compile Update="**\*.xaml.cs">
    <DependentUpon>%(Filename)</DependentUpon>
    <SubType>Code</SubType>
  </Compile>
</ItemGroup>
```

By default xaml files do not build correctly with Sdk-style projects, this section of the .csproj automatically collects all .xaml files in your project and ensures the correct compilation rules are applied to them.

```xml
<ItemGroup>
  <EmbeddedResource Update="VSPackage.resx">
    <MergeWithCTO>true</MergeWithCTO>
    <ManifestResourceName>VSPackage</ManifestResourceName>
  </EmbeddedResource>
</ItemGroup>

<ItemGroup>
  <VSCTCompile Include="BasicToolWindowPackage.vsct">
    <ResourceName>Menus.ctmenu</ResourceName>
  </VSCTCompile>
  <Content Include="Resources\BasicToolWindowCommand.png" />
  <Content Include="Resources\BasicToolWindowPackage.ico" />
</ItemGroup>
```

Embedded resources and VSCT files need special compilation steps that are specific to VSIX projects. The above rules apply the correct compliation rules for VSIX projects. These entries can be copied verbatim from a non-Sdk-style project.

```xml
<Import Sdk="Microsoft.NET.Sdk" Project="Sdk.targets" />
<Import Project="$(VSToolsPath)\VSSDK\Microsoft.VsSDK.targets" />
```

These last two lines import the build targets so that MSBuild can actually compile the VSIX. The order of these two lines is incredibly important. VSSDK targets _must_ come after the Sdk targets or the VSIX will not compile correctly.

# Reintroducing nuget packages

Now that the project file is in SDK format, it is time to reintroduce our VS nuget packages. Rather than adding each VS assembly directy we will use the new Microsoft.VisualStudio.Sdk meta package. This package makes it super easy to refer to all the assemblies present in a specific version of Visual Studio. Right now the SDK package is not visibly listed on nuget.org, but if you know the version you can manually add it anyways. Adding the SDK to your project is as simple as putting the following lines in your .csproj:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.VisualStudio.SDK" Version="15.9.3" />
  <PackageReference Include="Microsoft.VSSDK.BuildTools" Version="15.9.3039" />
</ItemGroup>
```

The VSSDK.Buildtools package is separate from the SDK since the build tools version does not have to match up with the minimum version of Visual Studio you wish to support.

# Restoring F5 functionality

There's one last piece to the puzzle, F5 functionality. By default VS does not have the rules to run a VSIX project in Sdk mode. To fix this we need to add a `launchSettings.json` file to our project. This file must be created under your project's `Properties` folder like so:

```
SdkDemoVsix
└── SdkDemoVsix
    └── Properties
        └── launchSettings.json
```

The content of the file should look like this:

```json
{
  "profiles": {
    "SdkDemoVsix": {
      "commandName": "Executable",
      "executablePath": "$(DevEnvDir)devenv.exe",
      "commandLineArgs": "/rootsuffix Exp"
    }
  }
}
```

# Summary

That's it! Your VSIX project is now running with the modern SDK-style project system. Your project should load up much faster in Visual Studio and you'll have nice conveniences like being able to edit your .csproj without unloading your project.

Before I go it would be remiss of me if I did not explain the downsides of switching to SDK-style projects. As an SDK-style VSIX project you will lose the following features:

- Item templates
- .vsixmanifest designer
- official support from VS

By switching to an SDK-style project you have entered unsupported and uncharted territory. Things may break or not work quite the way you expect them to, the only documentation will be your wits. However, if you wish to forgo these warnings and foray into the brave new world of SDK-style projects, I wish you good luck!

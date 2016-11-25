# Ideal Workflow for Fable + dotnetsdk

## 1. User creates repo directory

ES: :+1:

## 2. User types `npm init --yes` to create the `package.json` file

ES: :+1:

## 3. User install dependencies

Say: `npm install --save fable-compiler fable-core fable-powerpack fable-react react react-dom`. 

These will be saved in `package.json` and the package contents will be put in a `node_modules` folder.

> Note some of the dependencies are Fable related and other are pure JS.  
> Also note now it's recommended to install `fable-compiler` locally for better version management.  

ES: :+1:

## 4. Init fable project

Now the user would type: `./node_modules/.bin/fable init src/MyProject` (this command is not available yet) and this would create a skeleton `MyProject.fsproj` and `fableconfig.json` in the `src/MyProject` folder.

ES: this can be done by normal sdk `dotnet new` (templates will be installable by remote feeds, partial templates too probably)  
ES: example: `dotnet new -t fable` (or `fable_react` or `fable_elmish` etc).  
ES: this will be the way to go for whole sdk (c#/aspnet/etc)
ES: so there will be like
ES: `dotnet new --install http://suave.io/templates.feed` once (or no need if in nuget.org)
ES: `dotnet new -t suave_microservice --lang fsharp`

> Note users may have different projects in the same repo, but they may want to share the npm dependencies (that's why it's necessary to look for `node_modules` in parent directories too). 

ES: Np about that, search recursively ancestors untils node_modules is found :+1:

> Also note `MyProject.fsproj` must include a relative reference to `Fable.targets`. In this case it would be:

  ```xml
  <Import Project="../../node_modules/fable-compiler/Fable.targets" />
  ```

ES: No need for `Fable.targets` and `Import`. It will be auto included from a package.  
ES: That's how extensbility works in sdk (like for f# => `FSharp.NET.Sdk`)

  ```xml
  <PackageReference Include="Fable.Sdk" Version="1.*" />
  ```


## 5. User navigates to `.fsproj` folder: `cd src/MyProject`

ES: :+1:

## 6. User installs Fable dependencies

Like `../../node_modules/.bin/fable install fable-core fable-powerpack fable-react` (`fable install` command is not available yet).

ES: I can do `dotnet fable install fable-core fable-powerpack fable-react`  
ES: `dotnet fable` can be your nodejs executable or a normal f# .netcore console app (so xplat by default)  

This should put the following in the `.fsproj` file:

  ```xml
  <ItemGroup>
    <PackageReference Include="fable-core"><Npm>true</Npm></PackageReference>
    <PackageReference Include="fable-powerpack"><Npm>true</Npm></PackageReference>
    <PackageReference Include="fable-react"><Npm>true</Npm></PackageReference>
  </ItemGroup>
  ```

ES: Cannot use `PackageReference`, these are for nuget. You put your packages in npm.  
ES: We can use something similar.   

  ```xml
  <ItemGroup>
      <FableReference Include="fable-core" />
      <FableReference Include="fable-powerpack" />
      <FableReference Include="fable-react" />
  </ItemGroup>
  ```

ES: Each one will become a normal `<Reference Include="$(node_modules_dir)/fable-core/Fable.Core.dll" />`  
ES: That's done inside the target file automatically  
ES: v0 i'll do `<FableReference Include="Fable.Core" />` because i dont lose info, and i can infer path  
ES: v1 can be for sure `<FableReference Include="fable-core" />` it's just a bit more complicated the search algorithm (it's not a replaced :smile:)  
    

> Note this doesn't actually install the package contents nor manages the version, as this is already done by npm.  
> Other options would be to put the npm dependencies in `fableconfig.json` (to emulate a bit `paket.references`, but this may make IDE compatibility more difficult).  
> or just ask the user to edit `.fsproj` manually for now.

ES: pls dont, less files, it's better. I hope `paket.references` will be merged inside `fsproj` as `<PaketReference Include="Argu" />` too, so fsproj is enough  
ES: maybe you can also just move props inside fsproj directly (optional obj, `fableconfig.json` will override, so will works without fsproj too)  

## 6. build it!

So now, when building the project, the targets file should get the npm package references, look for the `node_modules` folder (either in the same folder as `.fsproj` or in one of the parent directories) and resolve them as follows 
This needs to work both for Fable and **IDEs**:

ES: For vscode/ides, that's a normal F# project (because include `FSharp.NET.Sdk`).  
ES: **Question**: The `FABLE_COMPILER` compiler directive is used only for build or also for normal f# (intellisense)?

  ```xml
  <ItemGroup>
    <Reference Include="../../node_modules/fable-core/Fable.Core.dll" />
    <Reference Include="../../node_modules/fable-powerpack/Fable.PowerPack.dll" />
    <Reference Include="../../node_modules/fable-react/Fable.React.dll" />
  </ItemGroup>
  ```

ES: Ok, the committed template already does that (not yet the recursive node_module)

> Note `Fable.PowerPack` has a capital letter in the middle of the word to the file match should
> be case insensitive (which is not the default case in unix): `File.Exists(sprintf "%s/%s/%s.dll" nodeModulesDir pkgName (pkgName.Replace("+", "."))`.  
> or we can just change the name of this package and follow the convention from now on.  
> Preferably the references must always start with "." and use forward slashes.

ES: Np about that, the target file can search it

About building the project using `dotnet build` or `../.../node_modules/.bin/fable`, I don't have a strong preference at the moment. 

ES: About `dotnet build` vs `dotnet fable` is the same for me. 
ES: You can leave these separated (one does .net compile, the other fable).  
ES: But using `dotnet build` by default will be the default also for flow.  
ES: Is easy anyway to try it what work best  

Fable has to read the `.fsproj` again to get the source files and references anyways.  

ES: That's something the sdk can be improve a lot. ref ##Enhancements.2

# Enhancements

## 1 - Auto create `<Reference` from `package.json`

ES: Because you use `package.json`, i can just read that to generate `<Reference`. Import everything by default (minus dev-deps obv)  
ES: But dunno about transitive deps. I can auto import all inside `node_modules` but need a convention for .dll otherwise i can pick not assemblies.   
ES: Exclusions can be done with with a prop `<NpmExclude>fable-installed-but-not-used</NpmExclude>`.  

## 2 - Sdk can give you FCS args

ES: I already know inside the target file, all the source/references/defines/etc.  
ES: I can give you (`fablecompiler`) the fsc compiler args (no need for projectcracker anymore!)
ES: As a note, i already compose the fsc args in `FSharp.NET.Sdk` for normal f#, that's what the `dotnet-compile-fsc` does  

# Examples

## WIP (ozmo)

ES: Ref `ozmo/ozmo.fsproj`  

**NOTE** require .net core sdk `1.0.0-preview4` not yet released to use `Version` attribute in `PackageReference` (rtw in days anyway)

ES: 1. `dotnet restore`
ES: 2. `dotnet build`

ES: - the produced `.js` file contains the fcs args now to check what the target file can see  
ES: - The `<FableReference` create `<Reference Path="node_module/...`  
ES: - it use `dotnet build` atm (generated js is copied in OutputDir), changing default extension for build artifact  

## A possibile final version (mario)

ES: Ref `mario/mario.fsproj`

ES: i added an example of what i think can be a final version  
ES: that's going to use next msbuild project enhancement (like remove sdk package, ref https://github.com/Microsoft/msbuild/issues/1392 )  
ES: All the `fableconfig.json` options can be moved inside the `fsproj` (but fableconfig.json)

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

AGC: This would be nice, I agree it's better to have a consistent experience with other dotnet projects.

ES: yes, also OOTB experience. and `fable init` can just call `dotnet new -t fable` so, you get both.  

> Note users may have different projects in the same repo, but they may want to share the npm dependencies (that's why it's necessary to look for `node_modules` in parent directories too). 

ES: Np about that, search recursively ancestors untils node_modules is found :+1:

> Also note `MyProject.fsproj` must include a relative reference to `Fable.targets`. In this case it would be:

  ```xml
  <Import Project="../../node_modules/fable-compiler/Fable.targets" />
  ```

ES: No need for `Fable.targets` and `Import`. It will be auto included from a package.  
ES: That's how extensbility works in sdk (like for f# => `FSharp.NET.Sdk`)

AGC: Would `Fable.Sdk` be the same as `fable-compiler`? If that's the case, does it need to be .NET application or can it be a node app as it's now?

ES: the `Fable.Sdk` package can just contains the `Fable.targets` file. The msbuild integration.  
ES: Inside `Fable.targets` i can call the existing `fable` in `PATH`. So can be the current nodejs app, like now.  
ES: Or if `fable-compiler` is a .net app, can be used and downloaded from nuget (like `fsc` compiler now). But can be done later, if at all.  

  ```xml
  <PackageReference Include="Fable.Sdk" Version="1.*" />
  ```


## 5. User navigates to `.fsproj` folder: `cd src/MyProject`

ES: :+1:

## 6. User installs Fable dependencies

Like `../../node_modules/.bin/fable install fable-core fable-powerpack fable-react` (`fable install` command is not available yet).

ES: I can do `dotnet fable install fable-core fable-powerpack fable-react`  
ES: `dotnet fable` can be your nodejs executable or a normal f# .netcore console app (so xplat by default)  

AGC: How would `dotnet fable` be translated? I mean, where's the executable that would be called?

ES: three options (normal sdk extensibility):  
ES: 1- In `PATH` exists `dotnet-fable.bat` (or `.sh`, or just `dotnet-fable`). You can add that in node package i think.  
ES: 2- Inside the msbuild as target with a conventional name like `dotnet fable` => `dotnet msbuild /t:Fable` (i need to check how, is a new feature)  
ES: 3- A dotnet sdk tool, so a `dotnet-fable.exe` netcoreapp1.0 packaged and referenced as `<DotNetCliToolReference` (like `dotnet-compile-fsc`)  
ES: All of these can just invoke forward to `../../node_modules/.bin/fable` if is already downloaded by npm. So a wrapper  

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
    
AGC: Looks good! :+1:

> Note this doesn't actually install the package contents nor manages the version, as this is already done by npm.  
> Other options would be to put the npm dependencies in `fableconfig.json` (to emulate a bit `paket.references`, but this may make IDE compatibility more difficult).  
> or just ask the user to edit `.fsproj` manually for now.

ES: pls dont, less files, it's better. I hope `paket.references` will be merged inside `fsproj` as `<PaketReference Include="Argu" />` too, so fsproj is enough  
ES: maybe you can also just move props inside fsproj directly (optional obj, `fableconfig.json` will override, so will works without fsproj too)  

AGC: That's a possibility but I'd keep `fableconfig.json` for now. I have some concerns about putting Fable compiler arguments in the .fsproj: a) it may become very verbose and not play well in all cases (for example if you include [Rollup options](http://fable.io/blog/Introducing-0-7.html#ES2015-Modules-and-Bundling)), b) intellisense support as we have now in VS Code may be difficult, c) thanks to `command-line-args` npm package, it's easy for me to define a [schema for arguments](https://github.com/fable-compiler/Fable/blob/master/src/fable/Fable.Client.Node/js/fable.js#L11-L34) that produces more or less the same result from the CLI or a json, I'm afraid supporting options in .fsproj may be maintenance burden because I have to edit the Fable.targets file (which still looks very scary to me :wink:) every time an option changes.

ES: the `Fable.target` seems scary because is filled with wip :D . But adding an option require push to nuget a new package version  
ES: But np, i understand the maintainance etc, dont want to change current working proj  

## 6. build it!

So now, when building the project, the targets file should get the npm package references, look for the `node_modules` folder (either in the same folder as `.fsproj` or in one of the parent directories) and resolve them as follows 
This needs to work both for Fable and **IDEs**:

ES: For vscode/ides, that's a normal F# project (because include `FSharp.NET.Sdk`).  
ES: **Question**: The `FABLE_COMPILER` compiler directive is used only for build or also for normal f# (intellisense)?

AGC: Yes. For example now I'm working in a solution that has some common files between the server and the client project. But for now the user can just define the directive in the `.fsproj` (as it's necessary to get IDE support).

ES: Ok, so when is built for fable, is passing `FABLE_COMPILER` always. got it.  

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

AGC: `dotnet build` could probably work :)

Fable has to read the `.fsproj` again to get the source files and references anyways.  

ES: That's something the sdk can be improve a lot. ref ##Enhancements.2

# Enhancements

## 1 - Auto create `<Reference` from `package.json`

ES: Because you use `package.json`, i can just read that to generate `<Reference`. Import everything by default (minus dev-deps obv)  
ES: But dunno about transitive deps. I can auto import all inside `node_modules` but need a convention for .dll otherwise i can pick not assemblies.   
ES: Exclusions can be done with with a prop `<NpmExclude>fable-installed-but-not-used</NpmExclude>`.  

AGC: This is something we can explore, but it may not be necessary in the first version. There can be many dependencies in `package.json` with only half or less than half being Fable-related. In this [example](https://github.com/fable-compiler/fable-react_native-demo/blob/master/package.json) it's easy to see because all Fable packages are prefixed with `fable-` but that may be not the case. Also, as npm installs everything locally you can have a repo like [Fable samples](https://github.com/fable-compiler/Fable/blob/master/samples/package.json) where you install all the common dependencies in the root directory but each project only uses some of them.

ES: Got it. :-1: so

## 2 - Sdk can give you FCS args

ES: I already know inside the target file, all the source/references/defines/etc.  
ES: I can give you (`fablecompiler`) the fsc compiler args (no need for projectcracker anymore!)
ES: As a note, i already compose the fsc args in `FSharp.NET.Sdk` for normal f#, that's what the `dotnet-compile-fsc` does  

AGC: This would be really, really interesting :clap: If we're putting Fable options in the `.fsproj` we need a way to distinguish F# compiler and Fable options. If for now we leave Fable options in `fableconfig.json` we just need to pass a first argument to indicate that what follows are F# compiler arguments. The only issue I can think of at the moment is how to read the project options again if the `.fsproj` file is modified in `--watch` mode, but we can ignore that at the moment (if users edit `.fsproj` they just need to restart watch mode).

ES: A good idea anyway is to use response file. That's because compiler argument can become very long.  
ES: That's a big problem with .netcore because use path from nuget cache, so command line become very very long  
ES: Windows sucks at that, and there is an hard limit (4k or so) for a single command line arguments (total, not single args)  
ES: That's why all compilers accept response file (.rsp) who just contains the arguments, splitted in multiple lines  
ES: so my idea is `fable --fcs-args "path/to/fsc-for-fable.rsp" --other-args` and `path/to/fsc-for-fable.rsp` contains the `FCS` args  
ES: it's really easy impl, inside `fable` you read the file, and use these for fcs.  
ES: normal response file (prefix `@` so `@path/to/file.rsp`) are just expanded when parsing the command line, so can contains all args  
ES: are amazing for dev/tests, because you write rsp, and can rerun `fsc`, and tweaking as needed (or comments too)  

# Examples

## WIP (ozmo)

ES: Ref `ozmo/ozmo.fsproj`  

**NOTE** require .net core sdk `1.0.0-preview4` not yet released to use `Version` attribute in `PackageReference` (rtw in days anyway)

ES: 1. `dotnet restore`
ES: 2. `dotnet build`

ES: - the produced `.js` file contains the fcs args now to check what the target file can see  
ES: - The `<FableReference` create `<Reference Path="node_module/...`  
ES: - it use `dotnet build` atm (generated js is copied in OutputDir), changing default extension for build artifact  

AGC: I run the sample, very cool!

## A possibile final version (mario)

ES: Ref `mario/mario.fsproj`

ES: i added an example of what i think can be a final version  
ES: that's going to use next msbuild project enhancement (like remove sdk package, ref https://github.com/Microsoft/msbuild/issues/1392 )  
ES: All the `fableconfig.json` options can be moved inside the `fsproj` (but fableconfig.json)

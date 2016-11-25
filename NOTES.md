# Ideal Workflow for Fable + dotnetsdk

- User creates repo directory
- User types `npm init --yes` to create the `package.json` file
- User install dependencies, say: `npm install --save fable-compiler fable-core fable-powerpack fable-react react react-dom`. These will be saved in `package.json` and the package contents will be put in a `node_modules` folder.

> Note some of the dependencies are Fable related and other are pure JS. Also note now it's recommended to install `fable-compiler` locally for better version management.

- Now the user would type: `./node_modules/.bin/fable init src/MyProject` (this command is not available yet) and this would create a skeleton `MyProject.fsproj` and `fableconfig.json` in the `src/MyProject` folder.

> Note users may have different projects in the same repo, but they may want to share the npm dependencies (that's why it's necessary to look for `node_modules` in parent directories too). Also note `MyProject.fsproj` must include a relative reference to `Fable.targets`. In this case it would be:

```xml
  <Import Project="../../node_modules/fable-compiler/Fable.targets" />
```

- User navigates to `.fsproj` folder: `cd src/MyProject`
- User installs Fable dependencies like `../../node_modules/.bin/fable install fable-core fable-powerpack fable-react` (`fable install` command is not available yet). This should put the following in the `.fsproj` file:

```xml
  <ItemGroup>
    <PackageReference Include="fable-core"><Npm>true</Npm></PackageReference>
    <PackageReference Include="fable-powerpack"><Npm>true</Npm></PackageReference>
    <PackageReference Include="fable-react"><Npm>true</Npm></PackageReference>
  </ItemGroup>
```

> Note this doesn't actually install the package contents nor manages the version, as this is already done by npm. Other options would be to put the npm dependencies in `fableconfig.json` (to emulate a bit `paket.references`, but this may make IDE compatibility more difficult) or just ask the user to edit `.fsproj` manually for now.

So now, when building the project, the targets file should get the npm package references, look for the `node_modules` folder (either in the same folder as `.fsproj` or in one of the parent directories) and resolve them as follows (this needs to work both for Fable and **IDEs**):

```xml
  <ItemGroup>
    <Reference Include="../../node_modules/fable-core/Fable.Core.dll" />
    <Reference Include="../../node_modules/fable-powerpack/Fable.PowerPack.dll" />
    <Reference Include="../../node_modules/fable-react/Fable.React.dll" />
  </ItemGroup>
```

> Note `Fable.PowerPack` has a capital letter in the middle of the word to the file match should be case insensitive (which is not the default case in unix): `File.Exists(sprintf "%s/%s/%s.dll" nodeModulesDir pkgName (pkgName.Replace("+", "."))`, or we can just change the name of this package and follow the convention from now on. Preferably the references must always start with "." and use forward slashes.

About building the project using `dotnet build` or `../.../node_modules/.bin/fable`, I don't have a strong preference at the moment. Fable has to read the `.fsproj` again to get the source files and references anyways.

(Published 28/02/2018 on https://blogg.itverket.no/cake/)

![](https://github.com/hmol/writings/blob/master/Simplifying-the-build-process-with-Cake/cake.jpg?raw=true)

# Simplifying the build process with Cake

Ensuring your application can be built and deployed easily in a repeatable manner, consistent across multiple environments, is an important success factor in development. Enter cake.

Cake is a cross platform build automation system for dotnet where you describe the build process with a C#-like scripting language.

# Why use cake?
When the build process is defined in a readable script, living inside source control, it will be easier for developers to maintain it. You don’t have to have much experience with advanced configuration of TeamCity or Jenkins.
You can run the scripts on your own machine or on any CI system such as AppVeyor, TeamCity, TFS, VSTS or Jenkins. The result will be the same.

# Getting started
To get started, look at the example project here: https://github.com/cake-build/example

In this example project, and all other cake projects, there are 3 important files:
1. `build.ps1` is the bootstrapper script which invokes cake and runs the scripts
2. `./tools/packages.config` contains the dependencies and external tools referenced in the script
3. `build.cake` is your build script that contains your build definition

Running a cake script from the console is done like this:
```csharp
PS C:\dev\MyApi\scripts> .\build.ps1
Preparing to run build script...
Running build script...

========================================
Default
========================================
Hello World!

Task                          Duration
--------------------------------------------------
Default                       00:00:00.0095166
--------------------------------------------------
Total:                        00:00:00.0095166
```

Cake has a quite simple syntax where each function you write is in the form of a task:
```csharp
Task("Build")
    .Does(() =>
{
     // build solution and write output
     Information("Hello World");
});
```

A common thing to do is to chain tasks together, where one task has a dependency on another task.
```csharp
Task("Build")
    .Does(() =>
{
     // build solution
});

Task("Test")
    .IsDependentOn("Build")
    .Does(() =>
{
     // test solution
});
```
When `Task("Test")` starts running, it will first invoke `Task("Build")` and wait for it to finish successfully before running. 

Cake also offers something called script aliases. These are helper functions accessible directly from the script. Here are some of the most used ones:

| Function                                             | What it does     |
| ---------------------------------------------------- |------------------|
| `CleanDirectory`| Deletes all files in directory|
| `GetFiles`  | Gets an array of files based on path in input parameter |
| `DotNetCoreRestore`  | Runs `dotnet restore` on defined project |
| `DotNetCoreBuild`  | Runs `dotnet build` in defined project|
| `DotNetCoreTest`  | Runs `dotnet test` in defined project|
| `DotNetCorePack`  | Runs `dotnet pack` on defined project|
| `DotNetCoreNuGetPush`  | Runs `dotnet nuget push`|

In addition to built-in functions, one can use external tools. See list of available tools and add-ins: https://cakebuild.net/dsl. For example; to make use of GitVersion the only thing needed is an import statement: `#tool "nuget:?package=GitVersion.CommandLine"`. When cake runs next time, it will figure out the dependencies, download and make them available for you in the script.

Knowing all this we can now try to create a simple build script that execute some common tasks: build and test the solution, pack it as a nuget pacakge and push to a nuget feed.

```csharp
// Input arguments
var target = Argument("target", "Default");
var projectBasePath = Argument("target", "");
var buildDir = Directory(projectBasePath) + "buildDir";

Task("Build")
    .Does(() =>
{
    var projects = GetFiles(projectBasePath + "/**/*.csproj");
    // Write to output (build log)
    Information("Found " + projects.Count + " projects in: " + projectBasePath);
    foreach (var project in projects)
    {
        DotNetCoreBuild(project.FullPath, settings);
    }
});

Task("Test")
    .Does(() =>
{
    var settings = new DotNetCoreTestSettings
    {
        WorkingDirectory = Directory(projectBasePath),
        NoBuild = true
    };
    var testProjects = GetDirectories(projectBasePath+"/**/*Tests");
    foreach(var testProject in testProjects)
    {
        DotNetCoreTest(testProject.FullPath, settings);
    }
});

// Create nuget packages for projects
Task("Pack")
    .Does(() =>
{
    var projects = GetFiles(projectBasePath + "/**/*.csproj");

    foreach (var project in projects)
    {
        if (project.FullPath.ToLower().Contains("tests"))
            continue;

        DotNetCorePack(project.FullPath, new DotNetCorePackSettings
        {
            OutputDirectory = buildDir
        };
    }
});

// Push nuget packages to nugetfeed
Task("Push")
	.Does(()=>
{
    var nugetFiles = GetFiles(buildDir.Path.FullPath+"/**/*.nupkg");

    foreach(var file in nugetFiles)
    {				
        var settings = new DotNetCoreNuGetPushSettings()
        {
            Source = "https://myget.org/F/mynugetfeed/api/v2/package",
            ApiKey = "z280ib18-db4c-4b71-ay97-6clq0b1la9dc"
        };
        DotNetCoreNuGetPush(file.FullPath, settings);
    }
});

Task("Default")
    .IsDependentOn("Build")
    .IsDependentOn("Test")
    .IsDependentOn("Pack")
    .IsDependentOn("Push")
	.Does(()=> { 
});

RunTarget(target);
```

In my opinion Cake offers a clean and simple solution to some of the problems one can encounter in large enterprise build/deploy environments.

Further reading: https://cakebuild.net/

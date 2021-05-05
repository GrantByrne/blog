---
title: "Getting Started With Hangfire"
date: 2021-05-02T22:00:44-04:00
tags: [csharp, scheduling, hangfire]
---

Putting together this guide for testing out hangfire to fill in some of the gaps I found through creating a greenfield hangfire application.

## Setting up a database to test with

Since I don't have a SQL Server running that I can just attach to. I'll have to run through the initial set up of SQL Express Edition

1. Navigate to the website: https://www.microsoft.com/en-us/sql-server/sql-server-downloads
2. Under the "Express" item at the bottom of the fold, hit the button "Download Now >"
3. Run the installer
4. Select "Basic" option
5. Hit the "Accept" button without reading anything because I actually want to get some work done.
6. Install it to the default directory

## Creating the Blank Project to Get Started

1. Open up Visual Studio 2019
2. Choose "Create a new Project"
3. Search for asp.net project type. Since I would like to use the dashboard for this, I will have to use at least a .net core project. So, I selected the ASP.NET Core Empty project
4. I named the application HangfireExperimentation
5. Selected the Target Framework to be .NET 5.0 (Current).

## Setting up the Test Database for Hangfire to Use

1. On the top menu go to View > SQL Server Object Explorer. This will open up a tab that allows you to navigate through the local database.
2. From this view: Expand out SQL Server > {You're localdb instance} > Databases
3. Right Click on Databases and select "Add New Database"
4. Name that Database HangfireExperimentation
5. Expand out that database
6. Right click on the database that was just created and select "Properties"
7. In the properties tab, copy the connection string and save it somewhere to use later. Mine was something like: 

{{< highlight xml >}}
Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=HangfireExperimentation;Integrated Security=True;Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False
{{< /highlight >}}

## Set up the Nuget Packages

1. In the solution explorer, right click on the project and select "Manage nuget Packages..."
2. Click on the "Browse" tab in the window
3. Search for "Hangfire"
4. Install the following packages
    - Hangfire.Core
    - Hangfire.SqlServer
    - Hangfire.AspNetCore

## Updating the appsettings.json file

1. Open up the appsettings.json file
2. Update the LogLevel to include hangfire
3. Add the connection string that you saved earlier to the list of connection strings

My appsettings.json file ended up looking like this:

{{< highlight json >}}
{
"Logging": {
    "LogLevel": {
    "Default": "Information",
    "Microsoft": "Warning",
    "Microsoft.Hosting.Lifetime": "Information",
    "Hangfire": "Information"
    }
},
"ConnectionStrings": {
    "HangfireConnection": "Data Source=(localdb)\\MSSQLLocalDB;Initial Catalog=HangfireExperimentation;Integrated Security=True;Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False"
},
"AllowedHosts": "*"
}
{{< /highlight >}}

## Update the Startup.cs file

1. Update the ConfigureServices() method so that it looks like this:

{{< highlight "C#" >}}
public void ConfigureServices(IServiceCollection services)
{
    services.AddHangfire(configuration => configuration
        .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
        .UseSimpleAssemblyNameTypeSerializer()
        .UseRecommendedSerializerSettings()
        .UseSqlServerStorage(ConfigurationManager.ConnectionStrings["HangfireConnection"].ConnectionString, new SqlServerStorageOptions
        {
            CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
            SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
            QueuePollInterval = TimeSpan.Zero,
            UseRecommendedIsolationLevel = true,
            DisableGlobalLocks = true
        }));

    services.AddHangfireServer();

    services.AddMvc(option => option.EnableEndpointRouting = false);
}
{{< /highlight >}}

2. Run through and add the missing references

3. Add in the Configuration field that ConfigureServices(...) uses to get the connection string

{{< highlight csharp >}}
public Startup(IConfiguration configuration)
{
    Configuration = configuration;
}

public IConfiguration Configuration { get; }
{{< /highlight >}}

4. Update the Configure(...) method to look like this:

{{< highlight csharp >}}
public void Configure(IApplicationBuilder app, 
    IWebHostEnvironment env, 
    IBackgroundJobClient backgroundJobs)
{
    app.UseStaticFiles();

    app.UseHangfireDashboard();
    backgroundJobs.Enqueue(() => Console.WriteLine("Hello world from Hangfire!"));

    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
{{< /highlight >}}

## Testing things out

1. Hit play on Visual Studio
2. Since this was created with blank project
3. Navigate to the hangfire dashboard. My url was: https://localhost:44321/hangfire
4. ...
5. PROFIT?!!
# AzureServerlessWorkshopUA
Azure Serverless workshop steps description. Community version of workshop that was run at Lviv and Odesa Microsoft .NET User Groups.


## Prerequisites

Good mood :).
VS 2017 or VS 2019.
[NET Core SDK 2.1 (2.2)](https://dotnet.microsoft.com/download/dotnet-core/2.2).
[Postman](https://www.getpostman.com/).
Azure subscription.
[Loader.io](https://loader.io/) account or artillery(local).
I will provide [JWKS endpoint](https://medium.com/@inthiraj1994/signature-verification-using-jwks-endpoint-in-wso2-identity-server-5ba65c5de086) and token, but you can use any OIDC of your choice.
GitHub nickname, so I can give you access to the workshop repo.

## Step 1. Migration to a function app. Start from 00_New_API_App and migrate to 01_New_Function_App folders.

1. Create .NET Core 2.2 empty Web API application.
2. Add following two classes (`FortuneTellerController` and `Person`) to API Project. Check if all endpoints work.

```csharp
        using System;
        using Microsoft.AspNetCore.Mvc;

        namespace WebApi.Controllers
        {
            [Route("api/[controller]")]
            [ApiController]
            public class FortuneTellerController : ControllerBase
            {
                [HttpGet("{name}")] //https://localhost:44328/api/FortuneTeller/Stas/
                public ActionResult<string> AskZoltar(string name)
                {
                    var rate = RatePrediction();

                    var prediction = $"Zoltar speaks! {name}, your rate will be '{rate}'.";

                    return prediction;
                }

                private static int RatePrediction()
                {
                    var random = new Random();
                    return random.Next(20, 50);
                }
            }
        }


        namespace WebApi
        {
            public class Person
            {
                public string Name { get; set; }
            }
        }
```

3. Create new Azure Functions application with HTTP trigger and `AuthorizationLevel.Function`.
4. Start new application and create Postman test project. Test local resources.
5. Convert `FortuneTellerController` to Function by moving controller and renaming method to function, as in example `Function1`.
Example result is below:

```csharp
        using System;
        using System.Threading.Tasks;
        using Microsoft.AspNetCore.Mvc;
        using Microsoft.Azure.WebJobs;
        using Microsoft.Azure.WebJobs.Extensions.Http;
        using Microsoft.AspNetCore.Http;
        using Microsoft.Extensions.Logging;

        namespace Functions
        {
            public class FortuneTellerController
            {
                [FunctionName("AskZoltar")] //https://localhost:44328/api/FortuneTeller/Stas/
                public async Task<IActionResult> Run(
                    [HttpTrigger(AuthorizationLevel.Function, "get", Route =  "AskZoltar/{name}")]
                    HttpRequest req,
                    string name,
                    ILogger log)
                {
                    var rate = RatePrediction();

                    var prediction = $"Zoltar speaks! {name}, your rate will be '{rate}'.";

                    return (ActionResult)new OkObjectResult(prediction);
                }

                private static int RatePrediction()
                {
                    var random = new Random();
                    return random.Next(20, 50);
                }
            }
        }
```

6. Start the function app locally and test it with postman.
7. Convert `ValuesController` the same way you did with `FortuneTellerController`.
8. Add new calls for Postman project and test all endpoints.
9. Lets add validation function for loader.io testing.
   Replace `Function1` class with the following code and rename the file.
   Replace string `loaderio-` with token from your account.

```csharp
        using System.Threading.Tasks;

        using Microsoft.AspNetCore.Http;
        using Microsoft.AspNetCore.Mvc;
        using Microsoft.Azure.WebJobs;
        using Microsoft.Azure.WebJobs.Extensions.Http;
        using Microsoft.Extensions.Logging;

        namespace Functions
        {
            public static class Loader
            {
                [FunctionName("LoaderActivation")]
                public static async Task<IActionResult> LoaderActivation(
                    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "loaderio-")] HttpRequest req,
                    ILogger log)
                {
                    log.LogInformation("Loader.io validation triggered.");
                    return (ActionResult)new OkObjectResult($"loaderio-");
                }
            }
        }
```

10. Update or add new file to solution - `host.json` with the following code.

```json
        {
          "version": "2.0",
          "extensions": {
            "http": {
              "routePrefix": ""
            }
          }
        }
```

11. Lets talk briefly about input and output triggers for functions. 
    Right click on solution and add new Azure Function with Queue Trigger, Blob trigger, Event Hub.
    Lets add following controller to further usage.

```csharp
        using Microsoft.Azure.WebJobs;
        using Microsoft.Extensions.Logging;

        namespace Functions
        {
            public class MessageController
            {
                [FunctionName("QueueFunction")]
                public async void QueueFunction(
                    [QueueTrigger("incoming-requests", Connection = "StorageConnectionString")] string incomingMessage,
                    [Queue("zoltar-results", Connection = "StorageConnectionString")] IAsyncCollector<string> outgoingMessages,
                    ILogger log)
                {
                    log.LogInformation("Processed message from incoming-requests.");

                    await outgoingMessages.AddAsync(incomingMessage);
                }
            }
        }
```

Comment out Queue function in `MessageController` to start project.
Add connection string to `local.settings.json`
Also add empty value.

```json
          "StorageConnectionString": "DefaultEndpointsProtocol=https;AccountName=",
          "APPINSIGHTS_INSTRUMENTATIONKEY": "f99435fd"
```

Start. This step concludes the first part of our workshop.
Nicely done!

## Step 2. Azure Infrastructure. 02_Function_App_Deployment folder.

1. Lets activate our Azure passes and create subscriptions.
2. We will create:
    - Azure Function app (Consumption plan)
    - Azure KeyVault
    - Managed identity for Functions App and connect it to KeyVault.
    - Blob Storage and Storage Queue
    - Application Insights
    - Azure SQL S0 instance.

3. Infrastructure as a code via Azure CLI.

```bash
        Azure CLI infrastructure script.
        -----------------

        # subscription switch and check
        subscriptionID=$(az account list --query "[?contains(name,'Pass')].[id]" -o tsv)
        echo "Test subscription ID is = " $subscriptionID
        az account set --subscription $subscriptionID
        az account show

        #----------------------------------------------------------------------------------
        # Resource group
        #----------------------------------------------------------------------------------

        location=northeurope
        postfix=$RANDOM

        # resource group
        groupName=ServWorkshopUA$postfix

        az group create --name $groupName --location $location
        #az group delete --name $groupName


        #----------------------------------------------------------------------------------
        # Storage account with Blob and Queue
        #----------------------------------------------------------------------------------

        location=northeurope
        accountSku=Standard_LRS
        accountName=${groupName,,}
        echo "accountName  = " $accountName

        az storage account create --name $accountName --location $location --kind StorageV2 \
        --resource-group $groupName --sku $accountSku --access-tier Hot  --https-only true

        accountKey=$(az storage account keys list --resource-group $groupName --account-name $accountName --query "[0].value" | tr -d '"')
        echo "storage account key = " $accountKey

        connString="DefaultEndpointsProtocol=https;AccountName=$accountName;AccountKey=$accountKey;EndpointSuffix=core.windows.net"
        echo "connection string = " $connString

        blobName=${groupName,,}

        az storage container create --name $blobName \
        --account-name $accountName --account-key $accountKey --public-access off

        queueName=${groupName,,}

        az storage queue create --name $queueName --account-key $accountKey \
        --account-name $accountName --connection-string $connString

        #----------------------------------------------------------------------------------
        # Application insights instance
        #----------------------------------------------------------------------------------

        insightsName=${groupName,,}
        echo "insightsName  = " $insightsName

        # drop this command with ctrl+c after 3 minutes of execution
        az resource create --resource-group $groupName --name $insightsName --resource-type "Microsoft.Insights/components" --location $location --properties '{"Application_Type":"web"}' --verbose

        insightsKey=$(az resource show -g $groupName -n $insightsName --resource-type "Microsoft.Insights/components" --query properties.InstrumentationKey --o tsv) 
        echo "Insights key = " $insightsKey
		
		# on your PC run CMD as administrator, then execute following commands and reboot PC.
		# just copy command output below to CMD and execute.
		echo "setx APPINSIGHTS_INSTRUMENTATIONKEY "$insightsKey

        #----------------------------------------------------------------------------------
        # Function app with staging slot and consumption plan
        #----------------------------------------------------------------------------------

        runtime=dotnet
        location=northeurope
        applicationName=${groupName,,}
        accountName=${groupName,,}
        echo "applicationName  = " $applicationName

        az functionapp create --resource-group $groupName \
        --name $applicationName --storage-account $accountName --runtime $runtime \
        --app-insights-key $insightsKey --consumption-plan-location $location

        az functionapp update --resource-group $groupName --name $applicationName --set dailyMemoryTimeQuota=400000

        az functionapp deployment slot create --resource-group $groupName --name $applicationName --slot staging

        az functionapp identity assign --resource-group $groupName --name $applicationName
		
		az functionapp config appsettings set --resource-group $groupName --name $applicationName --settings "MSDEPLOY_RENAME_LOCKED_FILES=1"

        managedIdKey=$(az functionapp identity show --name $applicationName --resource-group $groupName --query principalId --o tsv)
        echo "Managed Id key = " $managedIdKey


        #----------------------------------------------------------------------------------
        # Azure SQL Server and database.
        #----------------------------------------------------------------------------------

        location=northeurope
        serverName=${groupName,,}
        adminLogin=Admin$groupName
        password=Sup3rStr0ng$groupName
        databaseName=${groupName,,}
        serverSku=S0
        catalogCollation="SQL_Latin1_General_CP1_CI_AS"

        az sql server create --name $serverName --resource-group $groupName \
        --location $location --admin-user $adminLogin --admin-password $password

        az sql db create --resource-group $groupName --server $serverName --name $databaseName \
        --service-objective $serverSku --catalog-collation $catalogCollation

        outboundIps=$(az webapp show --resource-group $groupName --name $applicationName --query possibleOutboundIpAddresses --output tsv)
        IFS=',' read -r -a ipArray <<< "$outboundIps"

        for ip in "${ipArray[@]}"
        do
          echo "$ip add"
          az sql server firewall-rule create --resource-group $groupName --server $serverName \
          --name "WebApp$ip" --start-ip-address $ip --end-ip-address $ip
        done

        sqlClientType=ado.net

		#TODO add Admin login and remove password, set to variable.
        sqlConString=$(az sql db show-connection-string --name $databaseName --server $serverName --client $sqlClientType --o tsv)
        sqlConString=${sqlConString/Password=<password>;}
		sqlConString=${sqlConString/<username>/$adminLogin}
		echo "SQL Connection string is = " $sqlConString
		
		#----------------------------------------------------------------------------------
        # Key Vault with policies.
        #----------------------------------------------------------------------------------

        location=northeurope
        keyVaultName=${groupName,,}
        echo "keyVaultName  = " $keyVaultName

        az keyvault create --name $keyVaultName --resource-group $groupName --location $location 

        az keyvault set-policy --name $keyVaultName --object-id $managedIdKey \
        --certificate-permissions get list --key-permissions get list --secret-permissions get list

        az keyvault secret set --vault-name $keyVaultName --name FancySecret  --value 'SuperSecret'
        az keyvault secret set --vault-name $keyVaultName --name SqlConnectionString  --value "$sqlConString"
        az keyvault secret set --vault-name $keyVaultName --name SqlConnectionPassword  --value $password
		#az keyvault secret set --vault-name $keyVaultName --name StorageConnectionString  --value $connString
		
		# on your PC run CMD as administrator, then execute following commands and reboot PC.
		# just copy command output below to CMD and execute.
	
		echo "setx SqlConnectionString \""$sqlConString\"
		echo "setx SqlConnectionPassword "$password
		echo "setx StorageConnectionString \""$connString\"
		
        az functionapp deployment list-publishing-credentials --resource-group $groupName --name $applicationName
        url=$(az functionapp deployment source config-local-git --resource-group $groupName --name $applicationName --query url --output tsv)
        echo $url
```

4. Time to do some git magic! I am sure you know how to create a git repo. 

```
        git init
        git add
        git add .gitignore
        git commit
```

5. And add remote https://docs.microsoft.com/en-us/azure/app-service/scripts/cli-deploy-local-git

```bash
       git remote add azure https://
       git pull
       git push azure master
```

// TODO: add some comment about these links
 https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale 

## Step 3. DI and testing of the Azure Function App. 03_Function_App_Testing folder.

1. ILogger injection via Startup.cs and class constructor, instead of Function constructor.
```csharp
                using Functions;
                using Microsoft.Azure.Functions.Extensions.DependencyInjection;
                using Microsoft.Extensions.DependencyInjection;
                using Microsoft.Extensions.Logging;

                [assembly: FunctionsStartup(typeof(Startup))]

                namespace Functions
                {
                    public class Startup : FunctionsStartup
                    {
                        public override void Configure(IFunctionsHostBuilder builder)
                        {
                            builder.Services.AddLogging(options =>
                            {
                                options.AddFilter("Functions", LogLevel.Information);
                            });
                        }
                    }
                }
```
2. Create a new nUnit Test project and add following NuGet packages. 
```csharp                    
                    <PackageReference Include="Moq" Version="4.13.1" />
                    <PackageReference Include="nunit" Version="3.11.0" />
                    <PackageReference Include="NUnit3TestAdapter" Version="3.11.0" />
```                  
 3. Lets create sample classes for our project.
 ```csharp
                using NUnit.Framework;

                namespace Unit
                {
                    [SetUpFixture]
                    public class GlobalSetup
                    {
                        [OneTimeSetUp]
                        public void OneTimeSetup()
                        {

                        }

                        [OneTimeTearDown]
                        public void OneTimeTearDown()
                        {

                        }
                    }
                }
                
       and
       
                       using Functions;
                using Microsoft.AspNetCore.Mvc;
                using Microsoft.Extensions.Logging;
                using Moq;
                using NUnit.Framework;
                using System.Threading.Tasks;

                namespace Unit
                {
                    public class FortuneTellerTests
                    {
                        private Fixture fixture;

                        [SetUp]
                        public void Setup()
                        {
                            
                        }

                        [Test]
                        public async Task AskZoltar_WithNameParameter_ReturnOkResult()
                        {
                            var tellerLogger = new Mock<ILogger<FortuneTellerController>>();
                            var instance = new FortuneTellerController(tellerLogger.Object);

                            var result = await instance.AskZoltar(null, "Test");

                            Assert.That(result, Is.TypeOf<OkObjectResult>());
                        }
                    }
                }
``` 
 

ReSharper is used for Unit Test runs.

## Step 4. Entity Framework Core and Azure SQL integration. 04_Function_App_Entity_Core folder.

We will create a new project Data and Domain, model for database and run load test.

0. Crucial step it to add following code to Starup.cs of Functions project.
```csharp
          <Target Name="PostBuild" AfterTargets="PostBuildEvent">
            <Exec Command="copy /Y &quot;$(TargetDir)bin\$(ProjectName).dll&quot; &quot;$(TargetDir)$(ProjectName).dll&quot;" />
          </Target>
```        
   Update your local.settings.json with environment variables
```csharp   
                   {
                    "IsEncrypted": false,
                  "Values": {
                    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
                    "FUNCTIONS_WORKER_RUNTIME": "dotnet",
                    "StorageConnectionString": "DefaultEndpointsProtocol=https;AccountName=",
                    "APPINSIGHTS_INSTRUMENTATIONKEY": "f99435fd",
                    "SqlConnectionString": "value",
                    "SqlConnectionPassword":  "value" 
                  }
                }                
```     
                
1. Add project Domain with one class.
```csharp
                namespace Domain
                {
                    public class Person
                    {
                        public string Id { get; set; }
                        public string Name { get; set; }
                        public int Prediction { get; set; }
                    }
                }
```
2. Create Data project with the following dependencies and classes.
```csharp
                 <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="2.1.11" />
                    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="2.1.11" />
                    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="2.1.11">
                      <PrivateAssets>all</PrivateAssets>
                      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
                    </PackageReference>
```                    
Add FunctionDbContext
    
 ```csharp    
                 using Domain;

                using Microsoft.EntityFrameworkCore;

                namespace Data
                {
                    public class FunctionDbContext  : DbContext
                    {
                        public FunctionDbContext(DbContextOptions<FunctionDbContext> options) : base(options)
                        {
                        }

                        public DbSet<Person> Person { get; set; }

                        protected override void OnModelCreating(ModelBuilder modelBuilder)
                        {
                            FunctionDbContextConfig.Configure(modelBuilder);
                            base.OnModelCreating(modelBuilder);
                        }

                            /* open the command line via CMD, install tools (if there are none present)
						   * dotnet tool install --global dotnet-ef --version 2.2
							* dotnet ef migrations add InitialCreate --context FunctionDbContext
							* dotnet ef database update
						   */
                    }
                }
```
                
Add SqlConnectionBuilder class

```csharp 
                     namespace Data
                {
                    public class SqlConnectionBuilder
                    {
                        public static string GetConnectionString(string connectionString, string passwordKey)
                        {
                            var builder = new SqlConnectionStringBuilder(connectionString) { Password = passwordKey };

                            return builder.ConnectionString;
                        }
                    }
                }
```   
         
FunctionContextFactory
 
```csharp
                    using Microsoft.EntityFrameworkCore;
                using Microsoft.EntityFrameworkCore.Design;
                using System;

                namespace Data
                {
                    public class FunctionContextFactory : IDesignTimeDbContextFactory<FunctionDbContext>
                    {
                        public FunctionDbContext CreateDbContext(string[] args)
                        {

                            var sqlString = Environment.GetEnvironmentVariable("SqlConnectionString");
                            var password = Environment.GetEnvironmentVariable("SqlConnectionPassword");
                            var connectionString = SqlConnectionBuilder.GetConnectionString(sqlString, password);

                            var optionsBuilder = new DbContextOptionsBuilder<FunctionDbContext>();
                            optionsBuilder.UseSqlServer(connectionString);

                            return new FunctionDbContext(optionsBuilder.Options);
                        }
                    }
                }
                
```             
FunctionDbContextConfig
   
```csharp              
                using Domain;
                using Microsoft.EntityFrameworkCore;

                namespace Data
                {
                    public static class FunctionDbContextConfig
                    {
                        public static void Configure(ModelBuilder modelBuilder)
                        {
                            var person = modelBuilder.Entity<Person>();
                            person.Property(x => x.Id).ValueGeneratedOnAdd();
                            person.Property(x => x.Name);
                            person.Property(x => x.Prediction);
                        }
                    }
                }
```


3. Modify Startup.cs and uncomment DB contect section.
```csharp
                using Data;
                using Functions;
                using Microsoft.Azure.Functions.Extensions.DependencyInjection;
                using Microsoft.EntityFrameworkCore;
                using Microsoft.Extensions.DependencyInjection;
                using Microsoft.Extensions.Logging;
                using System;

                [assembly: FunctionsStartup(typeof(Startup))]

                namespace Functions
                {
                    public class Startup : FunctionsStartup
                    {
                        public override void Configure(IFunctionsHostBuilder builder)
                        {
                            builder.Services.AddLogging(options =>
                            {
                                options.AddFilter("Functions", LogLevel.Information);
                            });

                            var sqlString = Environment.GetEnvironmentVariable("SqlConnectionString");
                            var password = Environment.GetEnvironmentVariable("SqlConnectionPassword");
                            //var connectionString = SqlConnectionBuilder.GetConnectionString(sqlString, password);

                            //builder.Services.AddDbContextPool<FunctionDbContext>(
                            //builder.Services.AddDbContext<FunctionDbContext>(
                            //   options =>
                            //       {
                            //           if (!string.IsNullOrEmpty(connectionString))
                            //           {
                            //               options.UseSqlServer(connectionString, providerOptions => providerOptions.EnableRetryOnFailure());
                            //           }
                            //       });
                        }
                    }
                }
```
To run all migrations via command line via CMD. 
Make sure that you are in the Data folder of the project (04_Function_App_Entity_Core\Data)
If command dotnet returns errors, install tools.

- `dotnet tool install --global dotnet-ef --version 2.2`

If you encounter error that tools are not found during migration, 
you need to rename folders(displayed in exceptions) from core2.2-rtm32432 to core2.2

- `dotnet ef migrations add InitialCreate --context FunctionDbContext`
- `dotnet ef database update`


## Step 5. Error handling via Azure Storage Queue and retry via Polly. Azure Key Vault. 05_Function_App_ErrorHandling folder.   

1. Add following packages to Functions.csproj.

            <PackageReference Include="Microsoft.Azure.Functions.Extensions" Version="1.0.0" />
            <PackageReference Include="Microsoft.Azure.KeyVault" Version="3.0.4" />

2. Add following code to Startup.cs
```csharp
                     var fancySecret = string.Empty;

            if (EnvironmentHelper.IsProduction)
            {
                var azureServiceTokenProvider = new AzureServiceTokenProvider();

                var keyVaultClient = new KeyVaultClient(new KeyVaultClient.AuthenticationCallback(azureServiceTokenProvider.KeyVaultTokenCallback));
                var baseKeyVaultEndpoint = Environment.GetEnvironmentVariable("KeyVaultEndpoint");

                fancySecret = keyVaultClient.GetSecretAsync($"{baseKeyVaultEndpoint}/secrets/FancySecret").Result.Value;
            }
            else
            {
                fancySecret = Environment.GetEnvironmentVariable("FancySecret");
            }
```


3. Add class OperationResult to Domain project.
```csharp
                using System;

                namespace Domain
                {
                    public class OperationResult
                    {
                        public string ErrorMessage { get; private set; }

                        public Exception Exception { get; private set; }

                        public bool Success { get; private set; }

                        public static OperationResult CreateErrorResult(string errorMessage)
                        {
                            return new OperationResult { Success = false, ErrorMessage = errorMessage };
                        }

                        public static OperationResult CreateExceptionResult(Exception ex)
                        {
                            return new OperationResult
                                       {
                                           Success = false,
                                           ErrorMessage = string.Format("{0}{1}{2}", ex.Message, Environment.NewLine, ex.StackTrace),
                                           Exception = ex
                                       };
                        }

                        public static OperationResult CreateSuccessResult()
                        {
                            return new OperationResult { Success = true };
                        }
                    }
                }
```

			
Introduction to polly framework with retry policies, keeping failes request data in storage queue for replay.

4. Lets start with adding the Polly nuget.

                    <PackageReference Include="Polly" Version="7.1.1" />

5. Special helper for Helper injection into Polly policy on Startup.
```csharp
                        using Microsoft.Extensions.Logging;
                        using Polly;

                        namespace Functions
                        {
                            public static class PollyContextExtension
                            {
                                private static readonly string loggerKey = "LoggerKey";

                                public static Context WithLogger(this Context context, ILogger logger)
                                {
                                    context[loggerKey] = logger;
                                    return context;
                                }

                                public static ILogger TryGetLogger(this Context context)
                                {

                                    if (context.TryGetValue(loggerKey, out object logger))
                                    {
                                        return logger as ILogger;
                                    }

                                    return null;
                                }
                            }
                        }
 ```                       
   6. Updating Startup.cs with following code.
   
 ```csharp  
                builder.Services.AddSingleton(sp => {
                    return Policy
                        .Handle<Exception>()
                        .WaitAndRetryAsync(2, 
                            retryAttempt => TimeSpan.FromSeconds(8),
                            onRetry: (ex, retryCount, context) => {

                            var log = context.TryGetLogger();
                                log?.LogInformation($"Polly retry {retryCount} for exception {ex}");
                            });
                });
  ```              
  7. Lets update FortuneTellerController.cs with failure handling code.
  You can uncomment proper constructor and code, to allign it with your project.

```csharp

using Data;
using Domain;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;
using Microsoft.IdentityModel.Tokens;
using Newtonsoft.Json;
using Polly;
using Polly.Retry;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace Functions
{
    public class FortuneTellerController
    {
        private readonly ILogger log;

        private readonly IKeysHelper keysHelper;

        private readonly ISecurityHelper securityHelper;

        private readonly AsyncRetryPolicy retryPolicy;

        private readonly FunctionDbContext context;

        public FortuneTellerController(ILogger<FortuneTellerController> log,
                                       AsyncRetryPolicy retryPolicy,
                                       IKeysHelper keysHelper,
                                       ISecurityHelper securityHelper,
                                       FunctionDbContext context)
        {
            this.log = log;
            this.securityHelper = securityHelper;
            this.keysHelper = keysHelper;
            this.context = context;
            this.retryPolicy = retryPolicy;
        }

        [FunctionName("AskZoltar")] 
        public async Task<IActionResult> AskZoltar(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route =  "api/AskZoltar/{name}")]
            HttpRequest req,
            string name,
            [Queue("error-queue", Connection = "StorageConnectionString")] IAsyncCollector<string> messages)
        {
            if (!this.ValidateBearerToken(req, await this.keysHelper.GetKeys()))
            {
                return Unauthorized();
            }

            try
            {
                var pollyContext = new Context().WithLogger(this.log);
                var prediction = string.Empty;

                await this.retryPolicy.ExecuteAsync(async ctx => {
                    prediction = await this.ZoltarSpeaks(name);
                    throw new ArgumentException();
                }, pollyContext);

                return new OkObjectResult(prediction);
            }
            catch (Exception exception)
            {
                var person = new Person { Name = name };
                await messages.AddAsync(JsonConvert.SerializeObject(person));
                this.log.LogError(exception, "AskZoltar failed");
                return new BadRequestObjectResult($"{name}, Zoltar rejected your request.");
            }
        }

        [FunctionName("DataRestore")]
        public async Task DataRestore(
            [QueueTrigger("error-queue", Connection = "StorageConnectionString")]
            string message,
            CancellationToken cts)
        {
            var prediction = await this.ZoltarSpeaks(message);
            
            this.log.LogInformation($"Processed failed message '{message} .");

            throw new ArgumentException();
        }

        [FunctionName("PoisonDataRestore")]
        public async Task PoisonDataRestore(
            [QueueTrigger("error-queue-poison", Connection = "StorageConnectionString")]
            string message,
            CancellationToken cts)
        {
            var prediction = await this.ZoltarSpeaks(message);
            this.log.LogInformation($"Processed poison message '{message} .");
        }

        private async Task<string> ZoltarSpeaks(string name)
        {
            string prediction;
            var rate = RatePrediction();

            prediction = $"Zoltar speaks! {name}, your rate will be '{rate}'.";

            await this.SavePrediction(name, rate);

            this.log.LogInformation(prediction);
            return prediction;
        }

        private static int RatePrediction()
        {
            var random = new Random();
            return random.Next(20, 50);
        }

        private async Task SavePrediction(string name, int rate)
        {
            var person = new Person() { Name = name, Prediction = rate };
            await this.context.AddAsync(person);
            await this.context.SaveChangesAsync();
        }

        protected bool ValidateBearerToken(HttpRequest req, List<SecurityKey> signingKeys)
        {
            OperationResult result = this.securityHelper.ValidateTokenAndClaims(req, signingKeys);
            return result.Success == true ? true : false;
        }
        protected static IActionResult Unauthorized()
        {
            return (ActionResult)new UnauthorizedResult();
        }
    }
}
 ```



## Step 6. Api versioning. 06_Function_App_API_versioning

1. Add new dependency to your project.

                    <PackageReference Include="AzureFunctions.Extensions.Swashbuckle" Version="1.4.1" />

2. Unfortunately we need to add additional class for Swashbuckle DI.
 ```csharp
                        using AzureFunctions.Extensions.Swashbuckle;
                        using Functions;
                        using Microsoft.Azure.WebJobs;
                        using Microsoft.Azure.WebJobs.Hosting;
                        using System.Reflection;

                        [assembly: WebJobsStartup(typeof(SwashBuckleStartup))]
                        namespace Functions
                        {
                            internal class SwashBuckleStartup : IWebJobsStartup
                            {
                                public void Configure(IWebJobsBuilder builder)
                                {
                                    builder.AddSwashBuckle(Assembly.GetExecutingAssembly());
                                    // <PackageReference Include="Microsoft.Azure.WebJobs.Script.ExtensionsMetadataGenerator" Version="1.1.2" />
                                }
                            }
                        }
 ```                        
3. Lets add Swagger controller. Be aware that Function are using code authentication
 ```csharp
                        using AzureFunctions.Extensions.Swashbuckle;
                        using AzureFunctions.Extensions.Swashbuckle.Attribute;
                        using Microsoft.Azure.WebJobs;
                        using Microsoft.Azure.WebJobs.Extensions.Http;
                        using System.Net.Http;
                        using System.Threading.Tasks;

                        namespace Functions
                        {
                            public class SwaggerController
                            {
                                [SwaggerIgnore]
                                [FunctionName("Swagger")]
                                public static Task<HttpResponseMessage> Run(
                                    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "Swagger/json")] HttpRequestMessage req,
                                    [SwashBuckleClient]ISwashBuckleClient swashBuckleClient)
                                {
                                    return Task.FromResult(swashBuckleClient.CreateSwaggerDocumentResponse(req));
                                }

                                [SwaggerIgnore]
                                [FunctionName("SwaggerUi")]
                                public static Task<HttpResponseMessage> Run2(
                                    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "Swagger/ui")] HttpRequestMessage req,
                                    [SwashBuckleClient]ISwashBuckleClient swashBuckleClient)
                                {
                                    return Task.FromResult(swashBuckleClient.CreateSwaggerUIResponse(req, "swagger/json"));
                                }
                            }
                        }
 ```                        
   4. Update your Ask Zoltar function with Swagger attributes
   
  ```csharp  
                         [ProducesResponseType((int)HttpStatusCode.OK, Type = typeof(string))]
                        [ProducesResponseType((int)HttpStatusCode.BadRequest, Type = typeof(BadRequestObjectResult))]
                        [RequestHttpHeader("Authorization", isRequired: true)]

 ```
# Additional materials

https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-versions
https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue#hostjson-settings

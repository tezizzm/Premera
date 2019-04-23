# Exercise #4

## Goal

Explore fault and latency tolerance along with service health monitoring.

## Expected Results

Change the UI service so that external calls are wrapped in a Hystrix Command to provide fault and latency tolerance.  Bind existing UI application with an instance of Hystrix Service to allow monitoring of the external calls.

## Introduction

This exercise helps us understand how to wrap our external calls in Hystrix Commands and allows us to view metrics on the health of those calls.

1. Return back to your `bootcamp-store` project root and find the project file.  We will add the following nuget package:

    ```xml
    <PackageReference Include="RabbitMQ.Client" Version="5.1.0" />
    <PackageReference Include="Steeltoe.CircuitBreaker.Hystrix.MetricsStreamCore" Version="2.1.0" />
    <PackageReference Include="Steeltoe.CircuitBreaker.HystrixCore" Version="2.1.0" />
    ```

2. In the root of the project create a file called ProductService.cs with the below implementation.  This class will act as an abstraction to our product retrieval from our products API.  The logic for retrieving products has been moved to our RunAsync method and there is now a method to return placeholder values from the RunFallbackAsync method in cases of failures.

    ```c#
        using System;
        using System.Net.Http;
        using System.Threading.Tasks;
        using Microsoft.Extensions.Logging;
        using Steeltoe.CircuitBreaker.Hystrix;
        using Steeltoe.Common.Discovery;

        namespace bootcamp_store.Service
        {
            public sealed class ProductService : HystrixCommand<string[]>
            {
                private readonly DiscoveryHttpClientHandler _handler;
                private readonly ILogger<ProductService> _logger;

                public ProductService(IHystrixCommandOptions options, IDiscoveryClient client, ILogger<ProductService> logger) :
                    base(options)
                {
                    _logger = logger;
                    _handler = new DiscoveryHttpClientHandler(client);
                    IsFallbackUserDefined = true;
                }

                public async Task<string[]> RetrieveProducts()
                {
                    _logger.LogDebug("Retrieving Products from Product Service");
                    return await ExecuteAsync();
                }

                protected override async Task<string[]> RunAsync()
                {
                    var client = new HttpClient(_handler, false);
                    _logger.LogDebug("Processing rest api call to get products");
                    var result = await client.GetAsync("https://dotnet-core-api/api/products");
                    var products = await result.Content.ReadAsAsync<string[]>();

                    foreach (var product in products)
                    {
                        Console.WriteLine(product);
                    }

                    return products;
                }

                protected override Task<string[]> RunFallbackAsync()
                {
                    _logger.LogDebug("Processing products from fallback method");
                    var products = new[]
                    {
                        "Fallback Product One, Bulls Championship",
                        "Fallback Product Two, Notre Dame Football National Championship",
                        "Fallback Product Three, White Sox World Series!"
                    };

                    foreach (var product in products)
                    {
                        Console.WriteLine(product);
                    }

                    return Task.FromResult(products);
                }
            }
        }
    ```

3. Navigate to the Startup class and set the following using statements:

    ```c#
    using Steeltoe.CircuitBreaker.Hystrix;
    using bootcamp_store.Service;
    ```

4. In the ConfigureServices method use an extension method to add the service class and Hystrix metrics stream to the DI Container with the following lines of code.

    ```c#
    services.AddHystrixCommand<ProductService>("ProductService", Configuration);
    services.AddHystrixMetricsStream(Configuration);
    ```

5. In the Configure method add swagger to the middleware pipeline by adding the following code snippet

    ```c#
    //make sure the request context is set before adding the metrics stream middleware
    app.UseHystrixRequestContext();
    ...
    app.UseHystrixMetricsStream();
    ```

6. Navigate to the appsettings.json file and add an entry for hystrix like follows.  It configures the hystrix command and tells the stream to ignore certificates.

    ```json
        "hystrix": {
            "stream": {
                "validate_certificates": false
            },
            "command": {
                "ProductService": {
                "threadPoolKeyOverride": "ProductServiceTPool"
                }
            }
        }
    ```

7. Update the HomeController to now utilize the ProductService as follows:

    ```c#
        using System.Diagnostics;
        using System.Threading.Tasks;
        using Microsoft.AspNetCore.Mvc;
        using bootcamp_store.Models;
        using bootcamp_store.Service;
        using Microsoft.Extensions.Logging;

        namespace bootcamp_store.Controllers
        {
            public class HomeController : Controller
            {
                private readonly ProductService _productService;
                private readonly ILogger<HomeController> _logger;

                public HomeController(ProductService productService, ILogger<HomeController> logger)
                {
                    _productService = productService;
                    _logger = logger;
                }

                public async Task<IActionResult> Index()
                {
                    ViewData["products"] = await _productService.RetrieveProducts();
                    _logger.LogDebug("Retrieved Products");
                    return View();
                }

                [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
                public IActionResult Error()
                {
                    return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
                }
            }
        }
    ```

8. Run the following command to create an instance of the Circuit Breaker
 **note: service name and type may be different depending on platform/operator configuration**

    ```bat
    cf create-service p-circuit-breaker-dashboard standard myHystrixService
    ```

9. Navigate to the manifest.yml file and in the services section add an entry to bind the application to the newly created instance of the Hystrix Service.

    ```yml
        - myHystrixService
    ```

10. Run the cf push command to build, stage and run your application on PCF.  Ensure you are in the same directory as your manifest file and type `cf push`.

11. Go "manage" the `Circuit Breaker` instance from within Apps Manager. Notice the dashboard listings.  At this point you may see a generic "loading..." message. Once the UI application has finished updating navigate to it and refresh the page a couple of times.  In the Circuit Breaker instance you should start seeing activity being monitored.

*Optional: Explore advanced features of the Circuit Breaker:*

1. From Apps Manager stop the API application.  Once the API application has been stopped navigate to the UI application and refresh the page a couple of times.  Notice that the product listing has changed (the products are being loaded from the fallback method).  Also go back to the Circuit Breaker service and click Manage to see the circuit health.  

2. Things to note: all calls should now be failing.  Note that once the threshold has been hit calls completely bypass the initial call and immediately go to the fallback method.

3. Restart the API application.

4. Navigate to the Service folder inside the bootcamp-store folder.  Take note of the ProductService.cs file, within it you will find the definition of the ProductService which is a HystixCommand.  There are two protected methods RunAsync and RunFallbackAsync that implement the external call and fallback call respectively.

5. In the ProductService class add logic to raise an exception (ie. new Exception()) in the RunAsync method every fifth call prior to the code that calls the external service.  Once complete re-push the application.

6. Once the UI application has been updated use a tool (Postman, curl, etc) to hit the UI URL a large number of times.  You should notice over the course of monitoring that the Circuit should go between closed and opened as the health changes overtime.

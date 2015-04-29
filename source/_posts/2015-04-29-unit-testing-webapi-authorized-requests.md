---
title: "Unit Testing Web API: Authorized Requests"
date: 2015-04-29 11:35:41
subtitle: "How can we test Web API controllers requiring authorization?"
tags:
    - aspnet
    - csharp
---
##Why?
Since Web API has been out, I've been a huge fan - and one of those main reasons has been testability. In most of the applications I've ever built, I've specifically chosen to put the majority of heavy lifting under Web API's responsibility rather than ASP.NET MVC's (and yes, most of my deployments are MVC apps with Web API running inside). However, I wanted to share my approach to actually testing my Web API controllers and how my philosophy works because, as you'll see, there's often a lot more that needs to be tested than simply the controller.

## Oh, The Examples...
It seems like most of the examples out there of trying to Unit Test Web API Controllers (or most of ASP.NET ones, for that matter) seem to gloss over the fact that a lot of us need to test things that require authorization to take place at some point in the request pipeline. There's also a significant amount of us developing applications that are taking advantage of [IoC Containers](http://www.hanselman.com/blog/ListOfNETDependencyInjectionContainersIOC.aspx) and [separating our business logic](https://lostechies.com/jimmybogard/2013/10/10/put-your-controllers-on-a-diet-redux/) away from the controller (I'm not religious about this, but I have found it to be helpful for myself). So, hopefully the following example can be something that helps a few folks out. 

## The Philosophy
Ok, so first off I rarely (if ever) solely test controllers by themselves. As you'll see I keep the majority of my business logic outside of the controller and thus the controller really becomes the point in my application where values from an HTTP request are transitioned into parameters for my business logic. Hopefully, if you think about what is happening at the point a controller's action is being invoked, what I'm saying makes sense as a critical point in the framework-to-application logic junction. At such a critical point in our request's lifetime, I want to have tests around that ensure that if I ever change a route or my AJAX requests are failing - I know where I can replicate and fix things with tests.

So, for the most part my Web API controller and tests look something like the following (we'll break it down further). I do use [Mediatr](https://lostechies.com/jimmybogard/2014/09/09/tackling-cross-cutting-concerns-with-a-mediator-pipeline/) and [StructureMap](http://structuremap.github.io/) - but I would argue that you could accomplish what I'm doing in this post without them.  
```csharp
using Mediatr;

[Authorize]
public class SomeController : ApiController 
{
    private readonly IMediator _mediator;
    
    public SomeController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    public async Task<IHttpActionResult> Get(SomeListRequest req)
    {
        var result = await _mediator.SendAsync(req);
        
        return Ok(result);
    }
}
```
and the test
```csharp
using System;
using System.Web.Http;
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class SomeControllerTests : ApiControllerTestBase
{
    [TestMethod]
    public async Task SomeController_IndexWorks()
    {
        // arrange
        var vm      = new SomeListViewModel();
        var ctrller = CreateController();
        this.Mediator.SendAsyncOverride = (x => vm);
        this.Container.Inject(ctrller);

        // act
        var resp = await Client.GetAsync("/api/some/");

        // assert
        Assert.IsTrue(resp.IsSuccessStatusCode);
        
        var result = await resp.Content.ReadAsAsync<SomeListViewModel>();
        Assert.AreEqual(0, vm.Items.Count);
    }
    
    // helper method in case the dependencies get a bit gnarly
    private SomeController CreateController()
    {
        return new SomeController(this.Mediator);    
    }
}
```
The idea here is, hopefully, fairly straightforward. The `Mediator` does the hardwork of handing request parameters off to business-logic classes and then marshalls the business-logic result back to send as part of the HTTP response. It is the object that allows us to pull our business logic out of the controllers and into individual classes that *can* be more readily test-able. Here we simply use a fake `Mediator` and set it up to respond with our empty viewmodel. We then tell our IoC Container to use our already contstructed instance of our controller when it needs to resolve one and voila - we can now test an end-to-end HTTP request/response scenario against our Web API application and be sure it goes through our mocked-up controller.

##The Test Base Class
As you might be able to tell, the test above is relying quite a bit on its base class `ApiControllerTestBase`. What's going on there?

```csharp
using System;
using System.Text;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Web.Http;
using System.Threading;
using System.Threading.Tasks;
using Owin;
using Microsoft.Owin.Testing;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public abstract class ApiControllerTestBase
{
    public HttpConfiguration HttpConfig { get; set; }
    public TestServer Server { get; set; }
    public HttpClient Client { get; set; }
    public FakeMediator Mediator { get; set; }

    [TestInitialize]
    public override void Init()
    {
        // An override-able fake of a Mediatr.Mediator
        Mediator = new FakeMediator();

        var config = new HttpConfiguration();
        // make sure that Web API sets up the same routes and configuration
        // as the app will in production
        WebApiConfig.Register(config);

        // more to come on this one
        config.MessageHandlers.Add(new UnitTestMessageHandler());
        config.IncludeErrorDetailPolicy = IncludeErrorDetailPolicy.Always;

        // In-memory Web API Server for unit testing
        Server = TestServer.Create(app => 
            {
                app.UseWebApi(config);
            });

        Client = Server.HttpClient;
    }

    public override void Cleanup()
    {
        this.Client.Dispose();
        this.Server.Dispose();
        this.HttpConfig.Dispose();
    }
}
```
Effectively, there's nothing much more here than setting up and tearing down an in-memory instance of a Web API server each time a test is ran. We go through some steps to ensure the test server's configuration is the same as our actual application, and we inject a special message handler (more below). The `TestServer` class is actually a product of Microsoft's effort to make unit testing Web API even easier. You can get it with this NuGet Package: [Microsoft.Owin.Testing](https://www.nuget.org/packages/Microsoft.Owin.Testing/)
```powershell
Install-Package Microsoft.Owin.Testing
```

The `UnitTestMessageHandler` is a special `MessageHandler` I've created to help setup my requests within unit tests to give me a hand when I need one. I usually start out by just making sure the `IsLocal` value is in the request in case I have any logic that uses that within my app (i.e. don't call production API's, etc...): 

```csharp
public class UnitTestMessageHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
                            HttpRequestMessage request, 
                            CancellationToken cancellationToken)
    {
        // make all of our requests appear "local"
        request.Properties["MS_IsLocal"] = new Lazy<bool>(() => true);

        return base.SendAsync(request, cancellationToken);
    }
}
```
With this in place, I can add any other values into my requests to mimic production issues/requests as need be.

##Requests that require Authorization
So finally, the point of this entire post. What do we do when we have controllers or actions that require authorization (or not just even authorization, but also specific roles)? Well we need a way to mock a user for our requests and then tell the pipeline to think that user is authorized. This ultimately comes down to using the `IPrincipal` and `IIdentity` -based classes. Nowadays this will most likely involve setting up a fake `ClaimsIdentity` and `ClaimsPrincipal`. I do that by sticking those values on my test base class, where I can tweak them as need be for each test. The following will setup a real ClaimsPrincipal similar to what is created by [ASP.NET Identity](http://www.asp.net/identity/overview/getting-started/aspnet-identity-recommended-resources)
```csharp
[TestClass]
public abstract class ApiControllerTestBase
{
    // ...
    
    public ClaimsPrincipal User { get; set; }

    [TestInitialize]
    public override void Init()
    { 
        // ....
        
        this.User = SetupClaimsPrincipal();
    }
    
    private ClaimsPrincipal SetupClaimsPrincipal()
    {
        var claims = new List<Claim>();
            
        // UserId
        claims.Add(new Claim(
            issuer:@"LOCAL AUTHORITY",
            type:@"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier",
            value:@"e427434e-b922-4138-ba0b-6fd88188da12",
            valueType: @"http://www.w3.org/2001/XMLSchema#string"
        ));

        // UserName
        claims.Add(new Claim(
            issuer: @"LOCAL AUTHORITY",
            type: @"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
            value: @"john.doe@test.com",
            valueType: @"http://www.w3.org/2001/XMLSchema#string"
        ));

        // Provider
        claims.Add(new Claim(
            issuer: @"LOCAL AUTHORITY",
            type: @"http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider",
            value: @"ASP.NET Identity",
            valueType: @"http://www.w3.org/2001/XMLSchema#string"
        ));

        // SecurityStamp
        claims.Add(new Claim(
            issuer: @"LOCAL AUTHORITY",
            type: @"AspNet.Identity.SecurityStamp",
            value: @"a0605331-ac20-42de-9e2c-8bc32bf3299b",
            valueType: @"http://www.w3.org/2001/XMLSchema#string"
        ));

        // Role
        claims.Add(new Claim(
            issuer: @"LOCAL AUTHORITY",
            type: @"http://schemas.microsoft.com/ws/2008/06/identity/claims/role",
            value: @"Administrator",
            valueType: @"http://www.w3.org/2001/XMLSchema#string"
        ));

        var roleType = @"http://schemas.microsoft.com/ws/2008/06/identity/claims/role";
        var nameType = @"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name";
        var identity = new ClaimsIdentity(claims, "ApplicationCookie", nameType, roleType);
        var principal = new ClaimsPrincipal(identity);

        return principal;
    }
```
And then I will use this configured user within my setup logic for my base class as so: 
```csharp
[TestClass]
public abstract class ApiControllerTestBase
{
    // ...
    
    public ClaimsPrincipal User { get; set; }

    [TestInitialize]
    public override void Init()
    { 
        // ... other setup
        
        this.User = SetupClaimsPrincipal();
        
        // ... Web API config setup
        
        config.MessageHandlers.Add(new UnitTestMessageHandler(this.User));
        
        // ... other setup
    }
    
    // ...
}

public class UnitTestMessageHandler : DelegatingHandler
{
    public UnitTestMessageHandler(ClaimsPrincipal user)
    {
        this.User = user;
    }
    
    public ClaimsPrincipal User { get; set; }

    protected override Task<HttpResponseMessage> SendAsync(
                            HttpRequestMessage request, 
                            CancellationToken cancellationToken)
    {
        // setup the Request's principal
        var ctx = request.GetRequestContext();
        ctx.Principal = this.User;
        request.SetRequestContext(ctx);
        
        // make all of our requests appear "local"
        request.Properties["MS_IsLocal"] = new Lazy<bool>(() => true);

        return base.SendAsync(request, cancellationToken);
    }
}
```
And that's it. Once you've got that plumbing in place, you can test all kinds of different authorization scenarios with your Web API application and have fairly fine-grained control over what's being faked and what's not. I've personally gotten a lot of mileage on this style of setup in my test suites - and I hope that you find it valuable as well. If you have any further ideas - please sound off in the comments.
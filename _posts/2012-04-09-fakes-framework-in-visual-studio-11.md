---
layout: post
title: "Fakes Framework in Visual Studio 11"
date: 2012-04-09 04:48:29 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/04/08/fakes-framework-in-visual-studio-11/
---

Microsoft has introduced the Fakes Framework in Visual Studio 11, which is the replacement for both private accessors and the original [Moles](http://research.microsoft.com/en-us/projects/moles/) framework.

For those that are unfamiliar with Moles, it was an isolation framework that allowed us to replace dependencies in our unit tests. For example, if our legacy code uses the static method `Util.GetUser()` to retrieve information from a database, there isn't an easy way to change or remove that behavior. Moles allowed us to intercept the call to `Util.GetUser()` and redirect it to our own implementation. This is important because we want to keep our unit tests isolated, free from external dependencies such as the database. Keeping the results deterministic is also a good way to ensure that our test results are consistent.

The Fakes Framework introduces two types of fakes: stubs and shims. Stubs allow us to replace a method with our own implementation for interfaces and virtual methods. Shims allow us to override methods that usually can't be overridden, such as static methods and non-virtual methods. Shims also allow us to intercept calls to types found in the .NET base class library.

{% raw %}
```csharp
public class AuthenticationService
{
    private readonly IUserRepository repository;

    public AuthenticationService(IUserRepository repository)
    {
        this.repository = repository;
    }

    public bool Authenticate(string username, string password)
    {
        User user = repository.GetUser(username);
        return user.Password == password;
    }
}
```
{% endraw %}

For our test, we want to make sure that `Authenticate()` returns `false` when the password does not match what's returned from the repository. The `AuthenticationService` requires an instance of `IUserRepository`. We can use a stub to replace the method call `GetUser()`.

The Fakes Framework will generate strongly typed fakes for each assembly that we want to fake. For stubs, each type will be prefixed with `Stub`. In the test below, we'll be using the stub `StubIUserRepository`, which is an implementation of the interface `IUserRepository`. Each method/property on the stubbed type will contain delegates that are named very similar to the original method/property. We can change the implementation of the method `GetUser()` by assigning a delegate to the `GetUserString` field on the stubbed type.

{% raw %}
```csharp
[TestMethod]
public void Demo()
{
    var repository = new StubIUserRepository();

    repository.GetUserString = 
        username => new User { Name = "Malvin", Password = "secret" };

    var service = new AuthenticationService(repository);
    var isAuthenticated = service.Authenticate("Malvin", "password");

    Assert.IsFalse(isAuthenticated);
}
```
{% endraw %}

When `GetUser()` is called from `AuthenticationService`, the repository will return our hard-coded `User` with the name "Malvin" and password "secret".

However, there are cases when it isn't possible to replace the repository:

{% raw %}
```csharp
public class AuthenticationService
{
    public bool Authenticate(string username, string password)
    {
        UserRepository repository = new UserRepository();

        User user = repository.GetUser(username);

        return user.Password == password;
    }
}
```
{% endraw %}

Let's assume that we want to run the same test again. Instead of using stubs, we'll use a shim to intercept the call to `GetUser()`.

The Fakes Framework will generate strongly typed shims for each type in the original assembly. Each type are prefixed with `Shim`. Before we can use shims, we must create a new context under which the shims will be active.

In our test, we are specifying that for all calls to `GetUser()` for any instance of `UserRepository` while inside the `ShimContext`, we want to intercept that call and return our own `User`.

{% raw %}
```csharp
[TestMethod]
public void Demo()
{
    using (ShimsContext.Create())
    {
        ShimUserRepository.AllInstances.GetUserString =
            (repository, username) => new User { Name = "Malvin", Password = "secret" };

        var service = new AuthenticationService();
        var isAuthenticated = service.Authenticate("Malvin", "password");

        Assert.IsFalse(isAuthenticated);    
    }
}
```
{% endraw %}

For cases where we may not want to override the behavior for all instances of a type, we can create individual instances:

{% raw %}
```csharp
var repository = new ShimUserRepository();
repository.GetUserString = username => new User();
```
{% endraw %}

We can also replace static methods with our own implemention.

{% raw %}
```csharp
public static class Util
{
    public static User GetUser(string username)
    {
        // ...
    }
}
```
{% endraw %}

The shim:

{% raw %}
```csharp
ShimUtil.GetUserString = username => new User();
```
{% endraw %}

Shims also allow us to override the behavior of base classes. Here we have a `CachedUserRepository`, which caches any users returned from `UserRepository`. If a user is found in the cache, the cached user will be returned. If a user is not found in the cache, we will use the base class to retrieve the user and add it to the cache.

{% raw %}
```csharp
public class CachedUserRepository : UserRepository
{
    private static List<User> cache = new List<User>();

    public new User GetUser(string username)
    {
        User cachedUser = cache.FirstOrDefault(x => x.Name == username);
            
        if (cachedUser != null)
            return cachedUser;

        User user = base.GetUser(username);

        if (user != null)
            cache.Add(user);

        return user;
    }
}
```
{% endraw %}

Using shims, we can replace the the base method `base.GetUser()`. By creating a shim for the base class and passing the inheriting class to the shim's constructor, we can override the base class's behavior.

{% raw %}
```csharp
[TestMethod]
public void Demo()
{
    using (ShimsContext.Create())
    {
        var cachedRepo = new CachedUserRepository();
        var baseClass = new ShimUserRepository(cachedRepo);

        baseClass.GetUserString = 
            username => new User { Name = "Malvin", Password = "password" };

        var user = cachedRepo.GetUser("x");

        Assert.AreEqual("Malvin", user.Name);
    }
}
```
{% endraw %}

The new Fakes Framework is definitely a great tool for testing legacy code, but I would be hesitant to use it for anything else. Most existing mocking frameworks have a much cleaner API. Refactoring can be difficult since the framework generates strongly typed stubs and shims for an assembly. Renaming methods or classes in the original assembly will cause your tests to fail. I would also suggest that if you're using shims extensively in a new project, there's probably something wrong with your design.

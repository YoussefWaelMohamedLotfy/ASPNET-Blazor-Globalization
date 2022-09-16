# Blazor Globalization & Localization

Demonstration of Supporting multiple cultures in Blazor Server App.

[See Micorsoft's Documentation for more info on Globalization in Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/globalization-localization?view=aspnetcore-6.0&pivots=server)

## Steps to add Localization Features in your Blazor App

1. Register Localization Services to your DI Container.

```C#
builder.Services.AddLocalization();

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new List<CultureInfo>
    {
        new CultureInfo("en"),
        new CultureInfo("es"),
        new CultureInfo("ar"),
    };

    options.SetDefaultCulture("en");
    options.SupportedCultures = supportedCultures;
    options.SupportedUICultures = supportedCultures;
});
```

> The input parameter for *CultureInfo* Type should conform to the [ISO 639-1 standard](https://en.wikipedia.org/wiki/ISO_639-1)

2. Add Request Localization to your Request Pipeline.

```c#
app.UseRequestLocalization(app.Services.GetRequiredService<IOptions<RequestLocalizationOptions>>().Value);
```

3. Add Resource (.resx) Files for each Culture you want to support.

For example, Add a new Resource file `Resource.resx` or any name you want. This is the base Culture (English, in my case) that you support in your apps. If you want to add Spanish Culture to your app, simply create a new Resource file `Resource.es.resx`.

> Note that for each additional Resource File you add for a culture, you should add its Language Code before the file extension.

You can view more about what a Resource File is from [this link](https://fileinfo.com/extension/resx).

4. Create a Blazor Component for handling Culture Change.

The following snippet shows a minimal Component, where it contains only a `<label>` and `<select> tags for UI`.

The Component can be organized such that the `@code` section can be separated in its on file, named as its components name followed by `.cs` extension (CultureDropDown.razor.cs)

CultureDropDown.razor:
```html
@using System.Globalization
@inject NavigationManager Nav
@inject IServiceProvider serviceProvider
@inject IStringLocalizer<Resource> localizer

<label class="align-baseline">
    @localizer["CultureLabel"]
    <select @bind="Culture">
        @foreach (var culture in serviceProvider.GetRequiredService<IOptions<RequestLocalizationOptions>>().Value.SupportedCultures!)
        {
            <option value="@culture">@culture.NativeName</option>
        }
    </select>
</label>

@code
{
    protected override void OnInitialized()
    {
        Culture = CultureInfo.CurrentCulture;
    }

    private CultureInfo Culture
    {
        get => CultureInfo.CurrentCulture;
        set
        {
            if (CultureInfo.CurrentCulture != value)
            {
                var uri = new Uri(Nav.Uri).GetComponents(UriComponents.PathAndQuery, UriFormat.Unescaped);
                var cultureEscaped = Uri.EscapeDataString(value.Name);
                var uriEscaped = Uri.EscapeDataString(uri);

                Nav.NavigateTo($"Culture/Set?culture={cultureEscaped}&redirectUri={uriEscaped}", forceLoad: true);
            }
        }
    }
}
```

Then use the `CultureDropDown` tag in your `MainLayout.razor` or wherever it suits your needs.

> Don't forget to add any missing `using` statements or you will get Compilation Errors.

5. Add a Minimal API Endpoint to add a Culture Cookie to store User's Culture preference and redirect to the same page that you were in.

```c#
app.MapGet("Culture/Set", (string culture, string redirectUri, HttpContext context) =>
{
    if (culture is not null)
    {
        context.Response.Cookies.Append(CookieRequestCultureProvider.DefaultCookieName,
            CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(culture, culture)),
            new CookieOptions { Expires = DateTimeOffset.Now.AddDays(30) });
    }

    return Results.LocalRedirect(redirectUri);
});
```

> Of course, you can change the route pattern to whatever you want.

6. Inject the `IStringLocalizer<Resource>` in your razor components and access a specific resource record using its Key Name that you specified in the resource file.

Index.razor:
```html
@page "/"
@inject IStringLocalizer<Resource> localizer

<PageTitle>Index</PageTitle>
<h1>@localizer["helloworld"]</h1>
@localizer["welcome"]
```
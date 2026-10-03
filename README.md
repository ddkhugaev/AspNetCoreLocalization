# Гайд по двуязычному интерфейсу

## Каркас одноязычной локализации

Добавить сервисы локализации в `Program`
```c#
builder.Services.AddControllersWithViews()
    .AddViewLocalization();
```

Определить папку с локализацией в `Program`
```c#
builder.Services.AddLocalization(option => option.ResourcesPath = "Resources");
```

Определить поддерживаемые языки и язык по умолчанию в `Program`
```c#
builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new[] { "ru", "os", "en" };
    options.SetDefaultCulture(supportedCultures[0])
    .AddSupportedCultures(supportedCultures)
    .AddSupportedUICultures(supportedCultures);
});
```

Добавить локализацию в конвейер в `Program`
```c#
app.UseRequestLocalization();
```

Создать папку `Resources` в корне проекта

Создать для каждого языка по общему файлу в `Resources`

`SharedResource.ru.resx`

`SharedResource.os.resx`

`SharedResource.en.resx`

Создать класс-заглушку в корне проекта
```c#
public class SharedResource
{
}
```

Создать контроллер, меняющий язык в cookie
```c#
public class LanguageController : Controller
{
    public IActionResult Index(string culture)
    {
        Response.Cookies.Append(
            CookieRequestCultureProvider.DefaultCookieName,
            CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(culture)),
            new CookieOptions { Expires = DateTimeOffset.UtcNow.AddYears(1) }
        );
        
        var returnUrl = Request.Headers.Referer.ToString();
		return Redirect(string.IsNullOrEmpty(returnUrl) ? "/" : returnUrl);
    }
}
```

Добавить ссылки для смены языка в шапке
```html
<li><a class="nav-link text-dark" asp-area="" asp-controller="Language" asp-action="Index" asp-route-culture="ru">Русский</a></li>
<li><a class="nav-link text-dark" asp-area="" asp-controller="Language" asp-action="Index" asp-route-culture="os">Ирон</a></li>
<li><a class="nav-link text-dark" asp-area="" asp-controller="Language" asp-action="Index" asp-route-culture="en">English</a></li>
```

Подключить локализатор в `_ViewImports`
```html
@using Microsoft.AspNetCore.Mvc.Localization
@inject IHtmlLocalizer<SharedResource> SharedLocalizer
```

**Каркас одноязычной локализации готов**

## Каркас двуязычной локализации

Добавить метод для сохранения в cookie второго языка в `LanguageController`
```c#
public IActionResult Second(string culture)
    {
        Response.Cookies.Append(
            "SecondLanguage",
            culture,
            new CookieOptions
            {
                Expires = DateTimeOffset.UtcNow.AddYears(1),
                IsEssential = true
            }
        );
        
        var returnUrl = Request.Headers.Referer.ToString();
		return Redirect(string.IsNullOrEmpty(returnUrl) ? "/" : returnUrl);
    }
```

Добавить сервис для второго языка в `Helpers`
```c#
using System.Globalization;
using System.Resources;

namespace LingvoGameOs.Helpers
{
    public class SecondLocalizer
    {
        private readonly ResourceManager _rm;
        private readonly IHttpContextAccessor _http;

        public SecondLocalizer(IHttpContextAccessor http)
        {
            _http = http;
            _rm = new ResourceManager(
                "AspNetCoreLocalization.Resources.SharedResource",
                typeof(SharedResource).Assembly);
        }

        public string this[string key]
        {
            get
            {
                var lang = _http.HttpContext?.Request.Cookies["SecondLanguage"];
                if (string.IsNullOrEmpty(lang))
                    lang = "ru";

                return _rm.GetString(key, new CultureInfo(lang)) ?? key;
            }
        }
    }
}
```

Дать доступ к HttpContext вне контроллеров и зарегистрировать `SecondLocalizer` в `Program`
```c#
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<SecondLocalizer>();
```

Добавить сервис в `_ViewImports`
```html
@using AspNetCoreLocalization.Helpers
@inject SecondLocalizer SecondLocalizer
```

Добавить ссылки для смены второго языка
```html
<li><a class="nav-link text-dark" asp-controller="Language" asp-action="Second" asp-route-culture="ru">Русский (доп)</a></li>
<li><a class="nav-link text-dark" asp-controller="Language" asp-action="Second" asp-route-culture="os">Ирон (доп)</a></li>
<li><a class="nav-link text-dark" asp-controller="Language" asp-action="Second" asp-route-culture="en">English (доп)</a></li>
```

Добавить локализацию текста и всплывашки на главной странице (с любым тегом работает так же)
```html
<p>
    <span data-tooltip="@SecondLocalizer["hello"]">@SharedLocalizer["hello"] (наведи на меня курсор)</span>
</p>
```

Сделать фронт для всплывашки

**Каркас двуязычной локализации готов**

*Теперь добавляем в файлы ресурсов ключ для блока текста и перевод на все языки, а потом в представлениях обращаемся к локалайзерам по этому ключу. Перевод будет браться из файлов в зависимости от языка в cookie*

# Esia.NET
.NET библиотека для подключения к ЕСИА по протоколу OAuth 2.0 и OpenId Connect 1.0, сервисы REST.

Информация об ЕСИА на сайте Минкомсвязь России: [Единая система идентификации и аутентификации (ЕСИА)](http://minsvyaz.ru/ru/activity/directions/13/#section-description)

ESIA OAuth 2.0 client to russian government services for .NET framework.

## Documentation
Esia.NET состоит из следующих Nuget-пакетов:
- EsiaNET - основной http-клиент для работы с ЕСИА;
- EsiaNET.Identity - OWIN middleware для авторизации через ЕСИА. Зависит от EsiaNET.

### EsiaNET
#### Установка и использование
Для установки необходимо ввести следующую команду в Package Manager Console:
```
PM> Install-Package EsiaNET
```
Далее работа с ЕСИА осуществляется через класс EsiaClient. При создании экземпляра класса требуется указать параметры подключения к ЕСИА и сведения о системе-клиенте (класс EsiaOptions). Многие параметры имеют значения по умолчанию.
```C#
var esiaClient = new EsiaClient(new EsiaOptions
{
    ClientId = "YOUR_CLIENT_SYSTEM_ID",   // Идентификатор вашей систему. Обязателен
    Scope = "fullname id_doc",  // Требуемый scope. Обязателен
    CallbackUri = "/your_app_uri",  // Адрес возврата в ваше приложение после авторизации ЕСИА.
    RedirectUri = EsiaConsts.EsiaAuthTestUrl,   // Адрес перенаправления на страницу предоставления прав доступа в ЕСИА - либо тестовый, либо рабочий. По умолчанию - рабочий https://esia.gosuslugi.ru/aas/oauth2/ac
    TokenUri = EsiaConsts.EsiaTokenTestUrl,     // https-адрес ЕСИА для получения маркера доступа - либо тестовый, либо рабочий. По умолчанию - рабочий https://esia.gosuslugi.ru/aas/oauth2/te
    RestUri = EsiaConsts.EsiaRestTestUrl,       // Адрес REST-сервиса ЕСИА для получения данных - либо тестовый, либо рабочий. По умолчанию - рабочий https://esia.gosuslugi.ru/rs
    // Провайдер для получения сертификата системы-клиента. Обязан вернуть сертификат. В данном примере сертификат ищется на локальной машине по серийному номеру. Обязателен
    SignProvider = EsiaOptions.CreateSignProvider(() =>
    {
        // Будем искать сертификат в личном хранилище на локальной машине
        X509Store storeMy = new X509Store(StoreName.My, StoreLocation.LocalMachine);
        storeMy.Open(OpenFlags.OpenExistingOnly);
        X509Certificate2Collection certColl = storeMy.Certificates.Find(X509FindType.FindBySerialNumber, "SERIAL_ID", false);

        storeMy.Close();

        return certColl[0];
    },
        // Action должен вернуть сертификат ЕСИА тестовый или рабочий. В данном примере ищем сертификат по его пути, указанном в конфигурационном файле
        () => new X509Certificate2(WebConfigurationManager.AppSettings["EsiaCertFile"]))
});
```
В случае, если маркер доступа уже известен, то можно передать его в конструктор вторым параметром
```C#
EsiaToken token = <variable>;
var esiaClient = new EsiaClient(Esia.GetOptions(), token);
// или
string token = <variable>;
var esiaClient = new EsiaClient(Esia.GetOptions(), token);
```
#### Авторизация
Для осуществления авторизации через ЕСИА необходимо перенаправить пользователя на страницу авторизации ЕСИА с необходимыми параметрами. EsiaClient может только сгенерировать правильный адрес перенаправления, далее ваше приложение должно перенаправить пользователя на этот адрес требуемым вам способом. При создании экземпляра EsiaClient в параметрах обязательно требуется указание адреса возврата в ваше приложение (параметр CallbackUri). Ваше приложение, по этому адресу, должно получить ответ от ЕСИА.
```C#
var redirectUri = esiaClient.BuildRedirectUri();
// your redirect logic
```
Пакет EsiaNET.Identity встраивает логику авторизации (перенаправление в ЕСИА и получение ответа) в ваше приложение автоматически и интегрирует с ASP.NET Identity.

#### Возврат в ваше приложение
После осуществления пользователем авторизации через ЕСИА ваше приложение должно получить ответ по адресу возврата (реализуется вашим приложением). В случае использования EsiaNET.Identity это реализуется автоматически.
Далее вы должны обработать ответ и получить код авторизации (более подробно написано в методических рекомендациях). Имея код авторизации вы можете получить маркер доступа следующим образом:
```C#
var tokenResponse = await esiaClient.GetOAuthTokenAsync(authCode);
```
Вы можете проверить подпись полученного маркера доступа:
```C#
if ( !esiaClient.VerifyToken(tokenResponse.AccessToken) ) throw new Exception("Token signature is invalid");
```
И установить полученный маркер доступа в экземпляр класса EsiaClient (или сохранить для дальнейшего использования):
```C#
EsiaToken token = EsiaClient.CreateToken(tokenResponse);
esiaClient.Token = EsiaClient.CreateToken(token);
```
Теперь вы можете получить данные из REST сервисов ЕСИА.

#### Получение данных из ЕСИА
```C#
// Получим данные о пользователе
var personInfo = await esiaClient.GetPersonInfoAsync();

// Пользователь не подтвержден - выводим ошибку
if ( !personInfo.Trusted ) throw new Exception("Данные не подтверждены");

// Получаем контакты
var contacts = await esiaClient.GetPersonContactsAsync();

// Получаем документы пользователя
var docs = await esiaClient.GetPersonDocsAsync();

// Получаем адреса пользователя
var addrs = await esiaClient.GetPersonAddrsAsync();

// Получаем транспортные средства
var vehicles = await esiaClient.GetPersonVehiclesAsync();

// Получаем детей
var kids = await esiaClient.GetPersonKidsAsync();

foreach ( var child in kids )
{
    // Получаем документы ребенка
    var childDoc = await esiaClient.GetPersonChildDocsAsync(child.Id);
}
```
Если на предыдущем шаге вы сохранили маркер доступа, то по необходимости вы можете создать экземпляр класса EsiaClient с указанием параметров и маркера доступа и получить данные:
```C#
// Создаем ЕСИА клиента с маркером доступа для получения данных
var esiaClient = new EsiaClient(Esia.GetOptions(), (EsiaToken)esiaToken);

// Получим данные о пользователе
var personInfo = await esiaClient.GetPersonInfoAsync();
```

### EsiaNET.Identity
Coming soon

## Examples
### [ASP.NET Identity ESIA authorization example](https://github.com/xeltan/EsiaNET/tree/master/examples/ESIA.AspNetIdentityExample)

Пример интеграции Esia.NET и ASP.NET Identity для реализации авторизации через ЕСИА и получения данных.

Приложение создано из стандартного шаблона ASP.NET MVC Visual Studio. Вся настройка ЕСИА находится в файлах App_Start/Esia.cs и App_Start/Startup.Auth.cs, получение данных - в файле Controllers/HomeController.cs. Для запуска примера необходимо установить необходимые параметры вашей системы-клиента в файле App_Start/Esia.cs - это идентификатор системы и сертификат, а также сертификат ЕСИА (тестовый или рабочий).

## Support
Добро пожаловать в Issues или почту lettapost@gmail.com.
Информация здесь наполняется и скоро будет содержать более полную картину.

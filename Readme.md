# LibCore

[Текущий релиз](https://github.com/CryptoPro/libcore/releases).

**ВАЖНО!!! Для корректной работы библиотеки необходима актуальная версия КриптоПро CSP 5.0. 12900**

Также необходима установка [sdk и runtime dotnet 6](https://dotnet.microsoft.com/en-us/download/dotnet/6.0).

Поддерживается **только** dotnet 6.
т.е. в csproj должен быть явно указан
```xml
<TargetFramework>net6.0</TargetFramework>
```

В качестве целевой платформы в настоящий момент поддерживаются только linux-x64 и Windows-x64.

Интерфейсы и классы во многом аналогичны corefx и КриптоПро.NET.

### В настоящий момент в работе/не реализовано:

* 

## Известные проблемы
### 1. (исправлена) Ошибка разрешения сбоки `System.Security.Pkcs` при инициализации

**NB:** если данная проблема ещё встречаются - просьба сообщить открыв Issue на Github а данном проекте. В настоящий момент должна быть исправлена.

При использовании исправлений сборок `System.Security.Pkcs` перед инициализацией бибилиотеки
возможно появление runtime ошибок разрешения сборок `System.Security.Pkcs`. Для их исправления
необходимо создать объект класса `CmsSigner` для форсирования инициализации сборки.

```csharp
var signed = new CmsSigner();
LibCore.Initializer.Initialize();
```

### 2. (исправлена) Возможны проблемы при работе двухстороннего RSA TLS.

**NB:** если данная проблема ещё встречаются - просьба сообщить открыв Issue на Github а данном проекте. В настоящий момент должна быть исправлена.

Если необходимо использовать только RSA TLS, а LibCore ломает существующий код - возможно 
вызвать инициализацию без установки исправлений для System.Net.

```csharp
LibCore.Initializer.Initialize(
    LibCore.Initializer.DetouredAssembly.Xml |
    LibCore.Initializer.DetouredAssembly.Pkcs); 
```

### 3. Ошибка при многократном повторении вызово на CentOs/Red OS
На данных операционных системах возможно возникновение runtime ошибок при многократных вызовах, вызванных [tired compilation](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation). Отключить её можно через `csproj` проекта следующим образом

```xml
<PropertyGroup>
    <TieredCompilation>false</TieredCompilation>
</PropertyGroup>
```
[Подробнее](https://github.com/CryptoPro/libcore/issues/7)

### 4. Ошибка при работе EnvelopedCms на не-ГОСТовых сертификатах
В настоящий момент исправления `EnvelopedCms` работают только с ГОСТовыми ключами, ломая сценарий RSA. Если в проекте необходимо воспрользоваться RSA и/или ГОСТом одновременно - можно отключить установку исправлений для `EnvelopedCms`. 

```csharp
LibCore.Initializer.Initialize(
    debugFlags: LibCore.Initializer.DebugFlags.DisableEnvelopedCmsDetours);

```

После её отключения для шифрования/расшифрования на ГОСТовых ключах необходимо использовать класс `CpEnvelopedCms`, для RSA ключей - `EnvelopedCms`.


### 4. Ошибка при работе EnvelopedCms в Csp 5.0 12800

**NB:** Исправлена в 5.0 12900.

Наблюдается ошибка при работе `EnvelopedCms`. [Разбираемся](https://github.com/CryptoPro/libcore/issues/17).

Версия 12600 - рабочая. Версия 12900 содержит исправление.

## Примеры
Большинство примеров из КриптоПро.NET и corefx работают с небольшими изменениями.

 - [Установка и инициализация библиотеки](#init) 
 - [Загрузка сертификата из pfx файла](#file-pfx) 
 - [Загрузка сертификата из хранилища](#store-pfx) 
 - [Создание сертификата](#pfx-create)
 - [SignedXml](#signed-xml)
   - [Подпись](#signed-xml-sign)
   - [Проверка](#signed-xml-verify)
 - [EncryptedXml](#encrypted-xml) (СКОРО)
 - [SignedCms](#signed-cms)
   - [Attached](#signed-cms-attached)
   - [Detached](#signed-cms-detached)  
 - [EnvelopedCms](#enveloped-cms) (СКОРО)
 - [Работа с контейнерами и провайдерами](#container)
    - [Открытие контейнера ключа](#open-container)
    - [Создание ключевого контейнера](#create-container)
    - [Выставление OID](#set-oid)
 - [Ключевой транспорт, KeyWrap](#key)
   - [Symmetric KeyWrap](#key-wrap)
   - [Asymmetric agree (DH)](#key-agree)
   - [Asymmetric KeyExchange (key encryption)](#key-exchange)

## <a id="init"> Установка и инициализация библиотеки
Скачать nuget пакет `LibCore.Linux.XXXX.XX.XX.nupkg` или `LibCore.Windows.XXXX.XX.XX.nupkg` из текущего [релиза](https://github.com/CryptoPro/libcore/releases)
в папку по некоторому пути `packages_PATH`.

Изменить файл `%appdata%\NuGet\NuGet.Config` (Windows) или 
`~/.nuget/NuGet/NuGet.Config` (Linux), добавив в начало узла 
`packageSources` источник `<add key="local packages" value="packages_PATH" />`.

Пример:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="local packages" value="C:\packages" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
  </packageSources>
</configuration>
```

Для использования бибилиотеки необходимо однократно вызвать инициализацию:
```csharp
using LibCore;

namespace Sample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            LibCore.Initializer.Initialize();
            // ready to go!
        }
    }
}
```

В `Initialize(...)` можно указать только требуемые библиотеки, использование 
которых предполагается в рамках проекта.
```csharp
using LibCore;

namespace Sample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Инициализируем LibCore для использования XML подписи.
            // Исправления для TLS и PKCS не будут установлены.
            // Исправления для Csp, Algorithms и Primitives будут установлены
            // т.к. от них зависят исправления XML.
            LibCore.Initializer.Initialize(LibCore.Initializer.DetouredAssembly.Xml);
            // ready to go!
        }
    }
}
```

## <a id="store-pfx"> Загрузка сертификата из хранилища
```csharp
// unix, using CpX509Store
using (var store = new CpX509Store(StoreName.My, StoreLocation.CurrentUser))
{
    store.Open(OpenFlags.ReadOnly);
    var cert = store.Certificates.Find(X509FindType.FindBySubjectName, "G2012256", false)[0];
}
```

```csharp
// windows, using X509Store
using (var store = new X509Store(StoreName.My, StoreLocation.CurrentUser))
{
    store.Open(OpenFlags.ReadOnly);
    var cert = store.Certificates.Find(X509FindType.FindBySubjectName, "G2012256", false)[0];
}
```

## <a id="file-pfx"> Загрузка сертификата из pfx файла

```csharp
public static void ImportGostPfxWithNonPersistKey()
{
    using (var certificate = X509CertificateExtensions.Create(
        Gost2012_256Pfx, // string path of byte[] data
        "1", // password
        CpX509KeyStorageFlags.CspNoPersistKeySet))
    {
        var dataToBeSigned = new byte[] { 0 };

        var privateKey = (Gost3410_2012_256CryptoServiceProvider certificate.PrivateKey 
            as Gost3410_2012_256CryptoServiceProvider;
        
        var publicKey = certificate.PublicKey.Key 
            as Gost3410_2012_256CryptoServiceProvider;

        var signature = privateKey.SignData(
            dataToBeSigned, 
            CpHashAlgorithmName.Gost3411_2012_256);

        var result = publicKey.VerifyData(
            dataToBeSigned, 
            signature, 
            CpHashAlgorithmName.Gost3411_2012_256);
    }
}
```

## <a id="pfx-create"> Создание сертификата

```csharp
var certificateRequest = new CpCertificateRequest(
    cn,
    (Gost3410_2012_256)provider,
    CpHashAlgorithmName.Gost3411_2012_256);

certificateRequest.CertificateExtensions.Add(
    new X509KeyUsageExtension(
        X509KeyUsageFlags.DigitalSignature 
        | X509KeyUsageFlags.NonRepudiation 
        | X509KeyUsageFlags.KeyEncipherment,
        false));

var oidCollection = new OidCollection();
// Проверка подлинности клиента (1.3.6.1.5.5.7.3.2)
oidCollection.Add(new Oid("1.3.6.1.5.5.7.3.2"));

certificateRequest.CertificateExtensions.Add(
new X509EnhancedKeyUsageExtension(
    oidCollection,
    true));

certificateRequest.CertificateExtensions.Add(
new X509SubjectKeyIdentifierExtension(certificateRequest.PublicKey, false));

var cert = certificateRequest.CreateSelfSigned(
    DateTimeOffset.Now.AddDays(-45),
    (DateTimeOffset.Now.AddDays(45)));

byte[] file = cert.Export(X509ContentType.Pfx, new SecureString());
```

## <a id="signed-xml"> SignedXml

### <a id="signed-xml-sign"> Формирование подписи:

```csharp
static XmlDocument SignXmlFile(
    XmlDocument doc,
    AsymmetricAlgorithm Key,
    X509Certificate Certificate,
    string DigestMethod = CpSignedXml.XmlDsigGost3411_2012_256Url)
{
    // Создаем объект SignedXml по XML документу.
    SignedXml signedXml = new SignedXml(doc);

    // Добавляем ключ в SignedXml документ. 
       signedXml.SigningKey = Key;

    // Создаем ссылку на node для подписи.
    // При подписи всего документа проставляем "".
    Reference reference = new Reference();
    reference.Uri = "";

    // Явно проставляем алгоритм хэширования,
    // по умолчанию SHA1.
    reference.DigestMethod = DigestMethod;

    // Добавляем transform на подписываемые данные
    // для удаления вложенной подписи.
    XmlDsigEnvelopedSignatureTransform env =
        new XmlDsigEnvelopedSignatureTransform();
    reference.AddTransform(env);

    // Добавляем СМЭВ трансформ.
    // начиная с .NET 4.5.1 для проверки подписи, необходимо добавить этот трансформ в довернные:
    // signedXml.SafeCanonicalizationMethods.Add("urn://smev-gov-ru/xmldsig/transform");
    XmlDsigSmevTransform smev =
        new XmlDsigSmevTransform();
    reference.AddTransform(smev);

    // Добавляем transform для канонизации.
    XmlDsigC14NTransform c14 = new XmlDsigC14NTransform();
    reference.AddTransform(c14);

    // Добавляем ссылку на подписываемые данные
    signedXml.AddReference(reference);

    // Создаем объект KeyInfo.
    KeyInfo keyInfo = new KeyInfo();

    // Добавляем сертификат в KeyInfo
    keyInfo.AddClause(new KeyInfoX509Data(Certificate));

    // Добавляем KeyInfo в SignedXml.
    signedXml.KeyInfo = keyInfo;

    // Можно явно проставить алгоритм подписи: ГОСТ Р 34.10.
    // Если сертификат ключа подписи ГОСТ Р 34.10
    // и алгоритм ключа подписи не задан, то будет использован
    // XmlDsigGost3410Url
    // signedXml.SignedInfo.SignatureMethod =
    //     CPSignedXml.XmlDsigGost3410_2012_256Url;

    // Вычисляем подпись.
    signedXml.ComputeSignature();

    // Получаем XML представление подписи и сохраняем его 
    // в отдельном node.
    XmlElement xmlDigitalSignature = signedXml.GetXml();

    // Добавляем node подписи в XML документ.
    doc.DocumentElement.AppendChild(doc.ImportNode(
        xmlDigitalSignature, true));

    // При наличии стартовой XML декларации ее удаляем
    // (во избежание повторного сохранения)
    if (doc.FirstChild is XmlDeclaration)
    {
        doc.RemoveChild(doc.FirstChild);
    }

    return doc;
}
```

### <a id="signed-xml-verify"> Проверка подписи:

```csharp
static bool ValidateXmlFIle(XmlDocument xmlDocument)
{
    // Ищем все node "Signature" и сохраняем их в объекте XmlNodeList
    XmlNodeList nodeList = xmlDocument.GetElementsByTagName(
        "Signature", SignedXml.XmlDsigNamespaceUrl);

    // Проверяем все подписи.
    bool result = true;
    for (int curSignature = 0; curSignature < nodeList.Count; curSignature++)
    {
        // Создаем объект SignedXml для проверки подписи документа.
        SignedXml signedXml = new SignedXml(xmlDocument);

        // начиная с .NET 4.5.1 для проверки подписи, необходимо добавить СМЭВ transform в довернные:

        signedXml.SafeCanonicalizationMethods.Add("urn://smev-gov-ru/xmldsig/transform");

        // Загружаем узел с подписью.
        signedXml.LoadXml((XmlElement)nodeList[curSignature]);

        // Проверяем подпись и выводим результат.
        result &= signedXml.CheckSignature();
    }
    return result;
}
```

## <a id="signed-cms"> SignedCms

### <a id="signed-cms-attached"> Формирование и проверка присоединённой подписи:
```csharp
byte[] signature;
using (x509Certificate2 gostCert = GetGostX509Certificate2Somehow())
{
    var contentInfo = new ContentInfo(bytesToHash);
    var signedCms = new SignedCms(contentInfo, false);
    CmsSigner cmsSigner = new CmsSigner(gostCert);

    // Добавляем время подписания в атрибуты, если необходимо
    // cmsSigner.SignedAttributes.Add(new Pkcs9SigningTime(DateTime.Now));

    // Добавляем сертификат в атрибуты, если необходимо
    // cmsSigner.SignedAttributes.Add(new PkcsSigningCertificateV2(gostCert));

    signedCms.ComputeSignature(cmsSigner);
    signature = signedCms.Encode();
    Console.WriteLine($"CMS Sign: {Convert.ToBase64String(signature)}");
}

// Создаем SignedCms для декодирования и проверки.
SignedCms signedCmsVerify = new SignedCms();

// Декодируем подпись
signedCmsVerify.Decode(signature);

// Проверяем подпись
signedCmsVerify.CheckSignature(true);
```

### <a id="signed-cms-detached"> Формирование и проверка отсоединённой подписи:
```csharp
byte[] signature;
using (var gostCert = GetGostX509Certificate2Somehow())
{
    var contentInfo = new ContentInfo(bytesToHash);
    var signedCms = new SignedCms(contentInfo, true);
    CmsSigner cmsSigner = new CmsSigner(gostCert);
    signedCms.ComputeSignature(cmsSigner);
    signature = signedCms.Encode();
    Console.WriteLine($"CMS Sign: {Convert.ToBase64String(signature)}");
}

// Создаем объект ContentInfo по сообщению.
// Это необходимо для создания объекта SignedCms.
ContentInfo contentInfoVerify = new ContentInfo(bytesToHash);

// Создаем SignedCms для декодирования и проверки.
SignedCms signedCmsVerify = new SignedCms(contentInfoVerify, true);

// Декодируем подпись
signedCmsVerify.Decode(signature);

// Проверяем подпись
signedCmsVerify.CheckSignature(true);
```

## <a id="container"> Работа с контейнерами и провайдерами

### <a id="open-container"> Открытие контейнера ключа

```csharp
using (var provider =
    new Gost3410_2012_256CryptoServiceProvider(
        new CspParameters(
            80,
            "",
            "\\\\.\\HDIMAGE\\G2012256")))
{

}
```

### <a id="create-container"> Создание ключевого контейнера (требует гамму)

```csharp
using (var provider =
    new Gost3410_2012_256CryptoServiceProvider(
        new CspParameters()
        {
            Flags = CspProviderFlags.NoPrompt,
            KeyContainerName = $"\\\\.\\HDImage\\0000_test_{Guid.NewGuid()}",
            ProviderName = "Crypto-Pro GOST R 34.10-2012 Cryptographic Service Provider",
            ProviderType = 80,
            KeyPassword = new SecureString(),
            KeyNumber = keyNumber
        }))
{

}        
```

### <a id="set-oid"> Выставление OID

https://github.com/CryptoPro/corefx/issues/45

Провайдер для 2817 теперь можно задавать через новый конструктор
```csharp
public Gost28147CryptoServiceProvider(CspParameters parameters)
```

Выбор CipherOid для `Gost28147CryptoServiceProvider`, `Gost3410_XXXXCryptoServiceProvider` производится через одноименное свойство.

Список CipherOid - https://cpdn.cryptopro.ru/content/csp40/html/group___pro_c_s_p_ex_CP_PARAM_OIDS.html

*****************

Ниже оригинальные комментарии. Логика должна быть аналогичной КриптоПро.NET.

Для установки желаемого OID транспортного ключа необходимо выставить его через свойство CipherOid в объекте **открытого ключа**

Пример установки OID для транспортного ключа:
```csharp
var gost = (Gost3410_2012_256CryptoServiceProvider)cert.PrivateKey;
var gostRes = (Gost3410_2012_256CryptoServiceProvider)certRes.PrivateKey;      
      
var gostPk = (Gost3410_2012_256CryptoServiceProvider)cert.PublicKey.Key;
var gostResPk = (Gost3410_2012_256CryptoServiceProvider)certRes.PublicKey.Key;

gostPk.CipherOid = "1.2.643.2.2.31.1";
gostResPk.CipherOid = "1.2.643.2.2.31.1";

var agree = (GostSharedSecretCryptoServiceProvider)gost.CreateAgree(gostResPk.ExportParameters(false));
byte[] wrappedKeyBytesArray = agree.Wrap(symmetric, GostKeyWrapMethod.CryptoProKeyWrap);

var agreeRes = (GostSharedSecretCryptoServiceProvider)gostRes.CreateAgree(gostPk.ExportParameters(false));
var key = agreeRes.Unwrap(wrappedKeyBytesArray, GostKeyWrapMethod.CryptoProKeyWrap);
```

ps. При использовании метода ExportParameters(false) для **закрытого ключа** может не заполнится поле EncryptionParamSet возвращаемого объекта Gost3410Parameters и тогда будет использоваться Oid по умолчанию, вне зависимости от проставленного OID в закртытом ключе. Можно исправить руками, выставив в Gost3410Parameters.EncryptionParamSet нужное значение:
```csharp
var paramsPk = gostRes.ExportParameters(false);
paramsPk.EncryptionParamSet = gostResPk.CipherOid;            
var agree = (GostSharedSecretCryptoServiceProvider)gost.CreateAgree(paramsPk);
```

## <a id="key"> Ключевой транспорт, KeyWrap

см примеры в https://github.com/CryptoPro/corefx/issues/43

### <a id="key-wrap"> Symmetric KeyWrap

```csharp
var keyWrapMethod = GostKeyWrapMethod.CryptoPro12KeyWrap;
using (Gost28147 gost = new Gost28147CryptoServiceProvider())
{
    using (Gost28147 keyToWrap = new Gost28147CryptoServiceProvider())
    {
        var wrappedKey = gost.Wrap(keyToWrap, keyWrapMethod);
        var unwrappedKey = gost.Unwrap(wrappedKey, keyWrapMethod) as Gost28147;
        var iv = keyToWrap.IV;
    }
}
```

### <a id="key-agree"> Asymmetric agree (DH)

```csharp
var gost = (Gost3410_2012_512CryptoServiceProvider)cert.PrivateKey;
var gostRes = (Gost3410_2012_512CryptoServiceProvider)cert.PrivateKey;

var gostPk = (Gost3410_2012_512CryptoServiceProvider)cert.PublicKey.Key;
var gostResPk = (Gost3410_2012_512CryptoServiceProvider)cert.PublicKey.Key;

var symmetric = new Gost28147CryptoServiceProvider();

// set params, if needed
gostPk.CipherOid = "1.2.643.7.1.2.5.1.1";
gostResPk.CipherOid = "1.2.643.7.1.2.5.1.1";

var agree = (GostSharedSecretCryptoServiceProvider)gost.CreateAgree(gostResPk.ExportParameters(false));
byte[] wrappedKeyBytesArray = agree.Wrap(symmetric, GostKeyWrapMethod.CryptoProKeyWrap);

var agreeRes = (GostSharedSecretCryptoServiceProvider)gostRes.CreateAgree(gostPk.ExportParameters(false));
var key = agreeRes.Unwrap(wrappedKeyBytesArray, GostKeyWrapMethod.CryptoProKeyWrap);
```

### <a id="key-exchange"> Asymmetric KeyExchange (key encryption)

Отправитель:
```csharp
// Создаем случайный секретный ключ, который необходимо передать.
Gost28147 key = new Gost28147CryptoServiceProvider();
// Синхропосылка не входит в GostKeyTransport и должна
// передаваться отдельно.
IV = key.IV;

// Создаем форматтер, шифрующий на ассиметричном ключе получателя.
GostKeyExchangeFormatter Formatter = new GostKeyExchangeFormatter(cert.PublicKey.Key as Gost3410_2012_512);
// GostKeyTransport - формат зашифрованной для безопасной передачи
// ключевой информации.
GostKeyTransport encKey = Formatter.CreateKeyExchange(key);
```

Получатель:
```csharp
// Деформаттер для ключей, зашифрованных на ассиметричном ключе получателя.
GostKeyExchangeDeformatter Deformatter = new GostKeyExchangeDeformatter(cert.PrivateKey);
// Получаем ГОСТ-овый ключ из GostKeyTransport.
Gost28147 decryptedKey = (Gost28147)Deformatter.DecryptKeyExchange(encKey);
// Устанавливаем синхропосылку.
decryptedKey.IV = IV;
```

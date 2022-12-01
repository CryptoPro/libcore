# LibCore

��� ������ ���������� ���������� ��������� ���������� ������ ��������� CSP 5.0.

���������� � ������ �� ������ ���������� corefx � ���������.NET.

� ��������� ������ � ������/�� �����������:

* CMS ���������� (`EnvelopedCms`)
* XML ���������� (`EncryptedXml`)

## ��������� ��������
��� ������������� ����������� ������ `System.Security.Pkcs` ����� �������������� �����������
���������� ������� ������ ������ `CmsSigner` ��� ������������ ������������� ������.

```csharp
var signed = new CmsSigner();
LibCore.Initializer.Initialize();
```

## �������

 - [��������� � ������������� ����������](#init) 
 - [�������� ����������� �� pfx �����](#file-pfx) 
 - [�������� ����������� �� ���������](#store-pfx) 
 - [�������� �����������](#pfx-create)
 - [SignedXml](#signed-xml)
   - [�������](#signed-xml-sign)
   - [��������](#signed-xml-verify)
- [SignedCms](#signed-cms)
  - [Attached](#signed-cms-attached)
  - [Detached](#signed-cms-detached)  
 - [������ � ������������ � ������������](#container)
    - [�������� ���������� �����](#open-container)
    - [�������� ��������� ����������](#create-container)
    - [����������� OID](#set-oid)
 - [�������� ���������, KeyWrap](#key)
  - [Symmetric KeyWrap](#key-wrap)
  - [Asymmetric agree (DH)](#key-agree)
  - [Asymmetric KeyExchange (key encryption)](#key-exchange)

## <a id="init"> ��������� � ������������� ����������
������� nuget ����� `LibCore.Linux.XXXX.XX.XX.nupkg` ��� `LibCore.Windows.XXXX.XX.XX.nupkg`
� ����� �� ���������� ���� `packages_PATH`.

�������� ���� `%appdata%\NuGet\NuGet.Config` (Windows) ��� 
`~/.nuget/NuGet/NuGet.Config` (Linux), ������� � ������ ���� 
`packageSources` �������� `<add key="local packages" value="packages_PATH" />`.

������:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="local packages" value="C:\packages" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
  </packageSources>
</configuration>
```

��� ������������� ����������� ���������� ���������� ������� �������������:
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

� `Initialize(...)` ����� ������� ������ ��������� ����������, ������������� 
������� �������������� � ������ �������.
```csharp
using LibCore;

namespace Sample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // �������������� LibCore ��� ������������� XML �������.
            // ����������� ��� TLS � PKCS �� ����� �����������.
            // ����������� ��� Csp, Algorithms � Primitives ����� �����������
            // �.�. �� ��� ������� ����������� XML.
            LibCore.Initializer.Initialize(LibCore.Initializer.DetouredAssembly.Xml);
            // ready to go!
        }
    }
}
```

## <a id="file-pfx"> �������� ����������� �� ���������
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

## <a id="file-pfx"> �������� ����������� �� pfx �����

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

## <a id="pfx-create"> �������� �����������

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
// �������� ����������� ������� (1.3.6.1.5.5.7.3.2)
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

### <a id="signed-xml-sign"> ������������ �������:

```csharp
static XmlDocument SignXmlFile(
    XmlDocument doc,
    AsymmetricAlgorithm Key,
    X509Certificate Certificate,
    string DigestMethod = CpSignedXml.XmlDsigGost3411_2012_256Url)
{
    // ������� ������ SignedXml �� XML ���������.
    SignedXml signedXml = new SignedXml(doc);

    // ��������� ���� � SignedXml ��������. 
       signedXml.SigningKey = Key;

    // ������� ������ �� node ��� �������.
    // ��� ������� ����� ��������� ����������� "".
    Reference reference = new Reference();
    reference.Uri = "";

    // ���� ����������� �������� �����������,
    // �� ��������� SHA1.
    reference.DigestMethod = DigestMethod;

    // ��������� transform �� ������������� ������
    // ��� �������� ��������� �������.
    XmlDsigEnvelopedSignatureTransform env =
        new XmlDsigEnvelopedSignatureTransform();
    reference.AddTransform(env);

    // ��������� ���� ���������.
    // ������� � .NET 4.5.1 ��� �������� �������, ���������� �������� ���� ��������� � ���������:
    // signedXml.SafeCanonicalizationMethods.Add("urn://smev-gov-ru/xmldsig/transform");
    XmlDsigSmevTransform smev =
        new XmlDsigSmevTransform();
    reference.AddTransform(smev);

    // ��������� transform ��� �����������.
    XmlDsigC14NTransform c14 = new XmlDsigC14NTransform();
    reference.AddTransform(c14);

    // ��������� ������ �� ������������� ������
    signedXml.AddReference(reference);

    // ������� ������ KeyInfo.
    KeyInfo keyInfo = new KeyInfo();

    // ��������� ���������� � KeyInfo
    keyInfo.AddClause(new KeyInfoX509Data(Certificate));

    // ��������� KeyInfo � SignedXml.
    signedXml.KeyInfo = keyInfo;

    // ����� ���� ���������� �������� �������: ���� � 34.10.
    // ���� ���������� ����� ������� ���� � 34.10
    // � �������� ����� ������� �� �����, �� ����� �����������
    // XmlDsigGost3410Url
    // signedXml.SignedInfo.SignatureMethod =
    //     CPSignedXml.XmlDsigGost3410_2012_256Url;

    // ��������� �������.
    signedXml.ComputeSignature();

    // �������� XML ������������� ������� � ��������� ��� 
    // � ��������� node.
    XmlElement xmlDigitalSignature = signedXml.GetXml();

    // ��������� node ������� � XML ��������.
    doc.DocumentElement.AppendChild(doc.ImportNode(
        xmlDigitalSignature, true));

    // ��� ������� ��������� XML ���������� �� �������
    // (�� ��������� ���������� ����������)
    if (doc.FirstChild is XmlDeclaration)
    {
        doc.RemoveChild(doc.FirstChild);
    }

    return doc;
}
```

### <a id="signed-xml-verify"> �������� �������:

```csharp
static bool ValidateXmlFIle(XmlDocument xmlDocument)
{
    // ���� ��� node "Signature" � ��������� �� � ������� XmlNodeList
    XmlNodeList nodeList = xmlDocument.GetElementsByTagName(
        "Signature", SignedXml.XmlDsigNamespaceUrl);

    // ��������� ��� �������.
    bool result = true;
    for (int curSignature = 0; curSignature < nodeList.Count; curSignature++)
    {
        // ������� ������ SignedXml ��� �������� ������� ���������.
        SignedXml signedXml = new SignedXml(xmlDocument);

        // ������� � .NET 4.5.1 ��� �������� �������, ���������� �������� ���� transform � ���������:

        signedXml.SafeCanonicalizationMethods.Add("urn://smev-gov-ru/xmldsig/transform");

        // ��������� ���� � ��������.
        signedXml.LoadXml((XmlElement)nodeList[curSignature]);

        // ��������� ������� � ������� ���������.
        result &= signedXml.CheckSignature();
    }
    return result;
}
```

## <a id="signed-cms"> SignedCms

### <a id="signed-cms-attached"> ������������ � �������� ������������� �������:
```csharp
byte[] signature;
using (var gostCert = GostNonPersistCmsTests.GetGost2012_256Certificate())
{
    var contentInfo = new ContentInfo(bytesToHash);
    var signedCms = new SignedCms(contentInfo, false);
    CmsSigner cmsSigner = new CmsSigner(gostCert);
    signedCms.ComputeSignature(cmsSigner);
    signature = signedCms.Encode();
    Console.WriteLine($"CMS Sign: {Convert.ToBase64String(signature)}");
}

// ������� SignedCms ��� ������������� � ��������.
SignedCms signedCmsVerify = new SignedCms();

// ���������� �������
signedCmsVerify.Decode(signature);

// ��������� �������
signedCmsVerify.CheckSignature(true);
```

### <a id="signed-cms-detached"> ������������ � �������� ������������ �������:
```csharp
byte[] signature;
using (var gostCert = GostNonPersistCmsTests.GetGost2012_256Certificate())
{
    var contentInfo = new ContentInfo(bytesToHash);
    var signedCms = new SignedCms(contentInfo, true);
    CmsSigner cmsSigner = new CmsSigner(gostCert);
    signedCms.ComputeSignature(cmsSigner);
    signature = signedCms.Encode();
    Console.WriteLine($"CMS Sign: {Convert.ToBase64String(signature)}");
}

// ������� ������ ContentInfo �� ���������.
// ��� ���������� ��� �������� ������� SignedCms.
ContentInfo contentInfoVerify = new ContentInfo(bytesToHash);

// ������� SignedCms ��� ������������� � ��������.
SignedCms signedCmsVerify = new SignedCms(contentInfoVerify, true);

// ���������� �������
signedCmsVerify.Decode(signature);

// ��������� �������
signedCmsVerify.CheckSignature(true);
```

## <a id="container"> ������ � ������������ � ������������

### <a id="open-container"> �������� ���������� �����

```csharp
var provider =
    new Gost3410_2012_256CryptoServiceProvider(
        new CspParameters(
            80,
            "",
            "\\\\.\\HDIMAGE\\G2012256"));

using (var gost = new Gost3410_2012_256CryptoServiceProvider(cpsParams))
{

}
```

### <a id="create-container"> �������� ��������� ���������� (������� �����)

```csharp
var provider =
    new Gost3410_2012_256CryptoServiceProvider(
        new CspParameters()
        {
            Flags = CspProviderFlags.NoPrompt,
            KeyContainerName = $"\\\\.\\HDImage\\0000_test_{Guid.NewGuid()}",
            ProviderName = "Crypto-Pro GOST R 34.10-2012 Cryptographic Service Provider",
            ProviderType = 80,
            KeyPassword = new SecureString(),
            KeyNumber = keyNumber
        });
```

### <a id="set-oid"> ����������� OID

https://github.com/CryptoPro/corefx/issues/45

��������� ��� 2817 ������ ����� �������� ����� ����� �����������
```csharp
public Gost28147CryptoServiceProvider(CspParameters parameters)
```

����� CipherOid ��� `Gost28147CryptoServiceProvider`, `Gost3410_XXXXCryptoServiceProvider` ������������ ����� ����������� ��������.

������ CipherOid - https://cpdn.cryptopro.ru/content/csp40/html/group___pro_c_s_p_ex_CP_PARAM_OIDS.html

*****************

���� ������������ �����������. ������ ������ ���� ����������� ���������.NET.

��� ��������� ��������� OID ������������� ����� ���������� ��������� ��� ����� �������� CipherOid � ������� **��������� �����**

������ ��������� OID ��� ������������� �����:
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

ps. ��� ������������� ������ ExportParameters(false) ��� **��������� �����** ����� �� ���������� ���� EncryptionParamSet ������������� ������� Gost3410Parameters � ����� ����� �������������� Oid �� ���������, ��� ����������� �� �������������� OID � ��������� �����. ����� ��������� ������, �������� � Gost3410Parameters.EncryptionParamSet ������ ��������:
```csharp
var paramsPk = gostRes.ExportParameters(false);
paramsPk.EncryptionParamSet = gostResPk.CipherOid;            
var agree = (GostSharedSecretCryptoServiceProvider)gost.CreateAgree(paramsPk);
```

## <a id="key"> �������� ���������, KeyWrap

�� ������� � https://github.com/CryptoPro/corefx/issues/43

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

�����������:
```csharp
// ������� ��������� ��������� ����, ������� ���������� ��������.
Gost28147 key = new Gost28147CryptoServiceProvider();
// ������������� �� ������ � GostKeyTransport � ������
// ������������ ��������.
IV = key.IV;

// ������� ���������, ��������� �� ������������� ����� ����������.
GostKeyExchangeFormatter Formatter = new GostKeyExchangeFormatter(cert.PublicKey.Key as Gost3410_2012_512);
// GostKeyTransport - ������ ������������� ��� ���������� ��������
// �������� ����������.
GostKeyTransport encKey = Formatter.CreateKeyExchange(key);
```

����������:
```csharp
// ����������� ��� ������, ������������� �� ������������� ����� ����������.
GostKeyExchangeDeformatter Deformatter = new GostKeyExchangeDeformatter(cert.PrivateKey);
// �������� ����-���� ���� �� GostKeyTransport.
Gost28147 decryptedKey = (Gost28147)Deformatter.DecryptKeyExchange(encKey);
// ������������� �������������.
decryptedKey.IV = IV;
```

# Topic 16 — JSON & XML

> JSON is the primary data exchange format for REST APIs. XML is used in SOAP, config files, and legacy integrations. Both are tested in .NET senior roles.

---

## JSON (JavaScript Object Notation)

### Structure & Data Types
```json
{
    "id": "a1b2c3d4",
    "amount": 5000.50,
    "isActive": true,
    "notes": null,
    "tags": ["income", "salary", "monthly"],
    "account": {
        "id": "acc-001",
        "name": "BDO Savings",
        "balance": 125000.00
    },
    "transactions": [
        { "date": "2026-07-01", "amount": 50000 },
        { "date": "2026-07-05", "amount": -2000 }
    ]
}
```

**JSON data types:** string, number, boolean, null, object `{}`, array `[]`  
**Not valid JSON:** `undefined`, functions, `Date` objects (use ISO string), `NaN`, `Infinity`

---

## JSON in .NET (System.Text.Json)

```csharp
// Serialization:
var transaction = new TransactionDto
{
    Id = Guid.NewGuid(),
    Amount = 5000m,
    Date = DateTime.UtcNow,
    Description = "Salary"
};

string json = JsonSerializer.Serialize(transaction);

// With options (PascalCase → camelCase):
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
string json = JsonSerializer.Serialize(transaction, options);

// Deserialization:
var dto = JsonSerializer.Deserialize<TransactionDto>(json, options);

// Configure globally in ASP.NET Core:
builder.Services.ConfigureHttpJsonOptions(opt =>
{
    opt.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    opt.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});

// Attributes:
public class TransactionDto
{
    [JsonPropertyName("txId")]          // custom JSON name
    public Guid Id { get; set; }
    
    [JsonIgnore]                         // never serialize
    public string InternalCode { get; set; }
    
    [JsonInclude]                        // include non-public properties
    public string ComputedField { get; private set; }
    
    [JsonConverter(typeof(DecimalConverter))]
    public decimal Amount { get; set; }
}
```

---

## JSON Schema & Validation

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["amount", "type", "accountId"],
    "properties": {
        "amount": {
            "type": "number",
            "minimum": 0.01,
            "maximum": 1000000
        },
        "type": {
            "type": "string",
            "enum": ["income", "expense", "transfer"]
        },
        "description": {
            "type": "string",
            "maxLength": 500
        },
        "tags": {
            "type": "array",
            "items": { "type": "string" },
            "maxItems": 10
        }
    }
}
```

---

## JSON in JavaScript

```javascript
// Parse (string → object):
const data = JSON.parse('{"name":"Randolf","age":30}');
data.name; // "Randolf"

// Stringify (object → string):
const json = JSON.stringify({ name: "Randolf", age: 30 });
// '{"name":"Randolf","age":30}'

// Pretty print:
JSON.stringify(data, null, 2);

// With replacer (filter/transform properties):
JSON.stringify(data, ["name", "email"]); // only include name and email

// Reviver — transform values during parse:
JSON.parse(json, (key, value) => {
    if (key === "date") return new Date(value); // string → Date
    return value;
});

// Deep clone (simple, but loses undefined/functions/dates):
const clone = JSON.parse(JSON.stringify(originalObject));
// Better: structuredClone(originalObject) — native, handles Date/Map/Set
```

---

## XML (eXtensible Markup Language)

### Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Transactions xmlns="http://budgetph.com/transactions"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    
    <!-- This is a comment -->
    <Transaction id="tx-001" type="income">
        <Amount currency="PHP">50000.00</Amount>
        <Description><![CDATA[Salary & Benefits — June 2026]]></Description>
        <Date>2026-07-01</Date>
        <Account>
            <Id>acc-001</Id>
            <Name>BDO Savings</Name>
        </Account>
    </Transaction>
    
    <Transaction id="tx-002" type="expense">
        <Amount currency="PHP">2500.00</Amount>
        <Description>Groceries</Description>
        <Date>2026-07-05</Date>
    </Transaction>
    
</Transactions>
```

### XML vs JSON Comparison
| Feature | JSON | XML |
|---|---|---|
| Verbosity | Compact | Verbose |
| Data types | Built-in (number, bool, null) | Everything is string |
| Arrays | Native `[]` | Repeated elements |
| Attributes | N/A | Supported |
| Comments | ❌ Not supported | ✅ Supported |
| Schema validation | JSON Schema | XSD (XML Schema Definition) |
| Use cases | REST APIs, modern apps | SOAP, config, Office formats, legacy |
| Human readable | ✅ Easy | ⚠️ Harder with namespaces |

---

## XML in .NET

```csharp
// XDocument (LINQ to XML — recommended):
var doc = XDocument.Load("transactions.xml");

var transactions = doc
    .Descendants("Transaction")
    .Select(t => new
    {
        Id = t.Attribute("id")?.Value,
        Type = t.Attribute("type")?.Value,
        Amount = decimal.Parse(t.Element("Amount")?.Value ?? "0"),
        Description = t.Element("Description")?.Value
    })
    .ToList();

// Create XML:
var xmlDoc = new XDocument(
    new XDeclaration("1.0", "UTF-8", null),
    new XElement("Transactions",
        new XElement("Transaction",
            new XAttribute("id", "tx-001"),
            new XElement("Amount", "50000.00"),
            new XElement("Description", "Salary")
        )
    )
);
xmlDoc.Save("output.xml");

// XmlSerializer — serialize C# objects:
[XmlRoot("Transaction")]
public class TransactionXml
{
    [XmlAttribute("id")]
    public string Id { get; set; }
    
    [XmlElement("Amount")]
    public decimal Amount { get; set; }
    
    [XmlIgnore]
    public string InternalCode { get; set; }
}

var serializer = new XmlSerializer(typeof(TransactionXml));
using var writer = new StringWriter();
serializer.Serialize(writer, transaction);
string xml = writer.ToString();

// Deserialize:
using var reader = new StringReader(xml);
var tx = (TransactionXml)serializer.Deserialize(reader);

// XPath — query XML:
var navigator = doc.CreateNavigator();
var nodes = navigator.Select("//Transaction[@type='income']/Amount");
```

---

## SOAP Web Services (XML-based)

```xml
<!-- SOAP Request: -->
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Header>
        <auth:Token xmlns:auth="http://auth.example.com">abc123</auth:Token>
    </soap:Header>
    <soap:Body>
        <GetTransactions xmlns="http://banking.example.com">
            <AccountId>acc-001</AccountId>
            <StartDate>2026-07-01</StartDate>
            <EndDate>2026-07-31</EndDate>
        </GetTransactions>
    </soap:Body>
</soap:Envelope>
```

```csharp
// Call legacy SOAP service from .NET:
// Add Connected Service → WCF/SOAP → generates typed client
// Or use HttpClient directly:
var soapBody = $"""
    <?xml version="1.0"?>
    <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
        <soap:Body>
            <GetBalance>
                <AccountId>{accountId}</AccountId>
            </GetBalance>
        </soap:Body>
    </soap:Envelope>
    """;

using var request = new HttpRequestMessage(HttpMethod.Post, soapEndpoint);
request.Content = new StringContent(soapBody, Encoding.UTF8, "text/xml");
request.Headers.Add("SOAPAction", "GetBalance");
var response = await httpClient.SendAsync(request);
var responseXml = await response.Content.ReadAsStringAsync();
var responseDoc = XDocument.Parse(responseXml);
```

---

## JSON vs XML in API Design

```csharp
// Content negotiation — ASP.NET Core can return both:
builder.Services.AddControllers()
    .AddJsonOptions(opt => 
        opt.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase)
    .AddXmlSerializerFormatters(); // adds XML support

// Client sends: Accept: application/xml → gets XML
//               Accept: application/json → gets JSON
```

---

## Interview Q&A

**Q1: What are the limitations of JSON?**
> JSON doesn't support: `Date` objects (use ISO 8601 strings), comments, `undefined`, functions, `NaN`, `Infinity`, circular references (throws on `JSON.stringify`). It also has no schema enforcement at the format level (need JSON Schema for that).

**Q2: When would you use XML over JSON?**
> Legacy SOAP/WCF integrations, Microsoft Office file formats (.docx, .xlsx are zip+XML), configuration files (`.csproj`, `web.config`), document-centric data where mixed content matters (text with embedded elements), or when namespaces and document validation via XSD are required.

**Q3: What is the difference between `XDocument` and `XmlDocument` in .NET?**
> `XmlDocument` is the older DOM API (W3C standard). `XDocument` is the newer LINQ to XML API — cleaner, supports LINQ queries, functional construction, and is generally preferred. Use `XDocument` for new code.

**Q4: What is CDATA in XML?**
> `<![CDATA[...]]>` wraps content that contains characters that would otherwise need escaping in XML (like `<`, `>`, `&`). The parser treats everything inside as literal text, not markup. Useful for embedding HTML, code snippets, or free-form text in XML.

**Q5: How do you handle null values in JSON serialization in .NET?**
> Use `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull` to omit null properties, or `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` per property. For required-but-nullable fields, use `JsonIgnoreCondition.Never`.

# Microsoft.Azure.Documents.OData.Sql

Converts [OData V4](http://docs.oasis-open.org/odata/odata/v4.0/odata-v4.0-part1-protocol.html) queries to [DocumentDB SQL](https://azure.microsoft.com/en-us/documentation/articles/documentdb-sql-query/) queries. 

## Summary

This package supports most of the intersectional functionalities provided by OData V4 and DocumentDB SQL. For example, if you have a class looks like:
```
public class Company {
  public string englishName,
  public string countryCode,
  public int    revenue
}
```
To query all companies whose englishName contains "Limited", and sort them by countryCode in descending order, then select the revenue property from top 5 results, you can issue an OData query to your web service:
```
http://localhost/Company?$select=revenue&$filter=contains(englishName, 'Limited')&$orderby=countryCode desc&$top=5
```
The above query will then be translated to DocumentDB SQL:
```
SELECT TOP 5 c.revenue FROM c WHERE CONTAINS(c.englishName,'Limited') ORDER BY c.countryCode DESC 
```

### Supported OData to DocumentDB SQL mappings:

#### System Query Options::
[$select](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part1-protocol/odata-v4.0-errata03-os-part1-protocol-complete.html#_System_Query_Option_3) => SELECT

[$filter](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part1-protocol/odata-v4.0-errata03-os-part1-protocol-complete.html#_The_$filter_System) => WHERE

[$top](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part1-protocol/odata-v4.0-errata03-os-part1-protocol-complete.html#_The_$top_System_1) => TOP

[$orderby](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part1-protocol/odata-v4.0-errata03-os-part1-protocol-complete.html#_The_$select_System_1) => ORDER BY

#### Built-in Query Functions
contains()(field, 'value')	 => CONTAINS(c.field, 'value')

startswith()(field, 'value') => STARTSWITH(c.field, 'value')

endswith()(field, 'value')	 => ENDSWITH(c.field, 'value')

toupper()(field, 'value')    => UPPER(c.field, 'value')

tolower()(field, 'value')    => LOWER(c.field, 'value')

length()(field)              => LENGTH(c.field)

indexof(field,'value')       => INDEX_OF(c.field,'value')
          
substring(field,idx1,idx2)   => SUBSTRING(c.field,idx1,idx2)
 
trim(field)                  => LTRIM(RTRIM(c.englishName))

concat(field,'value')        => CONCAT(c.englishName,'value')

array_field/any(f:f eq 'value')   => ARRAY_CONTAINS(c.array_field,'value')

collection_field/any(f:f.property op 'value') => JOIN f in c.collection_field WHERE f.property op 'value'

## Installing

The nuget package of this project is published on Nuget.org [Download Page](https://www.nuget.org/packages/Lambda.Azure.CosmosDb.OData.Sql/). To install in Visual Studio Package Manager Console, run command: 
```
PM> Install-Package Lambda.Azure.CosmosDb.OData.Sql
```

## Usage

After installation, you can include the binary in your \*.cs file by
```
using Microsoft.Azure.Documents.OData.Sql;
```
-You can find a complete set of examles in [ODataToSqlSamples.cs](https://github.com/aboo/azure-documentdb-odata-sql/blob/master/azure-documentdb-odata-sql-samples/ODataToSqlSamples.cs)-

There are only two classes you need to work with in order to get the translation done: [SQLQueryFormatter](https://github.com/aboo/azure-documentdb-odata-sql/blob/master/azure-documentdb-odata-sql/ODataToSqlTranslator/SqlQueryFormatter.cs) and [ODataToSqlTranslator]https://github.com/aboo/azure-documentdb-odata-sql/blob/master/azure-documentdb-odata-sql/ODataToSqlTranslator/ODataToSqlTranslator.cs). 
#### SQLQueryFormatter
SQLQueryFormatter is where we do the property and function name translation, such as translating 'propertyName' to 'c.propertyName', or 'contains()' to 'CONTAINS()'. This class inherits from [QueryFormatterBase](https://github.com/aboo/azure-documentdb-odata-sql/blob/master/azure-documentdb-odata-sql/ODataToSqlTranslator/QueryFormatterBase.cs), which abstractly defines all required translations. One can derive from QueryFormatterBase to implement their own translation per need.

#### ODataToSqlTranslator
ODataToSqlTranslator does the overall translation, by taking in an implementation of QueryFormatterBase. Because the specific translation is defined in QueryFormatter, the translation performs differently according to the implementation in QueryFormatter. 
The default QueryFormatter is SQLQueryFormatter mentioned above:
```
var oDataToSqlTranslator = new ODataToSqlTranslator(new SQLQueryFormatter());
```
Once we have an instance of ODataToSqlTranslator, we can call .Translate(ODataQueryOptions, TranslateOptions, string) method:
```
var translatedSQL = oDataToSqlTranslator.(oDataQueryOptions, TranslateOptions.ALL, additionalWhereClause: null);
```
In the above example, the [ODataQueryOptions](https://msdn.microsoft.com/en-us/library/system.web.http.odata.query.odataqueryoptions(v=vs.118).aspx) is a fundamental class provided by ASP.NET [Web API 2](https://www.asp.net/web-api/overview/odata-support-in-aspnet-web-api/supporting-odata-query-options), there you can find detailed instruction on ODataQueryOptions. Additionally, you can also find ODataQueryOptions usage in the ODataToSqlSamples file. 
The second parameter is TranslateOptions, which defines what queries to be translated. Define TranslateOptions are:
```
SELECT_CLAUSE, JOIN_CLAUSE, WHERE_CLAUSE, ORDERBY_CLAUSE, TOP_CLAUSE, ALL
```
The options can be combined with bit operators such as ```(TranslateOptions.SELECT_CLAUSE | TranslateOptions.WHERE_CLAUSE)```. One common usage is ```(TranslateOptions.ALL & ~TranslateOptions.TOP)```, this combination enables all translation but TOP. The reason to disable TOP is that when performing pagination, DocumentDB ignores [continuation token in FeedOptions](https://msdn.microsoft.com/en-us/library/microsoft.azure.documents.client.feedoptions.requestcontinuation.aspx) if TOP exists. Therefore, the best practice is to use ```FeedOptions``` to perform TOP operation in DocumentDB.

## Release Notes

* 2.0.19 Added support for SELECT VALUE c
* 2.0.18 Added support for JOIN for entity with Id
* 2.0.16 Added support for JOIN
* 2.0.4 Added support for ARRAY_CONTAINS
* 2.0.3 Added support for DateTimeOffset
* 2.0.2 Added support for functions: length(), indexof(), substring(), trim(), concat()
* 2.0.1 Added support for functions: contains(), startswith(), endswith(), toupper() and tolower()
* 2.0.0 Breaking changes: Simplified usage with newly introuduced class ODataToSqlTranslator
* 1.0.0 Initial release
## Authors

* **Aboo Azarnoush** - Lambda Solutions
* **Ziyou Zheng** - Microsoft Universal Store Team -

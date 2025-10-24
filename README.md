# fch-cube
A set of .NET components creating cube.link conform datasets.

This repository gives an overview on the different components to publish cube.link conform datasets.

* [FCh.Cube.Dimension](https://github.com/swiss/fch-cube-dimension)
* [FCh.Cube.Rawdata](https://github.com/swiss/fch-cube-rawdata)

Both libraries are based on https://github.com/dotnetrdf/dotnetrdf

Register your namespaces on the graph.
```csharp
var graph = new Graph();
graph.NamespaceMap.AddNamespace("ex", new Uri("https://example.com/"));
```

# Libraries

## FCh.Cube.Dimension
This library supports the creation of "dimensions" (see: https://cube.link/#dimensions-0).

### Usage
#### Install NuGet Package
`dotnet add package FCh.Cube.Dimension`

#### Dependency Injection
Register the library's services using the extension method on `IServiceCollection`.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDimesionService();
```

Then inject `Swiss.FCh.Cube.Dimension.Contract.IDimensionService` into your class.

#### Create a Dimension
First, create your dimension entries using `Swiss.FCh.Cube.Dimension.Model.DimensionItem`.
Provide a key, a name and (if needed) additional properties.

```csharp
new DimensionItem(
	person.Id,
	new LingualLiteral($"{person.Surname} {person.GivenName}", "en"),
	[
		new AdditionalLingualProperty("schema:givenName", new LingualLiteral(person.GivenName)),
		new AdditionalLingualProperty("schema:familyName", new LingualLiteral(person.Surname))
	]);
```

Finally, use the `IDimensionService` to create the triples defining your dimension.

```csharp
var personTriples =
	_dimensionService.CreateDimension(
		personDimensionItems,
		graph,
		"ex:person",
		[new LingualLiteral("Name your dimension here", "en")],
		rdfTypes: ["http://schema.org/Person"]);
```

*Note:* `FCh.Cube.Dimension` will not push the triples triples to an RDF store. This has to be done using `dotnetRdf`.

## FCh.Cube.RawData
This library supports the creation of observations (https://cube.link/#Observation) building up your cube.
These observations can link to previously created dimensions values.

### Usage

#### Install NuGet Package
`dotnet add package FCh.Cube.RawData`

#### Dependency Injection
Register the library's services using the extension method on `IServiceCollection`.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddCubeRawData();
```

Then inject `Swiss.FCh.Cube.RawData.Contract.ICubeRawDataService` into your class.

#### Create Raw Data
A new data row can be created using `Swiss.FCh.Cube.RawData.Model.ObservationDataRow`.

```csharp
var dataRow = new ObservationDataRow
{
	KeyUri = "ex:myObservation/459",
	ValidFrom = new DateTime(2000, 1, 1),
	ValidTo = new DateTime(2025, 12, 31)
};
```

Links to dimension value can be added like this.

```csharp
dataRow.KeyDimensionLinks.Add(new KeyDimensionLink { PredicateUri = "ex:hasSomeDimension", Uri = "ex:dimension/2354" });
```

Additionally, raw values can be added as well.

```csharp
dataRow.Values.Add(new DimensionValue { Predicate = "schema:description", Value = "some text", LanguageTag = "en" });
```

Finally, the data can be transformed to triples using the ```ICubeRawDataService```.

```csharp
var rawDataTriples =
	_cubeRawDataService.CreateTriples(
		graph,
		"ex:myCube",
		"ex:myCube/observationSet",
		new List<ObservationDataRow> {});
```

*Note:* `FCh.Cube.RawData` will not push the triples triples to an RDF store. This has to be done using `dotnetRdf`.
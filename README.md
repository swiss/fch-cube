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

*Note:* `FCh.Cube.Dimension` will not push the triples triples to an RDF store. This has to be done using `dotnetRdf`. See https://dotnetrdf.org/docs/3.4.x/user_guide/writing_rdf.html for more information about `dotnetRdf`.

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

*Note:* `FCh.Cube.RawData` will not push the triples triples to an RDF store. This has to be done using `dotnetRdf`. See https://dotnetrdf.org/docs/3.4.x/user_guide/writing_rdf.html for more information about `dotnetRdf`.

# Visualization

## Using visualize.admin.ch
The data in "cube-form" can be visualized using [visualize.admin.ch](https://visualize.admin.ch).
Visualize can read the cube data and offers editors to create all sorts of diagrams, such as tables, bar charts, scatter plots, pie charts, etc.

## How to use visualize.admin.ch
### Start
Go to [visualize.admin.ch](https://visualize.admin.ch).
Users that are logged-in can save their visualizations for further editing later on. 

Start the process by selecting "Start a visualization".

<img src="./etc/documentation/images/01_visualize_create_visualization.png" alt="Start a Visualization" width="300">


### Select Data Set
Search for your data set and select it.

<img src="./etc/documentation/images/02_visualize_selecct_cube.png" alt="Select Cube" width="400" />

### Verify Data
Verify the selected data and confirm the selection.

<img src="./etc/documentation/images/03_visualize_start_visualization.png" alt="Confirm Selection" width="750" />

### Define Visualization
In this step, the actual visualization can be customized and diagram types can be chosen.

<img src="./etc/documentation/images/04_visualize_options.png" alt="Configure Visualization" width="750" />

### Share or Embed the Visualization
The visualization can be shared or embedded using iFrames.

<img src="./etc/documentation/images/05_visualize_embed_options.png" alt="Embed Visualization" width="250" />

## Using HTML / javascript
Displaying data on a website is another common usage of cube data.
Aside of [visualize.admin.ch](https://visualize.admin.ch), this can also be achieved by simply using HTML and javascript.

### Define a Sparql Query
Cube data can be queried using Sparql queries (same as with any RDF data).
[LINDAS](https://lindas.admin.ch/sparql/) is a suitable tool to try out Sparql queries against an endpoint of your choice.

Since the cube data is always structured in the same way, the queries to read such data, will also be following a similar pattern. 

```sparksql
PREFIX cube: <https://cube.link/>
PREFIX schema: <http://schema.org/>

SELECT * WHERE {
    <https://politics.ld.admin.ch/national-council-election/candidates/2023> cube:observationSet ?obsSet .
    ?obsSet cube:observation ?obs .

    ?obs <https://politics.ld.admin.ch/national-council-election/candidates/hasCanton> ?canton ;
         <https://politics.ld.admin.ch/national-council-election/candidates/hasCandidate> ?candidateUri ;
         <https://politics.ld.admin.ch/national-council-election/candidates/elected> ?elected ;
         <https://politics.ld.admin.ch/national-council-election/candidates/elected> "true"^^<http://www.w3.org/2001/XMLSchema#boolean> .

    OPTIONAL { ?candidateUri schema:birthDate    ?birth . }
}
```
*Example Sparql Query*

This demonstrates the basic structure of such a query.
* specify cube base URI
* select all the cube's observations
* select all the required properties said observations

A more generic template of such a query can thus look like this.

```sparksql
PREFIX cube: <https://cube.link/>
PREFIX schema: <http://schema.org/>

SELECT * WHERE {
    <YOUR_CUBE_BASE_URI> cube:observationSet ?obsSet .
    ?obsSet cube:observation ?obs .

    ?obs <DESIRED_PREDICATE_URI_1> ?predicate1 ;
         <DESIRED_PREDICATE_URI_2> ?predicate2  .
         [... more if needed...]
}
```
*Generic Sparql Query Pattern for Cubes*

### Use Javascript to Execute Query
Since Sparql queries are executed as HTTP requests, javascript is sufficient to do so.
The following code snippet shows, how an HTTP request with a Sparql query against an endpoint can be sent and how the data can be fetched.

```javascript
const query = '[your sparql query]';
const endpoint = '[your sparql endpoint]';

//HTTP request to Sparql endpoint
const response = await fetch(endpoint, {
    method: "POST",
    headers: {
        "Content-Type": "application/sparql-query",
        Accept: "application/sparql-results+json",
    },
    body: query,
});

const json = await response.json();
const rows = json.results.bindings;

//Fetching the results
const result = [];

for (let i = 0; i < rows.length; i++) {

    result.push({
        property1: rows[i].predicate1?.value,
        property2: rows[i].predicate2?.value
    });
}
```

_Simple javascript snippet to execute a Sparql query_

# Livingdocs Integration

A static HTML file (e.g. a visualization using the [HTML / javascript](#using-html--javascript) approach) can be embedded into a Livingdocs (CMS) page.
If the static HTML needs dynamic parameters from the CMS page's URL (e.g. `?committee=...`), these are passed into the iFrame via `postMessage`.

## Setup in Livingdocs

1. Add an **iFrame** element to the CMS page.
2. Set the **Sizing Mode** option to **Dynamic resizing**.
3. In the **URL Settings**, set the URL to the static HTML file.
4. Add a **FreeHtml** element to the CMS page (below the iFrame).
5. Add the following JavaScript snippet to the FreeHtml element.

```html
<script>
  let initInterval = null;
  let acknowledged = false;

  const iframeSrc =
    "https://www.assets-a.bk.admin.ch/livingdocs/apg/committee-statistic.html"; // adjust URL
  const language = "de"; // adjust language

  window.addEventListener("message", (event) => {
    if (event.data?.type === "ogd-init-success") {
      acknowledged = true;
      clearInterval(initInterval);
      initInterval = null;
    }
  });

  const params = new URLSearchParams(window.parent.location.search);
  let committee = params.get("committee"); // adjust parameter and session key
  if (committee) {
    sessionStorage.setItem("apg-committee", committee);
  } else {
    committee = sessionStorage.getItem("apg-committee");

    if (committee) {
      const url = new URL(parent.location.href);
      url.searchParams.set("committee", committee);
      parent.history.replaceState(parent.history.state, "", url);
    }
  }

  function appendParamToIframe() {
    const iframe = window.parent.document.querySelector(
      `iframe[src*="${iframeSrc}"]`,
    );
    if (!iframe?.contentWindow) {
      setTimeout(appendParamToIframe, 300);
      return;
    }

    if (initInterval) return;

    initInterval = setInterval(() => {
      if (!acknowledged) {
        iframe.contentWindow.postMessage(
          { type: "ogd-init", committee, language },
          "*",
        ); // adjust parameters
      }
    }, 500);
  }

  appendParamToIframe();
</script>
```

_FreeHtml snippet passing query parameters to the iFrame_

### How it works

- The FreeHtml element reads the parameter (here `committee`) from the parent page's query string.
- The value is stored in the `sessionStorage`. If the parameter is missing (e.g. after navigating back to the page), the stored value is used and written back to the parent's URL.
- The snippet looks up the iFrame by its `src` (retrying every 300 ms until it is rendered).
- It then sends an `ogd-init` message with the parameters to the iFrame every 500 ms, until the iFrame answers with `ogd-init-success`.

### Adjustments per page

- `iframeSrc`: must match the URL set in the iFrame's URL Settings.
- `language`: language of the CMS page.
- Parameter name (`committee`) and session key (`apg-committee`): adjust to the parameter(s) needed by the static HTML. Use a unique session key per use case.

## Receiving the Parameters in the Static HTML

The static HTML file has to listen for the `ogd-init` message and confirm it with `ogd-init-success`. Otherwise, the FreeHtml snippet keeps sending the message.

```javascript
setupIframeResizeMessaging(); // see "iFrame Resizing" below

window.addEventListener("message", async (event) => {
  if (event.data?.type !== "ogd-init") return;

  const { committee, language } = event.data;

  // confirm reception to stop the init interval
  event.source?.postMessage({ type: "ogd-init-success" }, "*");

  // load / render data using the received parameters
  await render(committee, language);

  // report new size after rendering
  postIframeSize();
});
```

_Example listener in the static HTML file_

## iFrame Resizing

With **Dynamic resizing**, the Livingdocs iFrame adjusts its size to the content. For this, the static HTML has to report its size to the parent page with an `iframe-resized` message (`height` and `width` in pixels).

Add the following snippet to the static HTML file.

```javascript
let lastPostedIframeHeight = null;
let lastPostedIframeWidth = null;

function postIframeSize(additionalHeight = 0) {
  const root = document.documentElement;
  const body = document.body;

  const height =
    Math.max(root.scrollHeight, root.offsetHeight, body ? body.scrollHeight : 0, body ? body.offsetHeight : 0) +
    additionalHeight;

  const width = Math.max(root.scrollWidth, root.offsetWidth, body ? body.scrollWidth : 0, body ? body.offsetWidth : 0);

  if (lastPostedIframeHeight === height && lastPostedIframeWidth === width) {
    return;
  }

  lastPostedIframeHeight = height;
  lastPostedIframeWidth = width;

  window.parent.postMessage(
    {
      type: "iframe-resized",
      height,
      width,
    },
    "*",
  );
}

function setupIframeResizeMessaging() {
  let requestAnimationFrameId = null;
  const schedulePost = () => {
    if (requestAnimationFrameId !== null) {
      return;
    }
    requestAnimationFrameId = window.requestAnimationFrame(() => {
      requestAnimationFrameId = null;
      postIframeSize();
    });
  };

  const resizeObserver = new ResizeObserver(() => {
    schedulePost();
  });

  resizeObserver.observe(document.documentElement);
  if (document.body) {
    resizeObserver.observe(document.body);
  }

  window.addEventListener("resize", schedulePost);
  window.addEventListener("load", schedulePost);

  if (document.fonts?.ready) {
    document.fonts.ready.then(schedulePost).catch(() => {});
  }

  schedulePost();
}
```

_iFrame resize snippet for the static HTML file_

### How it works

- `postIframeSize` measures the document's height and width (max of `scrollHeight`/`offsetHeight` resp. `scrollWidth`/`offsetWidth` of `<html>` and `<body>`) and sends them to the parent page.
  - The message is only sent if the size changed since the last message.
  - `additionalHeight` (optional) adds extra pixels to the height. Use it if the iFrame content gets cut off in the CMS page (e.g. `postIframeSize(20)`).
- `setupIframeResizeMessaging` sends the size automatically whenever the layout may change:
  - `ResizeObserver` on `<html>` and `<body>`
  - window `resize` and `load` events
  - web fonts loaded (`document.fonts.ready`)
  - Multiple triggers within the same frame are combined into one message (`requestAnimationFrame`).

### Usage

- Call `setupIframeResizeMessaging()` once on startup, **before** adding the `ogd-init` message listener (see [example above](#receiving-the-parameters-in-the-static-html)).
- Call `postIframeSize()` whenever data has been rendered, so that the new size is reported right away.

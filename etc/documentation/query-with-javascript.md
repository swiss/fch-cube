# Using HTML / javascript
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

The Neo4j template for Google Dataflow allows to import data into a Neo4j database through a Dataflow job, sourcing data from either a Google BigQuery dataset or CSV files hosted in Google Cloud Storage buckets.
It also allows to manipulate and transform the data at various steps of the import.
You can use the template for both first-time and incremental imports.

This tutorial guides you through importing an example BigQuery dataset into a Neo4j database using a Dataflow job.
It also provides a public dataset you can experiment with, before you go on and create an import job for you own dataset.


# Things you will need

Here is the high-level list of the things you will need throughout the tutorial.
The next few sections cover each item in more detail.

- A running Neo4j instance
- A Google account and a Google Cloud project
- A [Google Cloud Storage](https://console.cloud.google.com/storage/) bucket
- A dataset to import
- A [Google Dataflow](https://console.cloud.google.com/dataflow/) project

[!NOTE]
All Google-related resources should either belong to the same account, or to one which the Dataflow job has permissions to access.

## Neo4j instance

You need a running Neo4j instance which the data can flow into.

If you don't have an instance yet, you have two options:

- sign-up for a free [AuraDB](https://neo4j.com/cloud/aura-free/) instance
- install and self-host Neo4j in a location that is publicly accessible (see [Neo4j -> Installation](https://neo4j.com/docs/operations-manual/current/installation/)) with port 7687 open (Bolt protocol)

Either way, you then need to create a file containing your database connection information in JSON format (basic authentication is the only supported method).
This file will go into your Cloud Storage bucket.

The content of an example `neo4j-connection-info.json` file is the following:

```json
{
  "server_url": "neo4j+s://xxxx.databases.neo4j.io",
  "database": "neo4j",
  "username": "neo4j",
  "pwd": "verysecret"
}
```


## Google Cloud Storage bucket

You need a [Google Cloud Storage](https://console.cloud.google.com/storage/) bucket.
This is the one and only location from where the Dataflow job can source files (both configuration files and source CSVs, if any).

Go ahead and upload `neo4j-connection-info.json` to your Cloud Storage bucket.


## Dataset to import

You need a dataset that you want to import into Neo4j.
This can be either a [Google BigQuery](https://console.cloud.google.com/bigquery) dataset or a number of CSV files located in your Google Cloud Storage bucket.

This tutorial uses a subset of the `movies` dataset.
It contains entities `Person` and `Movie`, linked together by `DIRECTED` and `ACTED_IN` relationships.
In other words, each `Person` may have `DIRECTED` and/or `ACTED_IN` a `Movie`.
Both entities and relationships have extra details attached to each of them.
The data is sourced from the following files: [persons.csv](examples/persons.csv), [movies.csv](examples/movies.csv), [acted_in](examples/acted_in.csv), [directed](examples/directed.csv[directed.csv).

![Model for movies dataset](movies-model.png)

[!NOTE]
Since you are moving data from a relational database into a graph database, **the data model will have to change**.
Checkout [Graph data modeling guidelines](https://neo4j.com/docs/getting-started/data-modeling/guide-data-modeling/) to learn how to model for graph databases.

[!NOTE]
For importing CSV files into Neo4j, it can be easier and more efficient to use the Cypher clause [`LOAD CSV`](https://neo4j.com/docs/cypher-manual/current/clauses/load-csv/), or to parse the CSV in your favorite language and use one of [Neo4j's client libraries (drivers)](https://neo4j.com/docs/create-applications/) to insert the data into the database.


## Google Dataflow job

The [Google Dataflow](https://console.cloud.google.com/dataflow) job glues all the pieces together and performs the data import.
All the work that is now needed is to craft a _job specification_ file to provide Dataflow with all the information it needs to load the data into Neo4j.

![Google Dataflow worflow with Neo4j](google-dataflow.jpg)


# Create a job specification file

The job configuration file consists of a JSON object with four sections:

- `config` -- global flags affecting how the import is performed
- `sources` -- data source definitions (relational)
- `targets` -- data target definitions (graph: nodes/relationships)
- `actions` -- pre/post-load actions

```json
{
  "config": {},
  "sources": [
    { ... }
  ],
  "targets": [
    { ... }
  ],
  "actions": [
    { ... }
  ]
}
```

At a high level, the job will fetch data from `sources` and transform/import them into the `targets`.

Here below you can find an example job specification file that works out of the box to import the publicly-available  `movies` dataset.
In the next sections, we break it down and provide in-context information for each part. We recommend reading this guide side by side with the job specification example.

```json
{
  "config": {
    "reset_db": true,
    "index_all_properties": false
  },
  "sources": [
    {
      "type": "bigquery",
      "name": "movies",
      "query": "SELECT movieId, title FROM team-connectors-dev.movies.movies"
    },
    {
      "type": "bigquery",
      "name": "persons",
      "query": "SELECT person_tmdbId, name FROM team-connectors-dev.movies.persons"
    },
    {
      "type": "bigquery",
      "name": "directed",
      "query": "SELECT movieId, person_tmdbId FROM team-connectors-dev.movies.directed"
    },
    {
      "type": "bigquery",
      "name": "acted_in",
      "query": "SELECT movieId, person_tmdbId, role FROM team-connectors-dev.movies.acted_in"
    }
  ],
  "targets": [
    {
      "node": {
        "source": "movies",
        "name": "Movies",
        "mode": "merge",
        "transform": {
          "group": true
        },
        "mappings": {
          "labels": [
            "\"Movie\""
          ],
          "keys": [
            {"movieId": "movie_id"}
          ],
          "properties": {
            "unique": [],
            "indexed": [
              {"title": "title"}
            ],
            "strings": []
          }
        }
      }
    },
    {
      "node": {
        "source": "persons",
        "name": "Person",
        "mode": "merge",
        "transform": {
          "group": true
        },
        "mappings": {
          "labels": [
            "\"Person\""
          ],
          "keys": [
            {"person_tmdbId": "person_id"}
          ],
          "properties": {
            "unique": [],
            "indexed": [
              {"name": "name"}
            ],
            "strings": []
          }
        }
      }
    },
    {
      "edge": {
        "source": "directed",
        "name": "Directed",
        "mode": "merge",
        "transform": {
          "group": true
        },
        "mappings": {
          "type": "\"DIRECTED\"",
          "source": {
            "label": "\"Person\"",
            "key": "person_tmdbId"
          },
          "target": {
            "label": "\"Movie\"",
            "key": "movieId"
          },
          "properties": {
            "unique": [],
            "indexed": [],
            "strings": []
          }
        }
      }
    },
    {
      "edge": {
        "source": "acted_in",
        "name": "Acted_in",
        "mode": "merge",
        "transform": {
          "group": true
        },
        "mappings": {
          "type": "\"ACTED_IN\"",
          "source": {
            "label": "\"Person\"",
            "key": "person_tmdbId"
          },
          "target": {
            "label": "\"Movie\"",
            "key": "movieId"
          },
          "properties": {
            "unique": [],
            "indexed": [],
            "strings": [
              {"role": "role"}
            ]
          }
        }
      }
    }
  ]
}
```

## Configuration

The `config` object contains global configuration for the import job. The flags it supports are:

- `reset_db` (bool) -- whether to clear the target database before importing.
Deletes data, indexes, and constraints.
- `index_all_properties` (bool) -- whether to create indexes for all properties. See [Cypher -> Indexes for search performance](https://neo4j.com/docs/cypher-manual/current/indexes-for-search-performance/).

```json
"config": {
  "reset_db": false,
  "index_all_properties": false
}
```

## Sources

The `sources` section contains the definitions of the data sources, as a list. As a rough guideline, you can think `one table <=> one source`. The importer will leverage the data surfaced by the sources and make it available to the targets, which eventually map it into Neo4j.

Each source object can be of either type `bigquery` or `text`, depending on whether you want to import from a BigQuery dataset or CSV data. Regardless of type, each source must get a `name`, which the targets will later use to refer to it.

### BigQuery dataset

To import a BigQuery dataset, three attributes are compulsory.

```json
{
  "type": "bigquery",
  "name": "movies",
  "query": "SELECT movieId, title FROM team-connectors-dev.movies.movies"
}
```

- `type` (string) -- `bigquery`.
- `name` (string) -- a human-friendly label for the source (unique among all sources). You will use this to reference the source in the targets section.
- `query` (string) -- the dataset to extract from BigQuery, as an SQL query. Notice that:

  1. the source BigQuery table can have more columns than what you select in the query;
  2. multiple targets can use the same source, even filtering it for a subset of columns.

### CSV data

To import data from a CSV file, six attributes are compulsory.

```json
{
  "type": "text",
  "name": "movies",
  "uri": "<path-to-movies-csv>",
  "format": "EXCEL",
  "delimiter": ",",
  "ordered_field_names": "movieId,title"
}
```

- `type` (string) -- `text`.
- `name` (string) -- a human-friendly label for the source (unique among all sources). You will use this to reference the source in the targets section.
- `uri` (string) -- the Google Storage location of the CSV file (ex. `gs://neo4j-datasets/movies.csv`).
- `format` (string) -- any of [Apache's `CSVFormat` predefined formats](https://commons.apache.org/proper/commons-csv/apidocs/org/apache/commons/csv/CSVFormat.html).
- `delimiter` (string) -- CSV field delimiter.
- `ordered_field_names` (string) -- list of field names the CSV file contains, in order.

CSV files must fulfill some constraints:

- **they should not contain headers**. Column names should be specified in the `ordered_field_names` attributes, and files should contain data rows only.
- they should not contain empty rows.

## Targets

The `targets` section contains the definitions of the graph entities that will result from the import.
Neo4j represents objects with [nodes](https://neo4j.com/docs/getting-started/appendix/graphdb-concepts/#graphdb-node) (ex. movies, people) and connects them with [relationships](https://neo4j.com/docs/getting-started/appendix/graphdb-concepts/#graphdb-relationship) (ex. ACTED_IN, DIRECTED).
Each object in the targets section is keyed as either `node` or `edge` (synonym for _relationship_) and will generate a corresponding entity in Neo4j drawing data from a source.

By default, you do not have to think about dependencies between nodes and relationships, as the job imports all node objects _before_ importing any relationships (although you may alter this behavior).


### Node objects

Compulsory attributes for `node` objects are `source`, `mappings.labels`, and `mappings.keys`.

```json
{
  "node": {
    "source": "movies",
    "name": "Movies",
    "mode": "merge",
    "mappings": {
      "labels": [
        "\"Movie\""
      ],
      "keys": [
        {"movieId": "movie_id"}
      ],
      "properties": {
        "unique": [],
        "indexed": [
          {"title": "title"}
        ],
        "strings": []
      }
    },
    "transform": {
      "group": true
    }
  }
}
```

- **`source`** (string) -- the name of the source this target should draw data from. Should match one of the names from the `sources` objects.
- `name` (string) -- a human-friendly name for the target  (unique among all targets).
- `mode` (string) -- the creation mode in Neo4j. Either `merge` or `append` (default). See [Cypher -> `MERGE`](https://neo4j.com/docs/cypher-manual/current/clauses/merge/) and [Cypher -> `CREATE`](https://neo4j.com/docs/cypher-manual/current/clauses/create/) for info.
- `mappings` (object) -- details on how the source columns should be mapped into node details.
* **`labels`** (list of strings) -- [labels](https://medium.com/neo4j/graph-modeling-labels-71775ff7d121) to mark the nodes with. Note that they should be quoted and escaped.
* **`keys`** (list of objects) -- source columns that should be mapped into node properties _and_ that should get a node key constraint.
* `properties` (object) -- mapping of source columns into node properties.
** `unique` (list of objects) -- source columns that should be mapped into node properties _and_ that should get a node uniqueness constraint. These get mapped to string properties.
** `indexed` (list of objects) -- source columns that should be mapped into node properties _and_ that  should get an index on the corresponding node property. These get mapped to string properties.
** `strings`, `floats`, `ints`, `dates`, `points` (list of objects) -- source columns that should be mapped into node properties of the given type. The data type affects how the data is represented into Neo4j, but does not create type constraints.
- `transform` (object) -- if `"group": true`, the import will SQL `GROUP BY` on all fields specified in `keys` and `properties`. If set to `false`, any duplicate data in the source will be pushed into Neo4j, potentially raising constraints errors or making insertion less efficient. The object can also contain aggregation functions, see #Transformations.
- `execute_after` (string) -- target object after which the current target should run. Either `node` or `edge`. To be used in conjunction with `execute_after_name`.
- `execute_after_name` (string) -- the `name` of the target after which the current one should run.

The objects in `keys`, `unique`, `indexed`, and all the type properties (`strings`, `floats`, etc) have the format

```json
{"<column-name-in-source>": "<wished-node-property-name>"}
```

For example, `{"movieId": "movie_id"}` will map the source column `movieId` to the property `movie_id` in the new nodes.

Things to pay attention to:

- **make sure to quote and escape labels**.
- **names in `keys` should not also be listed in `unique`**, or the constraints will conflict.
- **source data must not have null values for `keys` columns**, or they will clash with the node key constraint. If the source is not clean in this respect, think of cleaning it upfront in the related `source.query` field by excluding all rows that wouldn't fulfill the constraints (ex. `WHERE movieId IS NOT NULL`).
- if `index_all_properties: true` in config, it is pointless to specify any columns in `properties.indexed`.


### Edge objects

Compulsory attributes for `edge` objects are `source`, `mappings.type`, `mappings.source`, and `mappings.target`.

```json
{
  "edge": {
    "source": "acted_in",
    "name": "Acted_in",
    "mode": "merge",
    "mappings": {
      "type": "\"ACTED_IN\"",
      "source": {
        "label": "\"Person\"",
        "key": "person_tmdbId"
      },
      "target": {
        "label": "\"Movie\"",
        "key": "movieId"
      },
      "properties": {
        "unique": [],
        "indexed": [],
        "strings": [
          {"role": "role"}
        ]
      }
    },
    "transform": {
      "group": true
    }
  }
}
```

- **`source`** (string) -- the name of the source this target should draw data from. Should match one of the names from the `sources` objects.
- `name` (string) -- a human-friendly name for the target  (unique among all targets).
- `mode` (string) -- the creation mode in Neo4j. Either `merge` or `append` (default). See [Cypher -> `MERGE`](https://neo4j.com/docs/cypher-manual/current/clauses/merge/) and [Cypher -> `CREATE`](https://neo4j.com/docs/cypher-manual/current/clauses/create/) for info.
- `mappings` (object) -- details on how the source columns should be mapped into node details.
* **`type`** (string) -- type to assign to the relationship . Note that it should be quoted and escaped.
* **`source`** (object) -- starting node for the relationship (identified by node label and key).
* **`target`** (object) -- ending node for the relationship (identified by node label and key).
* `properties` (object) -- mapping of source columns into relationship properties.
** `unique` (list of objects) -- source columns that should be mapped into relationship properties _and_ that should get a relationship uniqueness constraint. These get mapped to string properties.
** `indexed` (list of objects) -- source columns that should be mapped into relationship properties _and_ that should get an index on the corresponding relationship property. These get mapped to string properties.
** `strings`, `floats`, `ints`, `dates`, `points` (list of objects) -- source columns that should be mapped into node properties of the given type. The data type affects how the data is represented into Neo4j, but does not create type constraints.
- `transform` (object) -- if `"group": true`, the import will SQL `GROUP BY` on all fields specified in `mappings.source`, `mappings.target`, and properties. If set to `false`, any duplicate data in the source will be pushed into Neo4j, potentially raising constraints errors or making insertion less efficient. The object can also contain aggregation functions, see xref Transformations.
- `execute_after` (string) -- target object after which the current target should run. Either `node` or `edge`. To be used in conjunction with `execute_after_name`.
- `execute_after_name` (string) -- the `name` of the target after which the current one should run.

The objects in `unique`, `indexed`, and all the type properties (`strings`, `floats`, etc) have the format

```json
{"<column-name-in-source>": "<wished-relationship-property-name>"}
```

For example, `{"role": "role"}` will map the source column `role` to the property `role` in the new relationships.

Things to pay attention to:

- **make sure to quote and escape relationship types and node labels**.
- **`source.key` and `target.key` must take names from the source columns, not from the mapped graph properties**.
In the snippet above, notice how the key names are `person_tmdbId` and `movieId` even if the mapped property names in the related node objects are `person_id` and `movie_id`.
- if `index_all_properties: true` in config, it is pointless to specify any columns in `properties.indexed`.

## Transformations

Each target can optionally have a `transform` attribute containing aggregation functions. This can be useful to extract higher-level dimensions from a more granular source. Aggregations result in extra fields that become available for import into Neo4j.

The following example shows how the aggregations would work on a fictitious dataset (not the movies one).

```json
"transform": {
  "group": true,
  "aggregations": [
    {
      "expr": "SUM(unit_price*quantity)",
      "field": "total_amount_sold"
    },
    {
      "expr": "SUM(quantity)",
      "field": "total_quantity_sold"
    }
  ],
  "limit": 50
}
```

- `group` (bool) -- must be `true` for aggregations to work.
- `aggregations` (list of objects) -- aggregation functions are specified as SQL queries in the `expr` attribute, and the result is available under the name specified in `field`.
- `limit` (int) -- append a `LIMIT` clause (here `LIMIT 50`) to the generated SQL query that fetches the source data (defaults to no limit, encoded as `-1`).

## Pre/Post load actions

The `actions` section contains commands that can be run before or after specific steps of the import process. You may for example submit HTTP requests when steps complete, or execute SQL queries on the source, or Cypher statements on the Neo4j target.

```json
{
  "name": "Post load POST request",
  "execute_after": "edge",
  "execute_after_name": "Acted_in",
  "type": "http_post",
  "options": [
    {"url": "https://httpbin.org/post"},
    {"param1": "value1"}
  ],
  "headers": [
    {"header1": "value1"},
    {"header2": "value2"}
  ]
}
```

- `name` (string) -- a human friendly name for the action.
- `execute_after` (string) -- after what import step the action should run. Valid values are:
* `preloads` -- before any source is parsed
* `sources` -- after sources have been parsed
* `nodes` -- after all node objects have been imported
* `edges` -- after all edge objects have been imported
* `loads` -- after all entities (nodes+edges) have been imported
* `source`, `node`, `edge`, `action` -- after a specific source or node or edge or action object has been run, to be used in conjunction with `execute_after_name`
- `execute_after_name` (string) -- after which `source`/`node`/`edge`/`action` object the step should run.
- `type` (string) -- what action to run. Valid values are:
* `http_post` -- HTTP POST request (requires a `url` option)
* `http_get` -- HTTP GET request  (requires a `url` option)
* `bigquery` -- query to a BigQuery database (requires an `sql` option)
* `cypher` -- query to the target Neo4j database (requires a `cypher` option)
- `options` (list of objects) -- action options, such as `url`, `sql`, `cypher`.
- `headers` (list of objects) -- headers to send with the request.

## Variables

For production use cases it is common to supply date ranges or parameters based on dimensions, tenants, or tokens. Key-values can be supplied to replace `$` delimited tokens in SQL queries, URLs, or parameters. You can provide parameters in the `Options JSON` field when creating the Dataflow job, as a JSON object.

Variables must be escaped with the `$` symbol (ex. `$limit`). Replaceable tokens can appear in job specification files, in `readQuery` or `inputFilePattern` (source URI) command-line parameters, or in action options/headers.

# Run an import job

Once the job specification is ready, upload its json file to your Cloud Storage bucket and go to [Dataflow -> Create job from template](https://console.cloud.google.com/dataflow/createjob).

To create a job, specify the following fields:

- Job name -- a human-friendly name for the job
- Regional endpoint -- must match the region of your Google Cloud Storage bucket
- Dataflow template -- select `Google Cloud to Neo4j`
- Path to job configuration file -- the json job specification file (from Cloud storage buckets)
- Path to Neo4j connection metadata -- the json connection information file (from Cloud storage buckets)
- Options JSON -- values for variables used in the job specification file (or `{}` for none)

[!Dataflow job example](dataflow-job-example.png)

# CLI invocations

You can script execution of Dataflow jobs using the [Google Cloud CLI](https://cloud.google.com/dataflow/docs/guides/templates/using-flex-templates). Executing a job through the CLI is functionally equivalent to using the Dataflow user interface. Accepting the defaults yields a very minimalistic script:

```bash
export REGION=us-central1
gcloud dataflow flex-template run "test-bq-cli-`date +%Y%m%d-%H%M%S`" \
  --template-file-gcs-location="gs://dataflow-templates/latest/flex/Google_Cloud_to_Neo4j" \
  --region "$REGION" \
  --parameters jobSpecUri="<URI-to-job-specification-JSON-file>" \
  --parameters neo4jConnectionUri="<URI-to-neo4j-connection-JSON-file>"
```

The REST version looks like this:

```bash
export REGION=us-central1
curl -X POST "https://Dataflow.googleapis.com/v1b3/projects/<project-name>/locations/$REGION/flexTemplates:launch" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
-d '{
   "launch_parameter": {
      "jobName": "test-bq-rest-'$(date +%Y%m%d-%H%M%S)'",
      "parameters": {
        "jobSpecUri": "<URI-to-job-specification-JSON-file>",
        "neo4jConnectionUri": "<URI-to-neo4j-connection-JSON-file>"
      },
   "containerSpecGcsPath": "gs://dataflow-templates/latest/flex/Google_Cloud_to_Neo4j"
   }
}'
```

The `cli-script` folder in the [Neo4j Dataflow Template repository](https://github.com/GoogleCloudPlatform/DataflowTemplates/tree/main/v2/googlecloud-to-neo4j/docs/cli-scripts/dataflow-test) contains some example invocations.
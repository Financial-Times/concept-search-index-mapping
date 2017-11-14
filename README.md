# concept-search-index-mapping
Elasticsearch mapping for the concept search index.

The Docker image built by this project is based on the elasticsearch-reindexer. It applies the mapping.json file for the concepts index.

# Example Queries

Abouts/Mentions using the `edge_ngram` analyzer and a `0.1` boost on a term match:

```
{
   "query": {
      "bool": {
         "must": {
            "match": {
               "prefLabel.edge_ngram": {
                  "query": "donald trump"
               }
            }
         },
         "filter": {
            "terms": {
               "_type": [
                  "people",
                  "topics",
                  "organisations",
                  "locations"
               ]
            }
         },
         "should": [
            {
               "match": {
                  "prefLabel": {
                     "query": "donald trump",
                     "boost": 0.1
                  }
               }
            }
         ],
         "boost": 1
      }
   }
}
```

Abouts/Mentions with an additional boost using the `exact_match` analyzer:

```
{
   "query": {
      "bool": {
         "must": {
            "match": {
               "prefLabel.edge_ngram": {
                  "query": "new york"
               }
            }
         },
         "filter": {
            "terms": {
               "_type": [
                  "people",
                  "topics",
                  "organisations",
                  "locations"
               ]
            }
         },
         "should": [
            {
               "match": {
                  "prefLabel": {
                     "query": "new york",
                     "boost": 0.1
                  }
               }
            },
            {
               "term": {
                  "prefLabel.exact_match": {
                     "value": "new york",
                     "boost": 0.5
                  }
               }
            }
         ],
         "boost": 1
      }
   }
}
```

Abouts/Mentions using the `mentionsCompletion` suggester:

```
{
   "suggest": {
      "conceptSuggestion": {
         "text": "oita",
         "completion": {
            "field": "prefLabel.mentionsCompletion"
         }
      }
   }
}
```

Author boost using the `authorCompletionByContext` suggester:

```
{
   "suggest": {
      "conceptSuggestion": {
         "text": "martin wol",
         "completion": {
            "field": "prefLabel.authorCompletionByContext",
            "contexts": {
               "authorContext": {
                  "context": "true",
                  "boost": 2
               },
               "typeContext": "people"
            }
         }
      }
   }
}
```

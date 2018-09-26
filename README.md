# concept-search-index-mapping
Elasticsearch mapping for the concept search index.

The Docker image built by this project is based on the elasticsearch-reindexer. It applies the mapping.json file for the concepts index.

# Reindexing process tips & tricks
In order to trigger a reindexing using the newest values from [mapping.json](mapping.json) you should create a new github release and follow the progress of the deployment in jenkins.

## Things to do before triggering the jenkins job
* Check the indexes that already exist in the ES instance by running:
`GET <aws_es_endpoint>/_cat/indices?v`
* Check to see which index is un use:
`GET <aws_es_endpoint>/_cat/aliases`
* Delete the oldest index. Usually, there are two indexes available, one that is referenced by the `concepts` alias, and another one that should be an older version. This is needed because most of the time there is not enough disk space for the new index:
`DELETE <aws_es_endpoint>/<targeted_index_name>`

## Things to do during the reindexing process
Once triggered, here's how to monitor the reindexing process:
* Get the list of ES reindexing tasks:
`GET <aws_es_endpoint>/_tasks?detailed=true&actions=*reindex`
* Get the task identifier (which should be something similar to `r1A2WoRbTwKZ516z6NEs5A:36619`) and use it for fetching info just for your task:
`GET <aws_es_endpoint>/_tasks/r1A2WoRbTwKZ516z6NEs5A%3A36619` (note that this should be encoded accordingly `:` => `%3A`)

## Actual values
For being able to execute all the above commands, you will need the following:
* aws_es_endpoint: you can get this directly from AWS->Elasticsearch->Endpoint
* Credentials(AwsAccessKey, AwsSecretAccessKey): get them from LastPass -> `AWS ElasticSearch - Provisioning Setup`


# Example Queries

People search using

* the `edge_ngram` analyzer and a `0.1` boost on a term match
* the isFTAuthor `1.8` boost
* the `exact_match` analyzer  

```
{
   "size" : 20,
   "query": {
      "bool": {
         "must": {
            "match" : {
                "prefLabel.edge_ngram": {
                    "query":"Donald Gi"
                }
            }
         },
         "filter": {
            "term": {
               "_type": "people"
            }
         },
         "should": [
            {
               "term": {
                  "isFTAuthor": {
                    "value": "true",
                    "boost": 1.8
                  }
               }
            },
            {
               "match": {
                  "prefLabel": {
                    "query": "Donald Gi",
                    "boost": 0.1
                  }
               }
            },
            {
                "match": {
                   "prefLabel.exact_match": {
                      "query": "Donald Gi",
                      "boost": 0.3
                   }
                }
             }
         ],
         "minimum_should_match": 0,
         "boost": 1
      }
   }
}

```


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

The Abouts/Mentions search with all the bells & whistles - namely, `_type`, `scopeNote` and popularity (using `annotationsCount`) boosting:

```
{
  "query": {
    "bool": {
      "must": {
        "bool": {
          "should": [
            {
              "match": {
                "prefLabel.edge_ngram": "u"
              }
            },
            {
              "match": {
                "aliases.exact_match": "u"
              }
            }
          ],
          "minimum_should_match": 1
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
              "query": "u",
              "boost": 0.1
            }
          }
        },
        {
          "match": {
            "prefLabel.exact_match": {
              "query": "u",
              "boost": 0.3
            }
          }
        },
        {
          "match": {
            "aliases.exact_match": {
              "query": "u",
              "boost": 0.65
            }
          }
        },
        {
          "term": {
            "_type": {
              "value": "topics",
              "boost": 1.5
            }
          }
        },
        {
          "term": {
            "_type": {
              "value": "locations",
              "boost": 0.25
            }
          }
        },
        {
          "term": {
            "_type": {
              "value": "people",
              "boost": 0.1
            }
          }
        },
        {
          "exists": {
            "field": "scopeNote",
            "boost": 1.7
          }
        },
        {
        	"function_score": {
	            "field_value_factor": {
	                "field": "metrics.annotationsCount",
	                "modifier": "ln2p",
	                "missing": 0
	            },
	            "boost": 3.2
	        }
        }
      ],
      "boost": 1
    }
  }
}
```

Currently, only BRANDS search uses the suggester

@Deprecated Abouts/Mentions using the `mentionsCompletion` suggester:

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

@Deprecated Author boost using the `authorCompletionByContext` suggester:

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

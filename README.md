# PoC - Building a Search Engine from Scratch with Elasticsearch

This repo contains step-by-step instructions, tutorials, tips and other resources about how to create a search engine with Elasticsearch from scratch.

In this initial version, a Spanish dataset will be used, but the instructions below are equivalent (with a few changes) to another dataset in any other language.

Topics covered in this version:

* Index mapping
* Analyzers and normalizers
* Textual search with BM25 (query DSL)
* Semantic search
* Reranking
* A basic recommendation engine

## Prerequisites 

1. Download the [`spanish_products.ndjson`](datasets/spanish_products.ndjson) dataset from the [`datasets`](datasets) folder
   
2. Get an Elasticsearch cluster with, at least, the following resources (note: you can use the [Elastic Cloud trial](https://www.elastic.co/cloud/cloud-trial-overview) to get started): 
* 2 x 2 GB RAM Elasticsearch Hot/Content node
* 1 x 4 GB RAM Elasticsearch ML node
* 1 x 2 GB RAM Kibana

**NOTE: This document has been prepared considering Elasticsearch version 8.15.0.** In case of using a different version, certain features may not be available or may require some adaptation in some of the shared syntaxes.

3. An valid API Key in Cohere (or equivalent service / model) for rerank tasks

## Building the Search Engine Step-by-step

### Version 1 - Just importing the data

1. Upload the [`spanish_products.ndjson`](datasets/spanish_products.ndjson) in Kibana using the [UI](https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html#upload-data-kibana)

2. Leave all configuration options as default and choose `products_v1` as the name for this initial search engine

3. Go to Discover and start testing this initial version. Some queries you can test are:

  > vestido rojo

  > PANTALÓN NEGRO

You will see that the results are quite good despite the fact that no specific configuration has been made. In fact, this first version is not case sensitive, it searches the querystring in different fields (at least title and description)...

But it also has important shortcomings:

  > faldas

The above query does not return any results even though there is a category with the exact word (but capitalized) and the term "falda" (singular) does appear in numerous products. This is due to how the `category` and `tags` fields are being used and that no specific analyzer has been configured to “understand” Spanish.

  > elastic fashion

Again, it does not return any results even though there is a `vendor` named like this. Again it is due to how this field is configured.

  > pantalón de algodón

In this case, there are results like `Pantalón de tartán rojo` (5th) that do not even contain the keyword `algodón`. There are also results like `Pantalones negros con rasgaduras` which are made "100% algodón" and rank worse (9th and following). The reason is that the keyword `de` is being used to score results when it does not provide real value in the search.

  > strong

The `description` field contains HTML tags and, in this first version, you can search for any of them, returning results that are not expected by any future user.

In the next versions we will be correcting these deficiencies.

### Version 2 - Adjusting the mapping

4. Using Dev Tools, define the index mapping of the new version of our search engine:

```
PUT products_v2
{
    "mappings": {
      "properties": {
        "category": {
          "type": "keyword"
        },
        "compare_at_price": {
          "type": "short"
        },
        "description": {
          "type": "text"
        },
        "discount": {
          "type": "short"
        },
        "price": {
          "type": "short"
        },
        "product_id": {
          "type": "keyword"
        },
        "tags": {
          "type": "keyword"
        },
        "title": {
          "type": "text"
        },
        "url": {
          "type": "keyword"
        },
        "variants": {
          "properties": {
            "quantity": {
              "type": "short"
            },
            "size": {
              "type": "keyword"
            }
          }
        },
        "vendor": {
          "type": "text",
          "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
          
        }
      }
    }
}
```

5. Reindex:

```
POST _reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v2"
  }
}
```

6. Create a data view (from Kibana) with the same name of the index: `products_v2`


7. Test this new version from Discover: 

In this version we have improved:

* Mapping is now optimal in terms of resources: memory and storage. For example, numeric field were set to `short` instead of `long` 
* Certain fields (`vendor`) have become searchable. For example, the following query now returns valid results: 

  > elastic fashion

Even when using just one of the two keywords (`elastic` or `fashion`), relevant results are returned.

But all the problems identified above still exist for the following queries:

  > faldas

  > pantalón de algodón

  > strong

Further, we must now decide how we want our search engine to behave with queries like the following. Do we want to be able to search the `category` and `tags` fields using textual search or just exact search? Try both queries to check the results returned and decide how you prefer the search engine to behave.

  > faldas blancas
  > falda blanca

Another deficiency is that in this version (nor in the previous one) spelling mistakes derived from the use or not of a tilde were NOT solved: `algodon` vs `algodon`. Así, la siguiente consulta no devuelve los resultados esperados:

  > pantalones de algodon


### Version 3 - Defining analyzers and normalizers

8. Define the analyzers and normalyzers for versions 3 and 4 of our search engine. The difference between there are:
* version 3 will not accept textual searches in the fields `tags` and `category`
* version 4 will accept textual search in those fields

```
PUT products_v3
{
  "settings": {
    "analysis": {

      "filter": {
        "spanish_stemmer": {
          "language": "light_spanish",
          "type": "stemmer"
        },
        "spanish_stop": {
          "stopwords": "_spanish_",
          "type": "stop"
        }
      },

      "analyzer": {
        "my_text_analyzer": {
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "spanish_stop",
            "spanish_stemmer"
          ],
          "tokenizer": "standard"
        }
      },

      "normalizer": {
        "keyword_normalizer": {
          "char_filter": [],
          "filter": [
            "lowercase",
            "asciifolding"
          ],
          "type": "custom"
        }
      }  
    }
  },
  "mappings": {
    "properties": {
      "category": {
        "normalizer": "keyword_normalizer",
        "type": "keyword"
      },
      "compare_at_price": {
        "type": "short"
      },
      "description": {
        "analyzer": "my_text_analyzer",
        "type": "text"
      },
      "discount": {
        "type": "short"
      },
      "price": {
        "type": "short"
      },
      "product_id": {
        "type": "keyword"
      },
      "tags": {
        "normalizer": "keyword_normalizer",
        "type": "keyword"
      },
      "title": {
        "analyzer": "my_text_analyzer",
        "type": "text"
      },
      "url": {
        "type": "keyword"
      },
      "variants": {
        "properties": {
          "quantity": {
            "type": "short"
          },
          "size": {
            "normalizer": "keyword_normalizer",
            "type": "keyword"
          }
        }
      },
      "vendor": {
        "type": "text",
        "fields": {
          "keyword": {
            "normalizer": "keyword_normalizer",
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

```
PUT products_v4
{
  "settings": {
    "analysis": {

      "filter": {
        "spanish_stemmer": {
          "language": "light_spanish",
          "type": "stemmer"
        },
        "spanish_stop": {
          "stopwords": "_spanish_",
          "type": "stop"
        }
      },

      "analyzer": {
        "my_text_analyzer": {
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "spanish_stop",
            "spanish_stemmer"
          ],
          "tokenizer": "standard"
        }
      },

      "normalizer": {
        "keyword_normalizer": {
          "char_filter": [],
          "filter": [
            "lowercase",
            "asciifolding"
          ],
          "type": "custom"
        }
      }  
    }
  },
  "mappings": {
    "properties": {
      "category": {
        "type": "text",
        "analyzer": "my_text_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "compare_at_price": {
        "type": "short"
      },
      "description": {
        "type": "text",
        "analyzer": "my_text_analyzer"
      },
      "discount": {
        "type": "short"
      },
      "price": {
        "type": "short"
      },
      "product_id": {
        "type": "keyword",
        "normalizer": "keyword_normalizer"
      },
      "tags": {
        "type": "text",
        "analyzer": "my_text_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "title": {
        "type": "text",
        "analyzer": "my_text_analyzer"
      },
      "url": {
        "type": "keyword"
      },
      "variants": {
        "properties": {
          "quantity": {
            "type": "short"
          },
          "size": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "vendor": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      }
    }
  }
}
```

9.  Reindex the data from v1:

```
POST _reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v3"
  }
}
```

```
POST _reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v4"
  }
}
```

10. Create both data views using the same name as the indices: `products_v3` and `products_v4`

11. Check the results in Discover:

In this versions, we have improved:

* HTML tags are no longer searchable, as we have defined a `char_filter` to ignore them
* The nuew mapping is now optimal for this use case: searches can be made in any required field. Versions 3 and 4 differ in the mapping of `category` and `tags`
* Documents and queries are “understood”, although they are written in Spanish, so both are correctly processed by the defined parsers and normalizers. This means:
  * language aspects such as gender and number are considered
  * (basic) misspellings is supported
  * stop words are ignored

In other words, the following queries now work as expected:

  > strong

This query no longer returns any results.

  > faldas

  > faldas blancas

  > falda blanca

The above queries return the expected results, although there is a difference between versions 3 and 4 for the last two queries due to the differences in the mapping.

  > pantalón de algodón

  > pantalon de algodon

Words without tildes are understood correctly, so both queries return the same results. Also, the word `de` is no longer used for scoring. This improves the results considerably.

**Finally we have something functional!**


### Version 4 - You know, for SEARCH

Note: For the following examples, only version 4 of our search engine has been considered.

The current versions of our search engine work as expected, but since Discover, queries like the one below do not rank the best results at the top:

  > faldas azules con bolsillos

The reason is that the querystring contains 3 keywords that have the same weight in all the fields used to resolve the query. 

The above query in Discover is equivalent to the following [DSL query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html). Try to launch it from Dev Tools:


```
POST /products_v4/_search
{
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ],
  "fields": ["title", "description","category", "tags"],
  "size": 10,
  "_source": false,
  "query": {
    "bool": {
      "filter": [],
      "must": [
        {
          "multi_match": {
            "type": "best_fields",
            "query": "faldas azules con bolsillos",
            "lenient": true
          }
        }
      ],
      "should": [],
      "must_not": []
    }
  }
}
```

The following query uses [per-field boosting](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#field-boost) to position the blue skirts (`faldas azules`) in the first results:

```
POST /products_v4/_search
{
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ],
  "fields": ["title", "description", "category", "tags"],
  "size": 10,
  "_source": false,
  "query": {
    "bool": {
      "filter": [],
      "must": [
        {
          "multi_match": {
            "type": "best_fields",
            "query": "falda azul con bolsillos",
            "lenient": true,
            "fields": [ "category^2", "tags^2", "title", "description" ]
          }
        }
      ],
      "should": [],
      "must_not": []
    }
  }
}
```

This is just a simple example of how to tune relevance with Elasticsearch, but there are many other possibilities, such as [boosting depending on the source index](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-multiple-indices.html#index-boost), defining a [function](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html) and/or a [script](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-script-score) to modify the score calculation, [reciprocal rank fusion (RRF)](https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html) and many others.


### Version 5 - Applying Semantic Search

12.  Download, deploy and create an inference endpoint for the embedding calculation: 

```
PUT _inference/text_embedding/my-e5-model
{
  "service": "elasticsearch",
  "service_settings": {
    "num_allocations": 1,
    "num_threads": 1,
    "model_id": ".multilingual-e5-small_linux-x86_64" 
  }
}
```

13. Define a new mapping for a new version of the search engine (based on version 4)

```
PUT products_v5
{
  "settings": {
    "analysis": {

      "filter": {
        "spanish_stemmer": {
          "language": "light_spanish",
          "type": "stemmer"
        },
        "spanish_stop": {
          "stopwords": "_spanish_",
          "type": "stop"
        }
      },

      "analyzer": {
        "my_text_analyzer": {
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "spanish_stop",
            "spanish_stemmer"
          ],
          "tokenizer": "standard"
        }
      },

      "normalizer": {
        "keyword_normalizer": {
          "char_filter": [],
          "filter": [
            "lowercase",
            "asciifolding"
          ],
          "type": "custom"
        }
      }  
    }
  },
  "mappings": {
    "properties": {
      "category": {
        "type": "text",
        "analyzer": "my_text_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "compare_at_price": {
        "type": "short"
      },
      "description": {
        "type": "text",
        "analyzer": "my_text_analyzer"
      },
      "description_semantic": {
        "type": "semantic_text",
        "inference_id": "my-e5-model"
      },
      "discount": {
        "type": "short"
      },
      "price": {
        "type": "short"
      },
      "product_id": {
        "type": "keyword",
        "normalizer": "keyword_normalizer"
      },
      "tags": {
        "type": "text",
        "analyzer": "my_text_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "title": {
        "type": "text",
        "analyzer": "my_text_analyzer"
      },
      "url": {
        "type": "keyword"
      },
      "variants": {
        "properties": {
          "quantity": {
            "type": "short"
          },
          "size": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      },
      "vendor": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "normalizer": "keyword_normalizer"
          }
        }
      }
    }
  }
}
```

14. Define an ingest pipeline to clean up (now we are going to do it explicitly) HTML tags from the `description` field

```
PUT _ingest/pipeline/product_v5_clean_description
{
  "description": "Clean the HTML tags from the description into another field",
  "processors": [
    {
      "html_strip": {
        "field": "description",
        "ignore_failure": true,
        "ignore_missing": true,
        "target_field": "description_semantic"
      }
    }
  ]
}
```


15. Reindex

```
POST _reindex?wait_for_completion=false
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v5",
    "pipeline": "product_v5_clean_description"
  }
}
```

16. Use Dev Tools for test the following queries:

```
POST /products_v4/_search
{
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ],
  "fields": ["title", "description", "category", "tags"],
  "size": 10,
  "_source": false,
  "query": {
    "bool": {
      "filter": [],
      "must": [
        {
          "multi_match": {
            "type": "best_fields",
            "query": "tejanos",
            "lenient": true,
            "fields": [ "category^2", "tags^2", "title", "description" ]
          }
        }
      ],
      "should": [],
      "must_not": []
    }
  }
}
```

The query above (pointing to version 4) does not return any results, but an equivalent query (same query string) pointing to our new version 5 does:

```
GET products_v5/_search
{
  "query": {
    "semantic": {
      "field": "description_semantic",
      "query": "tejanos"
    }
  },
  "_source": false,
  "fields": ["title", "description", "category", "tags"]
}
```

Hybrid search (textual with BM25 and semantic) is also possible:

```
GET products_v5/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": {
                "title": "faldas"
              }
            }
          }
        },
        {
          "standard": {
            "query": {
              "semantic": {
                "field": "description_semantic",
                "query": "tejanos"
              }
            }
          }
        }
      ]
    }
  },
  "_source": false,
  "fields": ["title", "description", "category", "tags"]
}
```

17.  We can even take advantage of ML models with the “classic” BM25 search. That is: we can apply reranking to a BM25 query.

Note: define the `${cohere_key}` variable in Dev Tools or replace it with a valid value

```
# Define the rerank model
PUT _inference/rerank/cohere-rerank
{
    "service": "cohere",
    "service_settings": {
        "api_key": "${cohere_key}",
        "model_id": "rerank-multilingual-v3.0"
    },
    "task_settings": {
        "top_n": 100,
        "return_documents": true
    }
}

# Reference query (without rerank): 3 relevant results
POST /products_v4/_search
{
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ],
  "fields": ["title", "description", "category", "tags"],
  "_source": false,
  "query": {
    "bool": {
      "filter": [],
      "must": [
        {
          "multi_match": {
            "type": "best_fields",
            "query": "falda azul con bolsillos",
            "lenient": true,
            "fields": [ "category^1.5", "tags^1.5", "title", "description" ]
          }
        }
      ],
      "should": [],
      "must_not": []
    }
  }
}

# Reranking with BM25 -> 4 relevant results. 1 additional relevant result!!
POST /products_v5/_search
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": {
        "standard": {
          "query": {
            "bool": {
              "filter": [],
              "must": [
                {
                  "multi_match": {
                    "type": "best_fields",
                    "query": "falda azul con bolsillos",
                    "lenient": true,
                    "fields": [ "category^1.5", "tags^1.5", "title", "description" ]
                  }
                }
              ],
              "should": [],
              "must_not": []
            }
          }
        }
      },
      "field": "description",
      "inference_id": "cohere-rerank",
      "inference_text": "falda azul con bolsillos",
      "rank_window_size": 100,
      "min_score": 0.5
    }
  },
  "fields": ["title", "description", "category", "tags"],
  "_source": "false"
}


```


### Semantic Search as a Recommendation Engine

Finally, we are going to use Elasticsearch semantic search to build a recommendation engine based on products similar to what a user might be looking at on a product page. That is, we will return the N most similar products to a given one.

18.  In Dev Tools, launch a semantic query with any query string. Below an example:

```
GET products_v5/_search
{
  "query": {
    "semantic": {
      "field": "description_semantic",
      "query": "tejanos"
    }
  },
  "_source": false,
  "fields": ["title", "description", "category", "tags", "description_semantic.inference.chunks.embeddings"]
}
```

19.  From any of the results, copy the vector found in the `description_semantic.inference.chunks.embeddings` field and use it to complete the query below. 

```
GET products_v5/_search
{
  "knn": {
    "field": "description_semantic.inference.chunks.embeddings",
    "k": 10,
    "num_candidates": 100,
    "query_vector": [ ... REPLACE HERE ... ]
  },
  "_source": false,
  "fields": ["title", "description", "category", "tags"]
}
```

20. Launch the query once it is complete and check that the products returned are “similar” to the original (which is returned in the first position). This list of results can serve as a recommendation to visit other similar products.


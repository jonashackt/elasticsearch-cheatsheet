# elasticsearch-cheatsheet
Some helpful CURLs for interacting directly with es

#### show indices settings
```
curl -XGET 'http://localhost:9200/_all/_settings?pretty=true'
```

#### get elasticsearch version:
```
curl -XGET 'localhost:9200'
```

#### show cluster health status:
```
curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'
```

#### show nodes:
```
curl -XGET 'http://localhost:9200/_cat/nodes?pretty=true'
```

#### show shards:
```
curl -XGET http://localhost:9200/_cat/shards
```

#### show indices:
```
curl -XGET 'http://localhost:9200/_cat/indices?v'
```

#### deactivate shard allocation:
```
curl -XPUT http://localhost:9200/_cluster/settings -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}'
```

#### activate shard allocation:
```
curl -XPUT http://localhost:9200/_cluster/settings -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}'
```

#### reallocate an index
```
curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
        "commands" : [ {
              "allocate" : {
                  "index" : "logstash-2016.09.21", 
                  "shard" : 5, 
                  "node" : "Dormammu", 
                  "allow_primary" : true
              }
            }
        ]
    }'
```

####  reallocate all unallocated indices
```
for shard in $(curl -XGET http://localhost:9200/_cat/shards | grep UNASSIGNED | awk '{print $2}'); do
    curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
        "commands" : [ {
              "allocate" : {
                  "index" : "logstash-2016.06.02", 
                  "shard" : $shard, 
                  "node" : "Assassin", 
                  "allow_primary" : true
              }
            }
        ]
    }'
    sleep 5
done
```


#### flush for rolling cluster restart
```
curl -XPOST "http://localhost:9200/elasticsearch/_flush/synced"
```

#### set all indices number of replicas to 0 (if you only have on node!)
```
curl -XPUT localhost:9200/_settings -d '{
    "index" : {
        "number_of_replicas" : 0
    }
}'
```

#### create an index, e.g. when kibana says: "unable to fetch mapping"
```
curl -XPUT 'http://localhost:9200/logstash-2016.09.24/' -d '{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, 
            "number_of_replicas" : 0 
        }
    }
}'
```

#### get logstash template from es
```
 curl -XGET localhost:9200/_template/logstash?pretty=true
```

## Upgrade elasticsearch from 1.x to 2.x


Only start the upgrade-process, if status is green! check with:
```
curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'
```

When not green, first make sure it get´s there! Then start.


#### 1. deactivate shard allocation:
```
curl -XPUT http://localhost:9200/_cluster/settings -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}'
```

#### 2. flush for rolling cluster restart
```
curl -XPOST "http://localhost:9200/elasticsearch/_flush/synced"
```

#### 3. stop elasticsearch
```
sudo service elasticsearch stop
```

#### 4. install new elasticsearch version
```
deb http://packages.elasticsearch.org/elasticsearch/{{ elk_elasticsearch.version }}/debian stable main
apt-get install elasticsearch
```

#### 5. wait for yellow
```
curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'
```

#### 6. activate shard allocation again:
```
curl -XPUT http://localhost:9200/_cluster/settings -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}'
```

## and don´t forget logstash upgrade from 1.x to 2.x

https://www.elastic.co/guide/en/logstash/current/_upgrading_logstash_and_elasticsearch_to_2_0.html

#### 1. change logstash elasticsearch template

Because of a known issue (http://stackoverflow.com/questions/32761038/elk-unable-to-fetch-mapping-do-you-have-indices-matching-the-pattern, https://discuss.elastic.co/t/elasticseach-2-geoip-problem/33424/4), you have to manually change the logstash elasticsearch template. Therefor look into your logstash template with:

```
curl -XGET localhost:9200/_template/logstash?pretty=true
```

if it has a line  __"path" : "full",__ in it like:
```
{
    "order" : 0,
    "template" : "logstash-*",
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "string_fields" : {
            "mapping" : {
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string",
              "fields" : {
                "raw" : {
                  "ignore_above" : 256,
                  "index" : "not_analyzed",
                  "type" : "string"
                }
              }
            },
            "match_mapping_type" : "string",
            "match" : "*"
          }
        } ],
        "_all" : {
          "enabled" : true
        },
        "properties" : {
          "geoip" : {
            "path" : "full",
            "dynamic" : true,
            "type" : "object",
            "properties" : {
              "location" : {
                "type" : "geo_point"
              }
            }
          },
          "@version" : {
            "index" : "not_analyzed",
            "type" : "string"
          }
        }
      }
    },
    "aliases" : { }
  }
```

then we have to delete this line with:

```
curl -XPUT http://localhost:9200/_template/logstash -d '{
    "order" : 0,
    "template" : "logstash-*",
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "string_fields" : {
            "mapping" : {
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string",
              "fields" : {
                "raw" : {
                  "ignore_above" : 256,
                  "index" : "not_analyzed",
                  "type" : "string"
                }
              }
            },
            "match_mapping_type" : "string",
            "match" : "*"
          }
        } ],
        "_all" : {
          "enabled" : true
        },
        "properties" : {
          "geoip" : {
            "dynamic" : true,
            "type" : "object",
            "properties" : {
              "location" : {
                "type" : "geo_point"
              }
            }
          },
          "@version" : {
            "index" : "not_analyzed",
            "type" : "string"
          }
        }
      }
    },
    "aliases" : { }
}'



## Helpful Links

https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html

https://www.elastic.co/guide/en/elasticsearch/reference/1.4/cluster-nodes-shutdown.html

http://blog.florian-hopf.de/2015/02/fixing-elasticsearch-allocation-issues.html

http://blog.kiyanpro.com/2016/03/06/elasticsearch/reroute-unassigned-shards/

https://t37.net/how-to-fix-your-elasticsearch-cluster-stuck-in-initializing-shards-mode.html
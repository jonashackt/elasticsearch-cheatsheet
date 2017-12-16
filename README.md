# elasticsearch-cheatsheet

## How to delete logstash incides, if your disk space is low

[This is a good answer on so](https://stackoverflow.com/questions/33430055/removing-old-indices-in-elasticsearch), but i´ll give you a complete guide here:

If you´re on elasticsearch 2.x like me, the [4.x docs of curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/4.3/index.html) are a good choice :)

#### Install the corresponding elasticsearch-curator version:

* install 5.x, if you have elasticsearch 5.x via `sudo pip install -Iv elasticsearch-curator`
* install 4.3.1, if you have elasticsearch 2.x via `sudo pip install -Iv elasticsearch-curator==4.3.1`
* install 3.5.1, if you have elasticsearch 1.x via `sudo pip install -Iv elasticsearch-curator==3.5.1`

> If curator isn´t working (e.g. if you have a already installed but corrupt installation of curator, remove the package with a `sudo rm -rf /usr/local/lib/python2.7/dist-packages/curator`, https://stackoverflow.com/a/14572899/4964553)

#### Create a curator-configfile.yml (or download from here)

You can copy the contents from here - you only need to change the `unit_count: 14` to the quantity of days you don´t want to delete.

[curator-configfile.yml](https://github.com/jonashackt/elasticsearch-cheatsheet/blob/master/curator-configfile.yml):

```
---
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
client:
  hosts:
    - 127.0.0.1
  port: 9200
  url_prefix:
  use_ssl: False
  ssl_no_validate: False
  timeout: 30
  master_only: False

logging:
  loglevel: INFO
  logfile:
  logformat: default
```

#### Create a curator-actionfile.yml (or download from here)

[curator-actionfile.yml](https://github.com/jonashackt/elasticsearch-cheatsheet/blob/master/curator-actionfile.yml):

```
---
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
#
# Also remember that all examples have 'disable_action' set to True.  If you
# want to use this action as a template, be sure to set this to False after
# copying it.
actions:
  1:
    action: delete_indices
    description: >-
      Delete indices older than 45 days (based on index name), for logstash-
      prefixed indices. Ignore the error if the filter does not result in an
      actionable list of indices (ignore_empty_list) and exit cleanly.
    options:
      ignore_empty_list: True
      timeout_override:
      continue_if_exception: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 14
      exclude:
```

#### Copy both files to your linux box that runs elaticsearch

e.g. to folder `/home/userName/.curator`

#### Run curator

Start with a dry-run:

`curator --dry-run --config curator-configfile.yml curator-actionfile.yml`

If that looks good, delete your Indices with:

`curator --config curator-configfile.yml curator-actionfile.yml`

#### Optional: Setup regularly schedule to run the deletion

Put delete-logstash-indices bash script into `/etc/cron.daily` and you´re done with that issue!


## Some helpful CURLs for interacting directly with elasticsearch (mostly 2.x tested)

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

#### allocate UNASSIGNED shards

One specific:
```
curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
        "commands" : [ {
              "allocate" : {
                  "index" : "logstash-2017.12.09",
                  "shard" : 0,
                  "node" : "Human Torch II",
                  "allow_primary" : true
              }
            }
        ]
    }'
```

All of them:
```
curl -XGET http://localhost:9200/_cat/shards | grep UNASSIGNED | awk '{print 1,2}' | while read var_index var_shard; do
    curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
        "commands" : [ {
              "allocate" : {
                  "index" : "$var_index",
                  "shard" : $var_shard,
                  "node" : "Human Torch II",
                  "allow_primary" : true
              }
            }
        ]
    }'
    sleep 5;
done
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

#### activate elasticsearch logging

```
curl -XPUT 'http://localhost:9200/_cluster/settings/' -d '{
    "transient" : {
        "logger.discovery" : "DEBUG"
    }
}'
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
```

#### 2. update logstash config:

add the line __template_overwrite => true__ to the output-section of your __logstash.conf__:

```
output {
	elasticsearch {
		hosts => [ "localhost:9200" ]
		template_overwrite => true
	}
}
```

#### 3. see logstash logs

```
tail -f /var/log/logstash/logstash.log
```

#### 4. need more details in logstash logging while having problems with json filter?

Add the following to your logstash output section:
```
output {
  file {
        path => "/var/log/logstash/jsonparsefailure.debug.log"
        codec => "rubydebug"
    }
}
```

#### 5. Fix Parsed JSON object/hash requires a target configuration option

If you get {:timestamp=>"2016-09-29T11:10:05.559000+0200", :message=>"Parsed JSON object/hash requires a target configuration option", :source=>"message", :raw=>"", :level=>:warn}



## and finally: kibana also want´s to be updated:

If kibana says: "unable to fetch mapping", when you want to create an index, then you have to manually create an logstash-index in elasticsearch:
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

If the Settings/Indices Setup-Page has an empty __Time-field name__ dropdownbox, do these steps: http://stackoverflow.com/a/29535262/4964553


## Helpful Links

https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html

https://www.elastic.co/guide/en/elasticsearch/reference/1.4/cluster-nodes-shutdown.html

http://blog.florian-hopf.de/2015/02/fixing-elasticsearch-allocation-issues.html

http://blog.kiyanpro.com/2016/03/06/elasticsearch/reroute-unassigned-shards/

https://t37.net/how-to-fix-your-elasticsearch-cluster-stuck-in-initializing-shards-mode.html
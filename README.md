
# Elasticsearch: Time series data 

[Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/indices-aliases.html) and [Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.5/index.html) are used to simplify managing indices which need to be aged out, dropped or directed to specific infrastructure. All write operations will write to the index alias `log-write`. Curator will manage rotation of the alias and creation of new indices on the backend. All read operations will use the `log-read` alias.



### Index template

The alias `log-read` is applied to an [index template](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/indices-templates.html), all active `log-index-*` indices will be availible to it. Indices created following the index pattern will inherit the following definition. 

```json
curl -X PUT "localhost:9200/_template/log-index-template" -H 'Content-Type: application/json' -d'
{
    "index_patterns": ["log-index-*"],
    "settings" : {
        "number_of_shards" : 1
    },
    "aliases" : {
        "log-read": {}
    },
    "mappings" : {
        "log-mapping" : {
            "properties" : {
                "firstField" : {
                    "type" : "text" 
                },
                "secondField" : {
                    "type" : "text"
                },
                "thirdField" : {
                    "type" : "text"
                }           
            }
        }
    }
}
'
```
### Bootstrap

To prime Curator for managing the indicies the initial index needs creating and a `log-write` alias needs applying to it. Curator will then use the `log-write` alias to keep track of what it needs to rotate. This is a one-time operation

```
curl -X PUT "localhost:9200/log-index-2019_02_02_22_16"
```


Apply the `log-write` alias to the initial index

```json
curl -X POST "localhost:9200/_aliases" -H 'Content-Type: application/json' -d'
{
    "actions" : [
        { "add" : { "index" : "log-index-2019_02_02_22_16", "alias" : "log-write" } }
    ]
}
'
```

### Curator

The [rollover](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/rollover.html) action manages  mapping of the `log-write` alias for us. It creates a new index with its name derived from the [date math](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/date-math-index-names.html) function and a prefix which aligns with the template definition. The `log-write` alias is moved to the new index. The `log-write` alias is therefore not included in the template definition https://github.com/elastic/elasticsearch/pull/28110

```yaml
---
# elasticsearch templates are not managed by curator.
actions:
  1:
    action: rollover
    description: >-
      Rollover the index associated with alias 'name'.  Specify new index name using
      date math. Indicies older than the 'max_age' are rolled over. 
    options:
      name: log-write
      new_index: "<log-index-{now/m{YYYY_MM_dd_HH_mm}}>"
      conditions:
        max_age: 130s
        max_docs: 2
      wait_for_active_shards: 1
```


Running curator in a scheduled loop, the `log-write` alias is allways pointed to the latest index, while the `log-read` alias is available to all indices. 

![Alt text](output.png?raw=true "Title")

### Test Environment
Single node elasticsearch running in [Minikube](https://kubernetes.io/docs/setup/minikube/)
```
kubectl run elasticsearch --image=docker.elastic.co/elasticsearch/elasticsearch:6.4.1 --env="discovery.type=single-node" --port=9200
```
Proxy container traffic to localhost
```
while true do; kubectl port-forward <podname> 9200:9200; done
```
Curator was manually installed on the elasticsearch container from its Yum [repo](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/yum-repository.html), via a bash prompt to the container:
```
kubectl exec -it <podname> /bin/bash
```

Once installed the locale needs exporting (curator bug workaround)
```
export LC_ALL=en_US.UTF-8
```

---
layout: post
title: Elasticsearch - Replacing an index by alias.
---


### 1) create a new concrete index

`PUT myindex-1`


### 2) reindex the data from the old index to the new

```
  POST _reindex?slices=auto
  {
    "source": {"index": "myindex"},
    "dest": {"index": "myindex-1"}
  }
```


### 3) Follow the reindex task, it will disappear when done
```
  GET _tasks?actions=*reindex*
```
  
#### 3.1) alternatively you can also check the document count on both indices

```
  GET myindex/_count
  GET myindex-1/_count
```


### 4) When reindexing is done, remove the old index and replace it by an alias to the new index, all in one operation (no downtime)

```
  POST _aliases
  {
    "actions": [
      {
        "add": {
          "index": "myindex-1",
          "alias": "myindex"
        }
      },
      {
        "remove_index": {
          "index": "myindex"
        }
      }
    ]
  }
```



### 5) done, check that querying the old index name is now an alias that points to the new index

```
  GET myindex/_search
```
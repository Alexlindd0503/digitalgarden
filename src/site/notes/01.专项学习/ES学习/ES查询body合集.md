---
{"dg-publish":true,"permalink":"/01.专项学习/ES学习/ES查询body合集/","dg-note-properties":{"时间":"2026-03-23"}}
---

#ES #查询 #DSL #Elasticsearch

```ad-summary
title: 总结

- term 类查询用于精确匹配，bool 查询用于组合条件
- filter 和 must 都要求文档满足条件，区别是 filter 不打分，性能更好
- 聚合分指标聚合（avg/min/max/sum）和桶聚合（类似 group by）
```

## 1. term 类查询

### 精确查询

```json
{
  "query": {
    "term": {
      "jerseNo": "23"
    }
  }
}
```

### 非空查询

```json
{
  "query": {
    "exists": {
      "field": "teamNameEn"
    }
  }
}
```

### 前缀查询

```json
{
  "query": {
    "prefix": {
      "teamNameEn": "Rock"
    }
  }
}
```

### 通配符查询

`*` 匹配任意多个字符，`?` 匹配单个字符：
```json
{
  "query": {
    "wildcard": {
      "teamNameEn": "Ro*k"
    }
  }
}
```

### 正则查询

```json
{
  "query": {
    "regexp": {
      "teamNameEn": "Ro.*k"
    }
  }
}
```

### ids 查询

```json
{
  "query": {
    "ids": {
      "values": [1, 2]
    }
  }
}
```

## 2. 范围查询

### 数值范围

```json
{
  "query": {
    "range": {
      "playYear": {
        "gte": 2,
        "lt": 8
      }
    }
  }
}
```

### 日期范围

```json
{
  "query": {
    "range": {
      "birthDay": {
        "gte": "01/01/1999",
        "lt": "2022",
        "format": "dd/MM/yyyy||yyyy"
      }
    }
  }
}
```

## 3. bool 查询

bool 查询用来组合多个条件，四个子句：

| 子句 | 说明 |
|------|------|
| `must` | 必须满足，参与打分 |
| `filter` | 必须满足，不打分（性能更好，推荐用于过滤条件） |
| `must_not` | 不能满足 |
| `should` | 应该满足，不强制；配合 `minimum_should_match` 使用 |

### must

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "displayNameEn": "james"
          }
        }
      ]
    }
  }
}
```

### filter

```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "match": {
            "displayNameEn": "james"
          }
        }
      ]
    }
  }
}
```

### must + must_not + should 组合

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "displayNameEn": "james"
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "teamConferenceEn": {
              "value": "Eastern"
            }
          }
        }
      ],
      "should": [
        {
          "range": {
            "playYear": {
              "gte": 11,
              "lte": 20
            }
          }
        }
      ]
    }
  }
}
```

`should` 单独用时不强制匹配，配合 `minimum_should_match` 可以指定最少满足几个条件：
```json
{
  "query": {
    "bool": {
      "should": [...],
      "minimum_should_match": 2
    }
  }
}
```

## 4. 排序

多字段排序，先按 `playYear` 降序，再按 `heightValue` 升序：
```json
{
  "query": {
    "match": {
      "teamNameEn": "Rockets"
    }
  },
  "sort": [
    {
      "playYear": {
        "order": "desc"
      }
    },
    {
      "heightValue": {
        "order": "asc"
      }
    }
  ]
}
```

## 5. 聚合查询

### 指标聚合

`size: 0` 表示不返回文档，只返回聚合结果：
```json
{
  "query": {
    "match": {
      "teamNameEn": "Rockets"
    }
  },
  "aggs": {
    "avgAge": {
      "avg": {
        "field": "age"
      }
    }
  },
  "size": 0
}
```

支持的指标：`avg`、`min`、`max`、`sum`、`count`

### 桶聚合（group by）

按 `teamNameEn` 分组，统计每组数量：
```json
{
  "aggs": {
    "byTeam": {
      "terms": {
        "field": "teamNameEn"
      }
    }
  },
  "size": 0
}
```

桶聚合里还可以嵌套指标聚合，比如每个队的平均年龄：
```json
{
  "aggs": {
    "byTeam": {
      "terms": {
        "field": "teamNameEn"
      },
      "aggs": {
        "avgAge": {
          "avg": {
            "field": "age"
          }
        }
      }
    }
  },
  "size": 0
}
```

## 6. 面试相关

ES 的查询 DSL 是面试重点，如果你被问到查询优化，可以重点准备 [[01.专项学习/ES学习/ES面试题\|ES面试题]] 中的查询性能优化、filter vs must 区别等相关问题。

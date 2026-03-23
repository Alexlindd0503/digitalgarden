---
{"dg-publish":true,"permalink":"/01.专项学习/ES学习/ES的实战/","dg-note-properties":{"时间":"2026-03-23"}}
---

#ES #实战 #Bulk #查询

```ad-summary
title: 总结

- Bulk API 支持批量写入，比逐条写入快很多
- term 字段支持精确、前缀、通配符、正则、ids 等多种查询方式
- bool 查询组合 must/filter/must_not/should，filter 不打分性能更好
```

## 1. 批量导入数据

用 Bulk API 批量写入，详细使用见 [[01.专项学习/ES学习/文档的操作\|文档的操作]]：

```bash
curl -X POST "localhost:9200/_bulk" \
  -H 'Content-Type: application/json' \
  --data-binary @test.json
```

`@test.json` 是本地文件，Bulk 格式每两行一组（操作行 + 数据行）：
```json
{ "index": { "_index": "nba", "_id": "1" } }
{ "name": "哈登", "team_name": "火箭", "position": "得分后卫" }
{ "index": { "_index": "nba", "_id": "2" } }
{ "name": "库里", "team_name": "勇士", "position": "控球后卫" }
```

## 2. term 字段查询

详细 body 示例见 [[01.专项学习/ES学习/ES查询body合集\|ES查询body合集]]，这里汇总关键点：

- [[01.专项学习/ES学习/ES查询body合集#精确查询\|精确查询]]：完全匹配，对应 `keyword` 字段
- [[01.专项学习/ES学习/ES查询body合集#非空查询\|非空查询]]：字段存在且不为 null
- [[01.专项学习/ES学习/ES查询body合集#前缀查询\|前缀查询]]：字段值以指定前缀开头
- [[01.专项学习/ES学习/ES查询body合集#通配符查询\|通配符查询]]：`*` 多字符，`?` 单字符
- [[01.专项学习/ES学习/ES查询body合集#正则查询\|正则查询]]：正则表达式匹配
- [[01.专项学习/ES学习/ES查询body合集#ids 查询\|ids 查询]]：按文档 ID 批量查

## 3. 范围查询

- [[01.专项学习/ES学习/ES查询body合集#数值范围\|数值范围]]：`gte`、`gt`、`lte`、`lt`
- [[01.专项学习/ES学习/ES查询body合集#日期范围\|日期范围]]：支持自定义 format

## 4. bool 查询

- [[01.专项学习/ES学习/ES查询body合集#must\|must]]：必须满足，参与打分
- [[01.专项学习/ES学习/ES查询body合集#filter\|filter]]：必须满足，不打分，**过滤条件优先用 filter**
- [[01.专项学习/ES学习/ES查询body合集#must + must_not + should 组合\|must_not]]：不能满足
- [[01.专项学习/ES学习/ES查询body合集#must + must_not + should 组合\|should]]：应该满足，配合 `minimum_should_match` 使用

## 5. 排序与聚合

- [[01.专项学习/ES学习/ES查询body合集#排序\|sort 排序]]：支持多字段排序
- 指标聚合：[[01.专项学习/ES学习/ES查询body合集#指标聚合\|avg / min / max / sum]]
- 桶聚合：[[01.专项学习/ES学习/ES查询body合集#桶聚合（group by）\|terms 分组]]，类似 SQL 的 GROUP BY

## 6. 面试相关

ES 的实战问题在面试中很常见，如果你被问到批量写入或查询优化，可以重点准备 [[01.专项学习/ES学习/ES面试题\|ES面试题]] 中的写入性能优化、查询性能优化相关问题。

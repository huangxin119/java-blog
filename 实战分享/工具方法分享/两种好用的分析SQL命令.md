
[toc]

# 两种好用的分析SQL方法

工作中我们经常需要分析某些SQL的执行过程，方便了解SQL是否按照我们想要的那样去执行。下面给大家介绍两种分析SQL的好办法，建议
大家在分析SQL时优先采用explain命令，如果SQL较为复杂，可以采用optimizer_trace来进行分析。

## 简单易懂的explain命令

以以下表为例：

CREATE TABLE `t` (
`id` INT(11) NOT NULL,
`a` INT(11) DEFAULT NULL,
`b` INT(11) DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `a` (`a`)
)

需要分析SQL：

select * from t where a = 1 order by b desc limit 10;


### （1）使用方法

explain select * from t where a = 1 order by b desc limit 10;

### （2）看懂explain返回数据

返回数据如下(复杂的SQL会返回多条数据)：



      {
         "id": 1,
         "select_type": "SIMPLE",
         "table": "t",
         "partitions": null,
         "type": "ref",
         "possible_keys": "a",
         "key": "a",
         "key_len": "5",
         "ref": "const",
         "rows": ,
         "filtered": 100.00,
         "Extra": "Using index condition; Using filesort"
      }


**字段解析**

**id及其对应的table**----语句的序列号，当语句存在联表或者子查询时，会返回多条数据，SQL在执行时，优先执行id小的
SQL，id一样按照explain的顺序（自上而下）执行。

！！！在分析复杂的联表查询以及子查询时，一定要关注这两个字段，按照顺序逐步解析SQL执行过程。

**select_type**----查询操作类型（一般增删改操作力求简单，只有查询可能会比较复杂）

几种典型类型如下

--SIMPLE 简单查询，一般是那种没有联表以及子查询的简单SQL

--PRIMARY 主查询，子查询的最外层查询

--SUBQUERY 子查询

--UNION以及UNION RESULT UNION查询类型

--DERIVED 在FROM字段后续的子查询



**partitions**----分区表的命中情况，null代表该表不是分区表（分区表是MYSQL中的一个功能，可以通过按照某些字段进行分区，实现类似于分表的功能
，所以，为什么不直接分表呢）

**type**----访问类型，以下字段效率如下

```
NULL > system > const(直接取主键或者唯一键) > eq_ref（主键或唯一键联合查询） > ref（普通索引） > ref_or_null（普通索引含null） 
> index_merge（使用多个索引取交集或者并集） > range（范围查询） > index（遍历索引树） > ALL（全表）
```

**possible_keys**----SQL可能使用的索引

**key**----实际中用到的索引

**key_len**----索引字节长度

**ref**----使用索引比较的列值

**rows**----MYSQL估算需要扫描的行数，一般越少越好

**filtered**----经过索引筛选后，获取数据列占筛选后的列百分比，一般越高越好

**Extra**----一些MYSQL认为重要的额外信息

几种常见类型如下：

--Using filesort 进行了排序操作，如果排序的数据量较大性能会急剧下降。（MYSQL经常会为了避免排序选取一些排序字段作为索引）

--Using index 使用覆盖索引，不用回表了，这是MYSQL对你写SQL的表扬

--Using index condition 先使用索引，再过滤，再回表



### （3）1种更具体的explain format = json命令

输出内容如下，关注以下几个关键字段

```
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "1.20"  ##查询总成本
    },
    "ordering_operation": {
      "using_filesort": true,
      "table": {
        "table_name": "t",
        "access_type": "ref",
        "possible_keys": [
          "a"
        ],
        "key": "a",  ##使用索引a
        "used_key_parts": [
          "a"
        ],
        "key_length": "5",
        "ref": [
          "const"
        ],
        "rows_examined_per_scan": 1, ##扫描索引的行数
        "rows_produced_per_join": 1, ##扫描索引后满足其他各种条件的行数
        "filtered": "100.00",
        "index_condition": "(`test`.`t`.`a` <=> 1)",
        "cost_info": {
          "read_cost": "1.00", ##IO成本
          "eval_cost": "0.20", ##CPU成本
          "prefix_cost": "1.20",
          "data_read_per_join": "16"
        },
        "used_columns": [
          "id",
          "a",
          "b"
        ]
        ## 如果存在多种条件筛选，可能还有attached_condition属性表示这些附加筛选条件
      }
    }
  }
}
```

explain format = json最显著的是可以查看MYSQL估算的成本，以及执行过程。

### （4）explain的一些不足点

(1)rows不准确。rows是MYSQL通过对内存页进行抽样估算得出，是不准确的。

(2)SQL中的limit会被忽略。limit命令会被忽略（据说好像MYSQL8.0版本修复了，不确定？？？）

## optimizer_trace查看SQL执行具体轨迹

要查看MYSQL优化器具体的选取执行计划的执行过程，可以开启optimizer_trace查看SQL过程。

### （1）使用方法

```
SET optimizer_trace="enabled=on";
需要分析的SQL;
SET optimizer_trace="enabled=off";
select * from information_schema.OPTIMIZER_TRACE;
```
注意：有时候某些复杂的SQL的json数据太长被截断了，可以通过设置增大optimizer_trace_max_mem_size值来避免被截断。

### （2）看懂返回数据

返回数据如下


```json
{
  "steps": [
    //(1)SQL准备阶段，将通配符*改为具体字段
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `test`.`t`.`id` AS `id`,`test`.`t`.`a` AS `a`,`test`.`t`.`b` AS `b` from `test`.`t` where (`test`.`t`.`a` = 1) order by `test`.`t`.`b` desc limit 10"
          }
        ]
      }
    },
    //(2)SQL优化阶段，将通配符*改为具体字段
    {
      "join_optimization": {
        "select#": 1,  //代表select序号，如果是联表查询将会有多个序号
        "steps": [
          {
            //对条件语句的优化
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`test`.`t`.`a` = 1)",
              "steps": [
                {
                  "transformation": "equality_propagation",//等值条件句转换
                  "resulting_condition": "multiple equal(1, `test`.`t`.`a`)"
                },
                {
                  "transformation": "constant_propagation",//常量条件句转换
                  "resulting_condition": "multiple equal(1, `test`.`t`.`a`)"
                },
                {
                  "transformation": "trivial_condition_removal",//无效条件移除的转换
                  "resulting_condition": "multiple equal(1, `test`.`t`.`a`)"
                }
              ]
            }
          },
          {
            "substitute_generated_columns": {
            }
          },
          //表之间的相互依赖关系
          {
            "table_dependencies": [
              {
                "table": "`test`.`t`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          //显示可过滤字段
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`test`.`t`",
                "field": "a",
                "equals": "1",
                "null_rejecting": false
              }
            ]
          },
          //评估扫描成本
          {
            "rows_estimation": [
              {
                "table": "`test`.`t`",
                "range_analysis": {
                  //全表扫描
                  "table_scan": {
                    "rows": 100225,
                    "cost": 20272
                  },
                  //分析可用索引
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "a",
                      "usable": true,
                      "key_parts": [
                        "a",
                        "id"
                      ]
                    }
                  ],
                  //是否可以索引下推
                  "setup_range_conditions": [
                  ],
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  },
                  //分析各索引成本
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "a",
                        "ranges": [
                          "1 <= a <= 1"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": true
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    }
                  },
                  //选取最终索引
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "a",
                      "rows": 1,
                      "ranges": [
                        "1 <= a <= 1"
                      ]
                    },
                    "rows_for_plan": 1,
                    "cost_for_plan": 2.21,
                    "chosen": true
                  }
                }
              }
            ]
          },
          {
            //对比不同执行计划成本
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`test`.`t`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "a",
                      "rows": 1,
                      "cost": 1.2,
                      "chosen": true
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "a"
                      },
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`test`.`t`.`a` = 1)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`test`.`t`",
                  "attached": null
                }
              ]
            }
          },
          //DISTINCT、GROUP BY、ORDER BY语句的进一步优化。
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`test`.`t`.`b` desc",
              "items": [
                {
                  "item": "`test`.`t`.`b`"
                }
              ],
              "resulting_clause_is_simple": true,
              "resulting_clause": "`test`.`t`.`b` desc"
            }
          },
          {
            "added_back_ref_condition": "((`test`.`t`.`a` <=> 1))"
          },
          //最后优化后的结果
          {
            "refine_plan": [
              {
                "table": "`test`.`t`",
                "pushed_index_condition": "(`test`.`t`.`a` <=> 1)",
                "table_condition_attached": null
              }
            ]
          }
        ]
      }
    },
    {
      //（3）SQL执行阶段，主要是打印一些排序有关的信息
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "desc",
                "table": "`test`.`t`",
                "field": "b"
              }
            ],
            "filesort_priority_queue_optimization": {
              "limit": 10,
              "rows_estimate": 214577,
              "row_size": 18,
              "memory_available": 262144,
              "chosen": true
            },
            "filesort_execution": [
            ],
            "filesort_summary": {
              "rows": 1,
              "examined_rows": 1,
              "number_of_tmp_files": 0,
              "sort_buffer_size": 288,
              "sort_mode": "<sort_key, additional_fields>"
            }
          }
        ]
      }
    }
  ]
}

```

### （3）optimizer_trace局限性

（1）某些数据库不允许随意开启optimizer_trace配置。开启optimizer_trace会影响一点点性能。

（2）optimizer_trace没有explain直观，较为繁琐。

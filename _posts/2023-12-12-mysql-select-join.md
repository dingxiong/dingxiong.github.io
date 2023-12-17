---
title: Mysql select & join
date: 2023-12-12 11:52 -0800
categories: [database, mysql]
tags: [mysql, join]
---

QEP: query execution plan

## Join

### straight join

It is kind of option that forbid reordering tables when joining. According to
[official doc](https://dev.mysql.com/doc/refman/8.0/en/join.html),

> This can be used for those (few) cases for which the join optimizer processes
> the tables in a suboptimal order.

### Semi-join

TLDR: semi-join is a optimization step that transforms a sub-select into a
join.

This is a good blog about this topic
https://roylyseng.blogspot.com/2012/04/semi-join-in-mysql-56.html

The different semijoin strategy constants are defined
[here](https://github.com/mysql/mysql-server/blob/eb86b4016060d426858cc09873d12492f1be396e/sql/sql_const.h#L212).

Relevant code
[here](https://github.com/mysql/mysql-server/blob/9912892feecfa8f0450bb97b74ebaf37d16e375c/sql/sql_planner.cc#L4183-L4184).

### Choose table order

const table: table with zero or one row. See
[code](https://github.com/mysql/mysql-server/blob/2a57e948ca9b238262161ae854119f60c8fd347e/sql/sql_optimizer.cc#L5629).

Why const table is special? Because const table can be put at the most outside
order.

### system table

System tables are tables with exactly one row. See definition
[here](https://github.com/mysql/mysql-server/blob/83926c7fda58664b649f0731a973ad610985d36e/sql/dd_table_share.cc#L268).
It, together const table, is used for join optimization. Btw, `max_rows` and
`min_rows` are specified as table options when creating a table. Also, these
two options are not hard limit, and according to this
[post](https://bugs.mysql.com/bug.php?id=94651), it seems these two options are
legacy options. However, they seem play a role in table partition.

## Optimizer

## Optimizer tracer

Optimizer tracer tracks the optimization evidence and decision, and store the
result as a json object. The source code is scattered in these files:
[opt_trace.h](https://github.com/mysql/mysql-server/blob/44f859bda9930bf6a26c4fe94e2d8c0212bb26d1/sql/opt_trace.h#L779),
[opt_trace.cc](https://github.com/mysql/mysql-server/blob/44f859bda9930bf6a26c4fe94e2d8c0212bb26d1/sql/opt_trace.cc#L216),
[opt_trace_context.h](https://github.com/mysql/mysql-server/blob/b845ba26c825d8cf124b76c9738e88a9b0251eb0/sql/opt_trace_context.h#L382),
and
[opt_trace2server.cc](https://github.com/mysql/mysql-server/blob/b845ba26c825d8cf124b76c9738e88a9b0251eb0/sql/opt_trace2server.cc#L316).

The basic usage is in the
[class document](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_OPT_TRACE.html#INTRODUCTION).

```sql
SET SESSION OPTIMIZER_TRACE="enabled=on";
<sql>;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

The code is not hard to understand. It is essentially a json builder and a
linked list such that you know the indent level of current trace in the json
tree.

<details markdown="1">
<summary>example </summary>

```sql
mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE \G
*************************** 1. row ***************************
QUERY: select * from employees join dept_emp on dept_emp.emp_no = employees.emp_no limit 1
TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`emp_no` AS `emp_no`,`employees`.`birth_date` AS `birth_date`,`employees`.`first_name` AS `first_name`,`employees`.`last_name` AS `last_name`,`employees`.`gender` AS `gender`,`employees`.`hire_date` AS `hire_date`,`dept_emp`.`emp_no` AS `emp_no`,`dept_emp`.`dept_no` AS `dept_no`,`dept_emp`.`from_date` AS `from_date`,`dept_emp`.`to_date` AS `to_date` from (`employees` join `dept_emp` on((`dept_emp`.`emp_no` = `employees`.`emp_no`))) limit 1"
          },
          {
            "transformations_to_nested_joins": {
              "transformations": [
                "JOIN_condition_to_WHERE",
                "parenthesis_removal"
              ],
              "expanded_query": "/* select#1 */ select `employees`.`emp_no` AS `emp_no`,`employees`.`birth_date` AS `birth_date`,`employees`.`first_name` AS `first_name`,`employees`.`last_name` AS `last_name`,`employees`.`gender` AS `gender`,`employees`.`hire_date` AS `hire_date`,`dept_emp`.`emp_no` AS `emp_no`,`dept_emp`.`dept_no` AS `dept_no`,`dept_emp`.`from_date` AS `from_date`,`dept_emp`.`to_date` AS `to_date` from `employees` join `dept_emp` where (`dept_emp`.`emp_no` = `employees`.`emp_no`) limit 1"
            }
          }
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`dept_emp`.`emp_no` = `employees`.`emp_no`)",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "multiple equal(`dept_emp`.`emp_no`, `employees`.`emp_no`)"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "multiple equal(`dept_emp`.`emp_no`, `employees`.`emp_no`)"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "multiple equal(`dept_emp`.`emp_no`, `employees`.`emp_no`)"
                }
              ]
            }
          },
          {
            "substitute_generated_columns": {
            }
          },
          {
            "table_dependencies": [
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              },
              {
                "table": "`dept_emp`",
                "row_may_be_null": false,
                "map_bit": 1,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`employees`",
                "field": "emp_no",
                "equals": "`dept_emp`.`emp_no`",
                "null_rejecting": true
              },
              {
                "table": "`dept_emp`",
                "field": "emp_no",
                "equals": "`employees`.`emp_no`",
                "null_rejecting": true
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`employees`",
                "table_scan": {
                  "rows": 299423,
                  "cost": 929
                }
              },
              {
                "table": "`dept_emp`",
                "table_scan": {
                  "rows": 331143,
                  "cost": 737
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`employees`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "PRIMARY",
                      "usable": false,
                      "chosen": false
                    },
                    {
                      "rows_to_scan": 299423,
                      "filtering_effect": [
                      ],
                      "final_filtering_effect": 1,
                      "access_type": "scan",
                      "resulting_rows": 299423,
                      "cost": 30871.3,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 299423,
                "cost_for_plan": 30871.3,
                "rest_of_plan": [
                  {
                    "plan_prefix": [
                      "`employees`"
                    ],
                    "table": "`dept_emp`",
                    "best_access_path": {
                      "considered_access_paths": [
                        {
                          "access_type": "ref",
                          "index": "PRIMARY",
                          "rows": 1.10555,
                          "cost": 332526,
                          "chosen": true
                        },
                        {
                          "rows_to_scan": 331143,
                          "filtering_effect": [
                          ],
                          "final_filtering_effect": 1,
                          "access_type": "scan",
                          "using_join_cache": true,
                          "buffers_needed": 152,
                          "resulting_rows": 331143,
                          "cost": 9.9153e+09,
                          "chosen": false
                        }
                      ]
                    },
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 331028,
                    "cost_for_plan": 363397,
                    "chosen": true
                  }
                ]
              },
              {
                "plan_prefix": [
                ],
                "table": "`dept_emp`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "PRIMARY",
                      "usable": false,
                      "chosen": false
                    },
                    {
                      "rows_to_scan": 331143,
                      "filtering_effect": [
                      ],
                      "final_filtering_effect": 1,
                      "access_type": "scan",
                      "resulting_rows": 331143,
                      "cost": 33851.3,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 331143,
                "cost_for_plan": 33851.3,
                "rest_of_plan": [
                  {
                    "plan_prefix": [
                      "`dept_emp`"
                    ],
                    "table": "`employees`",
                    "best_access_path": {
                      "considered_access_paths": [
                        {
                          "access_type": "eq_ref",
                          "index": "PRIMARY",
                          "rows": 1,
                          "cost": 364257,
                          "chosen": true,
                          "cause": "clustered_pk_chosen_by_heuristics"
                        },
                        {
                          "rows_to_scan": 299423,
                          "filtering_effect": [
                          ],
                          "final_filtering_effect": 1,
                          "access_type": "scan",
                          "using_join_cache": true,
                          "buffers_needed": 33,
                          "resulting_rows": 299423,
                          "cost": 9.91521e+09,
                          "chosen": false
                        }
                      ]
                    },
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 331143,
                    "cost_for_plan": 398109,
                    "pruned_by_cost": true
                  }
                ]
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`dept_emp`.`emp_no` = `employees`.`emp_no`)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": null
                },
                {
                  "table": "`dept_emp`",
                  "attached": "(`dept_emp`.`emp_no` = `employees`.`emp_no`)"
                }
              ]
            }
          },
          {
            "finalizing_table_conditions": [
              {
                "table": "`dept_emp`",
                "original_table_condition": "(`dept_emp`.`emp_no` = `employees`.`emp_no`)",
                "final_table_condition   ": null
              }
            ]
          },
          {
            "refine_plan": [
              {
                "table": "`employees`"
              },
              {
                "table": "`dept_emp`"
              }
            ]
          }
        ]
      }
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ]
      }
    }
  ]
}
```

</details>

### How we get the row estimation

```
{
  "rows_estimation": [
    {
      "table": "`employees`",
      "table_scan": {
        "rows": 299423,
        "cost": 929
      }
    },
    {
      "table": "`dept_emp`",
      "table_scan": {
        "rows": 331143,
        "cost": 737
      }
    }
  ]
},

```

This tracer block is set at
[here](https://github.com/mysql/mysql-server/blob/2a57e948ca9b238262161ae854119f60c8fd347e/sql/sql_optimizer.cc#L5987).
Field `rows` comes from `stats.records`. See below.

```
mysql> SHOW TABLE STATUS like 'employees' \G
*************************** 1. row ***************************
           Name: employees
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 299379
 Avg_row_length: 50
    Data_length: 15220736
Max_data_length: 0
   Index_length: 0
      Data_free: 4194304
 Auto_increment: NULL
    Create_time: 2023-12-12 12:16:10
    Update_time: 2023-12-12 12:16:42
     Check_time: NULL
      Collation: utf8mb4_0900_ai_ci
       Checksum: NULL
 Create_options:
        Comment:
1 row in set (0.01 sec)
```

The cost part is more complicated. It calls function
[table_scan_cost](https://github.com/mysql/mysql-server/blob/83926c7fda58664b649f0731a973ad610985d36e/sql/handler.cc#L6081).
Basically, it is the product of `scan_time()` and `page_read_cost(1.0)`.
Function `scan_time()` returns the number of pages this table occupies. Note
this function is a virtual function. InnoDB overwrites the default
implementation. See
[code](https://github.com/mysql/mysql-server/blob/c12149baae15a972494a594f3eb9de2f9389a30e/storage/innobase/handler/ha_innodb.cc#L17169).
It returns `m_prebuilt->table->stat_clustered_index_size;`, i.e.,

```
mysql> select * from mysql.innodb_table_stats where table_name = 'employees' or table_name = 'dept_emp ';
+---------------+------------+---------------------+--------+----------------------+--------------------------+
| database_name | table_name | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
+---------------+------------+---------------------+--------+----------------------+--------------------------+
| employees     | dept_emp   | 2023-12-12 12:18:05 | 331143 |                  737 |                      353 |
| employees     | employees  | 2023-12-16 22:30:59 | 299423 |                  929 |                        0 |
+---------------+------------+---------------------+--------+----------------------+--------------------------+
```

Column `clustered_index_size` shows the number of disk pages this table
occupies. A side note: the row count is duplicated in `innodb_table_stats`
table and outcome of `show table status`.

Function `page_raed_cost(1.0)` returns the cost of reading one page. It is a
mix of cost of reading a page from memory and disk. Wait! Let's take a step
back: cost in what unit? It is unit-less. It is not milliseconds nor other time
unit. It is just a unit-less number to indicate the relative cost. See more
details from Mysql
[cost model](https://dev.mysql.com/doc/refman/8.0/en/cost-model.html). Below is
what I see in my localhost.

```
mysql> select * from mysql.server_cost;
+------------------------------+------------+---------------------+---------+---------------+
| cost_name                    | cost_value | last_update         | comment | default_value |
+------------------------------+------------+---------------------+---------+---------------+
| disk_temptable_create_cost   |       NULL | 2023-12-12 11:13:50 | NULL    |            20 |
| disk_temptable_row_cost      |       NULL | 2023-12-12 11:13:50 | NULL    |           0.5 |
| key_compare_cost             |       NULL | 2023-12-12 11:13:50 | NULL    |          0.05 |
| memory_temptable_create_cost |       NULL | 2023-12-12 11:13:50 | NULL    |             1 |
| memory_temptable_row_cost    |       NULL | 2023-12-12 11:13:50 | NULL    |           0.1 |
| row_evaluate_cost            |       NULL | 2023-12-12 11:13:50 | NULL    |           0.1 |
+------------------------------+------------+---------------------+---------+---------------+
6 rows in set (0.01 sec)

mysql> select * from mysql.engine_cost;
+-------------+-------------+------------------------+------------+---------------------+---------+---------------+
| engine_name | device_type | cost_name              | cost_value | last_update         | comment | default_value |
+-------------+-------------+------------------------+------------+---------------------+---------+---------------+
| default     |           0 | io_block_read_cost     |       NULL | 2023-12-12 11:13:50 | NULL    |             1 |
| default     |           0 | memory_block_read_cost |       NULL | 2023-12-12 11:13:50 | NULL    |          0.25 |
+-------------+-------------+------------------------+------------+---------------------+---------+---------------+
2 rows in set (0.01 sec)
```

You can see that `io_block_read_cost` is 4x more expensive than
`memory_block_read_cost`. In our case, Mysql has a fresh reboot, so nothing is
cached. `page_read_cost(1.0) = io_block_read_cost = 1`. This is how Mysql gets
the numbers for the `cost` field in the optimizer tracer output.

## Range optimization

A general introduction to this topic:
[official guide](https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html).
One thing to note is below sentense

> The optimizer attempts to use additional key parts to determine the interval
> as long as the comparison operator is =, <=>, or IS NULL. If the operator
> is >, <, >=, <=, !=, <>, BETWEEN, or LIKE, the optimizer uses it but
> considers no more key parts.

I did not know this. Previously, I thought Mysql will use as many prefixes as
possible.

### Skip scan

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

Questions:

1. what is the cost? how is it calculated. For const table, it is zero.
   - https://dev.mysql.com/doc/refman/8.0/en/cost-model.html
   - A few relevant source files:
     [opt_costconstants.h](https://github.com/mysql/mysql-server/blob/3fa9e4034d4dbdd596ecc25aaf87bb11ec56d5e3/sql/opt_costconstants.h#L201)
     [opt_costmodel.h](https://github.com/mysql/mysql-server/blob/3fa9e4034d4dbdd596ecc25aaf87bb11ec56d5e3/sql/opt_costmodel.h#L53).

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

example

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
                  "rows": 299379,
                  "cost": 315.608
                }
              },
              {
                "table": "`dept_emp`",
                "table_scan": {
                  "rows": 331143,
                  "cost": 735.488
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
                      "rows_to_scan": 299379,
                      "filtering_effect": [
                      ],
                      "final_filtering_effect": 1,
                      "access_type": "scan",
                      "resulting_rows": 299379,
                      "cost": 30253.5,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 299379,
                "cost_for_plan": 30253.5,
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
                          "cost": 331863,
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
                          "cost": 9.91384e+09,
                          "chosen": false
                        }
                      ]
                    },
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 330979,
                    "cost_for_plan": 362116,
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
                      "cost": 33849.8,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 331143,
                "cost_for_plan": 33849.8,
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
                          "cost": 145613,
                          "chosen": true,
                          "cause": "clustered_pk_chosen_by_heuristics"
                        },
                        {
                          "rows_to_scan": 299379,
                          "filtering_effect": [
                          ],
                          "final_filtering_effect": 1,
                          "access_type": "scan",
                          "using_join_cache": true,
                          "buffers_needed": 33,
                          "resulting_rows": 299379,
                          "cost": 9.91374e+09,
                          "chosen": false
                        }
                      ]
                    },
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 331143,
                    "cost_for_plan": 179463,
                    "chosen": true
                  }
                ]
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`emp_no` = `dept_emp`.`emp_no`)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`dept_emp`",
                  "attached": null
                },
                {
                  "table": "`employees`",
                  "attached": "(`employees`.`emp_no` = `dept_emp`.`emp_no`)"
                }
              ]
            }
          },
          {
            "finalizing_table_conditions": [
              {
                "table": "`employees`",
                "original_table_condition": "(`employees`.`emp_no` = `dept_emp`.`emp_no`)",
                "final_table_condition   ": null
              }
            ]
          },
          {
            "refine_plan": [
              {
                "table": "`dept_emp`"
              },
              {
                "table": "`employees`"
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

---
layout: post
title: Postgres -- Planner
date: 2024-09-07 14:13 -0700
categories: [database, postgres]
tags: [postgres]
---

Q1. What plan contains? How is it executed?

You know what? The part I love most about Postgres codebase is its comments.
Very informative and I learned a lot from them.

What does a query planner or optimizer do? It determines

1. Scan method
2. Join method
3. Join order

https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/optimizer/plan/planner.c#L279

https://www.postgresql.org/docs/16/custom-scan.html

Single table relation

1. Seq Scan
2. Bitmap Index Scan & Parallel Bitmap Heap Scan
3. Index Scan
4. Index Only Scan

Join

- innner
- full, left, right
- semi: a semi-join returns values from the left side of the relation that has
  a match with the right
- anti:

## Semi-join Optimization

Semi-join optimization is a step that converts top-level subqueries to joins.
For example

```
-- Create table my_table
CREATE TABLE my_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50)
);

-- Create table other_table
CREATE TABLE other_table (
    id INT PRIMARY KEY,
    description VARCHAR(100)
);

-- Insert sample data into my_table
INSERT INTO my_table (name) VALUES
('Alice'),
('Bob'),
('Charlie'),
('David');

-- Insert sample data into other_table
INSERT INTO other_table (id, description) VALUES
(1, 'Engineer'),
(2, 'Doctor'),
(3, 'Artist'),
(4, 'Teacher');
```

And a query

```
select t1.id, t1.name from my_table t1 where t1.id in (select id from other_table t2 where t2.id in (1, 3));
```

<details markdown="1">
<summary>Before optimization </summary>

```
   {QUERY
   :commandType 1
   :querySource 0
   :canSetTag true
   :utilityStmt <>
   :resultRelation 0
   :hasAggs false
   :hasWindowFuncs false
   :hasTargetSRFs false
   :hasSubLinks true
   :hasDistinctOn false
   :hasRecursive false
   :hasModifyingCTE false
   :hasForUpdate false
   :hasRowSecurity false
   :isReturn false
   :cteList <>
   :rtable (
      {RANGETBLENTRY
      :alias
         {ALIAS
         :aliasname t1
         :colnames <>
         }
      :eref
         {ALIAS
         :aliasname t1
         :colnames ("id" "name")
         }
      :rtekind 0
      :relid 24582
      :inh true
      :relkind r
      :rellockmode 1
      :perminfoindex 1
      :tablesample <>
      :lateral false
      :inFromCl true
      :securityQuals <>
      }
   )
   :rteperminfos (
      {RTEPERMISSIONINFO
      :relid 24582
      :inh true
      :requiredPerms 2
      :checkAsUser 0
      :selectedCols (b 8 9)
      :insertedCols (b)
      :updatedCols (b)
      }
   )
   :jointree
      {FROMEXPR
      :fromlist (
         {RANGETBLREF
         :rtindex 1
         }
      )
      :quals
         {SUBLINK
         :subLinkType 2
         :subLinkId 0
         :testexpr
            {OPEXPR
            :opno 96
            :opfuncid 65
            :opresulttype 16
            :opretset false
            :opcollid 0
            :inputcollid 0
            :args (
               {VAR
               :varno 1
               :varattno 1
               :vartype 23
               :vartypmod -1
               :varcollid 0
               :varnullingrels (b)
               :varlevelsup 0
               :varnosyn 1
               :varattnosyn 1
               :location 45
               }
               {PARAM
               :paramkind 2
               :paramid 1
               :paramtype 23
               :paramtypmod -1
               :paramcollid 0
               :location -1
               }
            )
            :location 51
            }
         :operName ("=")
         :subselect
            {QUERY
            :commandType 1
            :querySource 0
            :canSetTag true
            :utilityStmt <>
            :resultRelation 0
            :hasAggs false
            :hasWindowFuncs false
            :hasTargetSRFs false
            :hasSubLinks false
            :hasDistinctOn false
            :hasRecursive false
            :hasModifyingCTE false
            :hasForUpdate false
            :hasRowSecurity false
            :isReturn false
            :cteList <>
            :rtable (
               {RANGETBLENTRY
               :alias
                  {ALIAS
                  :aliasname t2
                  :colnames <>
                  }
               :eref
                  {ALIAS
                  :aliasname t2
                  :colnames ("id" "description")
                  }
               :rtekind 0
               :relid 24588
               :inh true
               :relkind r
               :rellockmode 1
               :perminfoindex 1
               :tablesample <>
               :lateral false
               :inFromCl true
               :securityQuals <>
               }
            )
            :rteperminfos (
               {RTEPERMISSIONINFO
               :relid 24588
               :inh true
               :requiredPerms 2
               :checkAsUser 0
               :selectedCols (b 8)
               :insertedCols (b)
               :updatedCols (b)
               }
            )
            :jointree
               {FROMEXPR
               :fromlist (
                  {RANGETBLREF
                  :rtindex 1
                  }
               )
               :quals
                  {SCALARARRAYOPEXPR
                  :opno 96
                  :opfuncid 65
                  :hashfuncid 0
                  :negfuncid 0
                  :useOr true
                  :inputcollid 0
                  :args (
                     {VAR
                     :varno 1
                     :varattno 1
                     :vartype 23
                     :vartypmod -1
                     :varcollid 0
                     :varnullingrels (b)
                     :varlevelsup 0
                     :varnosyn 1
                     :varattnosyn 1
                     :location 91
                     }
                     {ARRAYEXPR
                     :array_typeid 1007
                     :array_collid 0
                     :element_typeid 23
                     :elements (
                        {CONST
                        :consttype 23
                        :consttypmod -1
                        :constcollid 0
                        :constlen 4
                        :constbyval true
                        :constisnull false
                        :location 101
                        :constvalue 4 [ 1 0 0 0 0 0 0 0 ]
                        }
                        {CONST
                        :consttype 23
                        :consttypmod -1
                        :constcollid 0
                        :constlen 4
                        :constbyval true
                        :constisnull false
                        :location 104
                        :constvalue 4 [ 3 0 0 0 0 0 0 0 ]
                        }
                     )
                     :multidims false
                     :location -1
                     }
                  )
                  :location 97
                  }
               }
            :mergeActionList <>
            :mergeTargetRelation 0
            :mergeJoinCondition <>
            :targetList (
               {TARGETENTRY
               :expr
                  {VAR
                  :varno 1
                  :varattno 1
                  :vartype 23
                  :vartypmod -1
                  :varcollid 0
                  :varnullingrels (b)
                  :varlevelsup 0
                  :varnosyn 1
                  :varattnosyn 1
                  :location 62
                  }
               :resno 1
               :resname id
               :ressortgroupref 0
               :resorigtbl 24588
               :resorigcol 1
               :resjunk false
               }
            )
            :override 0
            :onConflict <>
            :returningList <>
            :groupClause <>
            :groupDistinct false
            :groupingSets <>
            :havingQual <>
            :windowClause <>
            :distinctClause <>
            :sortClause <>
            :limitOffset <>
            :limitCount <>
            :limitOption 0
            :rowMarks <>
            :setOperations <>
            :constraintDeps <>
            :withCheckOptions <>
            :stmt_location 0
            :stmt_len 0
            }
         :location 51
         }
      }
   :mergeActionList <>
   :mergeTargetRelation 0
   :mergeJoinCondition <>
   :targetList (
      {TARGETENTRY
      :expr
         {VAR
         :varno 1
         :varattno 1
         :vartype 23
         :vartypmod -1
         :varcollid 0
         :varnullingrels (b)
         :varlevelsup 0
         :varnosyn 1
         :varattnosyn 1
         :location 7
         }
      :resno 1
      :resname id
      :ressortgroupref 0
      :resorigtbl 24582
      :resorigcol 1
      :resjunk false
      }
      {TARGETENTRY
      :expr
         {VAR
         :varno 1
         :varattno 2
         :vartype 1043
         :vartypmod 54
         :varcollid 100
         :varnullingrels (b)
         :varlevelsup 0
         :varnosyn 1
         :varattnosyn 2
         :location 14
         }
      :resno 2
      :resname name
      :ressortgroupref 0
      :resorigtbl 24582
      :resorigcol 2
      :resjunk false
      }
   )
   :override 0
   :onConflict <>
   :returningList <>
   :groupClause <>
   :groupDistinct false
   :groupingSets <>
   :havingQual <>
   :windowClause <>
   :distinctClause <>
   :sortClause <>
   :limitOffset <>
   :limitCount <>
   :limitOption 0
   :rowMarks <>
   :setOperations <>
   :constraintDeps <>
   :withCheckOptions <>
   :stmt_location 0
   :stmt_len 107
   }
```

</details>

<details markdown="1">
<summary>After optimization </summary>

```
   {QUERY
   :commandType 1
   :querySource 0
   :canSetTag true
   :utilityStmt <>
   :resultRelation 0
   :hasAggs false
   :hasWindowFuncs false
   :hasTargetSRFs false
   :hasSubLinks true
   :hasDistinctOn false
   :hasRecursive false
   :hasModifyingCTE false
   :hasForUpdate false
   :hasRowSecurity false
   :isReturn false
   :cteList <>
   :rtable (
      {RANGETBLENTRY
      :alias
         {ALIAS
         :aliasname t1
         :colnames <>
         }
      :eref
         {ALIAS
         :aliasname t1
         :colnames ("id" "name")
         }
      :rtekind 0
      :relid 24582
      :inh false
      :relkind r
      :rellockmode 1
      :perminfoindex 1
      :tablesample <>
      :lateral false
      :inFromCl true
      :securityQuals <>
      }
      {RANGETBLENTRY
      :alias
         {ALIAS
         :aliasname ANY_subquery
         :colnames <>
         }
      :eref
         {ALIAS
         :aliasname ANY_subquery
         :colnames ("id")
         }
      :rtekind 1
      :subquery <>
      :security_barrier false
      :relid 0
      :inh false
      :relkind <>
      :rellockmode 0
      :perminfoindex 0
      :lateral false
      :inFromCl false
      :securityQuals <>
      }
      {RANGETBLENTRY
      :alias
         {ALIAS
         :aliasname t2
         :colnames <>
         }
      :eref
         {ALIAS
         :aliasname t2
         :colnames ("id" "description")
         }
      :rtekind 0
      :relid 24588
      :inh false
      :relkind r
      :rellockmode 1
      :perminfoindex 2
      :tablesample <>
      :lateral false
      :inFromCl true
      :securityQuals <>
      }
   )
   :rteperminfos (
      {RTEPERMISSIONINFO
      :relid 24582
      :inh true
      :requiredPerms 2
      :checkAsUser 0
      :selectedCols (b 8 9)
      :insertedCols (b)
      :updatedCols (b)
      }
      {RTEPERMISSIONINFO
      :relid 24588
      :inh true
      :requiredPerms 2
      :checkAsUser 0
      :selectedCols (b 8)
      :insertedCols (b)
      :updatedCols (b)
      }
   )
   :jointree
      {FROMEXPR
      :fromlist (
         {JOINEXPR
         :jointype 4
         :isNatural false
         :larg
            {FROMEXPR
            :fromlist (
               {RANGETBLREF
               :rtindex 1
               }
            )
            :quals <>
            }
         :rarg
            {FROMEXPR
            :fromlist (
               {RANGETBLREF
               :rtindex 3
               }
            )
            :quals (
               {SCALARARRAYOPEXPR
               :opno 96
               :opfuncid 65
               :hashfuncid 0
               :negfuncid 0
               :useOr true
               :inputcollid 0
               :args (
                  {VAR
                  :varno 3
                  :varattno 1
                  :vartype 23
                  :vartypmod -1
                  :varcollid 0
                  :varnullingrels (b)
                  :varlevelsup 0
                  :varnosyn 3
                  :varattnosyn 1
                  :location 91
                  }
                  {CONST
                  :consttype 1007
                  :consttypmod -1
                  :constcollid 0
                  :constlen -1
                  :constbyval false
                  :constisnull false
                  :location -1
                  :constvalue 32 [ -128 0 0 0 1 0 0 0 0 0 0 0 23 0 0 0 2 0 0 0
                   1 0 0 0 1 0 0 0 3 0 0 0 ]
                  }
               )
               :location 97
               }
            )
            }
         :usingClause <>
         :join_using_alias <>
         :quals (
            {OPEXPR
            :opno 96
            :opfuncid 65
            :opresulttype 16
            :opretset false
            :opcollid 0
            :inputcollid 0
            :args (
               {VAR
               :varno 1
               :varattno 1
               :vartype 23
               :vartypmod -1
               :varcollid 0
               :varnullingrels (b)
               :varlevelsup 0
               :varnosyn 1
               :varattnosyn 1
               :location 45
               }
               {VAR
               :varno 3
               :varattno 1
               :vartype 23
               :vartypmod -1
               :varcollid 0
               :varnullingrels (b)
               :varlevelsup 0
               :varnosyn 3
               :varattnosyn 1
               :location 62
               }
            )
            :location 51
            }
         )
         :alias <>
         :rtindex 0
         }
      )
      :quals <>
      }
   :mergeActionList <>
   :mergeTargetRelation 0
   :mergeJoinCondition <>
   :targetList (
      {TARGETENTRY
      :expr
         {VAR
         :varno 1
         :varattno 1
         :vartype 23
         :vartypmod -1
         :varcollid 0
         :varnullingrels (b)
         :varlevelsup 0
         :varnosyn 1
         :varattnosyn 1
         :location 7
         }
      :resno 1
      :resname id
      :ressortgroupref 0
      :resorigtbl 24582
      :resorigcol 1
      :resjunk false
      }
      {TARGETENTRY
      :expr
         {VAR
         :varno 1
         :varattno 2
         :vartype 1043
         :vartypmod 54
         :varcollid 100
         :varnullingrels (b)
         :varlevelsup 0
         :varnosyn 1
         :varattnosyn 2
         :location 14
         }
      :resno 2
      :resname name
      :ressortgroupref 0
      :resorigtbl 24582
      :resorigcol 2
      :resjunk false
      }
   )
   :override 0
   :onConflict <>
   :returningList <>
   :groupClause <>
   :groupDistinct false
   :groupingSets <>
   :havingQual <>
   :windowClause <>
   :distinctClause <>
   :sortClause <>
   :limitOffset <>
   :limitCount <>
   :limitOption 0
   :rowMarks <>
   :setOperations <>
   :constraintDeps <>
   :withCheckOptions <>
   :stmt_location 0
   :stmt_len 107
   }
```

</details>

## pg_hint_plan

According to installation guide,
https://github.com/ossc-db/pg_hint_plan/blob/master/docs/installation.md

```
$ tar xzvf pg_hint_plan-1.x.x.tar.gz
$ cd pg_hint_plan-1.x.x
```

Then we need to make some change to the Makefile.

```
# PG_CONFIG = pg_config
PG_CONFIG = ~/code/postgres/build_dir/install_dir/bin/pg_config
```

```
$ make
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -g -ggdb -Og -g3 -fno-omit-frame-pointer  -fvisibility=hidden -I. -I./ -I/Users/xiongding/code/postgres/build_dir/install_dir/include/server -I/Users/xiongding/code/postgres/build_dir/install_dir/include/internal  -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX14.5.sdk    -c -o pg_hint_plan.o pg_hint_plan.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -g -ggdb -Og -g3 -fno-omit-frame-pointer  -fvisibility=hidden pg_hint_plan.o -L/Users/xiongding/code/postgres/build_dir/install_dir/lib -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX14.5.sdk   -Wl,-dead_strip_dylibs   -fvisibility=hidden -bundle -bundle_loader /Users/xiongding/code/postgres/build_dir/install_dir/bin/postgres -o pg_hint_plan.dylib
```

```
$ make install
/opt/homebrew/bin/gmkdir -p '/Users/xiongding/code/postgres/build_dir/install_dir/share/extension'
/opt/homebrew/bin/gmkdir -p '/Users/xiongding/code/postgres/build_dir/install_dir/share/extension'
/opt/homebrew/bin/gmkdir -p '/Users/xiongding/code/postgres/build_dir/install_dir/lib'
/opt/homebrew/bin/ginstall -c -m 644 .//pg_hint_plan.control '/Users/xiongding/code/postgres/build_dir/install_dir/share/extension/'
/opt/homebrew/bin/ginstall -c -m 644 .//pg_hint_plan--1.3.0.sql .//pg_hint_plan--1.3.0--1.3.1.sql .//pg_hint_plan--1.3.1--1.3.2.sql .//pg_hint_plan--1.3.2--1.3.3.sql .//pg_hint_plan--1.3.3--1.3.4.sql .//pg_hint_plan--1.3.5--1.3.6.sql .//pg_hint_plan--1.3.4--1.3.5.sql .//pg_hint_plan--1.3.6--1.3.7.sql .//pg_hint_plan--1.3.7--1.3.8.sql .//pg_hint_plan--1.3.8--1.3.9.sql .//pg_hint_plan--1.3.9--1.3.10.sql .//pg_hint_plan--1.3.10--1.4.sql .//pg_hint_plan--1.4--1.4.1.sql .//pg_hint_plan--1.4.1--1.4.2.sql .//pg_hint_plan--1.4.2--1.4.3.sql .//pg_hint_plan--1.4.3--1.5.sql .//pg_hint_plan--1.5--1.5.1.sql .//pg_hint_plan--1.5.1--1.5.2.sql .//pg_hint_plan--1.5.2--1.6.0.sql .//pg_hint_plan--1.6.0--1.6.1.sql  '/Users/xiongding/code/postgres/build_dir/install_dir/share/extension/'
/opt/homebrew/bin/ginstall -c -m 755  pg_hint_plan.dylib '/Users/xiongding/code/postgres/build_dir/install_dir/lib/'
```

Then load it

```
postgres=# LOAD 'pg_hint_plan';
LOAD
```

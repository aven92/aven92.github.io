---
layout: post
title: PostgreSQL查看没有注释的表和字段
categories: PostgreSQL
---

<!--more-->

## 查看没有注释的表

    SELECT n.nspname as "Schema",
        c.relname as "Name",
        pg_catalog.obj_description(c.oid, 'pg_class') as "Description"
    FROM pg_catalog.pg_class c
    LEFT JOIN pg_catalog.pg_namespace n
    ON n.oid = c.relnamespace
    WHERE c.relkind IN ('r','')
        AND n.nspname <> 'pg_catalog'
        AND n.nspname <> 'information_schema'
        AND n.nspname !~ '^pg_toast'
        AND pg_catalog.pg_table_is_visible(c.oid)
        AND pg_catalog.obj_description(c.oid, 'pg_class') is null
    ORDER BY 1,2;

## 查看没有注释的字段

    SELECT b.nspname, b.relname, a.attname, pg_catalog.format_type(a.atttypid, a.atttypmod),
      pg_catalog.col_description(a.attrelid, a.attnum)
    FROM pg_catalog.pg_attribute a,
        (SELECT n.nspname, c.relname, c.oid
        FROM pg_catalog.pg_class c
        LEFT JOIN pg_catalog.pg_namespace n
        ON n.oid = c.relnamespace
        WHERE c.relkind IN ('r','')
            AND n.nspname <> 'pg_catalog'
            AND n.nspname <> 'information_schema'
            AND n.nspname !~ '^pg_toast'
            AND pg_catalog.pg_table_is_visible(c.oid)
        ) b
    WHERE a.attrelid = b.oid
        AND a.attnum > 0
        AND NOT a.attisdropped
        AND pg_catalog.col_description(a.attrelid, a.attnum) is null
    ORDER BY a.attrelid, a.attname;


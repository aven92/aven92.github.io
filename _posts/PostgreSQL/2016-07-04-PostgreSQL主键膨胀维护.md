---
layout: post
title: PostgreSQL主键膨胀维护
categories: PostgreSQL
---

<!--more-->

## 1. 维护前

    postgres=# \d+ test
                                                            Table
    "public.test"
      Column |            Type             |                     Modifiers
    | Storage | Stats target | Description
    --------+-----------------------------+----------------------------------------------------+---------+--------------+-------------
      id     | integer                     | not null default
    nextval('test_id_seq'::regclass) | plain   |              |
      val    | timestamp without time zone | default now()
    | plain   |              |
    Indexes:
         "test_pkey" PRIMARY KEY, btree (id)

## 2. 创建新的unique index

    postgres=# CREATE UNIQUE INDEX CONCURRENTLY ON test USING btree(id);
    CREATE INDEX

## 3. 替换主键

    postgres=# BEGIN;
    BEGIN
    postgres=# ALTER TABLE test DROP CONSTRAINT test_pkey;
    ALTER TABLE
    postgres=# ALTER TABLE test ADD CONSTRAINT test_id_idx PRIMARY KEY USING
    INDEX test_id_idx;
    ALTER TABLE
    postgres=# COMMIT;
    COMMIT

## 4. 维护后

    postgres=# \d+ test
                                                            Table
    "public.test"
      Column |            Type             |                     Modifiers
    | Storage | Stats target | Description
    --------+-----------------------------+----------------------------------------------------+---------+--------------+-------------
      id     | integer                     | not null default
    nextval('test_id_seq'::regclass) | plain   |              |
      val    | timestamp without time zone | default now()
    | plain   |              |
    Indexes:
         "test_id_idx" PRIMARY KEY, btree (id)

这样的好处是，可以在白天负载低的时候维护主键


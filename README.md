<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->
Superset
=========

[![Build Status](https://travis-ci.org/apache/incubator-superset.svg?branch=master)](https://travis-ci.org/apache/incubator-superset)
[![PyPI version](https://badge.fury.io/py/apache-superset.svg)](https://badge.fury.io/py/apache-superset)
[![Coverage Status](https://codecov.io/github/apache/incubator-superset/coverage.svg?branch=master)](https://codecov.io/github/apache/incubator-superset)
[![PyPI](https://img.shields.io/pypi/pyversions/apache-superset.svg?maxAge=2592000)](https://pypi.python.org/pypi/apache-superset)
[![Get on Slack](https://img.shields.io/badge/slack-join-orange.svg)](https://join.slack.com/t/apache-superset/shared_invite/enQtNDMxMDY5NjM4MDU0LWJmOTcxYjlhZTRhYmEyYTMzOWYxOWEwMjcwZDZiNWRiNDY2NDUwNzcwMDFhNzE1ZmMxZTZlZWY0ZTQ2MzMyNTU)
[![Documentation](https://img.shields.io/badge/docs-apache.org-blue.svg)](https://superset.incubator.apache.org)
[![dependencies Status](https://david-dm.org/apache/incubator-superset/status.svg?path=superset-frontend)](https://david-dm.org/apache/incubator-superset?path=superset-frontend)

<img
  src="https://cloud.githubusercontent.com/assets/130878/20946612/49a8a25c-bbc0-11e6-8314-10bef902af51.png"
  alt="Superset"
  width="500"
/>

A modern, enterprise-ready business intelligence web application.

[**Why Superset**](#why-superset) |
[**Database Support**](#database-support) |
[**Installation and Configuration**](#installation-and-configuration) |
[**Get Help**](#get-help) |
[**Contributor Guide**](#contributor-guide) |
[**Resources**](#resources) |
[**Superset Users**](#superset-users) |
[**License**](#license) |


## 我改了哪些地方？

### 1. SQL Lab 查看表字段注释

![显示表注释](https://github.com/aikuyun/superset/blob/master/table_commet.png)

用户希望在左侧筛选表的时候显示字段注释信息，目前只显示了`字段名`和`字段类型`。具体改造如下：

- 修改前端,对应文件：superset-frontend/src/SqlLab/components/ColumnElement.jsx，大概 64 行的位置开始
```js
return (
  <div className="clearfix table-column">
    <div className="pull-left m-l-10 col-name">
      {name}
      {icons}
        (<span className="text-muted">{col.type}</span>)
    </div>
    <div className="pull-right text-muted">
      <small>{col.comment}</small>
    </div>
  </div>
);
```

- 修改后端，对应文件: superset/views/database/api.py 中的 `get_table_metadata` 方法
> 对应 mysql,可以直接拿到 comment 字段，但是对应 hive 或者 presto 拿不到，所以改成 desc table 来取 comment 字段。

```python
def get_table_metadata(
    database: Database, table_name: str, schema_name: Optional[str]
) -> Dict:
    """
        Get table metadata information, including type, pk, fks.
        This function raises SQLAlchemyError when a schema is not found.


    :param database: The database model
    :param table_name: Table name
    :param schema_name: schema name
    :return: Dict table metadata ready for API response
    """
    keys: List = []
    columns = database.get_columns(table_name, schema_name)
    # define comment dict by tsl
    comment_dict = {}
    primary_key = database.get_pk_constraint(table_name, schema_name)
    if primary_key and primary_key.get("constrained_columns"):
        primary_key["column_names"] = primary_key.pop("constrained_columns")
        primary_key["type"] = "pk"
        keys += [primary_key]
    # get dialect name
    dialect_name = database.get_dialect().name
    if isinstance(dialect_name, bytes):
        dialect_name = dialect_name.decode()
    # get column comment, presto & hive
    if dialect_name == "presto" or dialect_name == "hive":
        db_engine_spec = database.db_engine_spec
        sql = ParsedQuery("desc {a}.{b}".format(a=schema_name, b=table_name)).stripped()
        engine = database.get_sqla_engine(schema_name)
        conn = engine.raw_connection()
        cursor = conn.cursor()
        query = Query()
        session = Session(bind=engine)
        query.executed_sql = sql
        query.__tablename__ = table_name
        session.commit()
        db_engine_spec.execute(cursor, sql, async_=False)
        data = db_engine_spec.fetch_data(cursor, query.limit)
        # parse list data into dict by tsl; hive and presto is different
        if dialect_name == "presto":
            for d in data:
                d[3]
                comment_dict[d[0]] = d[3]
        else:
            for d in data:
                d[2]
                comment_dict[d[0]] = d[2]
        conn.commit()

    foreign_keys = get_foreign_keys_metadata(database, table_name, schema_name)
    indexes = get_indexes_metadata(database, table_name, schema_name)
    keys += foreign_keys + indexes
    payload_columns: List[Dict] = []
    for col in columns:
        dtype = get_col_type(col)
        if len(comment_dict) > 0:
            payload_columns.append(
                {
                    "name": col["name"],
                    "type": dtype.split("(")[0] if "(" in dtype else dtype,
                    "longType": dtype,
                    "keys": [k for k in keys if col["name"] in k.get("column_names")],
                    "comment": comment_dict[col["name"]],
                }
            )
        elif dialect_name == "mysql":
            payload_columns.append(
                {
                    "name": col["name"],
                    "type": dtype.split("(")[0] if "(" in dtype else dtype,
                    "longType": dtype,
                    "keys": [k for k in keys if col["name"] in k.get("column_names")],
                    "comment": col["comment"],
                }
            )
        else:
            payload_columns.append(
                {
                    "name": col["name"],
                    "type": dtype.split("(")[0] if "(" in dtype else dtype,
                    "longType": dtype,
                    "keys": [k for k in keys if col["name"] in k.get("column_names")],
                    # "comment": col["comment"],
                }
            )
    return {
        "name": table_name,
        "columns": payload_columns,
        "selectStar": database.select_star(
            table_name,
            schema=schema_name,
            show_cols=True,
            indent=True,
            cols=columns,
            latest_partition=True,
        ),
        "primaryKey": primary_key,
        "foreignKeys": foreign_keys,
        "indexes": keys,
    }
```

### 修改过时方法

> 后台有一个报错信息：AttributeError: 'DataFrame' object has no attribute 'ix‘,原因是 pandas 在 1.0.0 之后，移除了 ix 方法

文件 superset/db_engine_specs/hive.py 中的 `_latest_partition_from_df` 方法

```python
@classmethod
  def _latest_partition_from_df(cls, df: pd.DataFrame) -> Optional[List[str]]:
      """Hive partitions look like ds={partition name}"""
      if not df.empty:
          return [df.iloc[:, 0].max().split("=")[1]]
      return None

```

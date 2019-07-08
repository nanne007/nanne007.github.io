---
title: "JDBC loadbalance 模式的小坑"
date: 2018-08-08T22:38:50+08:00
---

昨天在写代码插入数据到 tidb 的时候，时不时报错说 connection closed。一直没找到原因。
今天梳理代码，发现了这个坑。


用 jdbc 连接  tidb 的时候，

- 配置了 `jdbc:mysql:loadbalance` 模式（后面有多台 tidb，暂时还没有 load balance，只能先通过这种方式来搞）。
- 然后 jdbc 连接中没有显式的配置用哪个库。

这个时候，mysql client 底层在做 loadblance 的时候，会把之前 connection 的状态 sync 到新的 connection 上。这里面包括了：

- auto commit
- catalog
- isolation level
- max rows

``` java
  void syncSessionState(Connection source, Connection target, boolean readOnly) throws SQLException {
        if (target != null) {
            target.setReadOnly(readOnly);
        }

        if (source == null || target == null) {
            return;
        }

        boolean prevUseLocalSessionState = source.getUseLocalSessionState();
        source.setUseLocalSessionState(true);

        target.setAutoCommit(source.getAutoCommit());
        target.setCatalog(source.getCatalog());
        target.setTransactionIsolation(source.getTransactionIsolation());
        target.setSessionMaxRows(source.getSessionMaxRows());

        source.setUseLocalSessionState(prevUseLocalSessionState);
    }
```

需要注意了，连接配置中没有设置 catalog，所以会是空字符串。

`setCatalog` 拿到这个空字符串，会发送  __USE ``__  命令到 Server 端。

``` java
            StringBuilder query = new StringBuilder("USE ");
            query.append(StringUtils.quoteIdentifier(catalog, quotedId, getPedantic()));

            execSQL(null, query.toString(), -1, null, DEFAULT_RESULT_SET_TYPE, DEFAULT_RESULT_SET_CONCURRENCY, false, this.database, null, false);

            this.database = catalog;
```

TiDB（2.0.2） 对于这样的命令，会直接报错说找不到 database。

```
Server version: 5.7.10-TiDB-v2.0.2 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use ``;
ERROR 1049 (42000): Unknown database ''
```

``` go
func (e *SimpleExec) executeUse(s *ast.UseStmt) error {
	dbname := model.NewCIStr(s.DBName)
	dbinfo, exists := e.is.SchemaByName(dbname)
	if !exists {
		return infoschema.ErrDatabaseNotExists.GenByArgs(dbname)
	}
	e.ctx.GetSessionVars().CurrentDB = dbname.O
	// character_set_database is the character set used by the default database.
	// The server sets this variable whenever the default database changes.
	// See http://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_character_set_database
	sessionVars := e.ctx.GetSessionVars()
	terror.Log(errors.Trace(sessionVars.SetSystemVar(variable.CharsetDatabase, dbinfo.Charset)))
	terror.Log(errors.Trace(sessionVars.SetSystemVar(variable.CollationDatabase, dbinfo.Collate)))
	return nil
}
```

MySQL 会报错说 No database selected 。

```
Server version: 5.6.24-72.2-log Percona Server (GPL), Release 72.2, Revision 8d0f85b

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use ``;
ERROR 1046 (3D000): No database selected
```

在 mysql(tidb) 报错的情况下， load balance 就会 close 这个 connection，上层业务报错。

### 解决方法 ###

在 url 中，显式的指定库名，如 `jdbc:mysql:loadbalance/localhost:2379/testdb`

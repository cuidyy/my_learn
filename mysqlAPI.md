## 一、连接相关函数
1. **`mysql_init`**
   - 函数原型：`MYSQL *mysql_init(MYSQL *mysql)`
   - 功能：分配、初始化并返回新对象，用于连接MySQL服务器。如果`mysql`为`NULL`，则自动分配一个新对象；如果`mysql`不为`NULL`，则初始化该对象以便后续使用。
   - 返回值：成功返回指向`MYSQL`结构的指针，失败返回`NULL`。

2. **`mysql_real_connect`**
   - 函数原型：`MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, const char *db, unsigned int port, const char *unix_socket, unsigned long clientflag)`
   - 功能：建立与MySQL服务器的连接。`mysql`是之前初始化的`MYSQL`结构指针；`host`是服务器主机地址；`user`是用户名；`passwd`是密码；`db`是数据库名；`port`是端口号；`unix_socket`是Unix域套接字（在Windows下可设为`NULL`）；`clientflag`是客户端标志（一般设为`0`）。
   - 返回值：成功返回连接句柄（即`MYSQL`结构指针），失败返回`NULL`。

## 二、查询执行与结果获取函数
1. **`mysql_query`**
   - 函数原型：`int mysql_query(MYSQL *mysql, const char *query)`
   - 功能：执行SQL语句。`mysql`是连接句柄；`query`是要执行的SQL语句字符串。
   - 返回值：查询成功返回`0`，失败返回非`0`值。

2. **`mysql_store_result`**
   - 函数原型：`MYSQL_RES *mysql_store_result(MYSQL *mysql)`
   - 功能：从`mysql`对象中取出结果集。
   - 返回值：成功返回`MYSQL_RES`结构体指针，错误返回`NULL`。

## 三、结果集信息获取函数
1. **`mysql_num_fields`**
   - 函数原型：`unsigned int mysql_num_fields(MYSQL_RES *result)`
   - 功能：获取结果集中的列数。`result`是`mysql_store_result`的返回值。
   - 返回值：结果集中的列数。

2. **`mysql_fetch_fields`**
   - 函数原型：`MYSQL_FIELD *mysql_fetch_fields(MYSQL_RES *result)`
   - 功能：返回结果集中所有列的名字。
   - 返回值：返回`MYSQL_FIELD*`结构体指针，指向一个结构体数组，每个结构体包含列名等信息。

3. **`mysql_fetch_lengths`**
   - 函数原型：`unsigned long *mysql_fetch_lengths(MYSQL_RES *result)`
   - 功能：返回结果集内当前行各列的长度。
   - 返回值：返回无符号长整数数组，错误返回`NULL`。

## 四、结果集遍历函数
**`mysql_fetch_row`**
   - 函数原型：`MYSQL_ROW mysql_fetch_row(MYSQL_RES *result)`
   - 功能：遍历结果集的下一行。
   - 返回值：成功返回`MYSQL_ROW`（`char**`类型），指向一个指针数组，每个元素是`char*`类型，指向一列数据，失败返回`NULL`。

## 五、资源回收函数
1. **`mysql_free_result`**
   - 函数原型：`void mysql_free_result(MYSQL_RES *result)`
   - 功能：释放结果集。
   - 返回值：无。

2. **`mysql_close`**
   - 函数原型：`void mysql_close(MYSQL *mysql)`
   - 功能：关闭`mysql`实例。
   - 返回值：无。

## 六、事务操作函数
1. **`mysql_autocommit`**
   - 函数原型：`int mysql_autocommit(MYSQL *mysql, int mode)`
   - 功能：设置事务提交模式。`mysql`是连接句柄；`mode`为`0`表示禁用自动提交，非`0`表示启用自动提交。
   - 返回值：成功返回`0`，失败返回非`0`。

2. **`mysql_commit`**
   - 函数原型：`int mysql_commit(MYSQL *mysql)`
   - 功能：提交事务。
   - 返回值：成功返回`0`，失败返回非`0`。

3. **`mysql_rollback`**
   - 函数原型：`int mysql_rollback(MYSQL *mysql)`
   - 功能：数据回滚。
   - 返回值：成功返回`0`，失败返回非`0`。

## 七、错误处理函数
1. **`mysql_error`**
   - 函数原型：`const char *mysql_error(MYSQL *mysql)`
   - 功能：返回错误描述。
   - 返回值：错误描述字符串。

2. **`mysql_errno`**
   - 函数原型：`unsigned int mysql_errno(MYSQL *mysql)`
   - 功能：返回错误编号。
   - 返回值：错误编号。
title: Sqlite相关优化
date: 2017/10/12 13:44:00
author: 胡玮
tags:
- Android 
- Sqlite3
categories: 
- Sqlite相关优化
---
本文主要主要讲解了android上sqlite3的一些优化方案，总结出了常见的优化措施。通过策略的优化和sqlite３查询器的自身的优化，让你更好地使用和做好一些优化措施。
<!--more-->
## 一、策略优化
### 建立索引
提高查找性能

#### 创建索引的基本语法如下
```sql
CREATE INDEX index_name ON table_name;
```
#### 单列索引
```sql
CREATE INDEX index_name ON table_name (column_name);
```
#### 唯一索引
使用唯一索引不仅是为了性能，同时也为了数据的完整性。唯一索引不允许任何重复的值插入到表中。基本语法如下：
```sql
CREATE UNIQUE INDEX index_name
on table_name (column_name);
```
#### 组合索引
组合索引是基于一个表的两个或多个列上创建的索引。基本语法如下：
```sql
CREATE INDEX index_name
on table_name (column1, column2);
```
#### 隐式索引
隐式索引是在创建对象时，由数据库服务器自动创建的索引。索引自动创建为主键约束和唯一约束。
　
#### 测试重点
弄清楚不同数据量情况下建议索引和未建立索引的情况下的查找性能

#### 什么情况下要避免使用索引？
虽然索引的目的在于提高数据库的性能，但这里有几个情况需要避免使用索引。使用索引时，应重新考虑下列准则：
索引不应该使用在较小的表上。
索引不应该使用在有频繁的大批量的更新或插入操作的表上。
索引不应该使用在含有大量的 NULL 值的列上。
索引不应该使用在频繁操作的列上。

### 编译SQL语句
SQLite想要执行操作，需要将程序中的sql语句编译成对应的SQLiteStatement，比如select * from record这一句，被执行100次就需要编译100次。对于批量处理插入或者更新的操作，我们可以使用显式编译来做到重用SQLiteStatement。

想要做到重用SQLiteStatement也比较简单，基本如下：

编译sql语句获得SQLiteStatement对象，参数使用?代替
在循环中对SQLiteStatement对象进行具体数据绑定，bind方法中的index从1开始，不是0。
请参考如下简单的使用代码

```java
private void insertWithPreCompiledStatement(SQLiteDatabase db) {
    String sql = "INSERT INTO " + TableDefine.TABLE_RECORD + "( " + TableDefine.COLUMN_INSERT_TIME + ") VALUES(?)";
    SQLiteStatement  statement = db.compileStatement(sql);
    int count = 0;
    while (count < 100) {
        count++;
        statement.clearBindings();
        statement.bindLong(1, System.currentTimeMillis());
        statement.executeInsert();
    }
}
```

### 显式使用事务
在Android中，无论是使用SQLiteDatabase的insert,delete等方法还是execSQL都开启了事务，来确保每一次操作都具有原子性，使得结果要么是操作之后的正确结果，要么是操作之前的结果。

然而事务的实现是依赖于名为rollback journal文件，借助这个临时文件来完成原子操作和回滚功能。既然属于文件，就符合Unix的文件范型(Open-Read/Write-Close)，因而对于批量的修改操作会出现反复打开文件读写再关闭的操作。然而好在，我们可以显式使用事务，将批量的数据库更新带来的journal文件打开关闭降低到1次。

具体的实现代码如下：
```java
private void insertWithTransaction(SQLiteDatabase db) {
    int count = 0;
    ContentValues values = new ContentValues();
    try {
        db.beginTransaction();
        while (count++ < 100) {
            values.put(TableDefine.COLUMN_INSERT_TIME, System.currentTimeMillis());
            db.insert(TableDefine.TABLE_RECORD, null, values);
        }
          db.setTransactionSuccessful();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        db.endTransaction();
    }
}
```

### 提前获取列索引
当我们需要遍历cursor时，通常的做法是这样
```java
private void badQueryWithLoop(SQLiteDatabase db) {
    Cursor cursor = db.query(TableDefine.TABLE_RECORD, new String[]{TableDefine.COLUMN_INSERT_TIME}, null, null, null, null, null) ;
    while (cursor.moveToNext()) {
        long insertTime = cursor.getLong(cursor.getColumnIndex(TableDefine.COLUMN_INSERT_TIME));
    }
}
```
但是如果我们将获取ColumnIndex的操作提到循环之外，效果会更好一些，修改后的代码如下：
```java
private void goodQueryWithLoop(SQLiteDatabase db) {
    Cursor cursor = db.query(TableDefine.TABLE_RECORD, new String[]{TableDefine.COLUMN_INSERT_TIME}, null, null, null, null, null) ;
    int insertTimeColumnIndex = cursor.getColumnIndex(TableDefine.COLUMN_INSERT_TIME);
    while (cursor.moveToNext()) {
        long insertTime = cursor.getLong(insertTimeColumnIndex);
    }
    cursor.close();
}
```

### 用左连接代替子查询
连接查询的效率比子查询的效率要高

### ContentValues的容量调整(影响不大)
SQLiteDatabase提供了方便的ContentValues简化了我们处理列名与值的映射，ContentValues内部采用了HashMap来存储Key-Value数据，ContentValues的初始容量是8，如果当添加的数据超过8之前，则会进行双倍扩容操作，因此建议对ContentValues填入的内容进行估量，设置合理的初始化容量，减少不必要的内部扩容操作。

## 二，SQLITE查询计划器和优化
### 1.0 WHERE条件分析
查询中的WHERE子句被分解为“条件”，其中每个条件由AND运算符与其他运算符分隔。如果条件由or连接，索引将不可用。
为了使用索引，条件必须是以下形式之一：
- column = 表达式 
- 列 IS 表达式 
- 列 > 表达式 
- 列 > = 表达式 
- 列 < 表达式 
- 列 <= 表达式 
- 表达式 = 列 
- 表达式 > 列 
- 表达式 > = 列 
- 表达式 < column 
- 表达式 <= 列 
- 列 IN（ 表达式列表 ） 
- 列 IN（ 子查询 ） 
- 列 IS NULL 

如果索引的初始列（列a，b等）出现在WHERE子句中，则可以使用索引。 索引的初始列必须与=或IN或IS运算符一起使用。 使用最右边的列可以使用不等式。 对于所使用的索引的最右边的列，最多可以有两个不等式，必须将列的允许值夹在两个极限之间。

### 索引术语使用示例
对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND c is NULL AND d ='hello'
索引的前四列a，b，c和d将是可用的，因为这四列形成索引的前缀，并且都被等式约束所约束。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND c> 12 AND d ='hello'
只有索引的列a，b和c才可用。 d列不可用，因为它出现在c的右侧，c只受不平等约束。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND d ='hello'
只有索引的列a和b才可用。 d列不可用，因为列c不受约束，索引可用的列集合中没有间隙。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE b IN（1,2,3）AND c NOT NULL AND d ='hello'
索引根本不可用，因为索引（列“a”）的最左边的列不受约束。 假设没有其他索引，上面的查询将导致完整的表扫描。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 OR b IN（1,2,3）OR c NOT NULL或d ='hello'
该索引不可用，因为WHERE子句用OR而对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND c is NULL AND d ='hello'
索引的前四列a，b，c和d将是可用的，因为这四列形成索引的前缀，并且都被等式约束所约束。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND c> 12 AND d ='hello'
只有索引的列a，b和c才可用。 d列不可用，因为它出现在c的右侧，c只受不平等约束。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 AND b IN（1,2,3）AND d ='hello'
只有索引的列a和b才可用。 d列不可用，因为列c不受约束，索引可用的列集合中没有间隙。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE b IN（1,2,3）AND c NOT NULL AND d ='hello'
索引根本不可用，因为索引（列“a”）的最左边的列不受约束。 假设没有其他索引，上面的查询将导致完整的表扫描。

对于上面的索引和WHERE子句，如下所示：

   ... WHERE a = 5 OR b IN（1,2,3）OR c NOT NULL或d ='hello'
该索引不可用，因为WHERE子句用OR而不是AND连接。 此查询将导致完整的表扫描。 但是，如果添加的三个附加索引包含列b，c和d作为其最左边的列，则OR子句优化可能适用。

不是AND连接。 此查询将导致完整的表扫描。 但是，如果添加的三个附加索引包含列b，c和d作为其最左边的列，则OR子句优化可能适用。

### 2.0 BETWEEN优化
 如果WHERE子句的条件具有以下形式：
 
    expr1 BETWEEN expr2 和 expr3 
可以添加两个“虚拟”条件如下：

    expr1 > = expr2 AND expr1 <= expr3 
虚拟术语仅用于分析，不会导致生成任何VDBE代码。 如果两个虚拟条件最终被用作索引的约束，那么省略原始的BETWEEN项，并且不对输入行执行相应的分析。 因此，如果BETWEEN术语最终用作索引约束，则不会对该术语进行任何分析。 另一方面，虚拟条件本身不会导致对输入行执行分析。 因此，如果BETWEEN项不用作索引约束，而是必须用于分析输入行，则只会对expr1表达式进行一次评估。

### 3.0 OR优化
由OR而不是AND连接的WHERE子句约束可以通过两种不同的方式处理。 如果条件由包含公共列名称并由OR分隔的多个子句组成，如下所示：

    column = expr1 OR column = expr2 OR column = expr3 OR ... 
那么这个条件重写如下：

    列 IN（ expr1 ， expr2 ， expr3 ，...） 
重写的条件可能会继续使用IN运算符的常规规则约束索引。 请注意，每个OR连接子列中的列必须是相同的列，尽管该列可以发生在=运算符的左侧或右侧。

当且仅当先前描述的将OR转换为IN运算符不起作用时，才尝试进行第二个OR子句优化。 假设OR子句由多个子句组成，如下所示：


    expr1 OR expr2 OR expr3 
单个子项可能是单个比较表达式，如a = 5 OR x> y，或者它们可以是LIKE或BETWEEN表达式，或者子项可以是AND连接的子子项的括号列表。 分析每个子项，就好像它本身是整个WHERE子句一样，以查看子项是否可以自己索引。 如果一个OR子句的每个子项都是可分离的，那么OR子句可能被编码，以便使用单独的索引来评估OR子句的每个项。 考虑SQLite如何为每个OR子句使用单独的索引的一种方法是想象WHERE子句重写如下：

	rowid IN（SELECT rowid FROM table WHERE expr1 UNION SELECT rowid FROM table WHERE expr2 UNION SELECT rowid FROM table WHERE expr3 ）
  
上述重写的概念是概念性的; 包含OR的WHERE子句不会以这种方式进行重写。 OR子句的实际实现使用一种更有效的机制，即使对于“rowid”无法访问的WITHOUT ROWID表或表，它也可以正常工作。 但实现的本质是由上述语句捕获的：使用独立索引来从每个OR子句项中查找候选结果行，最终结果是这些行的并集。

请注意，在大多数情况下，SQLite只会对查询的FROM子句中的每个表使用单个索引。 这里描述的第二个OR子句优化是该规则的例外。 使用OR子句，OR子句中的每个子项可能使用不同的索引。

对于任何给定的查询，可以使用这里描述的OR子句优化的事实并不保证将被使用。 SQLite使用基于成本的查询计划器来估计各种竞争查询计划的CPU和磁盘I / O成本，并选择其认为最快的计划。 如果WHERE子句中有多个OR术语，或者单个OR子句子句中的某些索引不是很有选择性，那么SQLite可能会决定使用不同的查询算法甚至全表扫描更快。 应用程序开发人员可以在语句上使用EXPLAIN QUERY PLAN前缀来获取所选查询策略的高级概述。

### 4.0 LIKE优化
使用LIKE或GLOB运算符的WHERE子句术语有时可用于索引进行范围搜索，几乎就像LIKE或GLOB替代BETWEEN运算符一样。 这个优化有很多条件：

LIKE或GLOB的右侧必须是字符串文字或绑定到不以通配符开头的字符串文字的参数 。
ESCAPE子句不能出现在LIKE运算符上。
通过在左侧具有数值（而不是字符串或blob），不可能使LIKE或GLOB操作符为真。 这意味着：
LIKE或GLOB运算符的左侧是具有TEXT关联性的索引列的名称，或
右侧模式参数不以减号（“ - ”）或数字开头。
这个约束来自于数字不按字典顺序排列的事实。 例如：9 <10，'9'>'10'。
用于实现LIKE和GLOB的内置函数不能使用sqlite3_create_function（）API重载。
对于GLOB运算符，列必须使用内置的BINARY整理顺序进行索引。description
对于LIKE运算符，如果启用了case_sensitive_like模式，则列必须使用BINARY整理顺序进行索引，或者如果禁用了case_sensitive_like模式，则列必须使用内置的NOCASE顺序进行索引。
LIKE运算符有两种可以通过编译指令设置的模式。 默认模式是LIKE比较对于latin1字符的大小写区别不敏感。 因此，默认情况下，以下表达式为真：

   'a'LIKE'A'
但是如果启用了case_sensitive_like语法，则如下所示：

   PRAGMA case_sensitive_like = ON;
然后，LIKE运算符注意情况，上面的示例将判断为false。 请注意，case insensitivity仅适用于latin1字符 - 基本上是ASCII的较低127字节代码中的英文大写和小写字母。 国际字符集在SQLite中区分大小写，除非提供了应用程序定义的整理序列和类似（）SQL函数 ，以考虑非ASCII字符。 但是，如果提供了应用程序定义的整理顺序和/或类似（）SQL函数，则不会采用此处描述的LIKE优化。

默认情况下，LIKE运算符不区分大小写，因为这是SQL标准所要求的。 您可以使用编译器的SQLITE_CASE_SENSITIVE_LIKE命令行选项来更改编译时的默认行为。

如果使用内置的BINARY整理顺序对运算符左侧命名的列进行索引，并且启用了case_sensitive_like，则可能会出现LIKE优化。 或者如果列使用内置的NOCASE整理顺序进行索引，并且case_sensitive_like模式关闭，则可能会发生优化。 这些是LIKE操作员将被优化的唯一两种组合。

GLOB运算符总是区分大小写。 GLOB运算符左侧的列必须始终使用内置的BINARY整理顺序，否则不会尝试使用索引优化该运算符。

只有当GLOB或LIKE操作符的右侧是文字字符串或绑定到字符串文字的参数时 ，才会尝试LIKE优化。 字符串文字不能以通配符开头; 如果右侧以通配符开头，则尝试进行此优化。 如果右侧是一个绑定到字符串的参数 ，则只有使用sqlite3_prepare_v2（）或sqlite3_prepare16_v2（）编译了包含表达式的准备语句时 ，才会尝试进行此优化。 如果右侧是参数并且语句是使用sqlite3_prepare（）或sqlite3_prepare16（）准备的），则不尝试LIKE优化。 如果LIKE运算符上有ESCAPE短语，则不尝试LIKE优化。

通用前缀优化：假设LIKE或GLOB运算符右侧的非通配符的初始序列为x 。 我们使用单个字符来表示这个非通配符前缀，但是读者应该明白前缀可以由超过1个字符组成。 让y是与/ x /相同长度的最小字符串，但是比较大于x 。 例如，如果x是hello，那么y将是hellp 。 LIKE和GLOB优化包括添加两个虚拟术语：


    列 > = x AND 列 < y 
在大多数情况下，即使虚拟术语用于约束索引，原始LIKE或GLOB运算符仍然针对每个输入行进行评估。 这是因为我们不知道字符在x前缀的右边可能会施加什么额外的限制。 但是，如果在x右侧只有一个全局通配符，则原始的LIKE或GLOB测试将被禁用。 换句话说，如果模式是这样的：


    列 LIKE x ％ 
    列 GLOB x * 
 
那么当虚拟术语约束索引时，原始LIKE或GLOB测试被禁用，因为在这种情况下，我们知道索引选择的所有行将通过LIKE或GLOB测试。

请注意，当LIKE或GLOB运算符的右侧是参数并且使用sqlite3_prepare_v2（）或sqlite3_prepare16_v2（）准备语句时，则会在每次运行的第一个sqlite3_step（）调用时自动对该语句进行重新编译并重新编译，如果绑定到右侧参数自上一次运行以来已更改。 这种重新排序和重新编译本质上是在模式更改之后发生的相同操作。 重新编译是必要的，因此查询计划器可以检查绑定到LIKE或GLOB运算符右侧的新值，并确定是否使用上述优化。

### 5.0 跳过扫描优化
一般规则是，只有在索引的最左边列有WHERE-clause约束的情况下，索引才有用。 然而，在某些情况下，即使从WHERE子句中省略了索引的前几列，但是包含了后面的列，SQLite也可以使用索引。

考虑下列表格：

  CREATE TABLE people(
    name TEXT PRIMARY KEY,
    role TEXT NOT NULL,
    height INT NOT NULL, -- in cm
    CHECK( role IN ('student','teacher') )
  );
  CREATE INDEX people_idx1 ON people(role, height);

人员表中有一个大型组织中的每个人的入口。 每个人都是“学生”或“老师”，由“角色”领域决定。 我们记录每个人的厘米的高度。 作用和高度被索引。 请注意，索引的最左边的列不是很有选择性 - 它只包含两个可能的值。

现在考虑一个查询来查找组织中每个人的名字，这个名字是180厘米高或者更高：

   SELECT name FROM people WHERE height> = 180;
因为索引的最左边的列没有出现在查询的WHERE子句中，所以有人认为索引在这里是不可用的。 但是SQLite能够使用索引。 在概念上，SQLite使用索引，就像查询更像以下内容一样：

   SELECT name FROM people
    WHERE ROLE IN（SELECT DISTINCT ROLE FROM PEOPLE）
      AND height> = 180;
或这个：

   SELECT name FROM people WHERE role ='teacher'AND height> = 180
   UNION ALL
   SELECT name FROM people WHERE role ='student'AND height> = 180;
上面显示的替代查询方案只是概念性的。 SQLite不会真正转换查询。 实际的查询计划是这样的：SQLite定位“角色”的第一个可能的值，它可以通过将“people_idx1”索引倒回到开头并读取第一个记录来执行。 SQLite将这个第一个“角色”值存储在一个内部变量中，我们将在此称为“$ role”。 那么SQLite运行一个查询，如：“SELECT name FROM people WHERE role = $ role AND height> = 180”。 该查询在索引的最左侧列具有等式约束，因此索引可用于解析该查询。 一旦该查询完成，SQLite然后使用“people_idx1”索引来定位“role”列的下一个值，使用逻辑上类似于“SELECT ROLE FROM WHEREROLE > $ role LIMIT 1“的代码。 这个新的“ROLE”值将覆盖$角色变量，并且重复此过程，直到“ROLE”的所有可能值都被检查为止。

我们将这种索引使用称为“跳过扫描”，因为数据库引擎基本上是对索引进行全面扫描，但是通过偶尔跳过下一个候选值来优化扫描（使其小于“完整”）。

如果SQLite知道第一个或多个列包含许多重复值，SQLite可能会在索引上使用跳过扫描。 如果在索引的最左侧列中的重复次数太少，那么进行全表扫描比在索引上进行二进制搜索来定位下一个左列值更快。

SQLite可以知道索引的最左侧列有许多重复的唯一方法是如果ANALYZE命令已在数据库上运行。 没有ANALYZE的结果，SQLite必须猜测表中数据的“形状”，默认的猜测是索引最左侧列中的每个值平均有10个重复。 但是，当重复数量大约为18或更多时，跳过扫描只会变得有利的（它只能比全表扫描更快）。 因此，从未对尚未分析的数据库使用跳过扫描。

### 6.0 连接
内部连接的ON和USING子句在上述第1.0段之前的WHERE子句分析之前转换为WHERE子句的附加条款。 因此，使用SQLite，使用较旧的SQL89连接语法的较旧的SQL92连接语法没有计算优势。 他们最终在内部连接上完成了完全相同的事情。

对于一个左侧外部加入，情况比较复杂。 以下两个查询不等效：

   SELECT * FROM tab1 LEFT JOIN tab2 ON tab1.x = tab2.y;
   SELECT * FROM tab1 LEFT JOIN tab2 WHERE tab1.x = tab2.y;
对于内部连接，上述两个查询将是相同的。 但是特殊处理适用于OUTER连接的ON和USING子句：具体来说，如果连接的右表在空行上，ON或USING子句中的约束不适用，但是这些约束在WHERE子句中适用。 最终的效果是将WHERE子句中的LEFT JOIN的ON或USING子句表达式有效地将查询转换为普通的INNER JOIN（速度更慢）。

### 6.1 连接中的表的顺序
SQLite的当前实现仅使用循环连接。 也就是说，连接被实现为嵌套循环。

连接中嵌套循环的默认顺序是FROM子句中最左侧的表，以形成外部循环和最右边的表，以形成内部循环。 但是，如果这样做会帮助它选择更好的索引，SQLite将以不同的顺序嵌套循环。

内部连接可以自由重新排序。 然而，左外连接既不是交换也不是关联，因此不会被重新排序。 如果优化器认为是有利的，但是外部连接总是按照它们发生的顺序进行评估，则可能会重新排列外部连接的左侧和右侧的内部连接。

SQLite 特别处理CROSS JOIN运算符 。 CROSS JOIN算子在理论上是交换的。 但是SQLite选择从不重新排列CROSS JOIN中的表。 这提供了一种机制，程序员可以通过该机制强制SQLite选择特定的循环嵌套顺序。

当选择连接中的表的顺序时，SQLite使用有效的多项式时间算法。 因此，SQLite能够以微秒计算具有50或60路连接的查询。

加入重新排序是自动的，通常工作得很好，程序员不必考虑它，特别是如果使用ANALYZE来收集有关可用索引的统计信息。 但是有时候需要程序员的一些提示。 例如，考虑以下模式：
```sql
  CREATE TABLE node(
     id INTEGER PRIMARY KEY,
     name TEXT
  );
  CREATE INDEX node_idx ON node(name);
  CREATE TABLE edge(
     orig INTEGER REFERENCES node,
     dest INTEGER REFERENCES node,
     PRIMARY KEY(orig, dest)
  );
  CREATE INDEX edge_idx ON edge(dest,orig);
```
上面的模式定义了一个有向图，具有在每个节点上存储名称的能力。 现在考虑针对这个架构的查询：

```sql
  SELECT *
    FROM edge AS e,
         node AS n1,
         node AS n2
   WHERE n1.name = 'alice'
     AND n2.name = 'bob'
     AND e.orig = n1.id
     AND e.dest = n2.id;
```
此查询要求的是有关从标记为“alice”的节点到标记为“bob”的节点的边缘的所有信息。 SQLite中的查询优化器对于如何实现此查询基本上有两个选择。 （实际上有六种不同的选择，但是我们只会在这里考虑其中的两个。）伪代码下面说明这两个选择。

选项1：
```sql
  foreach n1 where n1.name='alice' do:
    foreach n2 where n2.name='bob' do:
      foreach e where e.orig=n1.id and e.dest=n2.id
        return n1.*, n2.*, e.*
      end
    end
  end
```
选项二：
```sql
  foreach n1 where n1.name='alice' do:
    foreach e where e.orig=n1.id do:
      foreach n2 where n2.id=e.dest and n2.name='bob' do:
        return n1.*, n2.*, e.*
      end
    end
  end
```
相同的索引用于加速两个实现选项中的每个循环。 这两个查询计划的唯一区别是循环嵌套的顺序。

那么哪个查询计划更好？ 事实证明，答案取决于在节点和边缘表中找到什么样的数据。

让alice节点的数量为M，bob节点数为N.考虑两种情况。 在第一种情况下，M和N都是2，但每个节点上有数千个边。 在这种情况下，优选选项1。 使用选项1，内部循环检查一对节点之间是否存在边缘，并输出结果（如果找到）。 但是因为每个只有2个alice和bob节点，所以内循环只需要运行4次，而且查询非常快。 备选方案2在这里需要更长的时间。 选项2的外部循环仅执行两次，但是由于存在大量边缘离开每个alice节点，所以中间循环必须迭代数千次。 这会慢得多 所以在第一种情况下，我们更喜欢使用选项1。

现在考虑M和N都是3500的情况。alice节点是丰富的。 但是假设这些节点中的每一个仅通过一个或两个边缘连接。 在这种情况下，优选选项2。 使用选项2，外部循环仍然需要运行3500次，但是中间循环仅对每个外部循环运行一次或两次，并且内部循环将仅针对每个中间循环运行一次（如果有的话）。 因此，内循环的总次数约为7000.另一方面，选项1必须同时运行其外循环和中间循环3500次，从而导致中间循环的1200万次迭代。 因此，在第二种情况下，备选方案2比选项1快了近2000倍。

因此，您可以看到，根据数据在表中的结构如何，查询计划1或查询计划2可能会更好。 SQLite默认选择哪个计划？ 从版本3.6.18开始，不运行ANALYZE ，SQLite将选择选项2.但如果运行ANALYZE命令以便收集统计信息，则如果统计信息指示替代方案可能运行得更快，则可能会做出不同的选择。

### 6.2使用SQLITE_STAT表手动控制查询计划
SQLite提供了高级程序员对优化器选择的查询计划进行控制的能力。 执行此操作的一种方法是在sqlite_stat1 ， sqlite_stat3和/或sqlite_stat4表中得出分析结果。 除了下一段所述的一种情况外，不建议采用这种方法。

对于使用SQLite数据库作为其应用程序文件格式的程序 ，当首次创建新数据库实例时， ANALYZE命令无效，因为数据库不包含收集统计信息的数据。 在这种情况下，可以在开发过程中构建包含典型数据的大型原型数据库，并在此原型数据库上运行ANALYZE命令来收集统计信息，然后将原型统计信息作为应用程序的一部分进行保存。 部署后，当应用程序创建新的数据库文件时，它可以运行ANALYZE命令，以便创建统计表，然后将从原型数据库获取的预计算的统计信息复制到这些新的统计信息表中。 这样，大型工作数据集的统计信息可以预先加载到新创建的应用程序文件中。

### 6.3使用CROSS JOIN手动控制查询计划
程序员可以通过使用CROSS JOIN运算符而不是JOIN，INNER JOIN，NATURAL JOIN或“，”连接来强制SQLite对连接使用特定的循环嵌套顺序。 虽然CROSS JOIN在理论上是可交换的，但SQLite选择永远不会在CROSS JOIN中重新排序表。 因此，CROSS JOIN的左侧表格将始终处于相对于右侧表格的外部循环中。

在以下查询中，优化器可以自由地对FROM子句的表进行重新排序，无论它看起来如何：

```sql
  SELECT *
    FROM node AS n1,
         edge AS e,
         node AS n2
   WHERE n1.name = 'alice'
     AND n2.name = 'bob'
     AND e.orig = n1.id
     AND e.dest = n2.id;
```
但是在以下相同查询的逻辑等价公式中，“CROSS JOIN”代替“，”表示表的顺序必须为N1，E，N2。
```sql
  SELECT *
    FROM node AS n1 CROSS JOIN
         edge AS e CROSS JOIN
         node AS n2
   WHERE n1.name = 'alice'
     AND n2.name = 'bob'
     AND e.orig = n1.id
     AND e.dest = n2.id;
```
在后一个查询中，查询计划必须是选项2 。 请注意，您必须使用关键字“CROSS”才能禁用表重新排序优化; INNER JOIN，NATURAL JOIN，JOIN和其他类似的组合工作就像一个逗号连接，优化器可以自由地对表进行重新排序。 （外部连接中的表重新排序也被禁用，但是这是因为外部联接不是关联的或可交换的。在OUTER JOIN中重新排序表会更改结果。）

有关使用CROSS JOIN手动控制连接的嵌套顺序的另一个现实世界示例，请参阅[The Fossil NGQP Upgrade Case Study][1]。 以后在同一文档中发现的[query planner checklist][2]提供了有关查询计划程序的手动控制的进一步指导。

### 7.0选择多个索引
查询的FROM子句中的每个表都最多可以使用一个索引（除非OR-clause优化发挥作用），SQLite力求在每个表上至少使用一个索引。 有时，两个或多个索引可能是在单个表上使用的候选者。 例如：
```sql
  CREATE TABLE ex2(x,y,z);
  CREATE INDEX ex2i1 ON ex2(x);
  CREATE INDEX ex2i2 ON ex2(y);
  SELECT z FROM ex2 WHERE x=5 AND y=6;
```
对于上面的SELECT语句，优化器可以使用ex2i1索引来查找包含x = 5的ex2行，然后根据y = 6项测试每一行。 或者可以使用ex2i2索引来查找包含y = 6的ex2行，然后根据x = 5项测试这些行中的每一行。

当面对两个或更多索引的选择时，SQLite会尝试使用每个选项来估计执行查询所需的总工作量。 然后选择给出估计工作最少的选项。

为了帮助优化器对使用各种索引所涉及的工作进行更准确的估计，用户可以选择运行ANALYZE命令。 ANALYZE命令扫描数据库的所有索引，其中可能有两个或多个索引之间的选择，并收集关于这些索引的选择性的统计信息。 此扫描收集的统计信息存储在特殊数据库表中，名称显示名称全部以“ sqlite_stat ”开头。 这些表的内容不会随着数据库的更改而更新，因此在进行重大更改后，重新运行ANALYZE可能会谨慎。 ANALYZE命令的结果仅适用于在ANALYZE命令完成后打开的数据库连接。

各种sqlite_stat N表包含有关各种索引的选择性的信息。 例如， sqlite_stat1表可能表示列x上的等式约束将搜索空间平均减少到10行，而列y上的等式约束将搜索空间平均减少到3行。 在这种情况下，SQLite更喜欢使用索引ex2i2，因为索引更有选择性。

### 7.1取消WHERE条件使用一元 - “+”
WHERE子句的条件可以手动取消资格，以便与索引一起使用，将一个一元的+运算符添加到列名中。 一元+是一个无操作的，不会在准备语句中生成任何字节码。 但是一元运算符将阻止该术语限制索引。 所以，在上面的例子中，如果查询被重写为：

   SELECT z FROM ex2 WHERE + x = 5 AND y = 6;
x列上的+运算符将阻止该项限制索引。 这将强制使用ex2i2索引。

请注意，一元+运算符也会从表达式中删除类型关联 ，并且在某些情况下，这可能会导致表达式含义的微妙变化。 在上面的例子中，如果列x具有TEXT亲和性，则比较“x = 5”将作为文本完成。 但是+操作符删除了亲和力。 所以比较“+ x = 5”会将列x中的文本与数值5进行比较，并将始终为假。

### 7.2范围查询
考虑一下略有不同的情况：
```sql
   CREATE TABLE ex2（x，y，z）;
   CREATE INDEX ex2i1 ON ex2（x）;
   CREATE INDEX ex2i2 ON ex2（y）;
   SELECT z FROM ex2 WHERE x BETWEEN 1 AND 100 AND y BETWEEN 1 AND 100;
```
进一步假设列x包含分布在0和1,000,000之间的值，列y包含跨越0到1,000之间的值。 在这种情况下，列x上的范围约束应该将搜索空间减少10,000，而列y的范围约束应该将搜索空间减少10倍。因此，ex2i1索引应该是首选的。

SQLite将做出这一决定，但只有当它已经使用SQLITE_ENABLE_STAT3或SQLITE_ENABLE_STAT4进行编译时。 SQLITE_ENABLE_STAT3和SQLITE_ENABLE_STAT4选项导致ANALYZE命令在sqlite_stat3或sqlite_stat4表中收集列内容的直方图，并使用此直方图更好地猜测最佳查询以用于范围约束（如上述）。 STAT3和STAT4之间的主要区别在于STAT3只记录索引最左侧列的直方图数据，而STAT4记录索引所有列的直方图数据。 对于单列索引，STAT3和STAT4工作相同。

直方图数据仅在约束的右侧是简单的编译时常数或参数而不是表达式时才有用。

直方图数据的另一个限制是它只适用于索引的最左边的列。 考虑这种情况：
```sql
   CREATE TABLE ex3（w，x，y，z）;
   CREATE INDEX ex3i1 ON ex2（w，x）;
   CREATE INDEX ex3i2 ON ex2（w，y）;
   SELECT z FROM ex3 WHERE w = 5 AND x BETWEEN 1 AND 100 AND y BETWEEN 1 AND 100;
```
这里的不等式在不是最左侧索引列的列x和y上。 因此，收集的最左列索引的直方图数据在帮助在列x和y之间的范围约束之间进行选择是无用的。

### 8.0覆盖指数
当进行索引查找的行时，通常的过程是对索引进行二进制搜索以找到索引条目，然后从索引中提取rowid ，并使用该rowid对原始表进行二进制搜索。 因此，典型的索引查找涉及两个二进制搜索。 但是，如果要从表中提取的所有列已经在索引本身中可用，SQLite将使用索引中包含的值，并且永远不会查找原始的表行。 这可以为每行节省一个二进制搜索，并且可以使许多查询的运行速度快两倍。

当索引包含查询所需的所有数据，而当原始表不需要查询时，我们将该索引称为“覆盖索引”。

### 9.0 ORDER BY优化
SQLite尝试使用索引来满足查询的ORDER BY子句。 当使用索引来满足WHERE子句约束或满足ORDER BY子句的时候，SQLite执行与上述相同的成本分析，并选择它认为会导致最快答案的索引。

SQLite还将尝试使用索引来帮助满足GROUP BY子句和DISTINCT关键字。 如果可以安排连接的嵌套循环，使得等于GROUP BY或DISTINCT的行是连续的，则GROUP BY或DISTINCT逻辑可以确定当前行是否是同一组的一部分，或者当前行通过将当前行与上一行进行比较，行是不同的。 这可以比将每行与所有先前行进行比较的替代方法快得多。

### 9.1部分ORDER BY通过索引
如果查询包含具有多个术语的ORDER BY子句，则可能是SQLite可以使用索引来按照ORDER BY中某些术语前缀的顺序排列，但ORDER BY中的后续术语不满足。 在这种情况下，SQLite会阻止排序。 假设ORDER BY子句有四个术语，并且查询的自然顺序按照前两个项的顺序排列出现。 由于每行由查询引擎输出并输入分拣机，所以将对应于ORDER BY的前两项的当前行中的输出与上一行进行比较。 如果它们已更改，则当前排序已完成并输出，并开始新的排序。 这导致排序稍快一些。 但是更大的优点是在内存中需要保留少行数量，减少内存需求，并且在核心查询运行完成之前可以开始出现输出。

### 10.0子查询变平
当子查询发生在SELECT的FROM子句中时，最简单的行为是将子查询计算到一个临时表中，然后根据transient表运行外部SELECT。 但是，这样的计划可能不是最佳的，因为临时表将不具有任何索引，外部查询（可能是连接）将被迫在临时表上执行全表扫描。

为了克服这个问题，SQLite尝试在SELECT的FROM子句中展开子查询。 这涉及将子查询的FROM子句插入到外部查询的FROM子句中，并重写外部查询中引用子查询结果集的表达式。 例如：

   SELECT a FROM（SELECT x + y as a FROM t1 WHERE z <100）WHERE a> 5 
将使用query flattening重写为：

   SELECT x + y As a FROM t1 WHERE z <100 AND a> 5
为了使查询平坦化发生，需要满足所有条件的长列表。 一些限制被标注为由斜体文本过时。 这些额外的约束保留在文档中以保留其他约束的编号。

- 子查询和外部查询都不使用聚合。
- 子查询不是聚合，也不是外部查询不是连接。
- 子查询不是左外连接的正确操作数，否则子查询本身不是连接。
- 子查询不是DISTINCT。
（纳入约束4）
- 子查询不使用聚合或外部查询不是DISTINCT。
- 子查询具有FROM子句。
- 子查询不使用LIMIT或外部查询不是连接。
- 子查询不使用LIMIT或外部查询不使用聚合。
- 子查询不使用聚合或外部查询不使用LIMIT。
- 子查询和外部查询都不具有ORDER BY子句。
（纳入约束3）
- 子查询和外部查询都不使用LIMIT。
- 子查询不使用OFFSET。
- 外部查询不是复合选择的一部分，否则子查询不具有LIMIT子句。
- 外部查询不是聚合或子查询不包含ORDER BY。
- 子查询不是复合选择，或者是完全由非聚合查询组成的UNION ALL复合子句和父查询：
- 本身不是复合选择的一部分，
不是聚合或DISTINCT查询，而且
不是加入。
- 父和子查询可能包含WHERE子句。 根据规则（11），（12）和（13），它们也可能包含ORDER BY，LIMIT和OFFSET条款。
- 如果子查询是复合选择，则父项的ORDER by子句的所有术语都必须是对子查询的列的简单引用。
- 子查询不使用LIMIT或外部查询没有WHERE子句。
如果子查询是复合选择，则不能使用ORDER BY子句。
- 子查询不使用LIMIT或外部查询不是DISTINCT。
- 子查询不是递归CTE。
- 父进程不是递归CTE，也不是子查询不是复合查询。

### 11.0 MIN / MAX优化
包含单个MIN（）或MAX（）聚合函数的查询可以通过执行单个索引查找而不是扫描整个表来满足其参数是索引的最左侧列。 例子：

   SELECT MIN（x）FROM table;
   SELECT MAX（x）+1 FROM表;

### 12.0自动索引
当没有索引可用于帮助评估查询时，SQLite可能会创建仅在单个SQL语句的持续时间内持续的自动索引。 由于构建自动索引的代价是O（NlogN）（其中N是表中的条目数），并且进行全表扫描的成本仅为O（N），所以只有SQLite才会创建自动索引期望在SQL语句过程中查找将运行多于logN次。 考虑一个例子：
```sql
  CREATE TABLE t1(a,b);
  CREATE TABLE t2(c,d);
  -- Insert many rows into both t1 and t2
  SELECT * FROM t1, t2 WHERE a=c;
```
在上面的查询中，如果t1和t2都有大约N行，那么没有任何索引，查询将需要O（N * N）个时间。 另一方面，在表t2上创建索引需要O（NlogN）时间，然后使用该索引来评估查询需要额外的O（NlogN）时间。 在没有ANALYZE信息的情况下，SQLite猜测N是一百万，因此它认为构建自动索引将是更便宜的方法。

自动索引也可能用于子查询：
```sql
  CREATE TABLE t1(a,b);
  CREATE TABLE t2(c,d);
  -- Insert many rows into both t1 and t2
  SELECT a, (SELECT d FROM t2 WHERE c=b) FROM t1;
```
在此示例中，t2表用于子查询以转换t1.b列的值。 如果每个表包含N行，SQLite希望子查询将运行N次，因此它将相信在t2之前首先构建一个自动临时索引更快，然后使用该索引来满足子查询的N个实例。

可以使用automatic_index pragma在运行时禁用自动索引功能。 默认情况下，自动索引处于打开状态，但可以更改此功能，以使默认情况下使用SQLITE_DEFAULT_AUTOMATIC_INDEX编译时选项关闭自动索引。 通过使用SQLITE_OMIT_AUTOMATIC_INDEX编译时选项进行编译，可以完全禁用创建自动索引的功能。

  [1]: http://www.sqlite.org/queryplanner-ng.html#fossilcasestudy
  [2]: http://www.sqlite.org/queryplanner-ng.html#howtofix

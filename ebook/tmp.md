**LikeSimplification**

LIKE是SQL中常见的操作，用于在WHERE子句中搜索列中的指定模式。使用的例子如下：

```sql
-- 从Persons表中选取姓名以“A”为开头的记录
SELECT * FROM Persons WHERE Name LIKE 'A%'

-- 从Persons表中选取城市名称以“g”为结尾的记录
SELECT * FROM Persons WHERE City LIKE '%g'
```

LIKE在Spark SQL逻辑计划中表示为：

```scala
case class Like(left: Expression, right: Expression) extends StringRegexExpression
```

其中left表示被匹配的列明，right表示匹配的目标正则表达式。

大家知道正则表达式匹配是比较耗时的操作，如果表中的每条记录都使用正则表达式取匹配的话，那计算速度就会变得很慢。本规则基于LIKE表达式中特定的匹配条件，将LIKE表达式进行适当转换，简化正则表达式匹配流程，提高计算效率。假设LIKE语句之后的模式为pattern，转换后的表达式为expr，它们的对应关系如下表所示：

| pattern               | expr                                               | 说明                                                         |
| --------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `"([^_%]+)%"`         | StartsWith                                         | %放在模式最后，表示在特定列过滤以特定字符串开头的记录，使用StartsWith替代 |
| `"%([^_%]+)"`         | EndsWith                                           | %放在模式最前，表示在特定列过滤以特定字符串结尾的记录，使用EndsWith替代 |
| `"([^_%]+)%([^_%]+)"` | And(GreaterThanOrEqual, And(StartsWith, EndsWith)) | %放在模式中间，表示在特定列过滤以特定字符串s1开头且以特定字符串s2结尾的记录。使用三个表达式的And连接来表示，记录的长度必须大于等于s1和s2的长度之和，使用GreaterThanOrEqual表示；必须以s1开头，使用StartsWith表示；必须以s2结尾，使用EndsWith表示 |
| `"%([^_%]+)%"`        | Contains                                           | %放在模式开头和结尾，表示在特定列过滤包含特定字符串的记录，使用Contains替代 |
| `"([^_%]*)"`          | EqualTo                                            | 不包含%，表示在特定列中过滤与特定字符串相等的记录，使用EqualTo替代 |

一个例子如下：

**BooleanSimplification**


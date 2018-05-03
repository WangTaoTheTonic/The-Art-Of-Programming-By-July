**OptimizeIn**

In操作的含义是判断一个值是否包含在一个表达式列表中，如果我们去看它的eval函数，会发现在具体的执行逻辑中，是遍历整个表达式进行是否包含的判断的：

```scala
    list.foreach { e =>
      val v = e.eval(input)
      if (v == null) {
        hasNull = true
      } else if (ordering.equiv(v, evaluatedValue)) {
        return true
      }
    }
```

从数据结构的角度来看，在列表较大时，循环遍历的效率肯定不如先将其转换为Set，使用分支判断取查找是否包含某个元素。本规则首先检查In的表达式列表中元素的类型是否都是字面值表达式，如果是，做以下操作：

1. 将表达式列表转换为表达式集合。

2. 如果表达式集合内元素的个数超过某个阈值（由`spark.sql.optimizer.inSetConversionThreshold`参数决定，默认值为10），则将In节点转换为InSet节点：

   ```scala
   case class InSet(child: Expression, hset: Set[Any]) extends UnaryExpression with Predicate
   ```
   其中child即为In节点中的value，hset即为转换后的表达式集合。

3. 如果表达式集合内元素的个数小于此阈值，并且小于原In节点中list内元素的个数，说明原In节点的list内有重复元素，使用去重后的列表代替原In节点的list表达式列表。

一个例子如下：

**ConstantFolding**

在逻辑计划的表达式中，有一些表达式是可折叠的（foldable）.通本规则检测逻辑计划中的此类表达式，并且将它们进行折叠求值。一个例子如下：

**ReorderAssociativeOperator**




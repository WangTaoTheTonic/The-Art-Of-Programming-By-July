[TOC]



# 逻辑计划的物化 - SparkPlanner

在前面的几个章节中，我们介绍了用户输入的一个SQL语句被分词、解析、分析、优化的过程。这些过程对应的操作对象叫做逻辑计划（即LogicalPlan），从逻辑上将SQL语句在Spark SQL中使用树的结构表示出来。逻辑计划要得到最后的执行，还需要通过一个叫做SparkPlanner的组件进行转换，转换后的实体称之为物理计划（physical plans，在Spark代码中使用SparkPlan类表示），可以通过调用物理计划的execute方法生成对应的RDD，然后交给Spark核心引擎去执行。

与前面介绍的Analyzer、Optimizer都是RuleExecutor的子类不同，SparkPlanner继承了SparkStrategies，而后者又是QueryPlanner的子类。类的关系图如下：

（待补充：几个类的关系图，凸显主要方法）


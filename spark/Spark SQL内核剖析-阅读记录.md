阅读sparkSQL 内核剖析-问题记录

# 第三章

catalyst 的重要概念，InternalRow + TreeNode + Expression 体系不懂，TreeNode解析成逻辑树，Expression 本身也是 TreeNode 类的子类

# 第四章

词法、语法解析器 ANTLR

ANTLR (Another Tool for Language Recognition ）是目前非常活跃的语法生成工具，用 Java 

语言编写，基于 **LL （＊）解析方式**  ，使用自上而下的递归下降分析方法。 ANTLR 可以用来 

产生词法分析器、语法分析器和树状分析器（Tree Parser ）等各个模块，其**文法定义使用类似** 

**EBNF (Extended Backus-Naur Form ）**的方式



antlr 和 sparkSQL的astbuilder到底是什么？

# 第五章

sparkSQL逻辑计划阶段，将SQL语句转换为树结构的逻辑算子树。

1.spark.sql的SQL语句经过sparkSQLParser解析成UNresolved LogicPlan，在经过Analyzer解析为Analyzed LogicPlan，在由Optimizer优化为Optimized LogicPlan，传递到下一阶段进行物理执行计划生成


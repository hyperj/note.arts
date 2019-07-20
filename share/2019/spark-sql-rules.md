# Spark SQL 规则

## Analyzer

```scala
 lazy val batches: Seq[Batch] = Seq(
    Batch("Hints", fixedPoint,
      new ResolveHints.ResolveJoinStrategyHints(conf), // BROADCAST、SHUFFLE_MERGE、SHUFFLE_HASH、SHUFFLE_REPLICATE_NL
      ResolveHints.ResolveCoalesceHints, // COALESCE、REPARTITION
      new ResolveHints.RemoveAllHints(conf)), // 移除无效 Hint
    Batch("Simple Sanity Check", Once,
      LookupFunctions), // 检查 Function 是否存在
    Batch("Substitution", fixedPoint,
      CTESubstitution, // CTE 替换
      WindowsSubstitution, // Window 替换
      EliminateUnions, // 消除 Union（只有一个时）
      new SubstituteUnresolvedOrdinals(conf)), // 替换 order by、group by 序号
    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions :: // 数据表函数，range
      ResolveAlterTable :: // ALTER TABLE v2
      ResolveTables :: // Table v2，表
      ResolveRelations :: // Table v1
      ResolveReferences :: // Reference 列
      ResolveCreateNamedStruct :: // named_struct
      ResolveDeserializer :: // Deserializer 反序列化
      ResolveNewInstance :: // 构造实例
      ResolveUpCast :: // 类型转换
      ResolveGroupingAnalytics :: // 多维分析，GroupingSets/Cube/Rollup
      ResolvePivot :: // 透视，行列转换
      ResolveOrdinalInOrderByAndGroupBy :: // 序号，order/sort by、group by
      ResolveAggAliasInGroupBy :: // 聚合中的别名，group by、grouping sets
      ResolveMissingReferences :: // 补充缺失的 Reference，sort by、having
      ExtractGenerator :: // 提取生成器，explode
      ResolveGenerate :: // 生成器
      ResolveFunctions :: // Function，函数
      ResolveAliases :: // Alias，别名
      ResolveSubquery :: // 子查询
      ResolveSubqueryColumnAliases :: // 子查询别名
      ResolveWindowOrder :: // Window Order
      ResolveWindowFrame :: // Window Frame
      ResolveNaturalAndUsingJoin :: // 使用 Join 替换 Natural 形式 
      ResolveOutputRelation :: // 处理 输出数据和表的差异
      ExtractWindowExpressions :: // 提取 Window
      GlobalAggregates :: // 全局聚合
      ResolveAggregateFunctions :: // 聚合函数
      TimeWindowing :: // 滑动时间窗口
      ResolveInlineTables(conf) :: // 内联表 VALUES()
      ResolveHigherOrderFunctions(catalog) :: // higher order function，lambda function
      ResolveLambdaVariables(conf) :: // lambda function 参数
      ResolveTimeZone(conf) :: // TimeZone
      ResolveRandomSeed :: // 随机数生成
      TypeCoercion.typeCoercionRules(conf) ++ // 强制类型转换
      extendedResolutionRules : _*), // 扩展规则
    Batch("Post-Hoc Resolution", Once, postHocResolutionRules: _*), // 扩展规则
    Batch("Nondeterministic", Once,
      PullOutNondeterministic), // 非确定性表达式
    Batch("UDF", Once,
      HandleNullInputsForUDF), // 处理为 null 的输入
    Batch("UpdateNullability", Once,
      UpdateAttributeNullability), // 更新可以为空属性
    Batch("Subquery", Once,
      UpdateOuterReferences), // 处理子查询的外部引用
    Batch("Cleanup", fixedPoint,
      CleanupAliases) // 清理没必要的别名
  )
```

## Optimizer

```scala
  /**
   * Defines the default rule batches in the Optimizer.
   *
   * Implementations of this class should override this method, and [[nonExcludableRules]] if
   * necessary, instead of [[batches]]. The rule batches that eventually run in the Optimizer,
   * i.e., returned by [[batches]], will be (defaultBatches - (excludedRules - nonExcludableRules)).
   */
  def defaultBatches: Seq[Batch] = {
    val operatorOptimizationRuleSet =
      Seq(
        // Operator push down
        PushProjectionThroughUnion, // 对于全部是确定性 Project 进行下推
        ReorderJoin, // 根据 Join 关联条件，进行顺序重排
        EliminateOuterJoin, // 消除非空约束的 Outer Join
        PushDownPredicates, // 谓词下推
        PushDownLeftSemiAntiJoin, // Project、Window、Union、Aggregate、PushPredicateThroughNonJoin 场景下，下推 LeftSemi/LeftAnti
        PushLeftSemiLeftAntiThroughJoin, // Join 场景下，下推 LeftSemi/LeftAnti
        LimitPushDown, // Limit 下推
        ColumnPruning, // 列裁剪
        InferFiltersFromConstraints, // 根据约束推断 Filter 信息
        // Operator combine
        CollapseRepartition, // 合并 Repartition
        CollapseProject, // 合并 Project
        CollapseWindow, // 合并 Window（相同的分区及排序）
        CombineFilters, // 合并 Filter
        CombineLimits, // 合并 Limit
        CombineUnions, // 合并 Union
        // Constant folding and strength reduction
        TransposeWindow, // 转置 Window，优化计算
        NullPropagation, // Null 传递
        ConstantPropagation, // 常量传递
        FoldablePropagation, // 可折叠字段、属性传播
        OptimizeIn, // 优化 In，空处理、重复处理
        ConstantFolding, // 常量折叠
        ReorderAssociativeOperator, // 排序与折叠变量
        LikeSimplification, // 优化 Like 对应的正则
        BooleanSimplification, // 简化 Boolean 表达式
        SimplifyConditionals, // 简化条件表达式，if/case
        RemoveDispensableExpressions, // 移除不必要的节点，positive
        SimplifyBinaryComparison, // 优化比较表达式为 Boolean Literal
        ReplaceNullWithFalseInPredicate, // 优化 Null 场景 Literal
        PruneFilters, // 裁剪明确的过滤条件
        EliminateSorts, // 消除 Sort，无关或重复
        SimplifyCasts, // 简化 Cast，类型匹配
        SimplifyCaseConversionExpressions, // 简化 叠加的大小写转换
        RewriteCorrelatedScalarSubquery, // 子查询改写为 Join
        EliminateSerialization, // 消除 序列化
        RemoveRedundantAliases, // 删除冗余的别名
        RemoveNoopOperators, // 删除没有操作的操作符
        SimplifyExtractValueOps, // 简化符合类型操作符
        CombineConcats) ++ // 合并 Concat 表达式
        extendedOperatorOptimizationRules // 扩展规则

    val operatorOptimizationBatch: Seq[Batch] = {
      val rulesWithoutInferFiltersFromConstraints =
        operatorOptimizationRuleSet.filterNot(_ == InferFiltersFromConstraints)
      Batch("Operator Optimization before Inferring Filters", fixedPoint,
        rulesWithoutInferFiltersFromConstraints: _*) ::
      Batch("Infer Filters", Once,
        InferFiltersFromConstraints) ::
      Batch("Operator Optimization after Inferring Filters", fixedPoint,
        rulesWithoutInferFiltersFromConstraints: _*) :: Nil
    }

    (Batch("Eliminate Distinct", Once, EliminateDistinct) ::
    // Technically some of the rules in Finish Analysis are not optimizer rules and belong more
    // in the analyzer, because they are needed for correctness (e.g. ComputeCurrentTime).
    // However, because we also use the analyzer to canonicalized queries (for view definition),
    // we do not eliminate subqueries or compute current time in the analyzer.
    Batch("Finish Analysis", Once,
      EliminateResolvedHint,
      EliminateSubqueryAliases,
      EliminateView,
      ReplaceExpressions,
      ComputeCurrentTime,
      GetCurrentDatabase(sessionCatalog),
      RewriteDistinctAggregates,
      ReplaceDeduplicateWithAggregate) ::
    //////////////////////////////////////////////////////////////////////////////////////////
    // Optimizer rules start here
    //////////////////////////////////////////////////////////////////////////////////////////
    // - Do the first call of CombineUnions before starting the major Optimizer rules,
    //   since it can reduce the number of iteration and the other rules could add/move
    //   extra operators between two adjacent Union operators.
    // - Call CombineUnions again in Batch("Operator Optimizations"),
    //   since the other rules might make two separate Unions operators adjacent.
    Batch("Union", Once,
      CombineUnions) ::
    Batch("OptimizeLimitZero", Once,
      OptimizeLimitZero) ::
    // Run this once earlier. This might simplify the plan and reduce cost of optimizer.
    // For example, a query such as Filter(LocalRelation) would go through all the heavy
    // optimizer rules that are triggered when there is a filter
    // (e.g. InferFiltersFromConstraints). If we run this batch earlier, the query becomes just
    // LocalRelation and does not trigger many rules.
    Batch("LocalRelation early", fixedPoint,
      ConvertToLocalRelation,
      PropagateEmptyRelation) ::
    Batch("Pullup Correlated Expressions", Once,
      PullupCorrelatedPredicates) ::
    Batch("Subquery", Once,
      OptimizeSubqueries) ::
    Batch("Replace Operators", fixedPoint,
      RewriteExceptAll,
      RewriteIntersectAll,
      ReplaceIntersectWithSemiJoin,
      ReplaceExceptWithFilter,
      ReplaceExceptWithAntiJoin,
      ReplaceDistinctWithAggregate) ::
    Batch("Aggregate", fixedPoint,
      RemoveLiteralFromGroupExpressions,
      RemoveRepetitionFromGroupExpressions) :: Nil ++
    operatorOptimizationBatch) :+
    Batch("Join Reorder", Once,
      CostBasedJoinReorder) :+
    Batch("Remove Redundant Sorts", Once,
      RemoveRedundantSorts) :+
    Batch("Decimal Optimizations", fixedPoint,
      DecimalAggregates) :+
    Batch("Object Expressions Optimization", fixedPoint,
      EliminateMapObjects,
      CombineTypedFilters,
      ObjectSerializerPruning,
      ReassignLambdaVariableID) :+
    Batch("LocalRelation", fixedPoint,
      ConvertToLocalRelation,
      PropagateEmptyRelation) :+
    Batch("Extract PythonUDF From JoinCondition", Once,
      PullOutPythonUDFInJoinCondition) :+
    // The following batch should be executed after batch "Join Reorder" "LocalRelation" and
    // "Extract PythonUDF From JoinCondition".
    Batch("Check Cartesian Products", Once,
      CheckCartesianProducts) :+
    Batch("RewriteSubquery", Once,
      RewritePredicateSubquery,
      ColumnPruning,
      CollapseProject,
      RemoveNoopOperators) :+
    // This batch must be executed after the `RewriteSubquery` batch, which creates joins.
    Batch("NormalizeFloatingNumbers", Once, NormalizeFloatingNumbers)
  }
```

## SparkPlan

## SparkStrategy

## Reference

- [一条 SQL 在 Apache Spark 之旅（上）](https://www.iteblog.com/archives/2561.html)
- [一条 SQL 在 Apache Spark 之旅（中）](https://www.iteblog.com/archives/2562.html)
- [一条 SQL 在 Apache Spark 之旅（下）](https://www.iteblog.com/archives/2563.html)
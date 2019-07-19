# Spark SQL 规则

## Analyzer

```scala
 lazy val batches: Seq[Batch] = Seq(
    Batch("Hints", fixedPoint,
      new ResolveHints.ResolveJoinStrategyHints(conf),
      ResolveHints.ResolveCoalesceHints,
      new ResolveHints.RemoveAllHints(conf)),
    Batch("Simple Sanity Check", Once,
      LookupFunctions),
    Batch("Substitution", fixedPoint,
      CTESubstitution,
      WindowsSubstitution,
      EliminateUnions,
      new SubstituteUnresolvedOrdinals(conf)),
    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions ::
      ResolveAlterTable ::
      ResolveTables ::
      ResolveRelations ::
      ResolveReferences ::
      ResolveCreateNamedStruct ::
      ResolveDeserializer ::
      ResolveNewInstance ::
      ResolveUpCast ::
      ResolveGroupingAnalytics ::
      ResolvePivot ::
      ResolveOrdinalInOrderByAndGroupBy ::
      ResolveAggAliasInGroupBy ::
      ResolveMissingReferences ::
      ExtractGenerator ::
      ResolveGenerate ::
      ResolveFunctions ::
      ResolveAliases ::
      ResolveSubquery ::
      ResolveSubqueryColumnAliases ::
      ResolveWindowOrder ::
      ResolveWindowFrame ::
      ResolveNaturalAndUsingJoin ::
      ResolveOutputRelation ::
      ExtractWindowExpressions ::
      GlobalAggregates ::
      ResolveAggregateFunctions ::
      TimeWindowing ::
      ResolveInlineTables(conf) ::
      ResolveHigherOrderFunctions(catalog) ::
      ResolveLambdaVariables(conf) ::
      ResolveTimeZone(conf) ::
      ResolveRandomSeed ::
      TypeCoercion.typeCoercionRules(conf) ++
      extendedResolutionRules : _*),
    Batch("Post-Hoc Resolution", Once, postHocResolutionRules: _*),
    Batch("Nondeterministic", Once,
      PullOutNondeterministic),
    Batch("UDF", Once,
      HandleNullInputsForUDF),
    Batch("UpdateNullability", Once,
      UpdateAttributeNullability),
    Batch("Subquery", Once,
      UpdateOuterReferences),
    Batch("Cleanup", fixedPoint,
      CleanupAliases)
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
        PushProjectionThroughUnion, // 对于全部是确定性投影进行下推
        ReorderJoin, // 根据 Join 关联条件，进行顺序重排
        EliminateOuterJoin, // 消除非空约束的 Outer Join
        PushDownPredicates, // 谓词下推
        PushDownLeftSemiAntiJoin, // Project、Window、Union、Aggregate、PushPredicateThroughNonJoin 场景下，下推 LeftSemi/LeftAnti
        PushLeftSemiLeftAntiThroughJoin, // Join场景下，下推 LeftSemi/LeftAnti
        LimitPushDown, // Limit 下推
        ColumnPruning, // 列裁剪
        InferFiltersFromConstraints, // 根据约束推断过滤信息
        // Operator combine
        CollapseRepartition, // 合并 Repartition
        CollapseProject, // 合并投影
        CollapseWindow, // 合并窗口
        CombineFilters,
        CombineLimits,
        CombineUnions,
        // Constant folding and strength reduction
        TransposeWindow,
        NullPropagation,
        ConstantPropagation,
        FoldablePropagation,
        OptimizeIn,
        ConstantFolding,
        ReorderAssociativeOperator,
        LikeSimplification,
        BooleanSimplification,
        SimplifyConditionals,
        RemoveDispensableExpressions,
        SimplifyBinaryComparison,
        ReplaceNullWithFalseInPredicate,
        PruneFilters,
        EliminateSorts,
        SimplifyCasts,
        SimplifyCaseConversionExpressions,
        RewriteCorrelatedScalarSubquery,
        EliminateSerialization,
        RemoveRedundantAliases,
        RemoveNoopOperators,
        SimplifyExtractValueOps,
        CombineConcats) ++
        extendedOperatorOptimizationRules

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
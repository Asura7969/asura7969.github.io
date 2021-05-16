---
title: Flink Retraction
date: 2020-12-24 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161046.png
---



# Flink Retraction Mechanism
**Flink Sql**最终生成**Physical Planning**后会对每个**RelNode**打标,判断当前**RelNode**是哪种**ChangelogMode**


**FlinkChangelogModeInferenceProgram**

```scala
  override def optimize(
      root: RelNode,
      context: StreamOptimizeContext): RelNode = {

    // step1: 确定每个节点的变更类型
    // 先从source节点开始标记节点属于哪种 ModifyKindSetTrait(I,U,D,Empty)
    val physicalRoot = root.asInstanceOf[StreamPhysicalRel]
    val rootWithModifyKindSet = SATISFY_MODIFY_KIND_SET_TRAIT_VISITOR.visit(
      physicalRoot,
      // we do not propagate the ModifyKindSet requirement and requester among blocks
      // set default ModifyKindSet requirement and requester for root
      ModifyKindSetTrait.ALL_CHANGES,
      "ROOT")

    // step2: 确定每个节点发送的消息类型(UA,UB)
    // 获取root节点（sink）的 ModifyKindSet
    val rootModifyKindSet = getModifyKindSet(rootWithModifyKindSet)
    // use the required UpdateKindTrait from parent blocks
    // 从 parent blocks 确定使用哪种 UpdateKindTrait
    val requiredUpdateKindTraits = if (rootModifyKindSet.contains(ModifyKind.UPDATE)) {
      if (context.isUpdateBeforeRequired) {
        Seq(UpdateKindTrait.BEFORE_AND_AFTER)
      } else {
        // update_before is not required, and input contains updates
        // try ONLY_UPDATE_AFTER first, and then BEFORE_AND_AFTER
        Seq(UpdateKindTrait.ONLY_UPDATE_AFTER, UpdateKindTrait.BEFORE_AND_AFTER)
      }
    } else {
      // there is no updates
      Seq(UpdateKindTrait.NONE)
    }
    // 每个节点确定 (UA,UB)
    val finalRoot = requiredUpdateKindTraits.flatMap { requiredUpdateKindTrait =>
      SATISFY_UPDATE_KIND_TRAIT_VISITOR.visit(rootWithModifyKindSet, requiredUpdateKindTrait)
    }

    // step3: sanity check and return non-empty root
    if (finalRoot.isEmpty) {
      val plan = FlinkRelOptUtil.toString(root, withChangelogTraits = true)
      throw new TableException(
        "Can't generate a valid execution plan for the given query:\n" + plan)
    } else {
      finalRoot.head
    }
  }

```





### 参考
> https://zhuanlan.zhihu.com/p/157265381

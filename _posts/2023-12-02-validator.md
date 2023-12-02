---
layout: post
title: Validator
subtitle: Radix Native Blueprints--Validator
# hero_image: /path/to/image.jpg
hero_darken: true
---
Validator(babylon)
================================

本文描述Babylon版本的Validator blueprint的设计及实现、还有functions和methods的使用细节。

## 背景
在Radix，一个验证器是在Radix网络通过`stake`方法被委托一点数量的节点。头部100个注册节点(按stake数量逆序)是活动节点集（Active Validator Set）,其职责是通过共识执行事务(transactions)。

每个验证器在共识管理器(Consensus Manager)注册。在每个epoch结束时，共识管理器基于几个规则选择一个新的验证器和stakes数作为活动节点集去运行下一个epoch.

验证器blueprint定义其关联的状态(On Ledger State)和修改(API)它们的逻辑部分。

#### Stake and PoV 的来源
* 个人用户(包括验证结点的所有者): 他能用Validator组件上的`stake`方法委托自己的XRD到验证器(并且同时接收 Stake Units(LSU))
* 网络发行(XRD铸造): 在每个epoch结束时，网络发出预定义的发行数量。每个活动验证器相应的一部分到它自己的stake中。
* 奖励，来自两个来源：
    * 事务执行费：每个事务被执行时要求支付一定数量的费用，它其中的一部分给当前主节点(Leader Validator)，一部分由所有活动节点共享。
    * 事务小费(Tips)，用户提交事务时可以指定基于百分比的小费，它是支付给主节点的。

一个验证器有单一的所有者角色：
* 所有者负有验证器的运维及设置参数的职责（比如指定被用于共识的公钥，或者验证费用百分比（网络发行收集到所者者)。
* 所有者能决定临时锁定自己的Stake Units内部金库延迟提取，它公开表达了对验证节点的可信度。

#### On-ledger State
`Validator` blueprint定义了2个字段：
* `state`，实际配置和子组件(用来管理 Staking And Owner's Stake Locking).
* `protocol_update_readiness_signal`元数据，当前节点准备升级的协议版本号。协议更新 流程仍在开发中。后续有文档专门说明。

`state`的高阶视角：
* 共识相关配置：
    * `key` - 一个公钥，在共识层相应的节点标识。`update_key` API了解更多。
    * `is_registered` - 表示所有者当前是否想让这个验证器作为活动节点集的一部分。如果这个值是`false`，即便这个节点的stake数量是最高的，它仍不会被共识管理器在为下一个epoch选择新的活动节点集时考虑。 `register/unregister`API了解更多。
    * `sorted_key` - 一个key值，被内部用于存储已注册节点stake数量降序的排序key.
        * 它只在验证器已注册并且有非0值的stake时有用，否则这个字段为None值。
        * 技术上来说，这个字段只是一个缓存并简化了实现。
* stake相关的配置和子组件
    * `accepts_delegated_stake` - 除了所有者外其它用户是否能委托stake到这个验证器。参考`update_accepts_delegated_stake` API.
    * `stake_unit_resource` - 当前验证器LSU资源地址，Validator组件实例化时确定。
    * `stake_xrd_vault_id` - 这个验证器的金库，持有当前stake。用户能通过 `stake/unstake` API与它进行交互。
    * `pending_xrd_withdraw_vault_id` - 处在提取进程中的金库。持有`unstaked`但时还没有被认领的xrd金库。参考`Staking/Unstaking/Claiming`
    * `claim_nft` - 一个NFT地址，用于从验证器unstake LSU的回执，在unstake延迟周期之后它可以交互成实际的XRDs. 参考 `claim_xrd` API
* Fee相关配置
    * `validator_fee_factor` - 一个从网络发行中收集到验证器所有者的一个百分比系数。
        **备注**: 它可能会发生实际值和此字段值不匹配的情况; 它被修改验证费后直接被履盖。 - 参考`validator_fee_change_request`说明。通过此收集到费用会自动锁定到所有者stake的金库之中。它表现为一个小数，而不是一个百分比。 (比如： 0.015表示"1.5%")
    * `validator_fee_change_request` - 最近一次请求改变`validator_fee_factor`(如果有).参考 `update_fee` API。如果费用升高时，它是延迟两周生效的。

        The value from this field is moved to the validator_fee_factor only when the next change is requested while this one became effective. For this reason, the requested-and-already-effective fee will be used by the Engine instead of the outdated validator_fee_factor (this only re-iterates the Important note seen above).
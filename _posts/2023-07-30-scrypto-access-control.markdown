---
layout: post
title: Radix Scrypto Access Control
subtitle: rcnet-v2 
# hero_image: /path/to/image.jpg
hero_darken: true
---
Scrypto(v0.11.0) Access Control
================================

With Scrypto, access control will feel relatable to how we intuitively think about defining who is allowed to do what. There are four cornerstones to Scrypto’s authorization model: Roles, Badges, AccessRules, and Proofs.

有了 Scrypto，访问控制将与我们定义谁被允许做什么的直观思维方式息息相关。Scrypto 的授权模型有四大基石： 角色（Roles）、徽章（Badges）、访问规则（AccessRules）和证明（Proofs）。

## Roles
角色被定义为你想允许的访问级别。比如你的dapp中可能定义3个级别的授权，admin, super admin, owner. 直觉上，这是三个继续的级别，owner有总体控制，管理和超级管理员有递增的权限控制。

``` rust
#[blueprint]
mod token_sale
    enable_method_auth! {
        roles {
            super_admin => updatable_by: []
            admin => updatable_by: [super_admin]
        },
        methods {
            buy => PUBLIC;
            create_admin => restrict_to: [super_admin, OWNER];
            change_price => restrict_to: [admin, super_admin, OWNER];
            redeem_profits => restrict_to: [OWNER];
        }
    struct GumballMachine {
        ..
    // -- snip --
    }
}
```
1. 角色定在模块下面，blueprint的struct的上面。
2. 角色命名符合你的应用场景。(OWNER不需要定义)
3. 角色可以被更新，我们可以选择哪个角色有权限更新另一个角色的访问规则。
4. 方法能被映射到每个角色。(method)
5. 方法能映射到`PUBLIC`，以满足每个人都可以调用。
6. 方法能映射给多个角色。

## Badges
当角色被定义时，相关的`badge(s)`也需要被定义。Badge不是原生类型，确切地说，它是一个主要用于验证的`resource`类型。任何时候一个用户或组件需要一些验证或授权的表单，在执行操作之前，指定的badge需要被出示。一个badge可能是可替换或不可替换资源，这依赖你的使用场景。比如你用一个NFT，如果你想它关联到某个人，或者你使用一个可替换token, 之个badge是许多用户或组件属于某个角色所提供的。

创建一个新的Badge类型，我们用`ResourceBuilder`就可以创建任何NFT或FT.

``` rust
// Using `DIVISIBILITY_NONE` to make sure nobody can divide a badge into multiple parts.
let owner_badge: Bucket = ResourceBuilder::new_fungible(OwnerRole::None)
    .divisibility(DIVISIBILITY_NONE)
    .metadata(metadata!(
        init {
            "name" => "Owner Badge", locked;
        }
    )
    .mint_initial_supply(1);

let super_admin_badge: Bucket = ResourceBuilder::new_fungible(OwnerRole::None)
    .divisibility(DIVISIBILITY_NONE)
    .metadata(metadata!(
        init {
            "name" => "Super Admin Badge", locked;
        }
    )
    .mint_initial_supply(2);

let admin_badge: Bucket = ResourceBuilder::new_fungible(OwnerRole::None)
    .divisibility(DIVISIBILITY_NONE)
    .metadata(metadata!(
        init {
            "name" => "Admin Badge", locked;
        }
    )
    .mint_initial_supply(1);
```





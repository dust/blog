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

有了`Scrypto`，访问控制将与我们定义谁被允许做什么的直观思维方式息息相关。`Scrypto`的授权模型有四大基石： `角色（Roles）`、`徽章（Badges）`、`访问规则（AccessRules）`和`证明（Proofs）`。

## Roles
角色被定义为你想允许的访问级别。比如你的dapp中可能定义3个级别的授权，`admin`, `super admin`, `owner`. 直觉上，这是三个继续的级别，`owner`有总体控制，管理和超级管理员有递增的权限控制。

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

1. 角色定在模块(`mod`)下面，`blueprint`的`struct`的上面。
2. 角色命名符合你的应用场景。(`OWNER`不需要定义)
3. 角色可以被更新，我们可以选择哪个角色有权限更新另一个角色的访问规则。
4. 方法能被映射到每个角色。(`method`)
5. 方法能映射到`PUBLIC`，以满足每个人都可以调用。
6. 方法能映射给多个角色。

## Badges
当角色被定义时，相关的`badge(s)`也需要被定义。`Badge`不是原生类型，确切地说，它是一个主要用于验证的`resource`类型。任何时候一个用户或组件需要一些验证或授权的表单，在执行操作之前，指定的`badge`需要被出示。一个`badge`可能是可替换或不可替换资源，这依赖你的使用场景。比如你用一个NFT，如果你想它关联到某个人，或者你使用一个可替换`token`, 这个`badge`是许多用户或组件属于某个角色所提供的。

创建一个新的`Badge`类型，我们用`ResourceBuilder`就可以创建任何NFT或FT.

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

## Access Rules
`AccessRule`用于定义围绕方法和资源行为的安全措施。更具体地说，`AccessRule`定义了在执行哪些方法调用或资源操作前需要哪些或多少`Badge`. 

``` rust
// -- snip --
.instantiate()
.prepare_to_globalize(
    OwnerRole::Fixed(
        rule!(require(owner_badge.resource_address())
    )
))
.roles(
    roles!(
        super_admin => rule!(require_amount(dec!(2), super_admin_badge.resource_address()));
        admin => rule!(require(admin_manager.resource_address()));
    )
)
.globalize();

```

1. OWNER与其它角色是分开定义的，这是因为Owner不仅可以调用组件的方法，还可以访问组件模块，如权限，元数据和版税。
2. 角色被构造在初始化阶段，在一个组件被全局化`(globalized, addressable)`之前。
3. 角色被映射到一个指定的已关联`badge`的`AccessRule`。


## Proofs
`badge`使用的一个重要惯例是，在正常使用情况下，`badge`实际上并不会从Vault中取出，取而代之的是创建一个证明`(Proof)`，用来证明某个行为者拥有该`badge`，或者至少可以使用该`badge`。

你可以从`Vault`或(罕见)`Bucket`中为特定资源创建`Proof`，然后不改变底层内容的所有权情况下使用该证明。例如，如果我的帐户有一个FLIX token表明我是Radflix的会员，我就可以创建一个这个Token的`Proof`，并向Radflix组件出示(这个`Proof`)，这样它就会允许我访问（某些会员的行为）。**我并实际传递它**，即使Radflix组件存在漏洞或恶意，它也无法控制生成`Proof`的底层Token。它就象现实世界闪亮的微章一样，无论你向谁展示它，都能看到你拥有它，并且检查它，但是你并没有实际把它交给他们，所以他们也无法把它据为已有。

`Proof`有一个它关联的数量（属性），并且数量为0时，不能创建`Proof`。也就是说，如果你有一个存放指定资源的`Vault`，但是`Vault`是空的，（那么）你无法创建该资源的`Proof`.

## The Authorization Zone
在`Transaction`模型中，每个`Transaction`都有可以通过`Manifest`访问的顶层，包含存储资源的`worktop和`存储`Proof`的授权区。从`manifest`调用任何Scrypto方法时，都会自动将该访问方法的规则(Access Rules)与授权区进行比较，如果符合规则，则允许访问该方法，调用成功。如果不符合规则，则会立即中止`Transaction`。不需要你（显式/主动）指定你认为满足规则所需的证明，系统会自动（帮你）计算出来。

同样的逻辑也适用于直接从`manifest`调用的组件，如果组件试图对资源进行特权操作，比如mint额外的供应，系统就会根据授权区的内容检查规则，如果符合规则，则操作成功，如果不符合规则，`Transaction`就会（立即）中止。

在绝大多数用例中，你不必考虑访问控制或授权区的问题。系统会自动处理。

## Passing by Intent
有时你的确需要你的代码去查看调用者使用了什么Proof。例如：你发行了一个NFT badge给你的系统的每一个会员，它包含了一些元数据，比如你的代码会关心用户ID。在这种情况下，你可以在方法中添加一个Proof参数，并且调用者可以在授权区中克隆一个Proof，然后像传递其它参数一样将它传递给方法。我们将这种明确提供证明方法为“passing by intent".

例如：
``` rust
pub fn query_ticket(&mut self, ticket: Proof) {
    // Make sure the provided proof is of the right resource address
    let checked_proof = ticket.check(self.ticket_manager.resource_address());

    // Get the data associated with the passed NFT proof
    let non_fungible: NonFungible<TicketData> = validated_ticket.non_fungible()
    let ticket_data: TicketData = non_fungible.data();

    info!("You inserted {} XRD into this component", ticket_data.xrd_amount);
}
```

在我们用non_fungible().data()获得Proof的数据之前, 我们必须先进行验证。如果不这么做，调用者可以传递任何带有`xrd_amount`字段的NFT，从而欺骗我们的组件。

## Limitations on Proof Visibility

由于Proofs提供了对特权方法或资源操作的授权，因此了解如何允许它们移动，谁能看到它们以及如何防止它们被恶意或有漏洞的组件滥用非常重要。

简而言之，Proof可以在调用栈中随意“向上”移动多次，但只能“向下”移动一次。也就是说你的Transaction manifest调用了Alpha组件上的一个方法，而这个方法将一个Proof返回到授权区，那么之后从你的Manifest调用组件Bravo就可以使用该证明。但是如果Bravo直接调用组件Charile，那么在确定是否可以调用Charlie的方法时，将不会考虑该证明，也不会考虑Charlie方法中任何特权资源操作。

这一逻辑同样适用于`pass a Proof by intent`,  直接在方法参数中传递Proof的情况。你调用的方法可以看到并使用这个Proof进行资源操作，但是不能将其传递给其它组件，也不能用它访问特权方法。

在执行过程中，任何被调用的组件都有自己的本地授权区域，可以添加或移除。因此，如果Bravo需要调用Charlie一个方法，并且需要一个badge去执行。它就必须创建一个badge的Proof，并且把它放在自己的本地授权区中。

这听起来比实际操作要复杂得多。你直接调用的东西可以使用你的受权区（如果直接从manifest调用，则使用Transaction授权区；如果是组件间调用，则使用组件的本地授权区）。如果你调用的组件想调用其它组件，那么它们就需要自己进行授权。换句话说，**如果你向某个组件传递了你的用户ID的badg的Proof，你就能确信它不用伪装成你调用其它组件。**


# Setting Access Rules
前面说了一般情况下如何在你的组件上设置验证身份验证，现在到了构造部分。我们需要将rule和AccessRule映射。前面简单提到过AccessRule允许我们指定角色访问权限方法所需的badge，AccessRule可以基于badge的存在高效地执行布尔条件，更确切地说，这些都是badge的Proof.

例如：

``` rust
// -- snip --
.instantiate()
.prepare_to_globalize(
    OwnerRole::Fixed(
        rule!(require(owner_badge.resource_address())
    )
))
.roles(
    roles!(
        super_admin => rule!(require_amount(dec!(2), super_admin_badge.resource_address()));
        admin => rule!(require(admin_manager.resource_address()));
    )
)
.globalize();

```

注意：OWNER（同样也包括admin)的AccessRule与superadmin的有所不同，对于owner，我们只需要owner_badge的proof，但是对于superadmin，我们不仅需要super_admin_badge的Proof，还需要两个Proof的数量。我们可以从中指定许多AccessRule要求：

| Rule	                                    | Description                                              |
| ----------------------------------------- | -------------------------------------------------------- |
| require(single resource)                  | TRUE if the specified resource is present                |
| require_any_of(list of resources)         | TRUE if any resource in the list is present     |
| require_all_of(list of resources)         | TRUE if every resource in the list is present   |
| require_n_of(n, list_of_resources)        | TRUE if at least n resources in the list are present |
| require_amount(quantity, single resource) | TRUE if the specified resource is present in at least the given quantity |
| allow_all or AccessRule::AllowAll         | TRUE always                              |
| deny_all or AccessRule::DenyAll           | FALSE always                             |

这些资源（或者资源列表）可以是指定的静态值（给出的精确资源地址）或者可能引用组件中的变量。 许多规则可能用逻辑运算符(&& , || )联合起来，并且用`()`嵌套。这里没有逻辑“非”操作符。

## 示例
这是一个定义复杂规则的示例，我们设置访问规则去访问组件上的方法。为了能去调用`ban_member`，调用者必须出示一个他们拥有一个admin badge或者moderator badge的Proof。去调用`destory`方法的调用者必须出示一个admin badge和两个moderator badges.

``` rust
#[blueprint]
mod rad_flix{
    enable_method_auth! {
        roles {
            auth1 => updatable_by: [];
            auth2 => updatable_by: [];
        },
        methods {
            ban_member => restrict_to: [admin, moderator]
            destroy => restrict_to: [admin, moderator]
        }

        // -- snip --

    pub fn instantiate() -> (Global<RadFlix>, Bucket, Bucket) {
        // Create the access badges
        let admin_badge: Bucket = ResourceBuilder::new_fungible(OwnerRole::None)
            .initial_supply(1);
        let moderator_badges: Bucket = ResourceBuilder::new_fungible(OwnerRole::None)
            .initial_supply(4);

        let admin_badge_address = admin_badge.resource_address();
        let moderator_badge_address = moderator_badges.resource_address();

        // Instantiate the component
        let component = Self {}
            .instantiate()
            .prepare_to_globalize(OwnerRole::None)
            .roles(
                roles!(
                    auth1 => rule!(require_any_of(vec![admin_badge_address, moderator_badge_address]));
                    auth2 => rule!(require(admin_badge_address) && require_amount(dec!(2), moderator_badge_address));
                )
            .globalize();

        (component, admin_badge, moderator_badges)
}
```


# Restricting Component Methods(限制组件方法)
组件包含一些方法，一些方法是公开的，有一些则要求一定程度的保护访问。Roles的构建是为了定义不同方法的访问层次。有些组件只要一个简单验证模型，在这种情况下，只需要定义一个角色。下面将介绍如何为组件设置验证。

## 设置组件验证
考虑一个`TokenSale`组件，顾名思义它被设计为销售token。TokenSale组件有几个方法，需要不同级别访问权限，下表总结了这些方法以及我们希望这些方法的访问级别：

| Method	     |  Description	                | Access Level      |
| -------------- | ---------------------------- | ----------------- |
| buy(payment)	 | Allows users to purchase a token. | Public       |
| create_admin	 | Creates a single admin badge. | Super Admin & Owner |
| change_price(new_price_per_token) | Changes the price of the tokens for sale.	 | Admin, Super Admin, & Owner |
| redeem_profits(amount) | Take profits from collected sales. | Owner   |


这些方法有不同级别的访问权限，我们希望在它们的访问权限之间创建一些边界。设置角色需要调用`enable_method_auth!`宏，我们调用这个宏在mod的下方和blueprint的上方，这是我们定义auth的地方。

``` rust
#[blueprint]
mod token_sale
    enable_method_auth! {
        roles {
            super_admin => updatable_by: [];
            admin => updatable_by: [super_admin];
        },
        methods {
            buy => PUBLIC;
            create_admin => restrict_to: [super_admin, OWNER];
            change_price => restrict_to: [admin, super_admin, OWNER];
            redeem_profits => OWNER;
        }
    struct TokenSale {
        ..
    // -- snip --
    }
}
```

1. `Roles`被定义为创建一个RoleList结构，这就是整个命名方案，有一个事情记在脑中：OWNER角色不需要去定义。
2. 在方法`methods`这个结构里， 方法被映射到它们关联的角色。这里定义了5个角色变量：`PUBLIC`, `OWNER`, `SELF`, `NOBODY`和`RoleList`.
3. 你可以映射一个方法到多个角色，这意味着这些角色都可以访问这个方法。

在定义了Role并将方法关联到各自的角色后，下一步就是构建构造器。它定义了角色访问方法所需要的badge。在此之前回顾一下角色变量及其含义：


|     Variant	  |  Description                                                                |
| --------------- | --------------------------------------------------------------------------- |
| PUBLIC          | Associating a method to the PUBLIC role sets the method to be freely accessed by anyone. |
| OWNER           | The OWNER is a unique role, which assumes control over the blueprint or component.  Methods assigned to the OWNER role not only assumes access to the method, but also to the component modules such as Authority, Metadata, and Royalties. |
|SELF             | The SELF role refers to the package or component itself. Scrypto has a concept of virtual badges which provides the package/component permission to act on behalf of itself.  |
| NOBODY          | Associating methods to NOBODY assumes that no one can access the method. |
| RoleList        | The RoleList is the list of roles defined in the roles struct. |

## 设置权限模块
在定义了角色并将方法映射到各自的角色后，我们就可以构建 Authority 模块了。在这个模块，能映射每个Role到一个AccessRule。实际上，我们是在为每个角色设置访问各自方法所需的Badge。这好比把不同的安全许可交给员工，其中有些Badge的安全级别高于其它的Badge.

``` rust
// -- snip --
.instantiate()
.prepare_to_globalize(
    OwnerRole::Fixed(
        rule!(require(owner_badge.resource_address())
    )
))
.roles(
    roles!(
        super_admin => rule!(
            require_amount(dec!(2), super_admin_badge.resource_address())
        );
        admin => rule!(
            require(admin_manager.resource_address())
        );
    )
)
.globalize();

```
	1. 前面提到过`OWNER`角色是唯一的，它不仅能访问受保护方法，更重要的，他们能访问组件的Authority, Metadata, Royalty模块。
	2. Roles被映射到AccessRule，这个访问规则规定了这个角色所需要的badge.
	3. 您可以为一个角色设置多个badge，以分散对特定角色的控制。

由于OWNER角色是角色系统的一部分，但其构造与其它角色不同，它可能会让人困惑。OWNER 和 SELF 这样的角色已经植入到引擎内部。 它们不需要定义，但是他们需要对与之关联的badge有一些规范。当然你的组件不必须要有一个OWNER, 这种情况下，你可以简单地传递 OwnerRole: None， 这样你或者任何人都无法配置Authority, Metadata和Royalty模块。

## 给Blueprint Package设置验证
上文介绍了如何在组件级别设置验证。不过，我们也可以在Blueprint Package一级设置验证。使用Blueprint Package时，无需定义角色。我们只需将方法映射到其 AccessRule 即可。

```
mod fund_manager {
    enable_function_auth! {
        instantiate_fund => rule!(require(fund_manager_badge.resource_address()));
    }
 // -- snip --
}
```

# 限制资源操作
资源可以对各种操作单独设置访问权限，开发者可以通过设置或调整这些权限来改变其行为。固定供应的资源是一种永久禁用铸造和燃烧的资源。而全局冻结资源则是指某些机构有能力在全球范围内更改提款或存款权限的资源。

| Rule      |    Description                                       |	Default        |
| --------- | ---------------------------------------------------- | ----------------- |
| minter    | Who may mint additional supply                       | AccessRule::DenyAll |
|burner     | Who may destroy some quantity	                       |AccessRule::DenyAll |
|withdrawer | Who may take the resource from a vault               |AccessRule::AllowAll |
|depositor  | Who may put the resource in a vault                  |AccessRule::AllowAll |
|recaller   | Who may remotely seize the resource from any vault   | AccessRule::DenyAll |
|freezer    | Who may remotely freeze withdraws/deposits or burns from any vault | AccessRule::DenyAll |
|non_fungible_data_updater (non-fungible only) | Who may update individual metadata on a non-fungible.	|AccessRule::DenyAll |

每个操作都可以定义任意的访问规则。每个操作还有一个配对的“_updater"角色，可用于更新该角色和updater角色本身的访问规则。例如: minter_update角色可更新minter和minter_updater.

系统以帐本外产品（如钱包）可以理解的方式公开这些访问规则。也就是说，如果某个token能够铸造更多供应量，但永不会燃烧，那么用户就可以在钱包中立即看到，而无需去阅读Scrypto代码。如果一个Token可以自由地转让，但存在一个机构可以将其改变为冻结状态，这对于用户来说，也直接可见。

下面用例子很好地描述。
我们将从可变供应的Token开始，在这种代币中，单一机构有能力铸造新币或烧毁持有的Token.

``` rust
ResourceBuilder::new_fungible(OwnerRole::None)
  .metadata(metadata!(
    init {
        "name" => "My Token", locked;
    }
  ))
  .mint_roles(mint_roles!(
     minter => rule!(require(badge_address));
     minter_updater => rule!(deny_all);
  ))
  .burn_roles(burn_roles!(
     burner => rule!(require(badge_address));
     burner_updater => rule!(deny_all);
  ))
  .initial_supply(100);
```

你为访问方法(.mint_roles()、burn_roles()等)提供的第一个参数执行操作的规则，第二参数是更新前一个规则时要检查的规则。在rule!宏中，我们指定了为执行操作而必须取值为“true"的逻辑，我们可以任意组合不同的条件，例如：

``` rust
  .mint_roles(mint_roles!(
     minter => rule!(require_n_of(2, "list_of_admins") || require(super_admin));
     minter_updater => rule!(deny_all);
  ))
```

在上面的例子中，为了铸造新币，你需要任意2个badge放到vec变量中，或者super admin badge.

让我们来看一个Token，在这个Token中，铸造是项特权操作--由所有者执行，但是任何持有者都可以自由燃烧。

``` rust
ResourceBuilder::new_fungible(
    OwnerRole::Fixed(rule!(require(admin)))
  )
  .metadata(metadata!(
    init {
        "name" => "Mutable supply, single mint authority, burnable by any holder", locked;
    }
  ))
  .mint_roles(mint_roles!(
     minter => OWNER;
     minter_updater => rule!(deny_all);
  ))
  .burn_roles(burn_roles!(
     burner => rule!(allow_all);
     burner_updater => rule!(deny_all);
  ))
  .initial_supply(100);
```


或者，你想分发一种Token，用户可以安全地收起，但是收起后无法转移：

``` rust
ResourceBuilder::new_fungible(OwnerRole::None)
  .metadata(metadata!(
    init {
        "name" => "Token which can not be withdrawn from a vault once stored", locked;
    }
  ))
  .withdraw_roles(withdraw_roles! {
      withdrawer => rule!(deny_all);
      withdrawer_updater => rule!(deny_all);
  })
  .initial_supply(1)
```

## 在创建资源之后更新访问规则
直到现在，我们指出的_updater的所有规则都设置为： rule!(deny_all).  这意味着它永远不会被改变，然而你可以用rule!()宏提供一个自定义规则替代 rule!(deny_all)，你在此提供的权限可以在将来更新规则，他们可以随意更改规则，也有权在任何时候将 _updater角色规则改为 rule!(deny_all)，使其永不再被更改。这意味着它将在钱包中显示为固定值。

锁定是单向过程，一旦被锁定后，不能再回到可修改的规则。

``` rust
// Initial creation, rule_admin is some badge address we have previously defined
let resource_address = ResourceBuilder::new_fungible(OwnerRole::None)
  .metadata(metadata!(
    init {
        "name" => "Globally freezable token", locked;
    }
  ))
  .withdraw_roles(withdraw_roles! {
      withdrawer => rule!(allow_all);
      withdrawer_updater => rule!(require(rule_admin));
  })
  .deposit_roles(deposit_roles! {
      depositor => rule!(allow_all);
      depositor_updater => rule!(require(rule_admin));
  })
  .create_with_no_initial_supply();

... // Later in the code

// `rule_admin_vault` is a vault that contains the badge allowed to make the following changes.
self.rule_admin_vault.authorize(|| {
  // Freeze the token, so no one may withdraw or deposit it
  let resource_manager = ResourceManager::from_address(resource_address);
  resource_manager.set_depositable(AccessRule::DenyAll);
  resource_manager.set_withdrawable(AccessRule::DenyAll);

  // ...or, make it so only a person presenting the proper badge can withdraw or deposit
  resource_manager.set_depositable(rule!(require(transfer_badge)));
  resource_manager.set_withdrawable(rule!(require(transfer_badge)));

  // Unfreeze the token!
  resource_manager.set_depositable(AccessRule::AllowAll);
  resource_manager.set_withdrawable(AccessRule::AllowAll);

  // Lock the token in the unfrozen state, so it may never again be changed
  resource_manager.lock_depositable();
  resource_manager.lock_withdrawable();
});

```

## authorize()方法
在上面的示例中，你看到用了rule_admin_vault.authorize方法，这个方法在Vaults和Buckets上可用，并且被用于在授权区临时放一个目标资源的proof去做验证。

这个方法以一个不带参数的闭包作为参数，并将Proof放在授权区之后运行，运行完闭包后，proof将从授权区移除。


## Transient tokens
还有一些更疯狂的事情，创建一个从来不能被作为存款的NFT。

``` rust 
ResourceBuilder::new_integer_non_fungible::<TransientToken>(OwnerRole::None)
  .metadata(metadata!(
    init {
        "name" => "Undepositable token", locked;
    }
  ))
  .mint_roles(mint_roles!(
     minter => rule!(require(admin));
     minter_updater => rule!(deny_all);
  ))
  .burn_roles(burn_roles!(
     burner => rule!(require(admin));
     burner_updater => rule!(deny_all);
  ))
  .deposit_roles(deposit_roles! {
      depositor => rule!(deny_all);
      depositor_updater => rule!(deny_all);
  })
  .create_with_no_initial_supply();
```

这个token被称为Transient（瞬时 ) token，你会好奇它有什么用呢？请记住Radix引擎保证一个Token从来不会在事务结束时“悬空”，（在操作结束时）任何飘浮在Vault外的资源都会导致Transaction中止。通过向你的调用者提供一个永远无法存入但由你控制烧毁权限的token，你可迫使他们最终调用一个可以烧毁token的方法，如果不这样做，那么事务将会失败。

这允许你实现一些很酷的功能，比如以面向资产的方式对闪电贷进行编程。示例。

闪电贷是一个瞬时资源效用的明显例子（借出资金，让调用者做他们想做的任何事，但他们必须在交易完成前还钱给你并加一定费用），这种模式也适合于拥有生态系统组件的操作，你希望激励人们一起使用它们。

例如，你可能有一个定期接收帐本外价格和费用的Oracle，消费者从它那里获取价格信息也需要费用。你也可以控制某种token的销售组件，你可能在Oracle上设置一个特殊入口能够免费返回价格，但也同时给出一个瞬时Token，面销毁瞬时Token的唯一方法就是将它传递给Token销售组件，只要有足够价值的购买行为，Token销售组件就会烧毁它。换句话说，你可以让人们免费使用你的Oracle，（只要）他们使用价格信息在你的其它组件上做交易。







---
title: "权限系统设计模型分析（DAC，MAC，RBAC，ABAC）"
date: 2021-07-14
draft: false
tags:
  - 权限系统
  - RBAC
  - ABAC
---
## 术语

这里对后面会用到的词汇做一个说明，老司机请直接翻到**常见设计模式**。

### 用户

发起操作的主体。

### 对象（Subject）

指操作所针对的客体对象，比如订单数据或图片文件。

### 权限控制表 (ACL: Access Control List)

用来描述权限规则或用户和权限之间关系的数据表。

### 权限 (Permission)

用来指代对某种对象的某一种操作，例如“添加文章的操作”。

### 权限标识

权限的代号，例如用“ARTICLE_ADD”来指代“添加文章的操作”权限。

## 常见设计模式

### 自主访问控制（DAC: Discretionary Access Control）

系统会识别用户，然后根据被操作对象（Subject）的权限控制列表（ACL: Access Control List）或者权限控制矩阵（ACL: Access Control Matrix）的信息来决定用户的是否能对其进行哪些操作，例如读取或修改。

而拥有对象权限的用户，又可以将该对象的权限分配给其他用户，所以称之为“自主（Discretionary）”控制。

这种设计最常见的应用就是文件系统的权限设计，如微软的NTFS。

DAC最大缺陷就是对权限控制比较分散，不便于管理，比如无法简单地将一组文件设置统一的权限开放给指定的一群用户。

![](/uploads/2021/12/806979267.png)

Windows的文件权限

### 强制访问控制（MAC: Mandatory Access Control）

MAC是为了弥补DAC权限控制过于分散的问题而诞生的。在MAC的设计中，每一个对象都都有一些权限标识，每个用户同样也会有一些权限标识，而用户能否对该对象进行操作取决于双方的权限标识的关系，这个限制判断通常是由系统硬性限制的。比如在影视作品中我们经常能看到特工在查询机密文件时，屏幕提示需要“无法访问，需要一级安全许可”，这个例子中，文件上就有“一级安全许可”的权限标识，而用户并不具有。

MAC非常适合机密机构或者其他等级观念强烈的行业，但对于类似商业服务系统，则因为不够灵活而不能适用。

![](/uploads/2021/12/245196935.png)

RedHat MLS

>

> [Red Hat: MLS](https://link.jianshu.com?t=https://www.centos.org/docs/5/html/Deployment_Guide-en-US/sec-mls-ov.html)

### 基于角色的访问控制（RBAC: Role-Based Access Control)

因为DAC和MAC的诸多限制，于是诞生了RBAC，并且成为了迄今为止最为普及的权限设计模型。

RBAC在用户和权限之间引入了“角色（Role）”的概念（暂时忽略Session这个概念）：

![](/uploads/2021/12/2221016572.png)

RBAC核心设计

>

> 图片来自[Apache Directory](https://link.jianshu.com?t=http://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html)

如图所示，每个用户关联一个或多个角色，每个角色关联一个或多个权限，从而可以实现了非常灵活的权限管理。角色可以根据实际业务需求灵活创建，这样就省去了每新增一个用户就要关联一遍所有权限的麻烦。简单来说RBAC就是：用户关联角色，角色关联权限。另外，RBAC是可以模拟出DAC和MAC的效果的。

例如数据库软件MongoDB便是采用RBAC模型，对数据库的操作都划分成了权限（[MongoDB权限文档](https://link.jianshu.com?t=https://docs.mongodb.com/manual/reference/privilege-actions/)）：


| 权限标识 | 说明 |
| - | - |
| find | 具有此权限的用户可以运行所有和查询有关的命令，如：aggregate、checkShardingIndex、count等。 |
| insert | 具有此权限的用户可以运行所有和新建数据有关的命令：insert和create等。 |
| collStats | 具有此权限的用户可以对指定database或collection执行collStats命令。 |
| viewRole | 具有此权限的用户可以查看指定database的角色信息。 |
| … |   |

基于这些权限，MongoDB提供了一些预定义的角色（[MongoDB预定义角色文档](https://link.jianshu.com?t=https://docs.mongodb.com/manual/reference/built-in-roles/)，用户也可以自己定义角色）：


| 角色 | find | insert | collStats | viewRole | … |
| - | - | - | - | - | - |
| read | ✔ |   | ✔ |   | … |
| readWrite | ✔ | ✔ | ✔ |   | … |
| dbAdmin | ✔ |   | ✔ |   | … |
| userAdmin |   |   |   | ✔ | … |

最后授予用户不同的角色，就可以实现不同粒度的权限分配了。

目前市面上绝大部分系统在设计权限系统时都采用RBAC模型。然而也有的系统错误地实现了RBAC，他们采用的是判断用户是否具有某个角色而不是判断权限，例如以下代码：

**

```php
<?php

if ($user->hasRole('hr')) {
    // 执行某种只有“HR”角色才能做的功能，例如给员工涨薪…
    // ...
}
```

如果后期公司规定部门经理也可以给员工涨薪，这时就不得不修改代码了。

以上基本就是RBAC的核心设计（RBAC Core）。而基于核心概念之上，RBAC规范还提供了扩展模式。

#### 角色继承(Hierarchical Role)

![](/uploads/2021/12/2238077731.png)

RBAC 1

>

> 带有角色继承的RBAC。图片来自[Apache Directory](https://link.jianshu.com?t=http://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html)

顾名思义，角色继承就是指角色可以继承于其他角色，在拥有其他角色权限的同时，自己还可以关联额外的权限。这种设计可以给角色分组和分层，一定程度简化了权限管理工作。

#### 职责分离(Separation of Duty)

为了避免用户拥有过多权限而产生利益冲突，例如一个篮球运动员同时拥有裁判的权限（看一眼就给你判犯规狠不狠？），另一种职责分离扩展版的RBAC被提出。

职责分离有两种模式：

* 静态职责分离(Static Separation of Duty)：用户无法同时被赋予有冲突的角色。
* 动态职责分离(Dynamic Separation of Duty)：用户在一次会话（Session）中不能同时激活自身所拥有的、互相有冲突的角色，只能选择其一。

![](/uploads/2021/12/1511202318.png)

RBAC 2

>

> 静态职责分离。图片来自[Apache Directory](https://link.jianshu.com?t=http://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html)

![](/uploads/2021/12/4120757801.png)

RBAC 3

>

> 动态职责分离。图片来自[Apache Directory](https://link.jianshu.com?t=http://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html)

讲了这么多RBAC，都还只是在用户和权限之间进行设计，并没有涉及到用户和对象之间的权限判断，而在实际业务系统中限制用户能够使用的对象是很常见的需求。例如华中区域的销售没有权限查询华南区域的客户数据，虽然他们都具有销售的角色，而销售的角色拥有查询客户信息的权限。

那么我们应该怎么办呢？

#### 用户和对象的权限控制

在RBAC标准中并没有涉及到这个内容（RBAC基本只能做到对一类对象的控制），但是这里讲几种基于RBAC的实现方式。

首先我们看看PHP框架[Yii 1.X的解决方案](https://link.jianshu.com?t=http://www.yiiframework.com/wiki/136/getting-to-understand-hierarchical-rbac-scheme/)（2.X中代码更为优雅，但1.X的示例代码更容易看明白）：

**

```php
<?php

$auth=Yii::app()->authManager;
 
$auth->createOperation('createPost','create a post');
$auth->createOperation('readPost','read a post');
$auth->createOperation('updatePost','update a post');
$auth->createOperation('deletePost','delete a post');

// 主要看这里。
// 这里创建了一个名为`updateOwnPost`的权限，并且写了一段代码用来检验用户是否为该帖子的作者
$bizRule='return Yii::app()->user->id==$params["post"]->authID;';
$task=$auth->createTask('updateOwnPost','update a post by author himself',$bizRule);
$task->addChild('updatePost');
 
$role=$auth->createRole('reader');
$role->addChild('readPost');
 
$role=$auth->createRole('author');
$role->addChild('reader');
$role->addChild('createPost');
$role->addChild('updateOwnPost');
 
$role=$auth->createRole('editor');
$role->addChild('reader');
$role->addChild('updatePost');
 
$role=$auth->createRole('admin');
$role->addChild('editor');
$role->addChild('author');
$role->addChild('deletePost');

```

实现效果：

![](/uploads/2021/12/946912186.gif)

Yii 1.X权限图

>

> 图片来自[Yii官方WiKi](https://link.jianshu.com?t=http://www.yiiframework.com/wiki/136/getting-to-understand-hierarchical-rbac-scheme/)

在这个Yii的官方例子中，`updateOwnPost`在判断用户是否具有`updatePost`权限的基础上更进一步判断了用户是否有权限操作这个特定的对象，并且这个判断逻辑是通过代码设置的，非常灵活。

不过大部分时候我们并不需要这样的灵活程度，会带来额外的开发和维护成本，而另一种基于模式匹配规则的对象权限控制可能更适合。例如判断用户是否对Id为123的文章具有编辑的权限，代码可能是这样的：

**

```php
<?php

// 假设articleId是动态获取的
$articleId = 123;

if ($user->can("article:edit:{$articleId}")) {
    // ...
}
```

而给用户授权则有多种方式可以选择：

**

```php
<?php

// 允许用户编辑Id为123的文章
$user->grant('article:edit:123');

// 使用通配符，允许用户编辑所有文章
$user->grant('article:edit:*');

```

虽然不及Yii方案的灵活，但某些场景下这样就够用了。

如果大家还有更好的方案，欢迎在评论中提出。

### 基于属性的权限验证（ABAC: Attribute-Based Access Control）

ABAC被一些人称为是权限系统设计的未来。

不同于常见的将用户通过某种方式关联到权限的方式，ABAC则是通过动态计算一个或一组属性来是否满足某种条件来进行授权判断（可以编写简单的逻辑）。属性通常来说分为四类：用户属性（如用户年龄），环境属性（如当前时间），操作属性（如读取）和对象属性（如一篇文章，又称资源属性），所以理论上能够实现非常灵活的权限控制，几乎能满足所有类型的需求。

例如规则：“允许所有班主任在上课时间自由进出校门”这条规则，其中，“班主任”是用户的角色属性，“上课时间”是环境属性，“进出”是操作属性，而“校门”就是对象属性了。为了实现便捷的规则设置和规则判断执行，ABAC通常有配置文件（XML、YAML等）或DSL配合规则解析引擎使用。XACML（eXtensible Access Control Markup Language）是ABAC的一个实现，但是该设计过于复杂，我还没有完全理解，故不做介绍。

总结一下，ABAC有如下特点：

1. 集中化管理
2. 可以按需实现不同颗粒度的权限控制
3. 不需要预定义判断逻辑，减轻了权限系统的维护成本，特别是在需求经常变化的系统中
4. 定义权限时，不能直观看出用户和对象间的关系
5. 规则如果稍微复杂一点，或者设计混乱，会给管理者维护和追查带来麻烦
6. 权限判断需要实时执行，规则过多会导致性能问题

既然ABAC这么好，那最流行的为什么还是RBAC呢？

我认为主要还是因为大部分系统对权限控制并没有过多的需求，而且ABAC的管理相对来说太复杂了。[Kubernetes便因为ABAC太难用，在`1.8`版本里引入了RBAC的方案](https://link.jianshu.com?t=http://blog.kubernetes.io/2017/04/rbac-support-in-kubernetes.html)。

>

> ABAC有时也被称为PBAC（Policy-Based Access Control）或CBAC（Claims-Based Access Control）。

## 结语

权限系统设计可谓博大精深，这篇文章只是介绍了一点皮毛。

随着人类在信息化道路上越走越远，权限系统的设计也在不断创新，但目前好像处在了平台期。

可能因为在RBAC到ABAC之间有着巨大的鸿沟，无法轻易跨越，也可能是一些基于RBAC的微创新方案还不够规范化从而做到普及。不过在服务化架构的浪潮下，未来这一块必然有极高的需求，也许巨头们已经开始布局了。

## 参考文档

[NTFS文件系统权限](https://link.jianshu.com?t=https://support.microsoft.com/en-us/help/949608/changes-to-the-default-ntfs-discretionary-access-control-list-dacl-set)

[Solaris权限模型](https://link.jianshu.com?t=https://docs.oracle.com/cd/E56344_01/html/E53956/rbac-28.html#scrolltoc)

[百度百科：访问控制](https://link.jianshu.com?t=https://baike.baidu.com/item/%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/8545517)

[Red Hat: Multi-Level Security (MLS)](https://link.jianshu.com?t=https://www.centos.org/docs/5/html/Deployment_Guide-en-US/sec-mls-ov.html)

[冰云：An Introduction To Role-Based Access Control](https://link.jianshu.com?t=http://files.cnblogs.com/files/Wenzy/an_introduction_to_rbac.pdf)

[NIST: Role-Based Access Control](https://link.jianshu.com?t=https://csrc.nist.gov/projects/role-based-access-control)

[MongoDB RBAC](https://link.jianshu.com?t=https://docs.mongodb.com/manual/core/authorization/)

[Stackoverflow: Group vs role Any real difference?](https://link.jianshu.com?t=https://stackoverflow.com/questions/7770728/group-vs-role-any-real-difference)

[Yii: Getting to Understand Hierarchical RBAC Scheme](https://link.jianshu.com?t=http://www.yiiframework.com/wiki/136/getting-to-understand-hierarchical-rbac-scheme/)

[Role-Based Access Control in Computer Security](https://link.jianshu.com?t=http://www.informit.com/articles/article.aspx?p=782116)

[(Syracuse University: Role-Based Access Control (RBAC)](https://link.jianshu.com?t=http://www.cis.syr.edu/~wedu/Teaching/cis643/LectureNotes_New/RBAC.pdf)

[Yii 2.0 Guide](https://link.jianshu.com?t=http://www.yiiframework.com/doc-2.0/guide-security-authorization.html)

[WIKIPEDIA: Computer access control](https://link.jianshu.com?t=https://en.wikipedia.org/wiki/Computer_access_control)

本文转载自[简书](https://www.jianshu.com/p/ce0944b4a903)

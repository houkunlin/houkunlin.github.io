---
title: 基于 MyBatisPlus 数据权限插件实现的自动数据权限（数据范围）查询限制条件功能
date: 2023-09-04 16:34:14
updated: 2023-09-04 16:34:14
tags:
---

**此功能已实现，经测试运行正常**

`MyBatisPlus` 本身有了一个 ` DataPermissionInterceptor` 数据权限插件，但是这个插件默认上是没有任何查询条件拼接的，需要自己去实现一个 `MultiDataPermissionHandler` 接口来创建所需要的条件表达式。

通过这个功能我们可以实现一个数据范围的功能，也就是根据用户的角色、权限不同，来限制用户查询不同的数据范围。

在 `MyBatisPlus` 的文档里面提到了他们的企业级增强特性里面也有这个功能 `企业高级特性-数据范围` 并且增加了注解的支持。

可是，没钱买企业级的来用怎么办，那就自己实现一个吧。

## 预期假设说明

#### 1.1 基本预期

现在，假设，我们想要的拼接效果如下：

仅本人的数据范围，自动拼接如下SQL

```sql
... AND user_id = 1000
... AND create_user_id = 1000
... AND create_by = 1000
```

所在部门的数据范围，自动拼接如下SQL

```sql
... AND dept_id = 10
... AND user_dept_id = 10
```

所在部门和子级部门的数据范围，自动拼接如下SQL

```sql
... AND dept_id IN (1,2,3,10)
... AND user_dept_id IN (1,2,3,10)
```

所在机构的数据范围，自动拼接如下SQL

```sql
... AND org_id = 10
... AND user_org_id = 10
```

所在机构和子级机构的数据范围，自动拼接如下SQL

```sql
... AND org_id IN (1,2,3,10)
... AND user_org_id IN (1,2,3,10)
```

需要的SQL拼接效果如上可能还不行，还得根据权限代码是否匹配来决定是否拼接条件，可以根据API接口的注解来决定是否忽略某个数据库表的拼接，可以自定义数据范围字段代码。

#### 1.2 根据权限代码来决定是否拼接条件

这个需求是参考了若依框架得来的，具体表现为：

1. 系统有角色A（权限a1/a2，范围本人）、角色B（权限b1/b2，范围部门）、角色C（权限c1/c2，范围机构）

2. 用户拥有角色B、C
3. 接口所需要的权限为b2
4. 期望拼接的结果为：查看部门范围数据，由于接口不需要c1/c2权限，因此角色C的数据范围配置不生效

#### 1.3 据API接口的注解来决定是否忽略某个数据库表的数据范围条件拼接

我们对某些特殊接口可能需要忽略数据范围限制，使数据范围配置不生效

1. 系统有角色A（权限a1/a2，范围本人）、角色B（权限b1/b2，范围部门）、角色C（权限c1/c2，范围机构）

2. 用户拥有角色B、C
3. 接口所需要的权限为b2，但是Controller配置了注解，使数据范围忽略处理
4. 期望拼接的结果为：查看全部数据

场景二：

1. 系统有角色A（权限a1/a2，范围本人）、角色B（权限b1/b2，范围部门）、角色C（权限c1/c2，范围机构）

2. 用户拥有角色B、C
3. 接口不需要权限，Controller没有注解
4. 期望拼接的结果为：查看部门和机构范围的数据

#### 1.4 自定义数据范围字段代码

对系统的某些表，他们的数据范围字段可能不规范，与通用的数据范围字段不同，例如，我们默认的数据范围字段为 `user_id/create_by` `org_id` `dept_id` ，但是某个表里面它就偏偏不是这个字段，它可能是 `userid/createby` `orgid` `deptid` 这样的，类似这种不规范让人很头疼，直接改表结构可能又太麻烦。

#### 1.5 代码弱侵入式，自动完成相关功能

参考了若依框架的代码，他是有一个 `BaseEntity` 基类，会把数据范围的条件放到这个基类的 `params` 字段里面，然后手写 XML 把条件加上去。

但是这种方式比较麻烦，代码侵入很强，每个需要数据范围权限的地方都需要去改SQL配置。

而我们想要的是，让他自动完成这个条件拼接，完全不用去定制具体的查询SQL。

并且，若依自定义了 `Spring Security` 注解的内容写法，他是 `@PreAuthorize("@ss.hasPermi('system:role:list')")` 这样的，引用了一个自己 Bean 去对权限代码进行判断，如果是在已有的项目上进行改造，则需要把所有的内容都替换掉。

那我们需要不修改这一部分代码，也就是使用 `Spring Security` 官方的用法，如：`@PreAuthorize("hasAuthority('system:role:list')")` 



## 重点难点分析

分析【1.1 基本预期】，引入了 ` DataPermissionInterceptor` 插件后完全可以达到自动拼接，难点在如何去判断当前用户访问的这个接口到底需要什么权限。

分析【1.2 根据权限代码来决定是否拼接条件】，难点在如何获取Controller上成功匹配的权限代码，如何接管 `@PreAuthorize("hasAuthority('system:role:list')")`  的内容解析处理。

分析【1.3 据API接口的注解来决定是否忽略某个数据库表的数据范围条件拼接】，难点在如何判断这个接口需要忽略数据范围的条件处理。

分析【1.4 自定义数据范围字段代码】，难点在如何根据不同的API用不同的字段，如何根据不同的表名用不同的字段。

分析【1.5 代码弱侵入式，自动完成相关功能】问题，在引入了 ` DataPermissionInterceptor` 插件后可以自动拼接查询条件，也就是可以不把拼接条件放到基类的`params` 里面存储。然后重写 `Spring Security` 的相关类对象，接管 `@PreAuthorize("hasAuthority('system:role:list')")`  的内容解析处理。



## 基本用法预期

在 Controller 上的用法

```java
// 忽略整个 Controller 接口的数据范围限制，并且忽略全部表的限制
@DataScope(ignore = true)
@PreAuthorize("hasAuthority('ROLE_USER')")
@RestController
@RequestMapping("/api/user/")
@RequiredArgsConstructor
public class UserWebApiController {
    // 获取当前用户的基本信息等接口，已经在代码中明确了只查询当前用户的数据，因此可以直接忽略数据范围的处理
}

@RestController
@RequestMapping("/api/dept/")
@RequiredArgsConstructor
public class UserWebApiController {
    /**
     * 获取全部的 <strong>单位部门</strong> 列表
     */
    // 忽略单个接口的限制，并且忽略全部表的限制
    @DataScope(ignore = true)
    @Operation(summary = "树形结构数据")
    @GetMapping("/tree")
    public Collection<DeptListVo> listTree(DeptQuery query) {
        // ......
    }
}
// 忽略对 sys_table1 这个表的处理
@DataScope(tableName = "sys_table1", ignore = true)
// 对 sys_table2 这个表的数据范围权限字段单独控制
@DataScope(tableName = "sys_table2", userColumn = "userid", orgColumn = "orgid", deptColumn = "deptid")
```

重写全局默认数据范围权限字段代码，并设置在全局对某个表进行单独设置

```java
public class CustomDataScopeColumnHandlerImpl extends DataScopeColumnHandlerDefaultImpl {

    @Override
    public String[] getUserIdColumn() {
        return new String[]{"user_id", "create_user_id", "create_by", "created_by"};
    }

    @Override
    public String[] getOrgIdColumn() {
        return new String[]{"org_id"};
    }

    @Override
    public String[] getDeptIdColumn() {
        return new String[]{"dept_id"};
    }

    @Override
    public List<DataScopeColumn> getDataScopeColumns() {
        return List.of(DataScopeColumn.builder()
                        .tableName("sys_table1")
                        .ignore(true)
                        .build(),
                DataScopeColumn.builder()
                        .tableName("sys_table2")
                        .userColumn("userid")
                        .orgColumn("orgid")
                        .deptColumn("deptid")
                        .ignore(false)
                        .build()
        );
    }
}
```



## 基本实现过程

### 重写 Spring Security 的注解内容解析处理功能

参照 `MethodSecurityExpressionRoot` 实现 `MethodSecurityExpressionOperations` 接口功能，把所有代码复制过来，修改 `hasAnyAuthorityName` 方法内容，当权限匹配成功的时候，把权限代码缓存到一个公共对象中

创建一个类继承 `DefaultMethodSecurityExpressionHandler` 类，重写 `createEvaluationContext` 和 `createSecurityExpressionRoot` 方法，返回我们的 `MethodSecurityExpressionOperations` 接口实现类

### MultiDataPermissionHandler 功能实现

从数据库中查询到当前用户的权限等信息，根据角色权限去判断是否需要增加限制条件，然后根据字段配置获取这个表的数据范围字段代码，最后拼接条件信息。

### 代码文件名称清单

- `CustomMethodSecurityExpressionRoot` 方法权限 `@PreAuthorize` 内容表达式根数据对象
- `CustomMethodSecurityExpressionHandler` 方法权限 `@PreAuthorize` 内容表达式处理器
- `DataScope` 数据范围注解
- `DataScopeAop` 数据范围AOP切面处理
- `DataScopeColumn` 与`DataScopeAop`数据范围注解的基本一致，用来提供全局配置
- `DataScopeColumnHandler` 数据范围字段处理接口
- `DataScopeColumnHandlerAbstract` 数据范围字段处理接口抽象实现
- `DataScopeColumnHandlerDefaultImpl` 数据范围字段处理接口默认实现
- `DataScopeColumnType` 数据范围字段代码类型：用户字段、部门字段、机构字段
- `DataScopeMyBatisPlusPlugin` MultiDataPermissionHandler的实现
- `DataScopes` 数据范围注解
- `DataScopeUtil` 抽取出来的几个公共方法
- `DataScopeVo` 数据范围功能处理时所需要的用户权限信息数据对象，包括用户id、部门id、子级部门id列表、机构id、子级机构id列表、角色列表、角色对应的权限代码列表
- `DataScopeType` 数据范围枚举类型：本人数据、部门数据、机构数据
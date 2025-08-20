在很多项目中都需要有严格的权限管理机制，需要有一套鉴权系统，比较典型的就是RBAC，但是我们是一个电商项目，在权限这一块没那么复杂，权限的分配也没那么多，所以就直接按照RBAC的模式写死了角色和权限。



我们实现了一套简单的权限控制体系，但是用到的技术方案也是业内通用的方案，即使用 SaToken 进行鉴权，并且在SpringCloudGateway中实现统一鉴权。



为了简单，我们把我们的角色和权限分为以下这些：



角色(UserRole)：

+ ADMIN：管理员
+ CUSTOMER：普通用户



权限（UserPermission）

+ BASIC：基本权限
+ AUTH：已实名认证权限
+ FROZEN：被冻结用户权限
+ NONE：没有任何权限



在 gateway 的应用中，我们定义一个StpInterfaceImpl，让他实现cn.dev33.satoken.stp.StpInterface这个接口。在其中实现权限和角色的获取：



```java
@Component
public class StpInterfaceImpl implements StpInterface {
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        UserInfo userInfo = (UserInfo) StpUtil.getSessionByLoginId(loginId).get((String) loginId);

        if (userInfo.getUserRole() == UserRole.ADMIN || userInfo.getState().equals(UserStateEnum.ACTIVE.name())) {
            return List.of(UserPermission.BASIC.name(), UserPermission.AUTH.name());
        }

        if (userInfo.getState().equals(UserStateEnum.INIT.name())) {
            return List.of(UserPermission.BASIC.name());
        }

        if (userInfo.getState().equals(UserStateEnum.FROZEN.name())) {
            return List.of(UserPermission.FROZEN.name());
        }

        return List.of(UserPermission.NONE.name());
    }

    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        UserInfo userInfo = (UserInfo) StpUtil.getSessionByLoginId(loginId).get((String) loginId);
        if (userInfo.getUserRole() == UserRole.ADMIN) {
            return List.of(UserRole.ADMIN.name());
        }
        return Collections.emptyList();
    }
}
```



这里的代码很简单，首先是角色的判断，如果是 ADMIN 的角色，那么我们就在getRoleList中返回 ADMIN 给他，如果不是，则返回 CUSTOMER 的角色给他。



接下来是权限的判断，我们根据用户的角色、状态、来进行权限的控制，有以下规则：

+ 已经认证用户：拥有全部权限，即 AUTH+BASIC
+ 未认证用户：只拥有 BASIC 权限
+ 被冻结用户：只拥有 FROZEN 权限



接下来，我们在定义一个 sa-token 的全局配置，这里进行权限的校验：

```java
/**
 * sa-token的全局配置
 */
@Configuration
public class SaTokenConfigure {

    @Bean
    public SaReactorFilter getSaReactorFilter() {
        return new SaReactorFilter()
                // 拦截地址
                .addInclude("/**")
                // 开放地址
                .addExclude("/favicon.ico")
                // 鉴权方法：每次访问进入
                .setAuth(obj -> {
                    // 登录校验 -- 拦截所有路由，并排除/auth/login 用于开放登录
                    SaRouter.match("/**").notMatch("/auth/**", "/collection/collectionList", "/collection/collectionInfo", "/wxPay/**").check(r -> StpUtil.checkLogin());

                    // 权限认证 -- 不同模块, 校验不同权限
                    SaRouter.match("/admin/**", r -> StpUtil.checkRole(UserRole.ADMIN.name()));
                    SaRouter.match("/trade/**", r -> StpUtil.checkPermission(UserPermission.AUTH.name()));

                    SaRouter.match("/user/**", r -> StpUtil.checkPermission(UserPermission.BASIC.name()));
                    SaRouter.match("/orders/**", r -> StpUtil.checkPermission(UserPermission.BASIC.name()));

                })
                // 异常处理方法：每次setAuth函数出现异常时进入
                .setError(this::getSaResult);
    }

    private SaResult getSaResult(Throwable throwable) {
        switch (throwable) {
            case NotLoginException notLoginException:
                return SaResult.error("请先登录");
            case NotRoleException notRoleException:
                if (notRoleException.getRole().equals("ADMIN")) {
                    return SaResult.error("请勿越权使用！");
                }
                return SaResult.error("您无权限进行此操作！");
            case NotPermissionException notPermissionException:
                if (notPermissionException.getPermission().equals("AUTH")) {
                    return SaResult.error("请先完成实名认证！");
                }
                return SaResult.error("您无权限进行此操作！");
            default:
                return SaResult.error(throwable.getMessage());
        }
    }
}
```



主要鉴权部分代码：

```java
// 权限认证 -- 不同模块, 校验不同权限
SaRouter.match("/admin/**", r -> StpUtil.checkRole(UserRole.ADMIN.name()));
SaRouter.match("/trade/**", r -> StpUtil.checkPermission(UserPermission.AUTH.name()));

SaRouter.match("/user/**", r -> StpUtil.checkPermission(UserPermission.BASIC.name()));
SaRouter.match("/orders/**", r -> StpUtil.checkPermission(UserPermission.BASIC.name()));
```



其实很简单，就是根据用户访问的不同模块（路径），来判断不同的权限即可。



并且为了让错误提示更加友好，定义了getSaResult方法，来对错误信息进行处理，这里为什么不用GlobalWebExceptionHandler这种形式呢？



主要是因为这种只有在 Spring MVC 应用中才能生效，而我们的 gateway 他不是个 web 应用，他是个 webflux应用。



这样，如果一个未做实名认证的用户尝试下单，即访问/trade/pay 接口，则会提示：



```json
{"code": 500, "msg": "请先完成实名认证！", "data": null}
```


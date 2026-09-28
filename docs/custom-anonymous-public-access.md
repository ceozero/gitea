# 公开资源匿名访问定制说明

本文件记录本实例相对 Gitea 官方源码的认证定制，供后续升级、拉取上游并合并时核对。

## 目标

当 `[service] REQUIRE_SIGNIN_VIEW = true` 时，默认仍要求登录访问 Gitea；仅在显式开启下列配置后，允许匿名读取公开资源：

```ini
[service]
REQUIRE_SIGNIN_VIEW = true
ALLOW_ANONYMOUS_PUBLIC_RAW = true
ALLOW_ANONYMOUS_PUBLIC_CONTAINER_PULL = true
```

两个扩展配置默认均为 `false`，不改变官方默认行为。

| 配置项 | 开放范围 | 不开放的范围 |
| --- | --- | --- |
| `ALLOW_ANONYMOUS_PUBLIC_RAW` | 公开仓库的 `GET /{owner}/{repo}/raw/...` | 私有仓库、仓库页面、写操作 |
| `ALLOW_ANONYMOUS_PUBLIC_CONTAINER_PULL` | 公开归属者的 OCI/Docker V2 镜像 manifest/blob 拉取 | 私有归属者、上传、修改、删除、`/_catalog`、`tags/list` |

Gitea 容器仓库使用 OCI/Docker V2 `/v2` 协议；本定制不增加 Docker V1 支持。

## 与官方源码的差异

所有生产代码差异均以 `本实例扩展` 注释标识。升级合并后可执行：

```powershell
rg -n "本实例扩展|ALLOW_ANONYMOUS_PUBLIC" modules routers services custom
```

| 文件 | 定制内容 |
| --- | --- |
| `modules/setting/service.go` | 新增并读取两个布尔配置项。 |
| `custom/conf/app.example.ini` | 声明两个配置项及其安全边界。 |
| `routers/web/web.go` | 将仓库 raw 路由从通用代码页面路由组拆出；配置开启时仅让该路由使用可选登录中间件。仍执行 `RepoAssignment`、`repo.MustBeNotEmpty` 与 `reqUnitCodeReader`。 |
| `routers/api/packages/container/container.go` | 强制登录模式下可为匿名公开镜像拉取签发并接受 Ghost token；目录/标签发现仍拒绝匿名请求。 |
| `routers/api/packages/api.go` | `/_catalog` 与 `tags/list` 改为使用发现权限中间件，避免公开拉取配置变成资源枚举开关。 |
| `services/context/package.go` | 公开镜像拉取开启时，允许容器 Ghost token 按归属者公开性获得只读包权限；私有归属者仍无权限。 |
| `modules/setting/service_test.go` | 覆盖两个配置项的默认值与启用后的读取结果。 |
| `tests/integration/signin_test.go` | 覆盖强制登录时公开仓库 raw 的匿名读取，以及私有仓库的拒绝访问。 |
| `tests/integration/api_packages_container_test.go` | 覆盖公开镜像匿名拉取、私有镜像拒绝及目录/标签枚举拒绝。 |

## 升级合并检查清单

1. 合并上游前保存当前分支或创建备份分支。
2. 逐个检查上述文件是否出现冲突，优先保留上游安全修复，再重新应用本实例扩展。
3. 特别检查 Gitea 是否调整了以下路由或权限实现：
   - `/{username}/{reponame}/raw`
   - `ContainerRoutes`、`ReqContainerAccess`、容器 token 签发
   - `PackageAssignment`、`determineAccessMode`
4. 不要将 `/api`、`/api/v1`、`/git` 或整个 `/v2` 加入匿名路径白名单；这些路径包含写接口、令牌接口或私有资源。
5. 上游若新增包级可见性或容器鉴权逻辑，应优先采用其安全模型，并确认本扩展仍只允许公开资源只读访问。

## 回归验证

每次合并后至少执行：

```powershell
go test -run '^TestLoadServiceRequireSignInView$' ./modules/setting/
go test -run '^TestRequireSignInView$' ./tests/integration/
go test -run '^TestPackageContainer$' ./tests/integration/
git diff --check
```

验证预期：

- 未登录可读取公开仓库 raw 文件。
- 未登录访问私有仓库 raw 文件失败。
- 未登录可使用匿名容器 token 拉取公开镜像 manifest/blob。
- 未登录无法拉取私有归属者镜像，无法写入镜像，也无法调用 `/_catalog` 或 `tags/list`。

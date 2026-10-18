# Scala 相关配置

## Nexus

~/.sbt/repositories
```
[repositories]
local
my-ivy-proxy-releases: <Nexus 上的 sbt-ivy 仓库地址>, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
my-maven-proxy-releases: <Nexus 上的 central 代理仓库地址>
```

.sbtopts
```
-Dsbt.override.build.repos=true
```

要点：

- 自定义仓库 id（`my-ivy-proxy-releases:`、`my-maven-proxy-releases:`）由自己命名，sbt 会把解析请求路由到对应 URL；仓库地址按实际部署环境替换。
- Ivy 模式串里的 `scala_[scalaVersion]`、`sbt_[sbtVersion]` 段可按成交互开启，用于在同一仓库中按 Scala / sbt 版本分区存放。
- `-Dsbt.override.build.repos=true` 让 sbt 忽略默认仓库，只使用 `~/.sbt/repositories` 中声明的仓库，这是强制走私有 Nexus 的标准做法。
- `local` 行保留本地 Ivy 缓存优先解析。

历史备注：原先记录的 `repo.typesafe.com/typesafe/ivy-releases/` 等公共端点已随发行渠道迁移而失效，此处不再保留具体地址。
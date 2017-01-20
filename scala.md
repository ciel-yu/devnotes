# Scala 相关配置

## Nexus

~/.sbt/repositories
```
[repositories]
local
my-ivy-proxy-releases: http://devsvr/nexus/repository/sbt-ivy/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
my-maven-proxy-releases: http://devsvr/nexus/repository/central/
```

.sbtopts
```
-Dsbt.override.build.repos=true
```

scala ivy
```
http://repo.typesafe.com/typesafe/ivy-releases/
http://repo.scala-sbt.org/scalasbt/sbt-plugin-releases
```
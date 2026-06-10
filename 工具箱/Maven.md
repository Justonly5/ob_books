
## setting
settings 结构：
```XML
<settings>
  <!-- repository 不能直接放这里，settings.xml 不支持顶层 repositories -->

  <mirrors>...</mirrors>      <!-- ← 可以直接放顶层 -->
  <servers>...</servers>      <!-- ← 可以直接放顶层 -->
  <proxies>...</proxies>      <!-- ← 可以直接放顶层 -->

  <profiles>
    <profile>
      <repositories>
	      <repository></repository>
      </repositories>  <!-- ← repository 只能放这里 -->
      <pluginRepositories>
	      <pluginRepository></pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>...</activeProfiles>
</settings>
```

### 各节点说明速查

| 节点                   | 是否必须 | 能否放顶层               | 说明                           |
| -------------------- | ---- | ------------------- | ---------------------------- |
| `localRepository`    | 否    | 是                   | 默认 `~/.m2/repository`        |
| `proxies`            | 否    | 是                   | 网络代理，内网不需要                   |
| `servers`            | 否    | 是                   | 认证信息，id 对应 repository/mirror |
| `mirrors`            | 否    | 是                   | 仓库拦截代理                       |
| `profiles`           | 否    | 是                   | 配置集容器                        |
| `repositories`       | 否    | **否，只能在 profile 内** | 依赖仓库                         |
| `pluginRepositories` | 否    | **否，只能在 profile 内** | 插件仓库                         |
| `activeProfiles`     | 否    | 是                   | 激活指定 profile                 |
| `pluginGroups`       | 否    | 是                   | 简化插件命令                       |


![[Pasted image 20260610102802.png]]


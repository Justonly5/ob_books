
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
      <repositories>...</repositories>  <!-- ← repository 只能放这里 -->
    </profile>
  </profiles>

  <activeProfiles>...</activeProfiles>
</settings>
```



![[Pasted image 20260610102802.png]]


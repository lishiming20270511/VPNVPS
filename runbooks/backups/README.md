# backups（占位）

此目录用于存放从 VPS 拉回本地的 3x-ui 数据库与 docker-compose.yml 备份。

⚠️ **所有真实备份文件由 `.gitignore` 排除，绝不提交到仓库。**

文件命名规范：

```
x-ui-<VPS-ID>-YYYY-MM-DD.db
compose-<VPS-ID>-YYYY-MM-DD.yml
```

备份频率与保留策略见 `../vpn-service-runbook.md` 末尾的「备份计划」一节。

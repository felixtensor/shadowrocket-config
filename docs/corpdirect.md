# CorpDirect.sgmodule

公司网段和内网域名直连。可选。

## 用法

复制 `CorpDirect.example.sgmodule` 为 `CorpDirect.sgmodule`，替换成真实的 CIDR 和域名后
启用。本地文件已被 `.gitignore` 排除，不会提交。

模块规则优先于主配置。

## 内网域名

规则和解析器两处都要配，缺一不可：

```text
[Rule]
DOMAIN-SUFFIX,corp.example,DIRECT

[Host]
*.corp.example = server:system
```

`server:system` 跟随当前网络的系统 DNS，也可以换成明确的公司 DNS，例如
`server:10.20.30.53`。

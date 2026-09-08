> 🌐 本文档由 [golang/go](https://github.com/golang/go) 翻译,英文原版见原项目。

## 工具 {#tools}

### Go 命令 {#go-command}

### Cgo {#cgo}

### Vet {#vet}

新增的 [`scannererr`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/scannererr)
分析器会检查在 [bufio.Scanner.Scan] 外层循环结束后是否没有处理 scanner 错误,
这种疏漏可能导致扫描错误或 I/O 错误被漏报。 <!-- /issue/17747/ -->

[`sqlrowserr`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/sqlrowserr)
分析器对 [sql.Rows.Next] 外层的循环执行类似的检查,
以便把迭代错误与"结果集较小"正确区分开。

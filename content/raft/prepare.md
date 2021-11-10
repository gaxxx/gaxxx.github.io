+++
title = 'Raft 环境 - 准备'
date = 2021-11-05
+++


[6.824 Go (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/go.html)

- 下载golang
- 获取代码 `git clone git://g.csail.mit.edu/6.824-golabs-2021 6.824`
- 不要将自己的代码实现贴到网上

```bash
sh-3.2$ cd 6.824/
sh-3.2$ cd src/raft/
sh-3.2$ go test
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.05s)
    config.go:400: expected one leader, got none
Test (2A): election after network failure ...
```

这样就进入了烧脑的第一步

[Back](/raft/)

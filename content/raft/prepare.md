+++
title = 'Raft Environment - Preparation'
date = 2021-11-05
+++


[6.824 Go (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/go.html)

- Download Go
- Get the code `git clone git://g.csail.mit.edu/6.824-golabs-2021 6.824`
- Do not post your implementation code online

```bash
sh-3.2$ cd 6.824/
sh-3.2$ cd src/raft/
sh-3.2$ go test
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.05s)
    config.go:400: expected one leader, got none
Test (2A): election after network failure ...
```

This gets you into the brain-burning first step.

[Back](/raft/)

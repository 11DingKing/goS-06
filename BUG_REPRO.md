# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

微电网黑启动完成后，之前排队的自动并网命令没有继续执行，请帮我修复。

外部电网掉电后先发起黑启动，此时再发自动并网命令，接口会正确返回 queued，重连回路也显示 locked。随后黑启动完成接口返回成功，locked 变为 false，但电网状态回到了 off_grid，而不是进入 syncing；再调用完成同步也无法回到 connected。直接从 off_grid 发并网命令是正常的。

期望：黑启动期间收到的并网命令继续保持排队；黑启动完成并释放重连回路后，该命令自动恢复，状态进入 syncing，随后完成同步可进入 connected。错误的黑启动 ID、幂等行为和普通并网流程保持不变。修复后请保证 go test ./... 全绿，不要修改或跳过测试。

## 含 Bug 版本

- 仓库：11DingKing/goS-06
- 仓库地址：https://github.com/11DingKing/goS-06.git
- parent SHA：5b80c84c8b2e6a3453539fdd296a9d8acbba49ba

## 复现步骤

```bash
git clone -- https://github.com/11DingKing/goS-06.git bug-repro
cd bug-repro
git checkout --detach 5b80c84c8b2e6a3453539fdd296a9d8acbba49ba
go test ./internal/domain/grid ./internal/app ./internal/server -run "^TestBlackStartPriorityOverReconnect$|^TestAppBlackStartPriorityAndLock$|^TestHTTPBlackStartFlow$" -count=1 -v
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/domain/grid ./internal/app ./internal/server -run "^TestBlackStartPriorityOverReconnect$|^TestAppBlackStartPriorityAndLock$|^TestHTTPBlackStartFlow$" -count=1 -v
=== RUN   TestBlackStartPriorityOverReconnect
    grid_test.go:43: expected syncing after queued reconnect, got off_grid
--- FAIL: TestBlackStartPriorityOverReconnect (0.00s)
FAIL
FAIL	ejina-microgrid/internal/domain/grid	0.051s
=== RUN   TestAppBlackStartPriorityAndLock
    app_test.go:49: expected syncing, got off_grid
--- FAIL: TestAppBlackStartPriorityAndLock (0.00s)
FAIL
FAIL	ejina-microgrid/internal/app	0.041s
=== RUN   TestHTTPBlackStartFlow
    server_test.go:119: expected syncing, got off_grid
    server_test.go:129: expected connected, got off_grid
--- FAIL: TestHTTPBlackStartFlow (0.01s)
FAIL
FAIL	ejina-microgrid/internal/server	0.066s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/domain/grid ./internal/app ./internal/server -run "^TestBlackStartPriorityOverReconnect$|^TestAppBlackStartPriorityAndLock$|^TestHTTPBlackStartFlow$" -count=1 -v
=== RUN   TestBlackStartPriorityOverReconnect
    grid_test.go:43: expected syncing after queued reconnect, got off_grid
--- FAIL: TestBlackStartPriorityOverReconnect (0.00s)
FAIL
FAIL	ejina-microgrid/internal/domain/grid	0.002s
=== RUN   TestAppBlackStartPriorityAndLock
    app_test.go:49: expected syncing, got off_grid
--- FAIL: TestAppBlackStartPriorityAndLock (0.00s)
FAIL
FAIL	ejina-microgrid/internal/app	0.002s
=== RUN   TestHTTPBlackStartFlow
    server_test.go:119: expected syncing, got off_grid
    server_test.go:129: expected connected, got off_grid
--- FAIL: TestHTTPBlackStartFlow (0.00s)
FAIL
FAIL	ejina-microgrid/internal/server	0.012s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向测试通过：go test ./internal/domain/grid ./internal/app ./internal/server -run '^TestBlackStartPriorityOverReconnect$|^TestAppBlackStartPriorityAndLock$|^TestHTTPBlackStartFlow$' -count=1 -v
全量回归 go test -timeout=120s -count=1 ./... 通过，go build ./... 与 go vet ./... 通过
黑启动期间并网请求保持 queued 和 locked；完成黑启动后解锁并进入 syncing；完成同步后进入 connected；不得修改或跳过既有测试

## AddObserver missing fields
`LastChangeHeight` missing when updating `totalObserverCountCurrentBlock` when `AddObserver`

### original code
[https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/observer/keeper/msg_server_add_observer.go#L51](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/observer/keeper/msg_server_add_observer.go#L51)


- I think this could be modified to:
```go
k.SetLastObserverCount(ctx, &types.LastObserverCount{Count: totalObserverCountCurrentBlock, LastChangeHeight: ctx.BlockHeight()})
```
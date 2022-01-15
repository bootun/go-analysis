Map剖析
===
## 初始化
## 编译时的map
节点的op类型
ir.OMAPLIT
walk.maplit
```Go
func maplit(n *ir.CompLitExpr, m ir.Node, init *ir.Nodes) {
    // make the map var
    // 虽然这里传入的op类型为ir.OMAKE,但最终生成代码时并
    // 不一定会有makemap,比如当map被分配至栈上，详细请看
    // walk.walkMakeMap
    a := ir.NewCallExpr(base.Pos, ir.OMAKE, nil, nil)
    ...
    // 如果要初始化的条目大于25,则将其放入数组循环赋值
    if len(entries) > 25 {
        
		// for i = 0; i < len(vstatk); i++ {
		//	map[vstatk[i]] = vstate[i]
		// }
        return
    }
    // 对于很少数量的条目，直接赋值

	// Build list of var[c] = expr.
}
```
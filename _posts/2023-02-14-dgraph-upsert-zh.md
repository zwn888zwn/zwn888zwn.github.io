---
layout    : post
title     : "DGraph查询并更新--Upsert"
date      : 2023-02-14
lastupdate: 2023-02-14
categories: dgraph
---

----

# DGraph查询并更新--Upsert

在操作DGraph图数据库时，一般是先查询出个结果，然后执行一段逻辑，再更新到数据库中。在单线程环境下这样做没问题，但有并发操作时，回出现查询到的结果已经被别的线程修改了，造成丢失更新。一般可以通过加锁来解决，但是这样做粒度有点大，随后查询资料后，发现DGraph支持Upsert方法（查询并更新）和Java中的CAS操作类似。

```go
func UpdateXXXX(ctx context.Context, ...) error {
	queryTemplate := `
		query {
			me(func: type(Access)) @filter(eq(res_id, %s) and eq(res_status, %s)){
				uid
				bw_cfg @filter(eq(bandwidth, %d)){
					v as uid
					bandwidth
				}
			}
		}
	`
	query := fmt.Sprintf(queryTemplate, access.ResID, bean.AccessStatusActive, expectedBw)
	cond := `@if(eq(len(v), 1))`

	if _, err := dgClient.UpsertConditional(ctx, query, cond, access); err != nil {
		return err
	}
	return nil
}
```

封装UpsertConditional方法。query为查询语句，condition为判断条件，一般查询的时候把可能并发修改的值作为查询条件，看期望值是否正确，如果正确就执行更新语句。当发现冲突后，报出异常，循环重试。

``` go
func (p *client) UpsertConditional(ctx context.Context, query, condition string, obj interface{}) (map[string]string, error) {
	dgraphClient, err := p.getDgraphClient()
	if err != nil {
		return nil, err
	}
	mu := &api.Mutation{}
	objBytes, err := json.Marshal(obj)
	if err != nil {
		logging.WithCtx(ctx).Errorf("graph client marshal [%v] failed: %v", obj, err)
		return nil, err
	}
	logging.WithCtx(ctx).Debugf("graph client mutate: %s", string(objBytes))
	mu.Cond = condition
	mu.SetJson = objBytes

	req := &api.Request{
		Query:     query,
		Mutations: []*api.Mutation{mu},
		CommitNow: true,
	}

	response, err := dgraphClient.NewTxn().Do(ctx, req)
	if err != nil {
		logging.WithCtx(ctx).Errorf("graph client mutate failed: %v", err)
		return nil, err
	}
	if len(response.Uids) > 0 {
		resultStr, _ := json.MarshalIndent(response.Uids, "", "\t")
		logging.WithCtx(ctx).Debugf("graph client mutate result: %s", resultStr)
	}
	logging.WithCtx(ctx).Debugf("graph client mutate success")
	return response.Uids, nil

}
```

## 参考链接

[Upsert Mutations](https://dgraph.io/docs/graphql/mutations/upsert/)

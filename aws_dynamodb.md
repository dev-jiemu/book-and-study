## Amazon DynamoDB

### 📖 Pagination
DynamoDB 에서 데이터를 조회할때 Scan, Query 의 경우, 응답값의 총 데이터는 1MB를 넘을 수 없음 <br/>
조회할 데이터가 많거나 용량이 커서 **<span style="color: red;">총 데이터량이 1MB을 넘는다면</span>** 일부 데이터만 응답하므로 페이징 처리를 통해 데이터를 가지고 온후 합치는 작업이 필요함 <br/>
~~어쩐지 일부 데이터가 안보인다 싶더라니 페이징처리 해줘야 했었어~~

### 📚 Paging 처리 (with Golang)
1. **dynamo.Query** method
``` go
var err error
var queryOut *dynamodb.QueryOutput

ctx, cancel := context.WithTimeout(context.Background(), time.Duration(5)*time.Second)
defer cancel()

queryIn := &dynamodb.QueryInput{
        TableName:                 aws.String("TableName"),
        IndexName:                 aws.String("IndexName"),
	FilterExpression:          expression.Filter(),
	ProjectionExpression:      expression.Projection(),
	ExpressionAttributeNames:  expression.Names(),
	ExpressionAttributeValues: expression.Values(),
	KeyConditionExpression:     expression.KeyCondition(),
}

// 이전 데이터에서 key 값이 있었다면 해당값을 설정
if len(lastEvaluatedKey) > 0 {
	queryIn.ExclusiveStartKey = lastEvaluatedKey
}

queryOut, _ = DbClient.Query(ctx, queryIn)
```

2. **dynamodb.NewQueryPaginator** method
``` go
var err error
var client *dynamodb.Client

ctx, cancel := context.WithTimeout(context.Background(), time.Duration(5)*time.Second)
defer cancel()

queryIn := &dynamodb.QueryInput{
        TableName:                 aws.String("TableName"),
        IndexName:                 aws.String("IndexName"),
	FilterExpression:          expression.Filter(),
	ProjectionExpression:      expression.Projection(),
	ExpressionAttributeNames:  expression.Names(),
	ExpressionAttributeValues: expression.Values(),
	KeyConditionExpression:     expression.KeyCondition(),
}

paging := dynamodb.NewQueryPaginator(client, queryIn)
	for paging.HasMorePages() {
		pageItems, err := paging.NextPage(ctx)
		if err != nil {
			fmt.Errorf("fail NextPage : %v", err)
		}
		if pageItems != nil {
			// 페이징 데이터 처리 관련 로직
		}
	}
```
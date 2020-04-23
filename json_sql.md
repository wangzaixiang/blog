# SQL on JSON 

目标：使用类似的 SQL 语法，进行 JSON 的操作，可以返回、构造出新的 JSON 结构。

使用场景：1. 作为JSON的界面配置的语言，例如完成协议的转换。


## grammar
jso: an JSON Object
jsa: an JSON array

1. select field1, field2 from jso 
2. select * from jsa limit 10
3. select chat_id, chat_module from ${jso}.data.*
4. select data*.{chat_id, chat_module} from ${jso}
5. { "name": "wangzx",
     "content": sql"select field1, field2 from ${jso}",
     "count": sql"select count(*) from ${jso.data}"
   }

## demo
```json
{
	"data":[
		{
			"chat_id":"31b8a265-39e1-4051-a6db-1f6ea40e08d9",
			"chat_module":["faq","task_engine"],
			"chat_module_detail":"[#E1_退货场景_整合版_三期_V2]",
			"chat_std_q":[],
			"chat_te_names":["#E1_退货场景_整合版_三期_V2"],
			"cih_acs_qs_flag":"2",
			"cih_source_page":"1",
			"create_time":"2020-04-21 23:54:15",
			"node_name":"订单商品列表显示",
			"user_id":"165947621",
			"wdqsds":"订单已出库，现客需修改：********不要了  【修改收货信息（否）】*【要求拒收（是）】*【会员要求重发（否）】*",
			"wdwono":"ZX202004212916584",
			"wtnm1":"配送问题",
			"wtnm2":"顺丰速运",
			"wtnm3":"配送更改",
			"wtnm_detail":"配送问题-顺丰速运-配送更改"
		}
   ],
	"limit":10,
	"page":1,
	"totalSize":17915
}
```

1. demo1  
```sql
  select limit, page, totalSize from $jso
```
  =>
```json
[  { "limit": 10,
    "page": 1,
    "totalSize": 17915
  }
]
```

```sql
select data.chat_id, data.chat_module from $jso 
-- [ {chat_id, chat_module}, {chat_id, chat_module}, ... ]

select data.{chat_id, chat_module} from $jso
-- [ { data: [ { chat_id, chat_module }, { } ] } ] 

```

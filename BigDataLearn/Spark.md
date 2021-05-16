[Spark Documentation scala](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/index.html)

# column

- [Column](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-Column.html)
- [Column api doc](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Column.html)
- [DF/DS api doc](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html)



# Spark Stream

## DStream

### DStream API

two types of API: stateless and stateful API

#### Stateful API

##### Windowing

##### State Tracking

- [blog post](https://ngorchakova.github.io/jvmwarstories/updateStateByKey-mapWithState/#:~:text=updateStateByKey%20is%20executed%20on%20the,the%20size%20of%20the%20batch)

- `updateStateByKey` is executed on the whole range of keys in DStream. As results performance of these operation is proportional to the size of the state;
  - 全局状态跟踪; memory cost big
- `mapWithState` is executed only on set of keys that are available in the last micro batch. As result performance is proportional to the size of the batch
  - 增量状态跟曾; memory cost less, more efficient
- adding a `DIM` layer for handling/flattening the dimension tables(Product Catogory, Seller Regions)
- zipper tables for slowly changing dimensions(payment types, product catogory) providing version rollback mechanism



### Documentation
- ods tables are hive external tables on HDFS files partitioned by the date
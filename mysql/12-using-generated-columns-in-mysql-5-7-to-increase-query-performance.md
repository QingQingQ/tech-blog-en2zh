# ʹ��MySQL 5.7������������߲�ѯ����

ԭ��: Using MySQL 5.7 Generated Columns to Increase Query Performance

����: Alexander Rubin

����: ��ҫ��@֪����

����ƪ�����У����ǽ��������ʹ��MySQL 5.7�������У��������У�����߲�ѯ���ܡ�
>In this blog post, we��ll look at ways you can use MySQL 5.7 generated columns (or virtual columns) to improve query performance.

### ˵��
��Լ����ǰ���ҷ�����һ����MySQL5.7�汾�Ϲ��������У������У��Ĳ������¡�����ʱ��ʼ������ΪMySQL5.7���а浱�У�����ϲ����һ�����ܵ㡣ԭ��ܼ򵥣��������еİ����£����ǿ��Դ���fine-grained indexes������������߲�ѯ���ܡ���Ҫ������һЩ���ɣ�����Ǳ�ڵؽ����Щʹ����GROUP BY �� ORDER BY�����ı����ѯ��
>About two years ago I published a blog post about Generated (Virtual) Columns in MySQL 5.7. Since then, it��s been one of my favorite features in the MySQL 5.7 release. The reason is simple: with the help of virtual columns, we can create fine-grained indexes that can significantly increase query performance. I��m going to show you some tricks that can potentially fix slow reporting queries with GROUP BY and ORDER BY.

### ����
���������Э��һλ�ͻ������������������ѯ�ϣ�
>Recently I was working with a customer who was struggling with this query:
```sql
SELECT 
CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', 
COUNT(*) as 'No. of API Calls', 
AVG(ExecutionTime) as 'Avg. Execution Time', 
COUNT(distinct AccountId) as 'No. Of Accounts', 
COUNT(distinct ParentAccountId) as 'No. Of Parents' 
FROM ApiLog 
WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59' 
GROUP BY CONCAT(verb, ' - ', replace(url,'.xml','')) 
HAVING COUNT(*) >= 1 ;
```

�����ѯ������һ����Сʱ������ʹ�úͳ��������� tmpĿ¼�������ļ����򣩡�
>The query was running for more than an hour and used all space in the tmp directory (with sort files).

��ṹ����:
>The table looked like this:
```sql
CREATE TABLE `ApiLog` (
`Id` int(11) NOT NULL AUTO_INCREMENT,
`ts` timestamp DEFAULT CURRENT_TIMESTAMP,
`ServerName` varchar(50)  NOT NULL default '',
`ServerIP` varchar(50)  NOT NULL default '',
`ClientIP` varchar(50)  NOT NULL default '',
`ExecutionTime` int(11) NOT NULL default 0,
`URL` varchar(3000)  NOT NULL COLLATE utf8mb4_unicode_ci NOT NULL,
`Verb` varchar(16)  NOT NULL,
`AccountId` int(11) NOT NULL,
`ParentAccountId` int(11) NOT NULL,
`QueryString` varchar(3000) NOT NULL,
`Request` text NOT NULL,
`RequestHeaders` varchar(2000) NOT NULL,
`Response` text NOT NULL,
`ResponseHeaders` varchar(2000) NOT NULL,
`ResponseCode` varchar(4000) NOT NULL,
... // other fields removed for simplicity
PRIMARY KEY (`Id`),
KEY `index_timestamp` (`ts`),
... // other indexes removed for simplicity
) ENGINE=InnoDB;
```

���Ƿ��ֲ�ѯû��ʹ��ʱ����ֶΣ���TS����������:
>We found out the query was not using an index on the timestamp field (��ts��):

```sql
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', COUNT(*)  as 'No. of API Calls',  avg(ExecutionTime) as 'Avg. Execution Time', count(distinct AccountId) as 'No. Of Accounts',  count(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY CONCAT(verb, ' - ', replace(url,'.xml',''))  HAVING COUNT(*)  >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: ALL
possible_keys: ts
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 22255292
     filtered: 50.00
        Extra: Using where; Using filesort
1 row in set, 1 warning (0.00 sec)
```

ԭ��ܼ򵥣����Ϲ�������������̫���ˣ�������Ӱ��һ������ɨ��ɨ���Ч�ʣ����������Ż�����������Ϊ�ģ���
>The reason for that is simple: the number of rows matching the filter condition was too large for an index scan to be efficient (or at least the optimizer thinks that):

```sql
mysql> select count(*) from ApiLog WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59' ;
+----------+
| count(*) |
+----------+
|  7948800 |
+----------+
1 row in set (2.68 sec)
```

��������21998514����ѯ��Ҫɨ�����������36%��7948800/21998514����
>Total number of rows: 21998514. The query needs to scan 36% of the total rows (7948800 / 21998514).

����������£���������ദ������

1. ����ʱ����к�group by�е���������
2. ����һ�������������������в�ѯ�ֶΣ�
3. ����group�д�������
4. ����������ɢ����ɨ��
>In this case, we have a number of approaches:
>1. Create a combined index on timestamp column + group by fields
>2. Create a covered index (including fields that are selected)
>3. Create an index on just GROUP BY fields
>4. Create an index for loose index scan

Ȼ�������������ϸ�۲��ѯ�С�group by�����֣����Ǻܿ����ʶ������Щ���������ܽ�����⡣���������ǵ�group by���֣�
>However, if we look closer at the ��GROUP BY�� part of the query, we quickly realize that none of those solutions will work. Here is our GROUP BY part:
```sql
GROUP BY CONCAT(verb, ' - ', replace(url,'.xml',''))
```

�������������⣺

1. ���Ǽ����У�����MySQL����ɨ��verb + url����������������Ҫ���������ֶΣ�Ȼ����������ַ����������ζ���ò���������
2. URL������Ϊ��varchar(3000) COLLATE utf8mb4_unicode_ci NOT NULL�������ܱ���ȫ��������ʹ��ȫinnodb_large_prefix= 1 ���������£�����UTF8�����µ�Ĭ�ϲ����������������������������group by��sql�Ż���û��ʲô������
>There are two problems here:
>1. It is using a calculating field, so MySQL can��t just scan the index on verb + url. It needs to first concat two fields, and then group on the concatenated string. That means that the index won��t be used.
>2. The URL is declared as ��varchar(3000) COLLATE utf8mb4_unicode_ci NOT NULL�� and can��t be indexed in full (even with innodb_large_prefix=1  option, which is the default as we have utf8 enabled). We can only do a partial index, which won��t be helpful for GROUP BY optimization.

������ҳ���ȥ��URL�����һ����������������innodb_large_prefix=1�����£�
>Here, I��m trying to add a full index on the URL with innodb_large_prefix=1:
```sql
mysql> alter table ApiLog add key verb_url(verb, url);
ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes
```

�ţ�ͨ���޸ġ�GROUP BY CONCAT(verb, �� �C ��, replace(url,��.xml��,��))��Ϊ  ��GROUP BY verb, url�� ��������������ǰ��ֶζ���� varchar��3000����СһЩ������ҵ���������������Ȼ�����⽫�ı�������URL�ֶβ���ɾ�� .xml��չ���ˡ�
>Well, changing the ��GROUP BY CONCAT(verb, �� �C ��, replace(url,��.xml��,��))�� to  ��GROUP BY verb, url�� could help (assuming that we somehow trim the field definition from  varchar(3000) to something smaller, which may or may not be possible). However, it will change the results as it will not remove the .xml extension from the URL field.

### �������
����Ϣ�ǣ���MySQL 5.7�������������С��������ǿ����ڡ�CONCAT(verb, �� �C ��, replace(url,��.xml��,��))��֮�ϴ���һ�������С���õĲ��֣����ǲ���Ҫִ��һ���������ַ��������ܴ���3000�ֽڣ������ǿ���ʹ��MD5��ϣ��������Ĺ�ϣ������SHA1 / SHA2����Ϊgroup by�Ķ���
>The good news is that in MySQL 5.7 we have virtual columns. So we can create a virtual column on top of ��CONCAT(verb, �� �C ��, replace(url,��.xml��,��))��. The best part: we do not have to perform a GROUP BY with the full string (potentially > 3000 bytes). We can use an MD5 hash (or longer hashes, i.e., sha1/sha2) for the purposes of the GROUP BY.

�����ǽ������:
>Here is the solution:
```sql
alter table ApiLog add verb_url_hash varbinary(16) GENERATED ALWAYS AS (unhex(md5(CONCAT(verb, ' - ', replace(url,'.xml',''))))) VIRTUAL;
alter table ApiLog add key (verb_url_hash);
```

�������������������ǣ�

1. ���������У�����Ϊvarbinary��16��
2. ��CONCAT(verb, �� �C ��, replace(url,��.xml��,��)�ϴ��������У�����ʹ��MD5��ϣת������ʹ��unhexת��32λʮ������Ϊ16λ�����ơ�
3. ������������д�������
>So what we did here is:
>1. Declared the virtual column with type varbinary(16)
>2. Created a virtual column on CONCAT(verb, �� �C ��, replace(url,��.xml��,��), and used an MD5 hash on top plus an unhex to convert 32 hex bytes to 16 binary bytes
>3. Created and index on top of the virtual column

�������ǿ����޸Ĳ�ѯ��䣬group by verb_url_hash�У�
>Now we can change the query and GROUP BY verb_url_hash column:
```sql
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml',''))
AS 'API Call', COUNT(*)  as 'No. of API Calls',
avg(ExecutionTime) as 'Avg. Execution Time',
count(distinct AccountId) as 'No. Of Accounts',
count(distinct ParentAccountId) as 'No. Of Parents'
FROM ApiLog
WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'
GROUP BY verb_url_hash
HAVING COUNT(*)  >= 1;
ERROR 1055 (42000): Expression #1 of SELECT list is not in
GROUP BY clause and contains nonaggregated column 'ApiLog.ApiLog.Verb'
which is not functionally dependent on columns in GROUP BY clause;
this is incompatible with sql_mode=only_full_group_by
```

MySQL 5.7���ϸ�ģʽ��Ĭ�����õģ����ǿ���ֻ�����β�ѯ�޸�һ�¡�

���ڽ��ͼƻ�����ȥ�ö��ˣ�
>MySQL 5.7 has a strict mode enabled by default, which we can change for that query only.

>Now the explain plan looks much better:

```sql
mysql> select @@sql_mode;
+-------------------------------------------------------------------------------------------------------------------------------------------+
| @@sql_mode                                                                                                                                |
+-------------------------------------------------------------------------------------------------------------------------------------------+
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
+-------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
mysql> set sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
Query OK, 0 rows affected (0.00 sec)
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', COUNT(*)  as 'No. of API Calls',  avg(ExecutionTime) as 'Avg. Execution Time', count(distinct AccountId) as 'No. Of Accounts',  count(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY verb_url_hash HAVING COUNT(*)  >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: index
possible_keys: ts,verb_url_hash
          key: verb_url_hash
      key_len: 19
          ref: NULL
         rows: 22008891
     filtered: 50.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

MySQL���Ա��������ٶȸ��졣�������ջ���Ҫɨ�����б��������˳����Ӧʱ�����Ը��ã�~ 38�������>һСʱ��
>MySQL will avoid any sorting, which is much faster. It will still have to eventually scan all the table in the order of the index. The response time is significantly better: ~38 seconds as opposed to > an hour.

### ��������
�������ǿ��Գ�����һ�������������⽫�൱��
>Now we can attempt to do a covered index, which will be quite large:

```sql
mysql> alter table ApiLog add key covered_index (`verb_url_hash`,`ts`,`ExecutionTime`,`AccountId`,`ParentAccountId`, verb, url);
Query OK, 0 rows affected (1 min 29.71 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

���������һ�������ʡ��͡�URL��������֮ǰ�Ҳ��ò�ɾ�������COLLATE utf8mb4_unicode_ci������ִ�мƻ�����������ʹ���˸���������
>We had to add a ��verb�� and ��url��, so beforehand I had to remove the COLLATE utf8mb4_unicode_ci from the table definition. Now explain shows that we��re using the index:
```sql
mysql> explain SELECT  CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call',  COUNT(*) as 'No. of API Calls',  AVG(ExecutionTime) as 'Avg. Execution Time',  COUNT(distinct AccountId) as 'No. Of Accounts',  COUNT(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY verb_url_hash  HAVING COUNT(*) >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: index
possible_keys: ts,verb_url_hash,covered_index
          key: covered_index
      key_len: 3057
          ref: NULL
         rows: 22382136
     filtered: 50.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```
��Ӧʱ���½���~12�룡���ǣ������Ĵ�С���ԵرȽ�verb_url_hash��������ÿ����¼16�ֽڣ�Ҫ��öࡣ
>The response time dropped to ~12 seconds! However, the index size is significantly larger compared to just verb_url_hash (16 bytes per record).

### ����
MySQL 5.7���������ṩһ���м�ֵ�ķ�������߲�ѯ���ܡ��������һ����Ȥ�İ��������������з���
>MySQL 5.7 generated columns provide a valuable way to improve query performance. If you have an interesting case, please share in the comments.

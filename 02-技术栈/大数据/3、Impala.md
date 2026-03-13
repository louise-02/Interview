# 1、常用函数

## 转换函数

```sql
cast(column_name as double)
```

## 数学函数

```sql
# 绝对值
abs(numeric_type a)

# 返回整数的二进制表示形式，即0和1位数字符串。
bin(bigint a)

# 返回大于或等于参数的最小整数
ceil(double a)
ceiling(double a)
dceil(double a)

# 将输入值进行基底转换并返回(进制转换)。
conv(bigint num,int from_base,int to_base)
conv(string num,int from_base,int to_base)

# 数学常量e
e()

# 常数e的n次方
exp(double a)
dexp(double a)

# 整数的的阶乘
factorial(integer_type a)

# 返回小于或等于参数的最大整数
floor(double a)
dfloor(double a)

# 得到任意值的64位哈希值
fnv_hash(type v)

# 返回表达式列表中的最大值
greatest(some-type a,[some-type b…])
# 返回表达式列表中的最小值
least(bigint a,[bigint b])

# 返回数字的余数的绝对值
pmod(bigint a, bigint b)
pmod(double a, double b)

# 求余数
fmod(double a, double b)
mod(numeric_typea, numeric_typeb)

# 截取数值，四舍五入
round(double a,[int d])
dround(double a,[int d])

# 求商（去掉余数）
quotient(bigint numerator, bigint denominator)
quotient(double numerator, double denominator)

# 删除所有小数点以后的数或删除N位小数
truncate(double_or_decimal a,[digits_to_leave])
dtrunc(double_or_decimal a,[digits_to_leave])
```

## 字符串函数

```sql
# 返回第一个不为null的v，全部为null则返回null
coalesce( v1, v2, ...)
```

## 日期函数

```sql
# 获取当前时间戳函数		输出类型
current_timestamp()	timestamp	返回客户端所在时区的当前时间戳
now()	timestamp	返回客户端所在时区的当前时间戳
unix_timestamp（）	bigint	返回客户端所在时区的当前时间戳的整数形式
utc_timestamp()	timestamp	返回客户端时间对应UTC时区的当前时间戳
timeofday()	string	根据本地系统的时间（包括任何时区指定）返回当前日期和时间的字符串表示形式。
 
# 时间计算函数
years_add(timestamp/date date, int/bigint years)    timestamp/date 增加指定年数
years_sub(timestamp/date date, int/bigint years)   timestamp/date 减少指定年数
months_add(timestamp/date date, int/bigint months) timestamp/date 增加指定月数
months_sub(timestamp/date date, int/bigint months) timestamp/date 减少指定月数
add_months(timestamp/date date, int/bigint months) timestamp/date 增加指定月数
weeks_add(timestamp/date date, int/bigint weeks)   timestamp/date 增加指定周数
weeks_sub(timestamp/date date, int/bigint weeks)   timestamp/date 减少指定周数
days_add(timestamp/date startdate, int/bigint days)    timestamp/date 增加指定天数
days_sub(timestamp/date startdate, int/bigint days)    timestamp/date 减少指定天数
date_add(timestamp/date startdate, int/bigint days)    timestamp/date 增加指定天数
date_sub(timestamp/date startdate, int/bigint days)    timestamp/date 减少指定天数
adddate(timestamp/date startdate, int/int days)    timestamp/date 增加指定天数
subdate(timestamp/date startdate，bigint/int days)  timestamp/date 减少指定天数
hours_add(timestamp date, int/bigint hours)    timestamp  增加指定小时
hours_sub(timestamp date, int/bigint hours)    timestamp  减少指定小时
minutes_add(timestamp date, int/bigint minutes)    timestamp  增加指定分钟
minutes_sub(timestamp date, int/bigint minutes)    timestamp  减少指定分钟
seconds_add(timestamp date, int/bigint seconds)    timestamp  增加指定秒数
seconds_sub(timestamp date, int/bigint seconds)    timestamp  减少指定秒数
milliseconds_add(timestamp t, int/bigint s）    timestamp  增加指定毫秒数
milliseconds_sub(timestamp t, int/bigint s）    timestamp  减少指定毫秒数
microseconds_add(timestamp t, int/bigint s)    timestamp  增加指定微秒数
microseconds_sub(timestamp t, int/bigint s)    timestamp  减少指定微秒数
nanoseconds_add(timestamp t, int/bigint s） timestamp  增加指定纳秒数
nanoseconds_sub(timestamp t, int/bigint s） timestamp  减少指定纳秒数
date_add(timestamp/date startdate, interval_expression)    timestamp/date 使用参数计算日期增量值（增加）
date_sub(timestamp/date startdate, interval_expression)    timestamp/date 使用参数计算日期增量值（减少）
interval_expression的表述可用如下：YEAR[S]，MONTH[S] ，WEEK[S] ，DAY[S]， HOUR[S]， MINUTE[S] ，SECOND[S] ，MILLISECOND[S] ，MICROSECOND[S]， NANOSECOND[S]

# 获取时间指定单位函数
year(timestamp/date date)	int	获取年
quarter(timestamp/date date)	int	获取季节（1,2,3,4）
month(timestamp/date date)	int	获取月
monthname(timestamp/date date)	string	获取月份名称
week(timestamp/date date)	int	获取周（1-53）
weekofyear(timestamp/date date)	int	获取周（1-53）
dayofweek(timestamp/date date)	int	获取天（本周第多少天,周日算第一天）
dayname(timestamp/datedate)	string	获取天（星期几）
next_day(timestamp/date date,String weekday)	timestamp/date	获取天（返回下一个指定星期几的日期）
day(timestamp/date date)	int	获取天（本月第多少天）
dayofmonth(timestamp/date date)	int	获取天（本月第多少天）
last_day(timestamp/date date)	timestamp/date	获取天（本月的最后一天日期）
dayofyear(timestamp/date date)	int	获取天（本年第多少天）
hour(timestamp date)	int	获取小时
minute(timestamp date)	int	获取分钟
second(timestamp date)	int	获取秒
millisecond (timestamp date)	int	获取毫秒
extract(timestamp/date date,String unit)	bigint	获取参数指定的时间单位
extract(unit from timestamp/date date)	bigint	获取参数指定的时间单位
date_part(string unit，timestamp timestamp)	bigint	获取参数指定的时间单位
trunc(timestamp/date date,String unit)	timestamp/date	获取截断为指定单位的时间
date_trunc(string unit，timestamp/date date)	timestamp/date	获取截断为指定单位的时间

extract 的时间单位可以使用如下：YEAR，QUARTER，MONTH，DAY，HOUR，MINUTE，SECOND，MILLISECOND，EPOCH（转成数字类型）
date_trunc的时间单位可以使用如下：MILLENNIUM（千年），CENTURY（百年），DECADE（十年），YEAR，MONTH，WEEK，DAY，HOUR， MINUTE，SECOND，MILLISECONDS（毫秒），MICROSECONDS（微妙）
trunc的时间单位可以使用如下：
SYYYY，YYYY，YEAR，SYEAR，YYY，YY，Y	年
Q	季节
MONTH，MON，MM，RM	月
WW	最近的日期是与一年中的第一天相同的日期
W	最近的日期是与该月的第一天相同的星期几
DDD，DD，J	天
DAY，DY，D	星期几（星期一）的开始
HH，HH12，HH24	小时
MI	分钟

# 时间比较函数
datediff(timestamp/date enddate, timestamp/date startdate)  int    返回endDate比startDate多多少天
int_months_between(timestamp/date t1, timestamp/date t2)   int    返回两个日期相差的整数月份个数
months_between(timestamp/date t1, timestamp/date t2)   double 返回浮点数的月数相差的数
date_cmp(DATE date1, DATE date2)   int    比较是否相等，返回-1,0,1,null四种数值
timestamp_cmp(timestamp t1，timestamp t2)   int    比较是否相等，返回-1,0,1,null四种数值

# 日期转换函数
to_date(timestamp date) string 返回时间戳对应的date
to_timestamp(bigint unixtime)  timestamp  返回整数对应的timestamp值
to_timestamp(string date，string pattern)   timestamp  返回字符串对应的timestamp值
to_utc_timestamp(timestamp t，string timezone)  timestamp  指定时区的时间戳转化为UTC时区的时间戳
from_timestamp(timestamp t，string pattern) string 把timestamp按照pattern进行格式化
from_timestamp(string date，string pattern) string 把date按照pattern进行格式化
from_unixtime(bigint unixtime) string 把时间戳秒数转化为本地地区中的字符串
from_unixtime(bigint unixtime，string pattern）  string 时间戳转化为本地时区字符串，pattern格式
from_utc_timestamp（timestamp t，string timezone）    timestamp  UTC时区指定时间戳转化为指定时区时间戳
unix_timestamp(string datetime)    bigint 把string类型的date或日期转化成时间戳Unix
unix_timestamp(timestamp datetime) bigint 把string类型的timestamp转化成时间戳Unix
unix_timestamp(string datetime，string pattern) bigint 日期按pattern转化成时间戳Unix

y   年
M  月
d  日
H  小时
m  分钟
s  秒
S  小数秒
+/-hh:mm   时区偏移
+/-hhmm    时区偏移
+/-hh  时区偏移

to_timestamp(1612399587)                         2021-02-04 08:46:27 
to_timestamp('2021/02/04','yyyy/MM/dd')          2021-02-04 00:00:00
to_utc_timestamp(now(),'Asia/Shanghai')          2021-02-04 00:46:27
from_timestamp(now(),'yyyy/MM')                  2021/02
from_timestamp('2021-02-04','yyyy/MM')           2021/02
from_unixtime(1612399587)                        2021-02-04 08:46:27
from_unixtime(1612399587,'yyyy/MM')              2021/02
from_utc_timestamp(now(),'Asia/Shanghai')        2021-02-04 16:46:27
unix_timestamp('2021-02-04')                     1612368000
unix_timestamp(now())                            1612399587
```

## 聚合函数

```sql
group_concat(create_date,',')
```

## 窗口函数

```sql
# 排名开窗函数
rank () over ()	分组排序生成排名（重复的话序号一样，然后跳过重复的序号）
row_number () over ()	分组排序生成排名（不区分重复）
dense_rank () over ()	分组排序生成排名（重复的话序号一样，然后顺排）

# 排名开窗函数（切片函数）
ntile () over ()	分组内将数据切片

# 排名开窗函数（序列分析函数）
cume_dist () over ()	小于等于当前行值的行数/总行数
percent_rank () over ()	(当前rank值-1) / (总行数-1)

# 排名开窗函数（lead+lag）
first_value（col）over（）	获取统计窗口内排名第一的列值
last_value（col）over（）	获取统计窗口内排名最后的列值

# 聚合开窗函数
count（sal） over （）	获取统计窗口内的指定列的数据量
max（sal） over （）	获取统计窗口内的指定列的最大值
min（sal） over （）	获取统计窗口内的指定列的最小值
avg（sal） over （）	获取统计窗口内的指定列的平均值
sum（sal） over （）	获取统计窗口内的指定列的和
```

# 2、同步 Hive 数据

```bash
for tab in ods_qa_line ods_it_fqms
do
	echo "------------------------------------------------------------------${tab}---------------------------------------------------------------"
  	echo "-----------------------------------------------------------------refresh---------------------------------------------------------------"
	hive -e "use ${tab}; show tables;" > ${tab}_hiveTables.log
	impala-shell -B -i hp4:21000 -l --auth_creds_ok_in_clear --user=languqms   --ldap_password_cmd="echo -n 'wgHefw@#14'" -q "use ${tab}; show tables;" > ${tab}_impalaTables.log


	arr=$(sort ${tab}_hiveTables.log ${tab}_impalaTables.log | uniq -d)

	for var in $arr
	do
		impala-shell -i hp4:21000 -l --auth_creds_ok_in_clear --user=languqms   --ldap_password_cmd="echo -n 'wgHefw@#14'" -q "refresh ${tab}.${var};"
	done

	echo "-----------------------------------------------------------------invalidate metadata---------------------------------------------------------------"

	arr=$(sort ${tab}_hiveTables.log  ${tab}_impalaTables.log | uniq -u)

	for var in $arr
	do
        	impala-shell -i hp4:21000 -l --auth_creds_ok_in_clear --user=languqms   --ldap_password_cmd="echo -n 'wgHefw@#14'" -q "invalidate metadata ${tab}.${var};"
	done

done
```

# 3、常用 Sql

## ddl

```sql
# 更改表的名称
ALTER TABLE [old_db_name.]old_table_name RENAME TO [new_db_name.]new_table_name
# 向表中添加列
ALTER TABLE name ADD COLUMNS (col_spec[, col_spec ...])
# 从表中删除列
ALTER TABLE name DROP [COLUMN] column_name
# 更改列的名称和类型
ALTER TABLE name CHANGE column_name new_name new_type
```

## 删除数据

```sql
# 方法1 not in 
insert overwrite table 库名.表名 select * from 库名.表名 where 字段 not in (数据1,数据2...)
# 方法2 in
insert overwrite table 库名.表名 select * from 库名.表名 where 字段  in (数据1,数据2...)
```

# 4、Jdbc

```bash
//官网下载jar包
https://www.cloudera.com/downloads/connectors/impala/jdbc/2-6-12.html

//将jar包打入本地仓库
mvn install:install-file
-DgroupId=com.cloudera.impala
-DartifactId=ImpalaJDBC41
-Dversion=1.0
-Dpackaging=jar
包所在的路径
-Dfile="D:\download\Access_JDBC30.jar"

//添加依赖
<dependency>
    <groupId>com.cloudera.impala</groupId>
    <artifactId>ImpalaJDBC41</artifactId>
    <version>1.0</version>
</dependency>

//jdbc连接
jdbc:impala://10.22.10.150:21050;AuthMech=3;UID=langutest;PWD=Langutest.123!;
```

# 5、命令行

```bash
# 命令
-i: 集群中任意一台impalad服务器，如果使用了该参数需要输入用户密码
-l: 使用ldap, --auth_creds_ok_in_clear 由于没有使用ssl，需要添加该参数
-u: 用户
-f: 在命令行运行一个sql脚本  -f query.sql
--ldap_password_cmd  密码命令行
-B 去格式化  没有--||等格式
# 登录
impala-shell -i cloudera01:21000 -l --auth_creds_ok_in_clear -u cdh01
impala-shell -i hp4:21000 -l --auth_creds_ok_in_clear -u languqms   --ldap_password_cmd="cat /home/languqms/pwd-impala"
```

# 99、常见问题

```sql
# 在Hive中或sqoop中导入数据后,impala中数据没有变化
refresh [table] partition [partition]    增量更新元数据
invalidate metadata [table]  用来清空（重置）元数据(慎用会对集群有影响)

# 分页查询
limit 10  offset 2  需要加排序顺序
```


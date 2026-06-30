# 1、yml 中使用

```yml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://${HOME_HOST:49.232.148.209}:3306/111?useSSL=false
    username: louise
    password: 123456
    
java -DHOME_HOST=127.0.0.1 -jar mysql.jar

# 获取pom中的属性
spring:
  application:
    name: @artifactId@
    
# security注解中
@PreAuthorize("#flag")  # 参数
@PreAuthorize("@myComponent.hasRole('user')")  # 容器中的对象调用方法
@PreAuthorize("hasAnyRole('user')")
```

# 2、代码示例

```java
public class SpringExpressionLanguageTest {
    /**
     * 获取变量
     */
    @Test
    public void valueTest(){
        // 表达式解析器
        SpelExpressionParser spelExpressionParser = new SpelExpressionParser();
        // 解析出一个表达式
        Expression expression = spelExpressionParser.parseExpression("#user.password");
        // 开始准备表达式运行环境
        StandardEvaluationContext ctx = new StandardEvaluationContext();

        ctx.setVariable("user",new TestEntity("123","321"));

        String value = expression.getValue(ctx, String.class);

        System.out.println(value);
    }


    /**
     * 执行方法
     */
    @Test
    public void methodTest(){
        ExpressionParser parser = new SpelExpressionParser();
        //Expression expression = parser.parseExpression("setUsername('999')"); 可以有参数
        Expression expression = parser.parseExpression("getUsername()");

        StandardEvaluationContext ctx = new StandardEvaluationContext();

        TestEntity testEntity = new TestEntity("123", "321");

        //设定需要执行方法的类
        ctx.setRootObject(testEntity);

        String value = expression.getValue(ctx, String.class);

        System.out.println("value = " + value);

        System.out.println(testEntity);
    }


  	@Data
    static class TestEntity{
        String username;
        String password;
    }
}
```

# 3、spel 介绍

```java
1. 解析器：ExpressionParser
Expression parseExpression(String expressionString) throws ParseException;
Expression parseExpression(String expressionString, ParserContext context) throws ParseException;
parseExpression：将字符串表达式转化为表达式对象
接口的实现：
SpelExpressionParser：SpEL解析器。实例是可重用的，且是线程安全的（最常用）。
TemplateAwareExpressionParser：模板感知表达式解析器，可以被不提供一流模板支持的表达式解析器子类化
InternalSpelExpressionParser：手写的SpEL解析器。 实例是可重用的，但不是线程安全的，是TemplateAwareExpressionParser的子类。

2.表达式：Expression
getValue：获取表达式值
setValue：设置对象值
接口的实现：
CompositeStringExpression：复合字符串表达式（里面封装一个表达式对象集合）
SpelExpression：SpEL表达式（最常用）
LiteralExpression：文字表达式

3.上下文：EvaluationContext
使用setRootObject方法来设置根对象，使用setVariable方法来注册自定义变量，使用registerFunction来注册自定义函数等。
接口的实现：
StandardEvaluationContext：标准上下文（最常用）
SimpleEvaluationContext：
ThymeleafEvaluationContext
MethodBasedEvaluationContext
CacheEvaluationContext

//使用流程
1.创建解析器：创建解析器ExpressionParser（如：SpelExpressionParser）
2.解析表达式：使用ExpressionParser的parseExpression来解析表达式得到表达式对象Expression（如：SpelExpression）。
3.构造上下文：创建上下文EvaluationContext（如：StandardEvaluationContext），设置需要的数据。
4.得到值：通过Expression的getValue方法根据上下文获得表达式值。
```

# 4、表达式

## 基本表达式

```java
1.字面量表达式
ExpressionParser parser = new SpelExpressionParser();
// 字符串
String str1 = parser.parseExpression("'test'").getValue(String.class);  //test
String str2 = parser.parseExpression("\"test\"").getValue(String.class); //test
// 数字类型
int int1 = parser.parseExpression("1").getValue(Integer.class);
long long1 = parser.parseExpression("1L").getValue(long.class);
float float1 = parser.parseExpression("1.1").getValue(Float.class);
double double1 = parser.parseExpression("1.1E+1").getValue(double.class);
int hex1 = parser.parseExpression("0xf").getValue(Integer.class);
// 布尔类型
boolean true1 = parser.parseExpression("true").getValue(boolean.class);
// null类型
Object null1 = parser.parseExpression("null").getValue(Object.class);

2.算数运算表达式
SpEL支持：加(+)、减(-)、乘(*)、除(/)、求余（%）、幂（^）等算数运算。
并且支持用英文替代符号，如：MOD等价%、DIV等价/，且不区分大小写。
ExpressionParser parser = new SpelExpressionParser();
int int1 = parser.parseExpression("3+2").getValue(Integer.class);// 5
int int2 = parser.parseExpression("3*2").getValue(Integer.class);// 6
int int3 = parser.parseExpression("3%2").getValue(Integer.class);// 1
int int4 = parser.parseExpression("3^2").getValue(Integer.class);// 9

3.关系运算表达式
SpEL支持：等于（==）、不等于(!=)、大于(>)、大于等于(>=)、小于(<)、小于等于(<=)，区间（between）等关系运算。
“EQ” 、“NE”、 “GT”、“GE”、 “LT” 、“LE”来表示等于、不等于、大于、大于等于、小于、小于等于
ExpressionParser parser = new SpelExpressionParser();
boolean b1 = parser.parseExpression("3==2").getValue(Boolean.class);// false
boolean b2 = parser.parseExpression("3>=2").getValue(Boolean.class);// true
boolean b3 = parser.parseExpression("2 between {2, 3}").getValue(Boolean.class);// true

4.逻辑运算表达式
SpEL支持：或(or)、且（and）、非(!或NOT)。
ExpressionParser parser = new SpelExpressionParser();
boolean b1 = parser.parseExpression("2>1 and false").getValue(Boolean.class);// false
boolean b2 = parser.parseExpression("2>1 or false").getValue(Boolean.class);// true
boolean b3 = parser.parseExpression("NOT false and (2>1 and 3>1)").getValue(Boolean.class);// true

5.字符串连接及截取表达式
SpEL支持字符串拼接和字符串截取（目前只支持截取一个字符）
ExpressionParser parser = new SpelExpressionParser();
String str1 = parser.parseExpression("'hello' + ' java'").getValue(String.class);// hello java
String str2 = parser.parseExpression("'hello java'[0]").getValue(String.class);// h
String str3 = parser.parseExpression("'hello java'[1]").getValue(String.class);// e

6.三目运算
ExpressionParser parser = new SpelExpressionParser();
Boolean b1 = parser.parseExpression("3 > 2 ? true : false").getValue(Boolean.class);// true

7.Elivis运算符
当表达式1为非null时则返回表达式1，当表达式1为null时则返回表达式2
ExpressionParser parser = new SpelExpressionParser();
String str1 = parser.parseExpression("'a' ?: 'b'").getValue(String.class);// a
String str2 = parser.parseExpression("null ?: 'b'").getValue(String.class);// b
Boolean b1 = parser.parseExpression("3 > 2 ?: false").getValue(Boolean.class);// true
Boolean b2 = parser.parseExpression("null ?: false").getValue(Boolean.class);// false
Boolean b3 = parser.parseExpression("false ?: true").getValue(Boolean.class);// false

8.正则表达式
ExpressionParser parser = new SpelExpressionParser();
Boolean b1 = parser.parseExpression("'123' matches '\\d{3}'").getValue(Boolean.class);// true
Boolean b2 = parser.parseExpression("'123' matches '\\d{2}'").getValue(Boolean.class);// false
```

## 类相关表达式

```java
1.类类型
SpEL支持使用T(Type)来表示java.lang.Class实例，Type必须是类全限定名，java.lang包除外
使用类类型表达式还可以进行访问类静态方法及类静态字段。
ExpressionParser parser = new SpelExpressionParser();
// java.lang包
Class class1 = parser.parseExpression("T(String)").getValue(Class.class);
// 其他包
Class class2 = parser.parseExpression("T(com.joker.pojo.User)").getValue(Class.class);
// 类静态字段访问
int result3 = parser.parseExpression("T(Integer).MAX_VALUE").getValue(int.class);
// 类静态方法调用
int result4 = parser.parseExpression("T(Integer).parseInt('2')").getValue(int.class);

2.类实例
SpEL支持类实例化，使用java关键字new，类名必须是全限定名，但java.lang包内的类型除外
ExpressionParser parser = new SpelExpressionParser();
// java.lang包
String str = parser.parseExpression("new String('str')").getValue(String.class);
// 其他包
Date date = parser.parseExpression("new java.util.Date()").getValue(Date.class);

3.instanceof
ExpressionParser parser = new SpelExpressionParser();
Boolean b1 = parser.parseExpression("'haha' instanceof T(String)").getValue(Boolean.class);// true
Boolean b2 = parser.parseExpression("123 instanceof T(String)").getValue(Boolean.class);// false
Boolean b3 = parser.parseExpression("123 instanceof T(java.util.Date)").getValue(Boolean.class);// false

4.变量定义及引用
使用#variableName引用通过EvaluationContext接口的setVariable(variableName, value)方法定义的变量；
使用#root引用根对象
使用#this引用当前上下文对象
ExpressionParser parser = new SpelExpressionParser();
// #variableName
EvaluationContext context = new StandardEvaluationContext();
context.setVariable("name1", "value1");
String str1 = parser.parseExpression("#name1").getValue(context, String.class);// value1
User user = new User();
user.setId("1");
context = new StandardEvaluationContext(user);
// #root（#root可以省略）
String str2 = parser.parseExpression("#root.id").getValue(context, String.class);// 1
String str3 = parser.parseExpression("id").getValue(context, String.class);// 1
// #this
String str4 = parser.parseExpression("#this.id").getValue(user, String.class);// 1

5.赋值
SpEL支持给自定义变量赋值，也允许给根对象赋值，直接使用#variableName=value即可赋值。
使用#variable=value给自定义变量赋值
使用#root=value给根对象赋值
使用#this=value给当前上下文对象赋值
ExpressionParser parser = new SpelExpressionParser();
// #variableName
EvaluationContext context = new StandardEvaluationContext();
User user = new User();
user.setId("1");
context.setVariable("name1", user);
String str1 = parser.parseExpression("#name1.id='2'").getValue(context, String.class);// 2
context = new StandardEvaluationContext(user);
// #root（#root可以省略）
String str2 = parser.parseExpression("#root.id='3'").getValue(context, String.class);// 3
String str3 = parser.parseExpression("id='4'").getValue(context, String.class);// 4
// #this
String str4 = parser.parseExpression("#this.id='5'").getValue(user, String.class);// 5

6.自定义函数
SpEL支持类静态方法注册为自定义函数。SpEL使用StandardEvaluationContext的registerFunction方法进行注册自定义函数（等同于使用setVariable）。
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext context = new StandardEvaluationContext();
Method parseInt = Integer.class.getDeclaredMethod("valueOf", String.class);
context.registerFunction("value", parseInt);
// 等同于Integer.valueOf("2").byteValue()
Byte b = parser.parseExpression("#value('2').byteValue()").getValue(context, Byte.class);// 2

7.对象属性获取及安全导航
对象属性获取：使用如object.property.property这种点缀式获取
安全导航：SpEL引入了Groovy语言中的安全导航运算符"(对象|属性)?.属性"，用来避免"?."前边的表达式为null时抛出空指针异常，而是返回null；修改对象属性值则可以通过赋值表达式或Expression接口的setValue方法修改。
安全导航运算符前面的#root可以省略，但后面的#root不可省略。
ExpressionParser parser = new SpelExpressionParser();
User user = new User();
user.setId("1");
StandardEvaluationContext context = new StandardEvaluationContext(user);
// 对象属性获取
String str1 = parser.parseExpression("id").getValue(context, String.class);// 1
String str2 = parser.parseExpression("Id").getValue(context, String.class);// 1
// 安全导航
User user1 = parser.parseExpression("#root?.#root").getValue(context, User.class);// {"id":"1"}
String str3 = parser.parseExpression("id?.#root.id").getValue(context, String.class);// 1
String str4 = parser.parseExpression("userName?.#root.userName").getValue(context, String.class);// null

8.对象方法调用
SpEL支持对象方法调用，使用方法跟Java语法一样。对于根对象可以直接调用方法。
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext context = new StandardEvaluationContext("user");
// 对象方法调用
String str1 = parser.parseExpression("#root.substring(1, 2)").getValue(context, String.class);// s
String str2 = parser.parseExpression("substring(1, 2)").getValue(context, String.class);// 1

9.Bean引用
SpEL支持使用@符号来引用Bean，在引用Bean时需要使用BeanResolver接口实现来查找Bean，Spring提供BeanFactoryResolver实现。
    @Autowired
    private ApplicationContext applicationContext;

    @GetMapping("test1")
    public ApiResult test1() {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext context = new StandardEvaluationContext();
        context.setBeanResolver(new BeanFactoryResolver(applicationContext));
        // 获取Spring容器中beanName为systemProperties的bean
        Properties systemProperties = parser.parseExpression("@systemProperties").getValue(context, Properties.class);
        System.out.println(JSON.toJSONString(systemProperties));
        // RedisProperties redisProperties = parser.parseExpression("@spring.redis-org.springframework.boot.autoconfigure.data.redis.RedisProperties").getValue(context, RedisProperties.class);
        // System.out.println(JSON.toJSONString(redisProperties));
        return ApiUtil.success();
    }
```

## 集合相关表达式

```java
1.内联数组定义
SpEL支持内联数组（Array）定义。但不支持多维内联数组初始化。
ExpressionParser parser = new SpelExpressionParser();
// 定义一维数组并初始化
int[] array1 = parser.parseExpression("new int[2]{1,2}").getValue(int[].class);
System.out.println(JSON.toJSONString(array1));
// 定义二维数组但不初始化
String[][] array2 = parser.parseExpression("new String[2][2]").getValue(String[][].class);
System.out.println(JSON.toJSONString(array2));
// 定义二维数组并初始化（会报错）
String[][] array3 = parser.parseExpression("new String[2][2]{{'1','2'},{'3','4'}}").getValue(String[][].class);
System.out.println(JSON.toJSONString(array3));

2.内联集合定义
SpEL支持内联集合（List）定义。使用{表达式，……}定义内联List，如{1,2,3}将返回一个整型的ArrayList，而{}将返回空的List
ExpressionParser parser = new SpelExpressionParser();
// 将返回不可修改的空List
List<Integer> list1= parser.parseExpression("{}").getValue(List.class);
// 对于字面量列表也将返回不可修改的List
List<Integer> list2 = parser.parseExpression("{1,2,3}").getValue(List.class);
//对于列表中只要有一个不是字面量表达式，将只返回原始List（可修改）
String expression3 = "{{1+2,2+4},{3,4+4}}";
List<List<Integer>> list3 = parser.parseExpression(expression3).getValue(List.class);
result3.get(0).set(0, 1);

3.数组、集合、字典元素访问
ExpressionParser parser = new SpelExpressionParser();
// 数组元素访问
int[] array1 = {1,2,3};
Integer int1 = parser.parseExpression("[0]").getValue(array1, int.class);
// 集合元素访问
Integer int2 = parser.parseExpression("{1,2,3}[0]").getValue(int.class);
List<Integer> list1 = Stream.of(1, 2, 3).collect(Collectors.toList());
Integer int3 = parser.parseExpression("[0]").getValue(list1, Integer.class);
// 字典元素访问
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
map.put("c", 3);
Integer int4 = parser.parseExpression("['b']").getValue(map, Integer.class);

4.数组、集合、字典元素修改
ExpressionParser parser = new SpelExpressionParser();
// 数组元素修改
int[] array = new int[] {1, 2, 3};
parser.parseExpression("[0] = 3").getValue(array, int.class);
// 集合元素修改
List<Integer> list = Stream.of(1, 2, 3).collect(Collectors.toList());
parser.parseExpression("[0] = 3").getValue(list, Integer.class);
// 字典元素修改
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
map.put("c", 3);
parser.parseExpression("['b'] = 3").getValue(map, Integer.class);

5.数组、集合、字典投影
SpEL支持数组、集合、字典投影。SpEL根据原集合中的元素中通过选择来构造另一个集合，该集合和原集合具有相同数量的元素。数组和集合类似，字典构造后是集合（不是字典）
SpEL使用list|map.![投影表达式]来进行投影运算。
数组、集合中的#this表示每个元素
字典中的#this表示每个Map.Entry，所有可以用#this.key、#this.value来获取键和值。
代码中.!后面的#this都可以省略，但.!前面的#root不可省略
ExpressionParser parser = new SpelExpressionParser();
// 数组投影（#this可省略）
int[] array = new int[] {1, 2, 3};
int[] array1 = parser.parseExpression("#root.![#this+1]").getValue(array, int[].class);
// 集合投影（#this可省略）
List<Integer> list = Stream.of(1, 2, 3).collect(Collectors.toList());
List list1 = parser.parseExpression("#root.![#this+1]").getValue(list, List.class);
// 字典投影（#this可省略）
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
map.put("c", 3);
List list2 = parser.parseExpression("#root.![#this.key+1]").getValue(map, List.class);
List list3 = parser.parseExpression("#root.![#this.value+1]").getValue(map, List.class);

6.数组、集合、字典选择
SpEL支持数组、集合、字典选择。SpEL根据原集合通过条件表达式选择出满足条件的元素并构造为新的集合。数组和字典类似。
SpEL使用“(list|map).?[选择表达式]”，其中选择表达式结果必须是boolean类型，如果true则选择的元素将添加到新集合中，false将不添加到新集合中。
ExpressionParser parser = new SpelExpressionParser();
// 数组选择
int[] array = new int[] {1, 2, 3};
int[] array1 = parser.parseExpression("#root.?[#this>1]").getValue(array, int[].class);
// 集合选择
List<Integer> list = Stream.of(1, 2, 3).collect(Collectors.toList());
List list1 = parser.parseExpression("#root.?[#this>1]").getValue(list, List.class);
// 字典选择
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
map.put("c", 3);
Map map1 = parser.parseExpression("#root.?[#this.key!='a']").getValue(map, Map.class);
Map map2 = parser.parseExpression("#root.?[#this.value>1]").getValue(map, Map.class);
```


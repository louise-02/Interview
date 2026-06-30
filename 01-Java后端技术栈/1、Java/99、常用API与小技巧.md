JDK 标准库常用 API 速查：**能抄的写法 + 踩坑**。第三方库（Hutool、Commons、Jackson）见 [工具类/2、日期时间与JSON工具](../99、Other/工具类/2、日期时间与JSON工具.md)。

---

# 1、日期时间（优先 java.time）

## 类怎么选

| 类 | 用途 |
| -- | ---- |
| `LocalDate` | 只有日期（生日、账期日） |
| `LocalTime` | 只有时间（开门 09:00） |
| `LocalDateTime` | 日期+时间，**无时区**（库表、业务时间多数够用） |
| `ZonedDateTime` | 带时区（跨国、夏令时） |
| `Instant` | UTC 时间戳（日志、接口 epoch） |
| `YearMonth` | 年月（2024-06 账单月） |
| `Period` | 日期间隔（2 年 3 月） |
| `Duration` | 时间间隔（2 小时 30 分） |

## LocalDate / LocalDateTime

```java
LocalDate today = LocalDate.now();
LocalDateTime now = LocalDateTime.now();
LocalTime time = LocalTime.of(9, 30, 0);

// 构造
LocalDate date = LocalDate.of(2024, 6, 12);
LocalDateTime dt = LocalDate.atTime(23, 59, 59);

// 格式化 / 解析（DateTimeFormatter 线程安全，可 static）
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = now.format(fmt);
LocalDateTime parsed = LocalDateTime.parse("2024-06-12 10:30:00", fmt);

// 加减
LocalDate nextWeek = today.plusWeeks(1);
LocalDateTime minusHours = now.minusHours(2);

// 比较
if (today.isBefore(LocalDate.of(2024, 12, 31))) { ... }
if (!today.equals(LocalDate.now())) { ... }
```

## Period / Duration / ChronoUnit

```java
// 两个日期之间：年/月/日
Period p = Period.between(LocalDate.of(2024, 1, 1), LocalDate.of(2024, 6, 12));
int months = p.getMonths();

// 两个时刻之间：时分秒
Duration d = Duration.between(start, end);
long minutes = d.toMinutes();

// 直接算天数/小时数
long days = ChronoUnit.DAYS.between(date1, date2);
long hours = ChronoUnit.HOURS.between(dt1, dt2);
```

## TemporalAdjusters（月初、月末、下一个周五）

```java
import static java.time.temporal.TemporalAdjusters.*;

LocalDate first = today.with(firstDayOfMonth());       // 本月 1 号
LocalDate last = today.with(lastDayOfMonth());         // 本月最后一天
LocalDate nextFri = today.with(next(DayOfWeek.FRIDAY)); // 下一个周五

// 当天起止（查库区间常用）
LocalDateTime startOfDay = today.atStartOfDay();                    // 00:00:00
LocalDateTime endOfDay = today.atTime(LocalTime.MAX);               // 23:59:59.999...
// 或：today.plusDays(1).atStartOfDay() 作为 [start, end) 右开区间
```

## YearMonth（按月）

```java
YearMonth ym = YearMonth.of(2024, 6);
LocalDate firstDay = ym.atDay(1);
LocalDate lastDay = ym.atEndOfMonth();
YearMonth next = ym.plusMonths(1);
```

## 时间戳与时区

```java
long millis = System.currentTimeMillis();
long millis2 = Instant.now().toEpochMilli();

LocalDateTime ldt = Instant.ofEpochMilli(millis)
        .atZone(ZoneId.systemDefault())
        .toLocalDateTime();

ZonedDateTime shanghai = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
Instant utc = Instant.now();

// LocalDateTime → Date（对接老接口）
Date legacy = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```

## 业务常见写法

```java
// 判断是否同一天
boolean sameDay = dt1.toLocalDate().equals(dt2.toLocalDate());

// 年龄（粗略）
int age = Period.between(birthday, LocalDate.now()).getYears();

// 本周一（以周一为一周开始）
LocalDate monday = today.with(DayOfWeek.MONDAY);
if (today.getDayOfWeek().getValue() < DayOfWeek.MONDAY.getValue()) {
    monday = monday.minusWeeks(1);
}
// JDK 8 也可：today.with(previousOrSame(DayOfWeek.MONDAY))
```

## 注意

| 点 | 说明 |
| -- | ---- |
| 新项目 | **业务计算用 `java.time`**，别用 `Date` / `Calendar` |
| 格式化 | `DateTimeFormatter` 可 static；**不要** static 共享 `SimpleDateFormat` |
| 时区 | 存库/接口约定 UTC 或固定 `Asia/Shanghai`，避免服务器默认时区不一致 |
| JSON | Spring Boot 配 `Jackson JavaTimeModule`，见 [工具类/2](../99、Other/工具类/2、日期时间与JSON工具.md) |

---

# 2、Date / Calendar / SimpleDateFormat（legacy）

老项目、老 SDK 里常见。**能改就改 java.time**；对接时做转换即可。

## java.util.Date

```java
Date now = new Date();                          // 当前时刻
long t = now.getTime();                         // 毫秒时间戳

// 很多方法已过时，别在新代码里 getYear()/getMonth()
//  month 从 0 开始，容易错
```

## Calendar（可变、非线程安全）

```java
Calendar cal = Calendar.getInstance();          // 默认时区
cal.set(2024, Calendar.JUNE, 12, 10, 30, 0);   // 注意：月份用 Calendar.JUNE，不是 6
cal.add(Calendar.DAY_OF_MONTH, 7);              // 加 7 天
cal.get(Calendar.YEAR);
cal.get(Calendar.MONTH);                        // 0～11，6 表示七月
Date date = cal.getTime();

// 月初
cal.set(Calendar.DAY_OF_MONTH, 1);
cal.set(Calendar.HOUR_OF_DAY, 0);
cal.set(Calendar.MINUTE, 0);
cal.set(Calendar.SECOND, 0);
cal.set(Calendar.MILLISECOND, 0);
```

| Calendar 字段 | 说明 |
| ------------- | ---- |
| `MONTH` | **0 起算**：0=一月，11=十二月 |
| `DAY_OF_WEEK` | 1=周日（美国习惯），与 `DayOfWeek` 不同 |
| `add` vs `roll` | `add` 进位到上级字段；`roll` 只在当前字段循环 |

**Calendar 的问题**：API 难记、可变、非线程安全、月份/星期易错 → 用 `LocalDate.plusDays()` / `TemporalAdjusters` 替代。

## SimpleDateFormat（非线程安全）

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String s = sdf.format(new Date());
Date date = sdf.parse("2024-06-12 10:30:00");
```

## ThreadLocal 复用 SimpleDateFormat

```java
private static final ThreadLocal<SimpleDateFormat> SDF =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

public static String format(Date date) {
    return SDF.get().format(date);
}
```

## legacy ↔ java.time 互转

```java
// Date → LocalDateTime
LocalDateTime ldt = date.toInstant()
        .atZone(ZoneId.systemDefault())
        .toLocalDateTime();

// LocalDateTime → Date
Date date = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());

// Calendar → LocalDateTime
LocalDateTime ldt2 = cal.toInstant()
        .atZone(ZoneId.systemDefault())
        .toLocalDateTime();
```

## sql 包

| 类 | 说明 |
| -- | ---- |
| `java.sql.Date` | 仅日期，JDBC `DATE` |
| `java.sql.Time` | 仅时间 |
| `java.sql.Timestamp` | 日期+时间+纳秒，JDBC `TIMESTAMP` |

JDBC 4.2+ 可直接用 `LocalDate` / `LocalDateTime`；老代码才频繁见到 `Timestamp`。

---

# 3、BigDecimal（金额 / 精确小数）

## 构造：用 String，别用 double

```java
// 错：new BigDecimal(0.1) 仍有精度问题
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
BigDecimal sum = a.add(b);                    // 0.3

// 也可
BigDecimal c = BigDecimal.valueOf(123, 2);    // 1.23
```

## 四则与比较

```java
BigDecimal price = new BigDecimal("19.99");
BigDecimal qty = new BigDecimal("3");

BigDecimal total = price.multiply(qty);
BigDecimal half = total.divide(new BigDecimal("2"), 2, RoundingMode.HALF_UP);

// 比较用 compareTo，不要用 equals（1.0 与 1.00 scale 不同 equals 为 false）
if (total.compareTo(BigDecimal.ZERO) > 0) { ... }
```

## 常用舍入

```java
amount.setScale(2, RoundingMode.HALF_UP);   // 四舍五入两位小数
amount.setScale(2, RoundingMode.DOWN);       // 截断
```

## 注意

| 点 | 说明 |
| -- | ---- |
| 金额 | 元转分：`amount.multiply(new BigDecimal("100")).longValue()` |
| 除法 | `divide` 必须指定 **scale + RoundingMode**，否则除不尽抛异常 |
| 比较 | **`compareTo`**，不用 `==` 或 `equals` 比大小 |

---

# 4、Stream

## 过滤、映射、收集

```java
List<String> names = users.stream()
        .filter(u -> u.getAge() >= 18)
        .map(User::getName)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
```

## List → Map

```java
// key 冲突会抛异常
Map<Long, User> map = users.stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));

// key 冲突保留后者
Map<Long, User> map2 = users.stream()
        .collect(Collectors.toMap(User::getId, Function.identity(), (a, b) -> b));

// 分组
Map<String, List<User>> byDept = users.stream()
        .collect(Collectors.groupingBy(User::getDept));
```

## flatMap（map + 压平）

**map**：一个元素 → **一个**新元素。  
**flatMap**：一个元素 → **零个或多个**新元素，再全部摊到同一条流里（先 map 成流，再 flatten）。

```java
// map：得到 Stream<Stream<String>>，嵌套
List<List<String>> nested = List.of(List.of("a", "b"), List.of("c"));
// nested.stream().map(List::stream)  // 不好用

// flatMap：摊平为一层
List<String> flat = nested.stream()
        .flatMap(List::stream)              // 等价 flatMap(list -> list.stream())
        .collect(Collectors.toList());      // [a, b, c]

// 业务：每个订单多个 tag → 所有 tag
List<String> tags = orders.stream()
        .flatMap(o -> o.getTags().stream())
        .distinct()
        .collect(Collectors.toList());
```

## findFirst / Optional 链

```java
Optional<User> first = users.stream()
        .filter(u -> "admin".equals(u.getRole()))
        .findFirst();
```

## 数值统计

```java
int max = list.stream().mapToInt(Integer::intValue).max().orElse(0);
double avg = list.stream().mapToInt(Integer::intValue).average().orElse(0);
```

## 注意

| 点 | 说明 |
| -- | ---- |
| 空集合 | `stream()` 对 null 会 NPE，先判空或用 `Optional.ofNullable(list).orElse(emptyList())` |
| 副作用 | 避免在 `forEach` 里改外部变量；需要副作用用普通 for 或明确文档 |
| parallelStream | 仅 **CPU 密集、数据量大、无共享可变状态** 时考虑；IO 密集不适合 |
| 已关闭流 | Stream 只能消费一次 |

---

# 5、Optional / Objects

`Optional<T>` 表示**可能有、可能没有**的值，主要用在**方法返回值**，避免直接返回 `null` 让调用方忘了判空。

**不要用 Optional 作实体字段、DTO 字段、方法参数**（除非 API 框架强制）；集合元素也不要包一层 Optional。

---

## Objects 常用

```java
Objects.requireNonNull(name, "name不能为空");   // null 抛 NPE
Objects.requireNonNullElse(name, "默认");       // null 时用默认值
Objects.equals(a, b);                           // null 安全 equals
Objects.hash(a, b, c);                          // 配合 equals 做 hashCode
```

---

## 创建 Optional

```java
Optional<String> a = Optional.of("x");           // 值不能为 null，否则 NPE
Optional<String> b = Optional.ofNullable(str);   // str 为 null → Optional.empty()
Optional<String> c = Optional.empty();           // 明确的「无值」
```

| 方法 | 说明 |
| ---- | ---- |
| `Optional.of(v)` | v **不能**为 null |
| `Optional.ofNullable(v)` | v 为 null 时得到 `empty` |
| `Optional.empty()` | 无值 |

---

## 判断与消费

```java
if (opt.isPresent()) { ... }        // 有值
if (opt.isEmpty()) { ... }           // Java 11+，无值

opt.ifPresent(v -> log.info("{}", v));

opt.ifPresentOrElse(
        v -> System.out.println(v),
        () -> System.out.println("无数据")
);                                   // Java 9+
```

> 链式写法优先于 `if (opt.isPresent()) opt.get()`。

---

## map / flatMap / filter

```java
// map：有值则变换，无值仍为 empty
Optional<String> name = Optional.ofNullable(user)
        .map(User::getName);

// flatMap：变换结果仍是 Optional 时用（避免 Optional<Optional<...>>）
Optional<String> deptName = Optional.ofNullable(user)
        .flatMap(u -> Optional.ofNullable(u.getDept()))
        .map(Dept::getName);

// filter：不满足条件变为 empty
Optional<User> adult = Optional.ofNullable(user)
        .filter(u -> u.getAge() >= 18);
```

---

## 取值：orElse / orElseGet / orElseThrow

```java
String n1 = opt.orElse("默认");                    // 无值返回字面默认值
String n2 = opt.orElseGet(() -> loadDefault());    // 无值时才执行 Supplier（懒加载）
String n3 = opt.orElseThrow();                     // 无值抛 NoSuchElementException
String n4 = opt.orElseThrow(() -> new BizException("用户不存在"));

// 不推荐：无值也调 get()
// String bad = opt.get();   // empty 时抛异常
```

| 方法 | 区别 |
| ---- | ---- |
| `orElse(默认)` | **无论有没有值**，默认值表达式都会参与求值（若写方法调用会白跑） |
| `orElseGet(Supplier)` | **只有 empty 时**才执行 Supplier，适合默认值要查库/计算 |
| `orElseThrow()` | 无值抛异常，业务上「必须有」时用 |

```java
// 错：user 有值时 loadDefault() 也会执行
opt.orElse(loadDefault());

// 对：仅 empty 时才 load
opt.orElseGet(this::loadDefault);
```

---

## 与 Stream 配合（Java 9+）

```java
List<String> names = users.stream()
        .map(u -> Optional.ofNullable(u.getNickName()))  // 不推荐包 Optional
        .flatMap(Optional::stream)                       // 有值留 1 个，无值过滤掉
        .collect(Collectors.toList());

// 更常见：flatMap + Optional.ofNullable(...).stream()
List<String> names2 = users.stream()
        .flatMap(u -> Optional.ofNullable(u.getNickName()).stream())
        .collect(Collectors.toList());
```

---

## 典型写法示例

```java
// Service 返回 Optional，调用方决定默认值或异常
public Optional<User> findById(Long id) {
    User user = userMapper.selectById(id);
    return Optional.ofNullable(user);
}

// 调用
User user = userService.findById(id)
        .orElseThrow(() -> new BizException("用户不存在"));

String display = userService.findById(id)
        .map(User::getName)
        .orElse("匿名");
```

---

## 注意

| 点 | 说明 |
| -- | ---- |
| 用途 | **方法返回值**可能为空时用；不是万能 null 包装器 |
| 实体字段 | **不要** `private Optional<String> name`；JSON 序列化、MyBatis 映射都麻烦 |
| 方法参数 | 一般直接允许 null 或 `@NonNull`，别用 Optional 参数 |
| `get()` | 先 `isPresent` 或改用 `orElse*`；别裸 `get()` |
| `of(null)` | 抛 NPE；不确定用 `ofNullable` |
| 集合 | `List<Optional<T>>` 几乎总是坏设计；用 `filter` / `flatMap` 过滤 null |

---

# 6、字符串与集合片段

```java
String joined = String.join(",", list);
List<String> copy = List.copyOf(list);           // 不可变，JDK9+
var map = Map.of("k1", "v1", "k2", "v2");         // 不可变，最多10对

Collections.sort(list, Comparator.comparing(User::getName));
list.sort(Comparator.comparing(User::getName).reversed());
```

---

# 7、ZIP 压缩与解压

## 压缩目录为 zip

```java
public static void zipDir(Path sourceDir, Path zipFile) throws IOException {
    try (ZipOutputStream zos = new ZipOutputStream(Files.newOutputStream(zipFile))) {
        Files.walk(sourceDir)
                .filter(path -> !Files.isDirectory(path))
                .forEach(path -> {
                    String entryName = sourceDir.relativize(path).toString()
                            .replace('\\', '/');
                    try {
                        zos.putNextEntry(new ZipEntry(entryName));
                        Files.copy(path, zos);
                        zos.closeEntry();
                    } catch (IOException e) {
                        throw new UncheckedIOException(e);
                    }
                });
    }
}
```

## 解压 zip 到目录

```java
public static void unzip(Path zipFile, Path targetDir) throws IOException {
    Files.createDirectories(targetDir);
    try (ZipInputStream zis = new ZipInputStream(Files.newInputStream(zipFile))) {
        ZipEntry entry;
        while ((entry = zis.getNextEntry()) != null) {
            Path out = targetDir.resolve(entry.getName()).normalize();
            // 防 Zip Slip：解压路径必须在目标目录内
            if (!out.startsWith(targetDir)) {
                throw new IOException("非法条目: " + entry.getName());
            }
            if (entry.isDirectory()) {
                Files.createDirectories(out);
            } else {
                Files.createDirectories(out.getParent());
                Files.copy(zis, out, StandardCopyOption.REPLACE_EXISTING);
            }
            zis.closeEntry();
        }
    }
}
```

## 压缩单个文件（简单）

```java
try (ZipOutputStream zos = new ZipOutputStream(Files.newOutputStream(Paths.get("out.zip")));
     InputStream in = Files.newInputStream(Paths.get("a.txt"))) {
    zos.putNextEntry(new ZipEntry("a.txt"));
    in.transferTo(zos);
    zos.closeEntry();
}
```

## 注意

| 点 | 说明 |
| -- | ---- |
| Zip Slip | 解压必须校验 `out.startsWith(targetDir)` |
| 中文文件名 | 老工具生成的 zip 可能是 GBK，需指定编码或改用 Apache Commons Compress |
| 大文件 | 流式读写，不要一次性读入内存 |

---

# 8、Base64 / 摘要（JDK 自带）

```java
String b64 = Base64.getEncoder().encodeToString(bytes);
byte[] raw = Base64.getDecoder().decode(b64);

MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] hash = md.digest(str.getBytes(StandardCharsets.UTF_8));
String hex = DatatypeConverter.printHexBinary(hash).toLowerCase(); // 或手动转 hex
```

常用 MD5/SHA 也可直接用 Commons Codec / Hutool，见 [工具类/1](../99、Other/工具类/1、Java与第三方常用工具库.md)。

---

# 9、反射：从父类泛型取 Class&lt;T&gt;

子类写死泛型实参时（如 `UserDao extends BaseDao<User>`），可在运行时拿到 `User.class`，MyBatis 泛型基类、JSON 反序列化类型常用。

## 写法

```java
@SuppressWarnings("unchecked")
public Class<T> getEntityClass() {
    Class<?> clazz = this.getClass();
    Type genericSuperclass = clazz.getGenericSuperclass();
    if (genericSuperclass instanceof ParameterizedType) {
        Type[] actualTypeArguments = ((ParameterizedType) genericSuperclass).getActualTypeArguments();
        return (Class<T>) actualTypeArguments[0];
    }
    return null;
}
```

## 使用示例

```java
public abstract class BaseService<T> {

    @SuppressWarnings("unchecked")
    protected Class<T> getEntityClass() {
        Type type = getClass().getGenericSuperclass();
        if (type instanceof ParameterizedType) {
            return (Class<T>) ((ParameterizedType) type).getActualTypeArguments()[0];
        }
        return null;
    }
}

public class UserService extends BaseService<User> {
    public void demo() {
        Class<User> clazz = getEntityClass();   // User.class
    }
}
```

## 注意

| 点 | 说明 |
| -- | ---- |
| 子类必须带具体泛型 | `extends BaseService<User>` 可以；裸写 `extends BaseService` 拿不到 |
| 多层继承 | 若中间还有泛型父类，可能要沿 `getGenericSuperclass()` 向上找 |
| 擦除 | 编译后泛型擦除，**靠子类声明的实参类型**才能反射出来 |
| 返回值 | 取不到时返回 `null`，调用方要判空或抛业务异常 |

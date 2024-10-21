[合集 \- 卓越工程(4\)](https://github.com)[1\.为什么需要依赖注入10\-07](https://github.com/OceanEyes/p/18450799)[2\.我在大厂做 CR——为什么建议使用枚举来替换布尔值10\-16](https://github.com/OceanEyes/p/18468830)3\.我在大厂做 CR——如何体系化防控空指针异常10\-21[4\.我在大厂做 CR——再谈如何优雅修改代码10\-07](https://github.com/OceanEyes/p/18450797)收起**阅读目录**

* [什么是空指针异常](#_label0)
* [CR 我们要做什么](#_label1)
* [再谈空指针防控手段](#_label2)
* [写在最后](#_label3):[veee加速器](https://liuyunzhuge.com)

大家好，我是木宛哥，今天和大家分享下——代码 CR 时针对恼人的空指针异常（`NullPointerException`）如何做到体系化去防控；


[回到顶部](#_labelTop)### 什么是空指针异常


从内存角度看，对象的实例化需要在堆内存中分配空间。如果一个对象没有被创建，那也就没有分配内存，当应用程序访问空对象时，实际上是访问一个“无效”的内存区域，从而导致系统抛出异常。


我们在 Java 编程时，空指针异常是一个常见的运行时错误，严重甚至会导致进程退出。所以这也是为什么我们要在 CR 时如此重视它的原因。


[回到顶部](#_labelTop)### CR 我们要做什么


木宛哥认为 CR 应该重点关注三点：


* 业务逻辑正确性
* 代码的可读性和可维护性
* 代码的健壮性和稳定性


OK，再回过头来看，针对空指针异常，在 CR 时，更多要从代码的健壮性和稳定性切入，可分为：


* **防御性去杜绝空指针异常的出现（大多是访问了不存在的对象）**
* **对三方框架或者 JDK 认知不完善导致的潜在空指针异常发生（更多靠评审参与者经验分享）**


#### 防御性编程


防御性编程是非常有必要的，一方面可以提高系统稳定性和健壮性，另一方面可以形成比较好的代码规范；同时也是非常重要的思想，每个人都会如此去实践，在 CR 时针对空指针异常的防御是 `common sense`；例如：


**1\.防御使用了未初始化的对象：**



```
MyObject obj = null;
if (obj!= null){
    obj.someMethod();
}

```

**2\.防御使用了对象没有初始化的字段；**



```
class MyClass {
   String name;
}
MyClass obj = new MyClass();
if (StringUtils.isNotBlank(obj.getName())){
    // do something
}

```

**3\.防御当调用方法返回 null 后，试图对返回的对象调用其方法或属性：**



```
MyObject obj = getMyObject(); // 假设返回 null
if (obj != null) {
    obj.someMethod();
}

```

**4\.防御访问了集合中不存在的元素：**


* Map.get() 方法返回 null
* Queue 的方法如 poll() 或 peek() 返回 null



```
Map dummyMap = new HashMap<>();
String value = dummyMap.get("key");
if (org.apache.commons.lang3.StringUtils.isNotBlank(value)){
    // do something
}

```

#### 三方框架或 JDK 使用不当引发空指针异常提前排雷


这一类更多是三方框架或 JDK 的内部机制不清楚导致的踩坑，只有踩了这种类，


才会恍然大悟：“哦，原来这样啊，下回得注意了”；


所以针对这类问题，更多需要评审参与人的经验去发现，需要团队去共创，共建知识体系，例如：在团队空间维护“ TOP 100 踩坑记”等等；


在上篇文章《为什么建议使用枚举来替换布尔值》中，木宛哥提到过 `Boolean` 为 `null` 时产生的第三种结果，易造成 `if` 条件判断拆箱引发空指针问题，今天再继续分享其他：


**1\.三目运算符拆箱空指针问题**



```
int var1 = 20;
Integer var2 = null;
boolean condition = true;
// 三目运算符拆箱问题，发生 NullPointerException
System.out.println(condition ? var2 : var1);

```

这里：`condition` 为 `true`，所以三目运算符选择了 `var2`（即 `null`）。即： `var2`（`Integer` 类型）赋值给 `num`（`Integer` 类型）。理论上在这里应该是 `num` 被赋值为 `null`。


但在 Java 中，三目运算符的返回类型需要通过类型来推导：


* 如果 `var1` 是 `int` 类型，而 `var2` 是 `Integer` 类型，三目运算符会将它们的类型推导合并，令返回值为 `int` 类型。
* 这意味着，如果 `condition` 为 `true`，则会尝试将 `var2`（`null`）拆箱成 `int`。由于 `null` 不能拆箱成 `int`，因此会抛出 `NullPointerException`


这类典型的问题更多需要在 CR 时提前暴露出来，保证一致的参数类型来避免拆箱；


**2\.日志打印使用 fastjson 序列化时造成的空指针问题**


大部分程序员编程开发习惯，喜欢打印参数到日志里，但有时候一个不起眼的 `log.info` 打印日志有可能导致接口异常；


如下打印日志，结果 `fastjson` 序列化异常，发生 `NullPointerException`



```
@Test
public void testJSONString() {
    Employee employee = new Employee("jack", 100);
    //fastjson 序列化异常，发生 NullPointerException
    LoggerUtil.info(logger,"{}",JSON.toJSONString(employee));
}

static class Employee {

    private EmployeeId employeeId;
    private String name;
    private Integer salary;

    public Employee(String name, Integer salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public Integer getSalary() {
        return salary;
    }

    public String getEmployeeId() {
        return this.employeeId.getId();
    }
}

static class EmployeeId {
    private String id;
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }
}

```

原因在于 `fastjson` 使用 `JSON.toJSONString(employee)` 序列化成 JSON 时，底层实际通过解析 `get` 开头方法来识别属性，即：调用 `get` 方法获取属性的 `value` 值；上述代码：`employeeId` 为 `null` ，但序列化时执行了 `getEmployeeId` 引发的空指针异常；


所以：特别是大家在实践 DDD 的时候，因为领域模型往往是充血模型，不仅有数据还包含了行为，对于行为可能习惯有 `get` 开头命名，要特别重视在打印领域模型时序列化问题；


**3\.对 Stream 流操作认知不完善导致的空指针异常**


如果 Stream 流中存在空值，需要非常小心。


例如，如果第一个元素恰好为 `null`，`findFirst()` 将抛出 `NullPointerException`。这是因为 `findFirst()` 返回一个 `Optional`，而 `Optional` 不能包含空值。



```
Arrays.asList(null, 1, 2).stream().findFirst();//发生 NullPointerException

```

`max()`、`min()` 和 `reduce()`，也表现出类似的行为。如果 `null` 是最终结果，则会抛出异常。



```
List list = Arrays.asList(null, 1, 2);
var comparator = Comparator.nullsLast(Comparator.naturalOrder());
System.out.println(list.stream().max(comparator));//发生 NullPointerException

```

再例如：我们在使用 `Stream` 流式编程时，如果流包含 `null`，可以转换为 `toList()` 或 `toSet()`；


然而，`toMap()` 要注意， 不允许空值（允许空Key）：



```
Employee employee1 = new Employee("Jack", 10000);
Employee employee2 = new Employee(null, 10000);
//toMap的Value不能为空，此处异常
Map salaryMap = Arrays.asList(employee1, employee2)
    .stream()
    .collect(Collectors.toMap(Employee::getSalary, Employee::getName));

```

以及：`groupingBy()` 不允许空 Key：



```
Employee employee1 = new Employee("Jack", 10000);
Employee employee2 = new Employee(null, 10000);

//groupingBy的Key不能为空，此处抛异常
Map> result = Stream.of(employee1, employee2)
    .collect(Collectors.groupingBy(Employee::getName));

```

可见在流中使用了空对象存在许多陷阱；所以，在 CR 时，要重点关注 Stream 流的数据来源，避免在流中存在 `null`，不确定的话建议用 `filter(Objects::nonNull)` 将它们过滤掉。


[回到顶部](#_labelTop)### 再谈空指针防控手段


上一章更多还是从防御空指针去解问题，但能保证每个人都是认知一样吗，同时在 CR 时也会有漏网之鱼；下面代码我想每个人都会这样去避免空指针，但难免在某个加班到凌晨的日子，脑袋一抽筋写反了:(



```
if("DEFAULT".equals(var)){
    //do something
}

```

所以，在这一章，木宛哥从数据来源切入，回答：**“能否数据天生就是存在非空的、方法天生就是不会返回 null”？**


从程序角度来看，是合理的；许多变量永远不包含 `null`，许多方法也永远不返回 `null`。我们可以分别称它们为“非空变量”和“非空方法”（`NonNull`)；


其他变量和方法在某些情况下可能会包含或返回 `null`，它们称为“可空”（`Nullable`）；


基于这个理论，在解空指针问题时，提供了另一种方式解法：


* 数据来自三方系统，控制权不在我们，故：不可信任，需要做好防御编程；
* 数据来自自身，控制权在我们，控制数据创建即非空，故：可信任；如下：


#### 尽可能屏蔽 null 值


对输入值进行校验——在公共方法和构造函数中。需在每个 `set` 字段的入口处添加 `Objects.requireNonNull()` 调用。`requireNonNull()` 方法会在其参数为 `null` 时抛出 `NullPointerException`。



```
public class Employee {
    public Employee(String name, Integer salary) {
        this.name = Objects.requireNonNull(name);
        this.salary = Objects.requireNonNull(salary);
    }
}

```

这样做有助于在入口处屏蔽 `null` 值的写入


如果你的方法接受集合作为输入，也可以在方法入口遍历该集合以确保它不包含 `null` 值：



```
public void check(Collection<String> data) { 
    data.forEach(Objects::requireNonNull);
}

```

**注：此处需要视具体集合的大小以及评估性能损耗；**


同样的类型场景，不详细举例了：


* 当你的类型是集合时，返回一个空容器，而不是返回 `null`，可以避免消费方出现空指针异常；
* 使用枚举常量来替换 `Boolean` 来避免拆箱引入的空指针异常
* 非法数据状态，直接短路抛出异常而不是返回 `null`；


#### 善用静态分析工具来辅助


介绍两个重要的注解：`@Nullable`和`@NotNull` 注解：


* `@Nullable` 注解意味着预期被注释的变量可能包含`null`，或者被注释的方法可能返回`null`；
* `@NotNull` 注释意味着预期的值绝不是`null`。并为静态分析提供了提示


这类注解，可以在静态分析工具实时分析潜在的异常；



```
interface Processor {
    @NotNull 
    String getNotNullValue();
    
    @Nullable
    String getNullable();
    
    public void process() {
        //此处警告：条件永远为假，不用多次一举
        if (getNotNullValue() == null) { 
            //do something
        } 
        //此处警告：trim() 调用可能导致 NullPointerException
        System.out.println(getNullable().trim()); 
    }
}

```

#### 再谈使用 Optional 替代 null 的一些注意事项


为了避免使用 `null` ，一些开发者倾向于使用 `Optional` 类型。可以将 `Optional` 想象成一个盒子，它要么是空的，要么包含一个非 `null` 的值：


获取Optional对象有三种标准方式：


* `Optional.empty()` —— 获取一个空的 `Optional`
* `Optional.of(value)` —— 获取一个非空的 `Optional`，如果值为 `null` 则抛出`NullPointerException`
* `Optional.ofNullable(value)` —— 如果值为 `null` 则获取一个空的 `Optional`，否则获取一个包含值的非空 `Optional`


使用 `Optional` 来预防空指针，大问题没有，**但有几个细节需要注意**：


**1\.勿滥用 ofNullable**


一些开发者喜欢在所有地方使用 `ofNullable()`，因为它被认为是更安全的，它从不抛出异常。但不能滥用，如果你已经知道你的值永远不会为`null`，最好使用 `Optional.of()`。在这种情况下，如果你看到一个异常，你会立即知道错了并且修复；


**2\.Optional 造成的代码可读性降低**


如下代码获取员工地址，虽然简洁，但可读性很差，对于嵌套特别深的情况下，我还是不建议使用 `Opinional`，毕竟代码除了给自己看还得让别人也一眼明白意图



```
String employeeAddress = Optional.ofNullable(employee)
        .map(Employee::getAddress)
        .map(Address::getStreet)
        .map(Street::getNo)
        .map(No::getNumber).orElseThrow(() -> new IllegalArgumentException("非法参数"));

```

#### 事后的异常监控


事前禁止写入 `null`，事中防御性编程空指针异常，但真的高枕无忧了吗?


未必，事后所以建立一套好的异常告警机制是非常重要的；


我建议针对关键字：`NullPointerException` 做单独的日志采集，同时配上相应的告警级别：理论上出现 1 次空指针异常就应该介入定位；


当然，特别是在发布周期内，如果 `N` 分钟内出现超过 `M` 次空指针异常那就肯定要快速定位和回滚了；


[回到顶部](#_labelTop)### 写在最后


欢迎关注我的公众号：编程启示录，第一时间获取最新消息；




| 微信 | 公众号 |
| --- | --- |
| image | image |



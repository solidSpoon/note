---
slug: java-functional-programming-explained
title: Java 函数式编程详解
authors: [solidSpoon]
tags: []
---

## 概要

首先一个简单的示例展示一下什么是函数式编程

假设我们有一个「Person」列表

```java
List<Person> people = List.of(
        new Person("John", MALE),
        new Person("Maria", FEMALE),
        new Person("Aisha", FEMALE),
        new Person("Alex", MALE),
        new Person("Alice", FEMALE)
);
```

「Person」的定义如下

```java
private record Person(String name, Gender gender) {}

enum Gender {
    MALE, FEMALE
}
```

如果我们想在列表中找到 FEMALE，我们可以使用这样的命令式方法

```java
List<Person> females = new ArrayList<>();

for (Person person : people) {
    if (FEMALE.equals(person.gender)) {
        females.add(person);
    }
}

for (Person female : females) {
    System.out.println(female);
}
```

但它在声明式方法中更简洁

```java
Predicate<Person> personPredicate = person -> FEMALE.equals(person.gender);
var females2 = people.stream().filter(personPredicate)
        .collect(Collectors.toList());
//          .forEach(System.out::println);
females2.forEach(System.out::println);
```

---

完整代码如下：

```java
public class Main {
    public static void main(String[] args) {
        List<Person> people = List.of(
                new Person("John", MALE),
                new Person("Maria", FEMALE),
                new Person("Aisha", FEMALE),
                new Person("Alex", MALE),
                new Person("Alice", FEMALE)
        );

        System.out.println("Imperative approach");
        // Imperative approach
        List<Person> females = new ArrayList<>();
        for (Person person : people) {
            if (FEMALE.equals(person.gender)) {
                females.add(person);
            }
        }
        for (Person female : females) {
            System.out.println(female);
        }

        // Declarative approach
        System.out.println("Declarative approach");
        Predicate<Person> personPredicate = person -> FEMALE.equals(person.gender);
        var females2 = people.stream().filter(personPredicate)
                .collect(Collectors.toList());
//                .forEach(System.out::println);
        females2.forEach(System.out::println);
    }

    private record Person(String name, Gender gender) {}
    enum Gender {
        MALE, FEMALE
    }
}
```

## Function and BiFunction

`Function` 表示接受一个参数 \<T\> 并产生结果 \<R\> 的函数。

```java
Function<T, R>
```

以下是 `Function` 的一些例子

```java
static Function<Integer, Integer> incrementByOneFunction = number -> number + 1;
static Function<Integer, Integer> multiplyBy10Function = number -> number * 10;
---usage
var increment2 = incrementByOneFunction.apply(1);
var multiply = multiplyBy10Function.apply(increment2);
```

酷，如果你看不懂，那么我们之前用命令式编程是这么写的

```java
static int incrementByOne(int number) {
    return number + 1;
}
---usage
var increment = incrementByOne(1);
```

更进一步，我们可以结合两个 Function

```java
var addBy1AndThenMultiplyBy10 = incrementByOneFunction.andThen(multiplyBy10Function);
---usage
var ans = addBy1AndThenMultiplyBy10.apply(4);
```

`BiFunction` 表示一个接受两个参数并产生结果的函数。

作为对比，这是一个传统的二参数方法

```java
static int incrementByOneAndMultiply(int number, int numberToMultiplyBy) {
    return (number + 1) * numberToMultiplyBy;
}
---usage
incrementByOneAndMultiply(4, 100);
```

在函数式编程中，我们这样写

```java
static BiFunction<Integer, Integer, Integer> incrementByOneAndMultiplyBiFunction =
        (numberToIncrementByOne, numberToMultiplyBy)
                -> (numberToIncrementByOne + 1) * numberToMultiplyBy;
---usage
incrementByOneAndMultiplyBiFunction.apply(4, 100);
```

---

完整代码

```java
public class _Function {
    public static void main(String[] args) {
        // Function takes 1 argument and produce 1 result
        var increment = incrementByOne(1);
        System.out.println(increment);

        var increment2 = incrementByOneFunction.apply(1);
        System.out.println(increment2);

        var multiply = multiplyBy10Function.apply(increment2);
        System.out.println(multiply);

        var addBy1AndThenMultiplyBy10 = incrementByOneFunction.andThen(multiplyBy10Function);
        var ans = addBy1AndThenMultiplyBy10.apply(4);
        System.out.println(ans);

        // BiFunction takes 2 argument and produce 1 result
        System.out.println(incrementByOneAndMultiply(4, 100));
        System.out.println(incrementByOneAndMultiplyBiFunction.apply(4, 100));
    }

    static Function<Integer, Integer> incrementByOneFunction = number -> number + 1;

    static Function<Integer, Integer> multiplyBy10Function = number -> number * 10;
    static int incrementByOne(int number) {
        return number + 1;
    }
    static BiFunction<Integer, Integer, Integer> incrementByOneAndMultiplyBiFunction =
            (numberToIncrementByOne, numberToMultiplyBy)
                    -> (numberToIncrementByOne + 1) * numberToMultiplyBy;
    static int incrementByOneAndMultiply(int number, int numberToMultiplyBy) {
        return (number + 1) * numberToMultiplyBy;
    }
}
```

## Consumer and BiConsumer

`Consumer` 表示接受单个输入参数并且不返回结果的操作。与大多数其他 Functional interface 不同，**`Consumer` 预计通过副作用进行操作**。

我们的 `Customer` 定义如下

```java
static record Customer(String customerName, String customerPhoneNumber) {}
---usage
var maria = new Customer("Maria", "99999");
```

在命令式编程中，我们这样编写代码

```java
static void greetConsumer(Customer customer) {
    System.out.println("Hello" + customer.customerName
            + ", thanks for registering phone number "
            + customer.customerPhoneNumber);
}
---usage
greetConsumer(maria);
```

在函数式编程中，我们这样编写代码

```java
static Consumer<Customer> greetCustomerConsumer = customer ->
        System.out.println("Hello" + customer.customerName
                + ", thanks for registering phone number "
                + customer.customerPhoneNumber);
---usage
greetCustomerConsumer.accept(maria);
```

`BiConsumer` 是 `Consumer` 的二参数版本，它表示一个接受两个输入参数并且不返回结果的操作。

我们通常编写下面这种命令式编程方法

```java
static void greetConsumerV2(Customer customer, boolean showPhoneNumber) {
    System.out.println("Hello" + customer.customerName
            + ", thanks for registering phone number "
            + (showPhoneNumber ? customer.customerPhoneNumber : "*********"));
}
---usage
greetConsumerV2(maria, false);
```

这是函数式编程版本

```java
static BiConsumer<Customer, Boolean> greetCustomerConsumerV2 = (customer, showPhoneNumber) ->
        System.out.println("Hello" + customer.customerName
                + ", thanks for registering phone number "
                + (showPhoneNumber ? customer.customerPhoneNumber : "*********"));
---usage
greetCustomerConsumerV2.accept(maria, false);
```

---

全部代码

```java
public class _Consumer {
    public static void main(String[] args) {
        var maria = new Customer("Maria", "99999");

        // Normal java function
        greetConsumer(maria);

        // Consumer Functional interface
        greetCustomerConsumer.accept(maria);

        greetCustomerConsumerV2.accept(maria, false);
        greetConsumerV2(maria, false);
    }
    static BiConsumer<Customer, Boolean> greetCustomerConsumerV2 = (customer, showPhoneNumber) ->
            System.out.println("Hello" + customer.customerName
                    + ", thanks for registering phone number "
                    + (showPhoneNumber ? customer.customerPhoneNumber : "*********"));

    static Consumer<Customer> greetCustomerConsumer = customer ->
            System.out.println("Hello" + customer.customerName
                    + ", thanks for registering phone number "
                    + customer.customerPhoneNumber);

    static void greetConsumer(Customer customer) {
        System.out.println("Hello" + customer.customerName
                + ", thanks for registering phone number "
                + customer.customerPhoneNumber);
    }
    static void greetConsumerV2(Customer customer, boolean showPhoneNumber) {
        System.out.println("Hello" + customer.customerName
                + ", thanks for registering phone number "
                + (showPhoneNumber ? customer.customerPhoneNumber : "*********"));
    }

    static record Customer(String customerName, String customerPhoneNumber) {
    }
}
```

## Predicate

`Predicate` 表示一个布尔值函数

在命令式编程中通过这样写达到相同目的

```java
static boolean isPhoneNumberValid(String phoneNumber) {
    return phoneNumber.startsWith("07") && phoneNumber.length() == 11;
}
---usage
var phoneNumberValid = isPhoneNumberValid("07000000000");
```

在函数式编程中，你可以这样写

```java
static Predicate<String> isPhoneNumberValidPredicate = phoneNumber ->
        phoneNumber.startsWith("07") && phoneNumber.length() == 11;

static Predicate<String> containsNumber3 = phoneNumber ->
        phoneNumber.contains("3");
---usage
System.out.println(isPhoneNumberValidPredicate.test("09009877300"));

System.out.println(
        "Is phone number valid and contains number 3 = " +
                isPhoneNumberValidPredicate.and(containsNumber3).test("07009877300")
);

var isPhoneNumberValidAndContainsNumber3 = isPhoneNumberValidPredicate.or(containsNumber3);
System.out.println(
        "Is phone number valid or contains number 3 = " +
                isPhoneNumberValidAndContainsNumber3.test("07000000000")
);
```

还记得我们在概要中写的代码 `stream().filter()` 吗？

```java
var females2 = people.stream().filter(person -> FEMALE.equals(person.gender))
        .collect(Collectors.toList());
```

`filter` 接收的参数就是 `Predicate` ，在 idea 中使用快捷键 「Ctrl + Alt + V」将它的参数提取成变量，我们就会看到 `Predicate`

```java
Predicate<Person> personPredicate = person -> FEMALE.equals(person.gender);
var females2 = people.stream().filter(personPredicate)
        .collect(Collectors.toList());
```

---

完整代码

```java
public class _Predicate {
    public static void main(String[] args) {
        System.out.println("Without predicate");
        var phoneNumberValid = isPhoneNumberValid("07000000000");
        System.out.println(phoneNumberValid);
        System.out.println(isPhoneNumberValid("0700000000"));
        System.out.println(isPhoneNumberValid("09009877300"));

        System.out.println("With Predicate");
        System.out.println(isPhoneNumberValidPredicate.test("07000000000"));
        System.out.println(isPhoneNumberValidPredicate.test("0700000000"));
        System.out.println(isPhoneNumberValidPredicate.test("09009877300"));

        System.out.println(
                "Is phone number valid and contains number 3 = " +
                        isPhoneNumberValidPredicate.and(containsNumber3).test("07009877300")
        );

        var isPhoneNumberValidAndContainsNumber3 = isPhoneNumberValidPredicate.or(containsNumber3);
        System.out.println(
                "Is phone number valid or contains number 3 = " +
                        isPhoneNumberValidAndContainsNumber3.test("07000000000")
        );

    }

    static boolean isPhoneNumberValid(String phoneNumber) {
        return phoneNumber.startsWith("07") && phoneNumber.length() == 11;
    }

    static Predicate<String> isPhoneNumberValidPredicate = phoneNumber ->
            phoneNumber.startsWith("07") && phoneNumber.length() == 11;

    static Predicate<String> containsNumber3 = phoneNumber ->
            phoneNumber.contains("3");
}
```

## Supplier

`Supplier` 不接收任何参数并提供一个结果

在命令式编程中我们可以这样写

```java
static String getDbConnectionUrl() {
    return "jdbc://localhost:5432/users";
}
---usage
System.out.println(getDbConnectionUrl());
```

函数式编程的版本

```java
static Supplier<String> getDbConnectionUrlSupplier = () ->
        "jdbc://localhost:5432/users";

static Supplier<List<String>> getDbConnectionListUrlSupplier = () ->
        List.of(
                "jdbc://localhost:5432/users",
                "jdbc://localhost:5432/customer"
        );
---usage
System.out.println(getDbConnectionUrlSupplier.get());
System.out.println(getDbConnectionListUrlSupplier.get());
```

---

完整代码

```java
public class _Supplier {
    public static void main(String[] args) {
        System.out.println(getDbConnectionUrl());
        System.out.println(getDbConnectionUrlSupplier.get());
        System.out.println(getDbConnectionListUrlSupplier.get());

    }

    static String getDbConnectionUrl() {
        return "jdbc://localhost:5432/users";
    }

    static Supplier<String> getDbConnectionUrlSupplier = () ->
            "jdbc://localhost:5432/users";
    static Supplier<List<String>> getDbConnectionListUrlSupplier = () ->
            List.of(
                    "jdbc://localhost:5432/users",
                    "jdbc://localhost:5432/customer"
            );

}
```

## Stream

首先将前文的定义 Persion 的代码复制过来

```java
private record Person(String name, Gender gender) {
}

enum Gender {
    MALE, FEMALE, PREFER_NOT_TO_SAY
}

List<Person> people = List.of(
        new Person("John", MALE),
        new Person("Maria", FEMALE),
        new Person("Aisha", FEMALE),
        new Person("Alex", MALE),
        new Person("Alice", FEMALE),
        new Person("Bob", PREFER_NOT_TO_SAY)
);
```

通过 Stream 来调用

```java
people.stream()
        .map(Person::name)
        .mapToInt(String::length)
//        .collect(Collectors.toSet())
        .forEach(System.out::println);
```

我们可以把每一步的参数提取成变量，方便观察它们的类型

```java
Function<Person, String> personStringFunction = Person::name;
        ToIntFunction<String> length = String::length;
        IntConsumer println = System.out::println;
        people.stream()
                .map(personStringFunction)
                .mapToInt(length)
//                .collect(Collectors.toSet())
                .forEach(println);
```

Stream 的其他用法

```java
Predicate<Person> personPredicate = person -> FEMALE.equals(person.gender);
var containOnlyFemales = people.stream()
        .allMatch(personPredicate);
System.out.println(containOnlyFemales);

var personHaveFemales = people.stream()
        .anyMatch(personPredicate);
//      .noneMatch(personPredicate);
System.out.println(personHaveFemales);
```

## Optional

`Optional` 会改变你处理空指针的方式

示例：

```java
var value = Optional.ofNullable(null)
        .orElseGet(() -> "default value");
var value2 = Optional.ofNullable("hello")
        .orElseGet(() -> "default value");
```

示例2：

```java
Supplier<IllegalStateException> exception = () -> new IllegalStateException("exception");
var value3 = Optional.ofNullable("hello")
        .orElseThrow(exception);
```

示例3：

```java
Optional.ofNullable("john.gmail.com")
        .ifPresent(email ->
                System.out.println("Sending email to " + email));
Optional.ofNullable(null)
        .ifPresentOrElse(email -> System.out.println("Sending email to " + email),
                () -> System.out.println("Can not send email"));
```

---

完整代码

```java
public class Main {
    public static void main(String[] args) {
        var value = Optional.ofNullable(null)
                .orElseGet(() -> "default value");
        var value2 = Optional.ofNullable("hello")
                .orElseGet(() -> "default value");
        System.out.println(value2);
        Supplier<IllegalStateException> exception = () -> new IllegalStateException("exception");
        var value3 = Optional.ofNullable("hello")
                .orElseThrow(exception);
        Optional.ofNullable("john.gmail.com")
                .ifPresent(email ->
                        System.out.println("Sending email to " + email));
        Optional.ofNullable(null)
                .ifPresentOrElse(email -> System.out.println("Sending email to " + email),
                        () -> System.out.println("Can not send email"));
    }
}
```

## Combinator Pattern

我们有一个 Customer 类定义如下

```java
public record Customer(
        String name,
        String email,
        String phoneNumber,
        LocalDate dob
) {}
---usage
Customer customer = new Customer(
        "Alice",
        "alice@gmail.com",
        "+089998879",
        LocalDate.of(2000, 1, 1)
);
```

我们想验证此人的信息是否合法。在命令式编程中，我们可以这样验证

```java
public class CustomerValidatorService {
    private boolean isEmailValid(String email) {
        return email.contains("@");
    }

    private boolean isPhoneNumberValid(String phoneNumber) {
        return phoneNumber.startsWith("+0");
    }

    private boolean isAdult(LocalDate dob) {
        return Period.between(dob, LocalDate.now()).getYears() > 16;
    }

    public boolean isValid(Customer customer) {
        return isEmailValid(customer.email())
                && isPhoneNumberValid(customer.phoneNumber())
                && isAdult(customer.dob());
    }
}
---usage
System.out.println(new CustomerValidatorService().isValid(customer));
```

可以看到，当我们需要添加验证项或者需要根据不同的用户启用不同的验证策略时，上面的方法需要改动太多的代码。这种方法的另一个缺点是：当验证失败时，我们无法知道对象的哪个属性没有通过验证，该方法只是返回一个失败的结果，这个结果并不包含细节。

我这里介绍的解决方案叫做 Combinator Pattern

为了能够返回方法的详细信息，我们首先定义一个枚举类

```java
enum ValidationResult {
    SUCCESS,
    PHONE_NUMBER_NOT_VALID,
    EMAIL_NOT_VALID,
    IS_NOT_AN_ADULT
}
```

我们使用 `CustomerRegistrationValidator interface` 扩展 `Functional interface`

```java
public interface CustomerRegistrationValidator
        extends Function<Customer, ValidationResult> {

    static CustomerRegistrationValidator isEmailValid() {
        return customer -> customer.email().contains("@") ?
                SUCCESS : EMAIL_NOT_VALID;
    }
    static CustomerRegistrationValidator isPhoneNumberValid() {
        return customer -> customer.phoneNumber().startsWith("+0") ?
                SUCCESS : PHONE_NUMBER_NOT_VALID;
    }
    static CustomerRegistrationValidator isAnAdult() {
        return customer ->
                Period.between(customer.dob(), LocalDate.now()).getYears() > 16 ?
                        SUCCESS : IS_NOT_AN_ADULT;
    }

    /**
     * test lazy load
     */
    static CustomerRegistrationValidator printSomething() {
        return customer ->{
            System.out.println("print something");
            return SUCCESS;
        };
    }

    default CustomerRegistrationValidator and (CustomerRegistrationValidator other) {
        return customer -> {
            ValidationResult result = this.apply(customer);
            return result.equals(SUCCESS) ? other.apply(customer) : result;
        };

    }

    enum ValidationResult {
        SUCCESS,
        PHONE_NUMBER_NOT_VALID,
        EMAIL_NOT_VALID,
        IS_NOT_AN_ADULT
    }
}
```

用法很简单

```java
var result = isEmailValid()
        .and(isPhoneNumberValid())
        .and(isAnAdult())
        .apply(customer);
System.out.println(result);
if (result != ValidationResult.SUCCESS) {
    throw new IllegalStateException(result.name());
}
```

使用这种方法，我们可以灵活地组合多个验证。当验证失败时，该方法会返回失败的原因

此外，它是延迟加载，也就是直到调用 `apply()` 时才会真正执行

```java
var result2 = isEmailValid()
        .and(isPhoneNumberValid())
        .and(isAnAdult())
        .and(printSomething());
System.out.println("not load before apply()");
result2.apply(customer);
```

---

完整代码

调用方法

```java
public class Main {
    public static void main(String[] args) {
        Customer customer = new Customer(
                "Alice",
                "alice@gmail.com",
                "+089998879",
                LocalDate.of(2000, 1, 1)
        );

        System.out.println(new CustomerValidatorService().isValid(customer));

        // If valid we can store customer in db
        // ...

        // Using combinator pattern
        var result = isEmailValid()
                .and(isPhoneNumberValid())
                .and(isAnAdult())
                .apply(customer);
        System.out.println(result);
        if (result != ValidationResult.SUCCESS) {
            throw new IllegalStateException(result.name());
        }

        // Lazy lode
        var result2 = isEmailValid()
                .and(isPhoneNumberValid())
                .and(isAnAdult())
                .and(printSomething());
        System.out.println("not load before apply()");
        result2.apply(customer);

    }
}
```

命令式编程

```java
public class CustomerValidatorService {
    private boolean isEmailValid(String email) {
        return email.contains("@");
    }

    private boolean isPhoneNumberValid(String phoneNumber) {
        return phoneNumber.startsWith("+0");
    }

    private boolean isAdult(LocalDate dob) {
        return Period.between(dob, LocalDate.now()).getYears() > 16;
    }

    public boolean isValid(Customer customer) {
        return isEmailValid(customer.email())
                && isPhoneNumberValid(customer.phoneNumber())
                && isAdult(customer.dob());
    }

}
```

函数式编程

```java
public interface CustomerRegistrationValidator
        extends Function<Customer, ValidationResult> {

    static CustomerRegistrationValidator isEmailValid() {
        return customer -> customer.email().contains("@") ?
                SUCCESS : EMAIL_NOT_VALID;
    }
    static CustomerRegistrationValidator isPhoneNumberValid() {
        return customer -> customer.phoneNumber().startsWith("+0") ?
                SUCCESS : PHONE_NUMBER_NOT_VALID;
    }
    static CustomerRegistrationValidator isAnAdult() {
        return customer ->
                Period.between(customer.dob(), LocalDate.now()).getYears() > 16 ?
                        SUCCESS : IS_NOT_AN_ADULT;
    }

    /**
     * test lazy load
     */
    static CustomerRegistrationValidator printSomething() {
        return customer ->{
            System.out.println("print something");
            return SUCCESS;
        };
    }

    default CustomerRegistrationValidator and (CustomerRegistrationValidator other) {
        return customer -> {
            ValidationResult result = this.apply(customer);
            return result.equals(SUCCESS) ? other.apply(customer) : result;
        };

    }

    enum ValidationResult {
        SUCCESS,
        PHONE_NUMBER_NOT_VALID,
        EMAIL_NOT_VALID,
        IS_NOT_AN_ADULT
    }
}
```

entity

```java
public record Customer(
        String name,
        String email,
        String phoneNumber,
        LocalDate dob
) {}
```

## Callbacks

由于 Java 的函数式接口，我们现在可以像 JavaScript 一样使用 callback

在 JavaScript 中，我们像这样定义带有回调的函数

```jsx
function hello(firstName, lastName,callback) {
    console.log(firstName);
    if (lastName) {
        console.log(lastName);
    } else {
        callback();
    }
}
```

我们可以在 Chrome 控制台中调用它

```jsx
hello("john", null, function(){console.log("no lastname provided")})
```

现在我们可以在 Java 中做同样的事情

```java
static void hello(String firstName, String lastName, Consumer<String> callback) {
    System.out.println(firstName);
    if (lastName != null) {
        System.out.println(lastName);
    } else {
        callback.accept(firstName);
    }
}

static void hello2(String firstName, String lastName, Runnable callback) {
    System.out.println(firstName);
    if (lastName != null) {
        System.out.println(lastName);
    } else {
        callback.run();
    }
}
```

usage

```java
hello("John", "Montana", null);
hello("John", null, value -> {
    System.out.println("no lastName provided for " + value);
});

hello2("John", null,
        () -> System.out.println("no lastName provided"));
```

---

完整代码

```java
public class Callbacks {
    public static void main(String[] args) {
        hello("John", "Montana", null);
        hello("John", null, value -> {
            System.out.println("no lastName provided for " + value);
        });
        hello2("John", null,
                () -> System.out.println("no lastName provided"));


    }

    static void hello(String firstName, String lastName, Consumer<String> callback) {
        System.out.println(firstName);
        if (lastName != null) {
            System.out.println(lastName);
        } else {
            callback.accept(firstName);
        }
    }

    static void hello2(String firstName, String lastName, Runnable callback) {
        System.out.println(firstName);
        if (lastName != null) {
            System.out.println(lastName);
        } else {
            callback.run();
        }
    }

    /*
    Callback function in js:
    function hello(firstName, lastName,callback) {
        console.log(firstName);
        if (lastName) {
            console.log(lastName);
        } else {
            callback();
        }
    }

    Invoke it:
    hello("john", null, function(){console.log("no lastname provided")})
     */
}
```

## 函数式编程的特性

- 无状态
- 纯函数
- 无副作用
- 高阶特性
    - 函数将一个或多个函数作为参数。
    - 函数返回另一个函数作为结果。

---
layout: post
title:  "Parameter conversion with TableTest"
date:   2025-09-06 18:52:41 +0200
tags: table-driven-testing tabletest tdd junit java kotlin parameterized-tests testing test-design
---
In my previous [post][table-driven-testing], I showed how TableTest makes table-driven tests concise and readable in JUnit. That example used three columns: a scenario description, example years, and whether the year is a leap year.

```java
@TableTest("""
    Scenario                        | Year | Is Leap Year?
    Not divisible by 4              | 2001 | false
    Divisible by 4                  | 2004 | true
    Divisible by 100 but not by 400 | 2100 | false
    Divisible by 400                | 2000 | true
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

How does TableTest map the text table to method parameters? How do values in the 'Year' column become an `int year`, and values in 'Is Leap Year?' become a `boolean isLeapYear`? And why doesn’t the 'Scenario' column need a matching parameter in the test method?


### Scenario Names

TableTest allows one extra column not declared as a parameter. By default, the left-most column is treated as a scenario description. It does not have to be named "Scenario". It can be named anything you like.

The description is also used as the test display name when the test runs (example from IntelliJ IDEA):

![Scenario as test display name in IntelliJ IDEA][table-test-display-name-intellij]

If you prefer to bind the scenario to a parameter or use another column as the display name, see the [user guide][table-test-user-guide-scenario].


### Automatic Parameter Conversion

TableTest supports JUnit's built-in implicit conversions for parameterized tests with String-based value sources. Numeric strings convert to numeric types (e.g., `int`, `double`), and `"true"`/`"false"` convert to `boolean`. Other standard types are supported too (dates, times, URLs, file paths). See the [JUnit user guide][junit-implicit-conversion] for the full list.

In addition, TableTest supports collections:
- Lists: `[1, 2, 3]`
- Sets: `{1, 2, 3}`
- Maps: `[key1: value1, key2: value2]`

Elements inside collections are also converted, including nested collections.

```java
@TableTest("""
    Student Grades                               | Highest Grade?
    [Alice: [95, 87, 92], Bob: [78, 85, 90]]     | 95
    [Charlie: [98, 89, 91], David: [45, 60, 70]] | 98
    """)
void testStudentGrades(Map<String, List<Integer>> studentGrades, int expectedHighestGrade) {
    // test implementation
}
```

### Custom Parameter Conversion

If you want conversions not covered by implicit rules, add a factory method. For example, to read `Yes/No` instead of `true/false`:

```java
@TableTest("""
    Scenario                        | Year | Is Leap Year?
    Not divisible by 4              | 2001 | No
    Divisible by 4                  | 2004 | Yes
    Divisible by 100 but not by 400 | 2100 | No
    Divisible by 400                | 2000 | Yes
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}

public static Boolean parseBoolean(String input) {
    return input.equalsIgnoreCase("yes");    
}
```

Factory methods must be public, static, and accept a single argument. To reuse them across test classes, put them in a shared superclass or reference external classes with `@FactorySources`. See the [user guide][table-test-user-guide-factorysource] for details.

The factory parameter type does not have to be `String`; it can be any type supported by automatic or custom conversion:

```java
@TableTest("""
        Grades                                       | Highest Grade?
        [Alice: [95, 87, 92], Bob: [78, 85, 90]]     | 95
        [Charlie: [98, 89, 91], David: [45, 60, 70]] | 98
        """)
void testStudentGrades(Map<String, Grades> studentGrades, int expectedHighestGrade) {
    // test implementation
}

public static Grades parseGrades(List<Integer> input) {
    return new Grades(input);
}
```

### Value Sets

One more feature from the previous post is value sets: providing multiple values for a single parameter in one row.

```java
@TableTest("""
    Scenario                        | Year               | Is Leap Year?
    Not divisible by 4              | {1, 2001, 30001}   | No
    Divisible by 4                  | {4, 2004, 30008}   | Yes
    Divisible by 100 but not by 400 | {100, 2100, 30300} | No
    Divisible by 400                | {400, 2000, 30000} | Yes
    Year 0 (ISO proleptic calendar) | 0                  | Yes
    Negative input                  | -1                 | No
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

The benefit is twofold. It clearly specifies that multiple values of an input parameter are applicable for the same scenario. This can help clarify the behaviour of the system under test. 

It also makes it easier to extend test coverage while still keeping the table concise. 

TableTest runs the test once per value in the set. If multiple columns use sets, all combinations are tested (cartesian product). Test display names include the values under test:

![Test display name in IntelliJ IDEA when using value sets][value-set-display-name-intellij]


### Summary

- TableTest supports a "Scenario" column to describe each row.
- Automatic conversion of table values to test parameter types reduces boilerplate code.
- Custom conversion methods can be added and reused across test classes.
- Value sets make it clear that multiple values are applicable for a single scenario, expanding coverage without bloating the table.

<br>
<br>

_Previous post: [Readable Table-Driven Tests in JUnit with TableTest][table-driven-testing]_  
_Next post: [Table-Driven Testing with TableTest: A Realistic Example][tabletest-realistic-example]_

[table-driven-testing]: {% post_url 2025-08-31-table-driven-testing %}
[tabletest-realistic-example]: {% post_url 2025-10-28-tabletest-realistic-example %}
[table-test-display-name-intellij]: /assets/2025-09-06-parameter-conversion-with-tabletest/scenario-as-test-display-name-in-intellij.png
[value-set-display-name-intellij]: /assets/2025-09-06-parameter-conversion-with-tabletest/value-set-display-name-in-intellij.png
[table-test-user-guide-scenario]: https://github.com/nchaugen/tabletest/blob/main/USERGUIDE.md#scenario-names
[junit-implicit-conversion]: https://docs.junit.org/current/user-guide/#writing-tests-parameterized-tests-argument-conversion-implicit
[table-test-user-guide-factorysource]: https://github.com/nchaugen/tabletest/blob/main/USERGUIDE.md#factory-sources

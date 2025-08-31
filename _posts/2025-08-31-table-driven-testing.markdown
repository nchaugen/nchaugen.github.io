---
layout: post
title:  "Table-Driven Testing with TableTest"
date:   2025-08-31 19:42:41 +0200
categories: table-driven-testing tabletest tdd
---
A lot of software is about evaluating inputs against rules: validating fields, classifying data, and triggering actions when thresholds are crossed. Automated testing of this functionality often leads to repetitive tests with near-identical setup, execution, and assertions.

Table-driven tests solve this by separating test logic from test data. One parameterized test method runs many examples — each example is a row in a table. This allows adding or refining behaviour by editing rows, not writing new methods.

With this approach, should changes in the software design require the test implementation to change, there will be fewer test methods to update.

### Example

For a simple example, consider a function for deciding if a particular year is a leap year or not. A parameterized test method for this using Java and JUnit could look like this:

```java
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

A parameterized test needs to be driven with input data for each variation to be tested. I prefer to organize the test input in a readable table format.

The leap year test could be driven by a table like the following:

| Year | Is Leap Year? |
|------|---------------|
| 2001 | false         |
| 2004 | true          |
| 2100 | false         |
| 2000 | true          |

Not being satisfied with the built-in mechanisms for expressing tables in JUnit, I have developed an extension called [TableTest][tabletest] to make this easier. Using JUnit with TableTest, a table-driven test can be written as follows:

```java
@TableTest("""
    Year | Is Leap Year?
    2001 | false
    2004 | true
    2100 | false
    2000 | true
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

To help describe the test cases, add a short description of what is being tested in each row. This column does not have to be declared as a parameter in the test method:

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

Starting with a few rows, it is straightforward to add more tests as new situations are discovered or to clarify corner cases:

```java
@TableTest("""
    Scenario                        | Year  | Is Leap Year?
    Not divisible by 4              | 2001  | false
    Divisible by 4                  | 2004  | true
    Divisible by 100 but not by 400 | 2100  | false
    Divisible by 400                | 2000  | true
    Year 0                          | 0     | true
    Far future leap                 | 2800  | true
    Very far future leap            | 30000 | true
    Very far future not leap        | 30100 | false
    Negative input                  | -1    | false
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

Long tables can be hard to read. With TableTest is it possible to group examples with the same expected result into a single row: 

```java
@TableTest("""
    Scenario                        | Year               | Is Leap Year?
    Not divisible by 4              | {1, 2001, 30001}   | false
    Divisible by 4                  | {4, 2004, 30008}   | true
    Divisible by 100 but not by 400 | {100, 2100, 30300} | false
    Divisible by 400                | {400, 2000, 30000} | true
    Year 0                          | 0                  | true
    Negative input                  | -1                 | false
    """)
public void testLeapYear(int year, boolean isLeapYear) {
    assertEquals(isLeapYear, Year.isLeap(year));
}
```

The rules for leap year calculation are limited in number and well-known. This is not always the case with rules in business software.

I have found that tables like these can become useful descriptions of how the business and, consequently, the software work. In many situations I have brought the tables into discussion with business experts to clarify how values can vary and how the software should respond in each situation.

### Summary

Table-driven tests turn rule-heavy behaviour into a readable contract:
- Start small to introduce the rule
- Expand with more rows as you learn
- Collapse using sets to keep the table short without losing coverage

This style makes tests easier to maintain, doubles as documentation, and invites conversations with non-developers about how the system should behave — exactly where tests add the most value.

[TableTest][tabletest] is a small library that makes this approach easy to use with JUnit, both for Java and Kotlin code.

[tabletest]: https://github.com/nchaugen/tabletest

---
layout: post
title:  "Table-Driven Testing with TableTest: A Realistic Example"
date:   2025-10-28 21:30:00 +0100
tags: table-driven-testing tabletest tdd junit java kotlin parameterized-tests testing test-design
---
[TableTest][tabletest] is a JUnit extension for driving parameterized tests from readable tables. Previous posts have [introduced TableTest][table-driven-testing] and explained how table values are [converted to test parameter types][parameter-conversion]. In this post we'll show how to design TableTests for a realistic example. 

### Levelled discount

A public transport ticketing system needs functionality for a levelled discount scheme for single tickets. The more you travel, the cheaper the tickets will become. Discounts increase stepwise based on the number of ticket purchases in the last 30 days, capped at 40%:

| Ticket number | Discount |
|:--------------|:---------|
| 1-4           | 0%       |
| 5-9           | 5%       |
| 10-14         | 10%      |
| 15-19         | 15%      |
| 20-24         | 20%      |
| 25-29         | 25%      |
| 30-34         | 30%      |
| 35-39         | 35%      |
| 40+           | 40%      |

This is the discount scheme used by the Norwegian public transport company [Ruter][ruter]. Let's see how we can implement these rules using TableTest. We will not cover the implementation of the discount calculation but focus on the test design.


### Calculating the discount

We begin with a function for calculating the discount percentage given the number of purchases in the last 30 days. Starting with the test, we set up a base case:

```java
@TableTest("""
    Purchases last 30 days | Discount?
    0                      | 0%
    """)
void next_purchase_discount(
    int countPurchasesLast30Days,
    PercentageDiscount expectedDiscount
){
    // TODO
}
```

Before implementing the test body, we can run the test to check that the table parses correctly and that TableTest can convert table values to test parameters. The test fails with an exception:

```text
io.github.nchaugen.tabletest.junit.TableTestException: 
Built-in conversion of value "0%" to type PercentageDiscount failed. 
Are you missing a factory method for this conversion? 
Locations searched for public static factory methods: LevelledDiscountTest
```

TableTest does not know how to convert the string "0%" to a PercentageDiscount object. We need to provide a factory method to help with the conversion:

```java
public static PercentageDiscount parseDiscount(String input) {
    String digits = input.substring(0, input.length() - 1);
    return new PercentageDiscount(Integer.parseInt(digits));
}
```

Parsing is simple: strip the percentage sign and parse the digits. Next we add the test body and create the function calculating the discount:

```java
@TableTest("""
    Purchases last 30 days | Discount?
    0                      | 0%
    """)
void next_purchase_discount(
    int countPurchasesLast30Days, 
    PercentageDiscount expectedDiscount
) {
    assertEquals(
        expectedDiscount.percentage(),
        LevelledDiscount.calculateDiscountPercentage(countPurchasesLast30Days)
    );
}
```

When the test passes for the base case, we continue to add more scenarios to the table and extend the discount calculation to handle them. 

We use the TableTest set syntax to group inputs with the same expected result. TableTest will verify that the expectation holds true for all the values in the set. 

The final table ends up like this:

```java
@TableTest("""
    Purchases last 30 days | Discount?
    {0, 1, 2, 3}           | 0%
    {4, 5, 6, 7, 8}        | 5%
    {9, 10, 11, 12, 13}    | 10%
    {14, 15, 16, 17, 18}   | 15%
    {19, 20, 21, 22, 23}   | 20%
    {24, 25, 26, 27, 28}   | 25%
    {29, 30, 31, 32, 33}   | 30%
    {34, 35, 36, 37, 38}   | 35%
    {39, 40, 100, 1000}    | 40%
    """)
void next_purchase_discount(
    int purchasesLast30Days, 
    PercentageDiscount expectedDiscount
) {
    assertEquals(
        expectedDiscount.percentage(), 
        LevelledDiscount.calculateDiscountPercentage(purchasesLast30Days)
    );
}
```

Notice how the TableTest table resembles the table in the discount scheme description. It is good practice to strive for tables to communicate well. This makes it easier to validate that the implementation is behaving correctly, and it makes the test better documentation.


### Displaying the test results

The default test result display name for parameterized tests is a comma-separated list of parameter values: 

![Test run with default display names][test-run-default-display]

To make it more descriptive, we can add a @DisplayName and annotate a parameter with @Scenario to use its toString as the display name:

```java
@DisplayName("Next purchase levelled discount")
@TableTest("""
    Purchases last 30 days | Discount?
    {0, 1, 2, 3}           | 0%
    {4, 5, 6, 7, 8}        | 5%
    // ...
    """)
void next_purchase_discount(
    int purchasesLast30Days, 
    @Scenario PercentageDiscount expectedDiscount
) {
    // ...
}
```

This makes the test results more readable. When using value sets, TableTest will add the value being tested to the display name: 

![Test run with @Scenario and @DisplayName][test-run-improved-display]

Alternatively, we can add a leftmost scenario column, as we will see in the next section.


### Counting purchases

Next, we want to calculate the number of purchases in the last 30 days given a purchase history. Again we start with a simple scenario:

```java
@TableTest("""
    Scenario     | Purchases | Count?
    No purchases | []        | 0
    """)
void count_purchases_last_30_days(
    List<Purchase> purchases,
    int expectedCount
) {
    // TODO
}
```

Success! Maybe a bit unexpectedly as we didn't think TableTest knew how to create Purchase objects. It doesn't, but with an empty list, the need for conversion never arises.

A purchase is represented by the following type:

```java
public record Purchase(
    LocalDateTime timeOfPurchase, 
    TicketType ticketType, 
    Discount discount, 
    Price purchasePrice, 
    Price ticketFaceValue
) {}
```

In the table we decide to represent a purchase by its timestamp as that is the piece of data relevant to the counting functionality: 

```java
@TableTest("""
    Scenario     | Purchases               | Count?
    No purchases | []                      | 0
    One purchase | ["2025-09-01T00:00:00"] | 1
    """)
void count_purchases_last_30_days(
    List<Purchase> purchases, 
    int expectedCount
) {
    // TODO
}
```

We add the next scenario, and sure enough, now it fails with a message saying it doesn't know how to convert the string timestamp to a Purchase object. A new factory method is needed:

```java
public static Purchase createPurchase(
    LocalDateTime purchaseTimestamp
) {
    return new Purchase(
        purchaseTimestamp,
        TicketType.SINGLE,
        Discount.NONE,
        new Price(BigDecimal.ONE, Currency.getInstance("NOK")),
        new Price(BigDecimal.ONE, Currency.getInstance("NOK"))
    );
}
```

As we are using the Java standard notation for the timestamp, our factory method can accept a LocalDateTime object instead of the string. TableTest will use a [built-in converter][junit-implicit-conversion] to transform the string to a LocalDateTime object before invoking the factory method.

The remaining fields in Purchase are not relevant for the counting logic, so we use constant values for these in this test.

Adding the test body, we realize that we need to specify the time we should count purchases relative to. This improves the design of the production code, separating the responsibility of time from the business logic and avoids test failure when the date is no longer in the 30-day window.

We add a new column for the current time and add more scenarios: 

```java
@TableTest("""
    Scenario              | Time now            | Purchases                                      | Count?
    No purchases          | 2025-09-30T23:59:59 | []                                             | 0
    Purchase too old      | 2025-09-30T23:59:59 | ["2025-08-01T00:00:00"]                        | 0
    Purchase just outside | 2025-09-30T23:59:59 | ["2025-08-31T23:59:59"]                        | 0
    Purchase just inside  | 2025-09-30T23:59:59 | ["2025-09-01T00:00:00"]                        | 1
    All purchases inside  | 2025-09-30T23:59:59 | ["2025-09-02T00:00:00", "2025-09-03T00:00:00"] | 2
    One inside, one not   | 2025-09-30T23:59:59 | ["2025-08-02T00:00:00", "2025-09-03T00:00:00"] | 1
    """)
void count_purchases_last_30_days(
    LocalDateTime timeNow, 
    List<Purchase> purchases, 
    int expectedCount
) {
    assertEquals(
        expectedCount,
        LevelledDiscount.countPurchasesLast30Days(timeNow, purchases)
    );
}
```

We need to wrap timestamps in quotes inside lists to avoid parse errors. The LocalDateTime format uses colons, which collide with TableTest’s map syntax `[key:value]`. 


### Making the table more readable

As the table grows, it becomes clear that absolute timestamps are hard to compare visually. A notation for relative timestamps solves this:

- T-60d = 60 days ago
- T-30d1s = 30 days and 1 second ago
- T-30d1h2m3s = 30 days and 1 hour, 2 minutes and 3 seconds ago

We rewrite the table to use relative timestamps. This also makes the current time column redundant:

```java
@TableTest("""
    Scenario              | Time of past purchases | Count?
    No purchases          | []                     | 0
    Purchase too old      | [T-60d]                | 0
    Purchase just inside  | [T-29d23h59m59s]       | 1
    Purchase just outside | [T-30d]                | 0
    All purchases inside  | [T-28d, T-27d]         | 2
    One inside, one not   | [T-45d, T-15d]         | 1
    """)
void count_purchases_last_30_days(
    List<Purchase> purchases,
    int expectedCount
) {
    assertEquals(
        expectedCount,
        LevelledDiscount.countPurchasesLast30Days(TIME_NOW, purchases)
    );
}
```

The relative timestamp notation makes the counting logic more obvious. We add custom parsing for the relative timestamp syntax and make sure to fix `TIME_NOW` to a constant to avoid test flakiness:

```java
private static final LocalDateTime TIME_NOW = LocalDateTime.now();

public static LocalDateTime parseRelativeTimestamp(String input) {
    Matcher matcher = T_MINUS_SYNTAX.matcher(input);
    if (!matcher.matches()) {
        throw new IllegalArgumentException("Invalid syntax for relative timestamp: " + input);
    }
    return TIME_NOW
        .minusDays(intValue(matcher, TimeUnit.DAYS))
        .minusHours(intValue(matcher, TimeUnit.HOURS))
        .minusMinutes(intValue(matcher, TimeUnit.MINUTES))
        .minusSeconds(intValue(matcher, TimeUnit.SECONDS));
}

private static final Pattern T_MINUS_SYNTAX = Pattern.compile("T-(\\d*)d?(\\d*)h?(\\d*)m?(\\d*)s?");

private enum TimeUnit {
    DAYS, HOURS, MINUTES, SECONDS;

    int group() {
        return ordinal() + 1;
    }
}

private static int intValue(Matcher matcher, TimeUnit unit) {
    String value = matcher.group(unit.group());
    return value.isEmpty() ? 0 : Integer.parseInt(value);
}
```

The relative timestamp parser may warrant its own class (and tests!) for reuse in other TableTests. We will not cover that here, but TableTest supports importing factory method parsers from other classes.

Note that we provide an alternative parser for LocalDateTime, not a new factory method for Purchase. As before, TableTest will look for another converter to transform the string to LocalDateTime before calling the Purchase factory method. Our new custom parser will take precedence over the built-in version that was used before. 


### Summary

We now have the building blocks for the levelled discount logic covered by two TableTests. Hopefully, this example illustrates how TableTest can be used to craft tests that are concise, maintainable, and communicate well. 

We put some effort into making the tables convey the business rules as clearly as possible. We did this by adding scenario descriptions, considering the value notation, and providing custom parsers. I believe this is a worthwhile investment. In many situations, I have brought tables like these into discussions with business experts to clarify the business rules. I also find that I keep referring back to the tables when I need to clarify how the software works. 

<br>
<br>
_Previous post: [Parameter Conversion with TableTest][parameter-conversion]_

[tabletest]: https://github.com/nchaugen/tabletest
[table-driven-testing]: {% post_url 2025-08-31-table-driven-testing %}
[parameter-conversion]: {% post_url 2025-09-06-parameter-conversion-with-tabletest %}
[ruter]: https://ruter.no/en/about-our-tickets/reis-discount-on-single-tickets
[junit-implicit-conversion]: https://docs.junit.org/current/user-guide/#writing-tests-parameterized-tests-argument-conversion-implicit
[test-run-default-display]: /assets/2025-10-28-tabletest-realistic-example/test-run-default-display.png
[test-run-improved-display]: /assets/2025-10-28-tabletest-realistic-example/test-run-with-displayname-and-scenario.png

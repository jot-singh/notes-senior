# Java Streams & Functional Programming

## Concept Overview

Java 8 introduced functional programming features including lambda expressions, method references, functional interfaces, and the Stream API. These enable declarative, composable data processing pipelines with potential for parallel execution.

## Core Mechanics & Implementation

### Lambda Expressions

```java
// Lambda syntax
(parameters) -> expression
(parameters) -> { statements; }

// Examples
Runnable r = () -> System.out.println("Hello");
Comparator<String> c = (a, b) -> a.length() - b.length();
Function<String, Integer> f = s -> s.length();

// Method references
list.forEach(System.out::println);           // Instance method on parameter
list.stream().map(String::toUpperCase);      // Instance method on parameter
list.stream().map(String::length);           // Instance method on parameter
list.stream().filter(Objects::nonNull);      // Static method
list.stream().map(User::new);                // Constructor reference
```

### Functional Interfaces

```java
// Core functional interfaces
Function<T, R>      // T -> R
BiFunction<T, U, R> // (T, U) -> R  
Predicate<T>        // T -> boolean
Consumer<T>         // T -> void
Supplier<T>         // () -> T
UnaryOperator<T>    // T -> T
BinaryOperator<T>   // (T, T) -> T

// Usage examples
Function<String, Integer> parse = Integer::parseInt;
Predicate<String> notEmpty = s -> !s.isEmpty();
Consumer<String> print = System.out::println;
Supplier<UUID> idGenerator = UUID::randomUUID;

// Composition
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> combined = trim.andThen(upper);

Predicate<Integer> positive = n -> n > 0;
Predicate<Integer> even = n -> n % 2 == 0;
Predicate<Integer> positiveEven = positive.and(even);
```

### Stream Operations

```java
// Creating streams
Stream<String> stream = list.stream();
Stream<String> parallel = list.parallelStream();
Stream<Integer> of = Stream.of(1, 2, 3);
Stream<Integer> range = IntStream.range(0, 100).boxed();
Stream<String> lines = Files.lines(path);

// Intermediate operations (lazy)
stream
    .filter(s -> s.length() > 3)        // Keep matching elements
    .map(String::toUpperCase)           // Transform elements
    .flatMap(s -> Arrays.stream(s.split(""))) // Flatten nested streams
    .distinct()                         // Remove duplicates
    .sorted()                           // Natural order
    .sorted(Comparator.reverseOrder())  // Custom order
    .limit(10)                          // First N elements
    .skip(5)                            // Skip first N
    .peek(System.out::println);         // Debug/side effects

// Terminal operations (trigger execution)
long count = stream.count();
Optional<String> first = stream.findFirst();
Optional<String> any = stream.findAny();      // Better for parallel
boolean allMatch = stream.allMatch(predicate);
boolean anyMatch = stream.anyMatch(predicate);
String reduced = stream.reduce("", (a, b) -> a + b);
```

### Collectors

```java
// Collecting results
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
Map<String, User> map = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

// Joining strings
String joined = stream.collect(Collectors.joining(", "));

// Grouping
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Grouping with downstream collector
Map<Department, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// Partitioning
Map<Boolean, List<Employee>> partition = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));

// Statistics
DoubleSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingDouble(Employee::getSalary));
// stats.getAverage(), stats.getMax(), stats.getCount()

// Custom collector
String result = stream.collect(
    StringBuilder::new,
    StringBuilder::append,
    StringBuilder::append
).toString();
```

### Optional

```java
// Creating Optional
Optional<String> empty = Optional.empty();
Optional<String> of = Optional.of("value");        // Throws if null
Optional<String> nullable = Optional.ofNullable(value);

// Using Optional
String result = optional
    .filter(s -> s.length() > 0)
    .map(String::toUpperCase)
    .orElse("default");

String required = optional.orElseThrow(() -> 
    new IllegalStateException("Value required"));

optional.ifPresent(System.out::println);
optional.ifPresentOrElse(
    System.out::println,
    () -> System.out.println("Empty")
);

// Avoid these anti-patterns
if (optional.isPresent()) {
    return optional.get();  // Use orElse/map instead
}

Optional<String> opt = Optional.of(nullable);  // Use ofNullable
```

### Parallel Streams

```java
// Parallel processing
long count = list.parallelStream()
    .filter(expensive::test)
    .count();

// When to use parallel:
// - Large data sets (>10,000 elements)
// - CPU-intensive operations
// - Independent operations (no shared state)
// - Splittable sources (ArrayList good, LinkedList bad)

// When NOT to use parallel:
// - Small collections
// - I/O-bound operations
// - Operations with ordering requirements
// - Operations modifying shared state

// Custom thread pool for parallel streams
ForkJoinPool customPool = new ForkJoinPool(4);
long result = customPool.submit(() ->
    list.parallelStream()
        .filter(predicate)
        .count()
).get();
```

### Common Patterns

```java
// Null-safe stream from collection
public static <T> Stream<T> safeStream(Collection<T> collection) {
    return collection == null ? Stream.empty() : collection.stream();
}

// Flatten nested optionals
Optional<String> nested = outer.flatMap(o -> o.getInner());

// Group and aggregate
Map<String, DoubleSummaryStatistics> salesByRegion = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getRegion,
        Collectors.summarizingDouble(Order::getAmount)
    ));

// Find max by property
Optional<Employee> highest = employees.stream()
    .max(Comparator.comparing(Employee::getSalary));

// Convert map entries
Map<String, String> transformed = original.entrySet().stream()
    .collect(Collectors.toMap(
        e -> e.getKey().toLowerCase(),
        e -> e.getValue().toUpperCase()
    ));
```

## Interview Questions

**Q1: What is the difference between intermediate and terminal operations?**

Intermediate operations are lazy, return streams, and don't process until terminal operation invoked (filter, map, sorted). Terminal operations are eager, trigger pipeline execution, return non-stream result (collect, forEach, count, reduce).

**Q2: When should you use parallel streams?**

Use when: large collections (>10,000), CPU-intensive operations, independent operations, ArrayList/arrays (good splitting). Avoid when: small data, I/O operations, ordering matters, shared mutable state, LinkedList source.

**Q3: Explain the difference between map() and flatMap().**

map() transforms each element 1:1 (Stream<T> → Stream<R>). flatMap() transforms and flattens, each element can produce 0+ elements (Stream<T> → Stream<R> where T produces Stream<R>). Use flatMap for nested collections or Optionals.

**Q4: What is a functional interface?**

Interface with exactly one abstract method (can have default/static methods). Can be implemented with lambdas. Annotate with @FunctionalInterface for compiler check. Examples: Runnable, Callable, Comparator, Function, Predicate.

**Q5: How does Optional help with null handling?**

Explicitly represents presence/absence of value. Forces handling of both cases. Provides map/flatMap/filter for transformations. orElse/orElseGet for defaults. orElseThrow for required values. Avoid isPresent()/get() pattern.

**Q6: Explain reduce() operation.**

Combines stream elements into single result. Takes identity value and BinaryOperator. Example: `stream.reduce(0, Integer::sum)`. Three-argument form for parallel: identity, accumulator, combiner. Associative operation required for parallel.

**Q7: What are the advantages of streams over loops?**

Declarative style (what not how), composable operations, lazy evaluation, easy parallelization, cleaner code for transformations. Loops better for: early termination with complex conditions, modifying external state, simple iterations.

**Q8: How do collectors work?**

Collector defines: supplier (create result container), accumulator (add element), combiner (merge containers for parallel), finisher (transform to final result), characteristics (CONCURRENT, UNORDERED, IDENTITY_FINISH). Use Collectors factory methods or implement custom.

# The Template Method Pattern in Java

## Introduction

The **Template Method** is a behavioral design pattern first described in the classic *Gang of Four* (GoF) book *Design Patterns* (1994).  
It defines the **skeleton of an algorithm** in a base class while allowing subclasses to redefine certain steps of that algorithm without changing its overall structure.  

This approach is useful when several classes share common logic, but some parts vary and should be implemented differently depending on the subclass.

---

## Core Idea

- The base (abstract) class defines the general **algorithm structure**.  
- Certain steps are implemented in the base class, while others are left as **abstract methods**.  
- Subclasses override these abstract methods to provide specific behavior.  
- The algorithm’s high-level logic is preserved, ensuring consistency.

ASCII diagram of the structure:

```
+-----------------------+
|   AbstractClass       |
|-----------------------|
| + templateMethod()    |  <-- defines the skeleton
| + step1()             |  <-- concrete
| + step2()             |  <-- abstract (to override)
| + step3()             |  <-- abstract (to override)
+-----------------------+
          ^
          |
   -----------------
   |               |
+--------+   +------------+
|SubclassA|   | SubclassB |
| step2() |   | step2()   |
| step3() |   | step3()   |
+---------+   +-----------+
```

---

## Example 1: Caffeine Beverages (Tea and Coffee)

A classic demonstration is making beverages. The steps are similar, but the details differ depending on whether you brew tea or coffee.

```java
abstract class CaffeineBeverage {
    // Template method - defines the skeleton
    public final void prepareRecipe() {
        boilWater();
        brew();
        pourInCup();
        addCondiments();
    }

    void boilWater() {
        System.out.println("Boiling water");
    }

    void pourInCup() {
        System.out.println("Pouring into cup");
    }

    // Abstract steps to be filled in by subclasses
    abstract void brew();
    abstract void addCondiments();
}

class Tea extends CaffeineBeverage {
    @Override
    void brew() {
        System.out.println("Steeping the tea");
    }

    @Override
    void addCondiments() {
        System.out.println("Adding lemon");
    }
}

class Coffee extends CaffeineBeverage {
    @Override
    void brew() {
        System.out.println("Dripping coffee through filter");
    }

    @Override
    void addCondiments() {
        System.out.println("Adding sugar and milk");
    }
}

public class BeverageDemo {
    public static void main(String[] args) {
        CaffeineBeverage tea = new Tea();
        tea.prepareRecipe();

        System.out.println("---");

        CaffeineBeverage coffee = new Coffee();
        coffee.prepareRecipe();
    }
}
```

**Output:**
```
Boiling water
Steeping the tea
Pouring into cup
Adding lemon
---
Boiling water
Dripping coffee through filter
Pouring into cup
Adding sugar and milk
```

---

## Example 2: Data Processor with Collections

Another example involves processing a list of strings. The base class defines the overall process:  
- load the data,  
- filter it,  
- transform it,  
- print the results.  

Each subclass customizes the filtering and transformation logic.

```java
import java.util.*;

abstract class DataProcessor {
    public final void process(List<String> data) {
        List<String> filtered = filter(data);
        List<String> transformed = transform(filtered);
        print(transformed);
    }

    // Steps with default or abstract behavior
    abstract List<String> filter(List<String> data);
    abstract List<String> transform(List<String> data);

    void print(List<String> data) {
        System.out.println("Result: " + data);
    }
}

class UppercaseProcessor extends DataProcessor {
    @Override
    List<String> filter(List<String> data) {
        // keep only words with length >= 4
        List<String> result = new ArrayList<>();
        for (String s : data) {
            if (s.length() >= 4) result.add(s);
        }
        return result;
    }

    @Override
    List<String> transform(List<String> data) {
        List<String> result = new ArrayList<>();
        for (String s : data) {
            result.add(s.toUpperCase());
        }
        return result;
    }
}

class ReverseProcessor extends DataProcessor {
    @Override
    List<String> filter(List<String> data) {
        // keep only words that start with a vowel
        List<String> result = new ArrayList<>();
        for (String s : data) {
            if ("AEIOUaeiou".indexOf(s.charAt(0)) >= 0) result.add(s);
        }
        return result;
    }

    @Override
    List<String> transform(List<String> data) {
        List<String> result = new ArrayList<>();
        for (String s : data) {
            result.add(new StringBuilder(s).reverse().toString());
        }
        return result;
    }
}

public class DataProcessorDemo {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("apple", "cat", "orange", "dog", "umbrella");

        DataProcessor p1 = new UppercaseProcessor();
        p1.process(words);

        DataProcessor p2 = new ReverseProcessor();
        p2.process(words);
    }
}
```

**Output:**
```
Result: [APPLE, ORANGE, UMBRELLA]
Result: [elppa, egnaro, allerbmu]
```

Here the `process` method (template) is fixed, while each subclass changes the details of filtering and transforming.

---

## When to Use Template Method

- When you have **several variations of an algorithm** that share a common structure.  
- When you want to **avoid code duplication** by centralizing the skeleton logic in one place.  
- When you want to **enforce a consistent workflow** but still allow customization at specific points.  
- Common in frameworks: the framework defines the flow, the user provides details.

---

## Summary

The **Template Method** pattern provides a powerful way to separate **what is fixed** in an algorithm from **what can vary**.  
- The base class ensures consistency.  
- The subclasses provide flexibility.  
- Examples: brewing beverages, processing collections, generating random pets (Bruce Eckel’s example).  

This pattern is simple yet effective, and still relevant in modern Java programming.

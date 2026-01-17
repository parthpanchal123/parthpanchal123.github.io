---
date: "2026-01-17T14:36:57+05:30"
draft: false
title: "Write your own Spring Framework ♨️"
summary: "Unveil the magic behind IOC and Dependency Injection."
cover:
  image: "/images/spring-blog-cover.png"
tags:
  [
    "java",
    "backend",
    "spring-framework",
    "inversion-of-control",
    "dependency-injection",
  ]
comments: true 
---

Ever felt like the Spring Framework is just pure magic? You put an `@Autowired` annotation, and _tadaa_ the object appears 🤌.

But today, we look behind the _parda_. We are going to build **Mini-Spring**!

_(Spoiler Alert: It’s not magic, purely **Jugaad** with HashMaps. Don't tell the interviewers 🤫)_

---

## 🦁 Why is Spring the King?

Before we start cooking, let's talk about the ingredient.
**Spring Framework** is the absolute **Baahubali** of the Java world. 💪

If you use a Banking App, book a flight, or watch a movie on streaming, chances are **Spring** is running behind the scenes.

- **Backbone of Finance**: Major banks (J.P. Morgan, Citi, HDFC, etc.) trust it because it is secure like a fortress.
- **Enterprise King**: It’s like the steel frame of a skyscraper. You _could_ build a building with just bricks (plain Java), but if you want to build the **Burj Khalifa**, you need a steel frame.

It handles all the heavy lifting—connecting to databases, security, web servers—so developers like you and me can focus on the business logic.

---

## 🧐 The Secret Sauce: Inversion of Control (IoC)

Imagine you are making **Masala Chai**. ☕

- **Without Spring (Manual Mode)**: You grow the tea leaves, milk the cow, light the stove, and boil it all yourself every single time. _Itni mehnat nahi karni life me!_ 😩
- **With Spring (Smart Mode)**: You just sit at the table and say **"Bhaiya, ek cutting chai!"**, and the **Chaiwala (Container)** hands it to you.

This Chaiwala is what we call an **IoC (Inversion of Control) Container**. You don't worry about _how_ the chai is made; you just get it.

> **Its like**
>
> _Me trying to manage dependencies manually:_
> 😵‍💫 `ServiceA a = new ServiceA(new ServiceB(new ServiceC()));`
>
> _Me using Spring:_
> 😎 `@Autowired private ServiceA a;` -> _Rishta Wahi, Soch Nayi_

---

## 🛠️ The Blueprint: How Mini-Spring Works

Our Mini-Spring is the "Gully Cricket" version of the real Spring. Real Spring is the IPL 🏏; ours is played with a tennis ball. But hey, _concepts remain the same_!

Here are the players in our team:

### 1. The Tag (`@MyService`) 🏷️

First, we need to tell our code, _"Me bhi ek player hoon. You can select me for the team."_
In real Spring, this is `@Component`. In our Mini-Spring, it's `@MyService`.

```java
@MyService
public class SmartBulb { ... }
```

_It says : "Main bhi ek player hoon. You can select me for the team."_

### 2. The Demand (`@MyAutowired`) 🧲

This is the **Arranged Marriage** logic. When our container sees this sticker, it knows this class needs a partner to function.

```java
@MyAutowired
private PowerGrid grid;
```

_It says : "Mummy, ladka/ladki dhoond do . I can't live alone, hehe ."_

### 3. The Head of Family (`MiniSpring` Container) 👴

This is the **Bauji** (Big Boss). He does three main jobs:

1.  **Scan**: Searches your package for anyone wearing the `@MyService` badge. 🕵️
2.  **Register**: Keeps a note of everyone in his _khata_.
3.  **Inject**: Connects the families (objects) using **Reflection**. _Rishta pakka!_ 🤝

---

## 🧬 Singleton vs. Prototype

Our Mini-Spring understands two types of lifestyles :

1.  **Singleton (Default)** 🧘‍♂️

    - **Concept**: _I dont need to explain this. I mean look at your life :p_.
    - **In Code**:
      ```java
      @MyService // Default is "singleton"
      public class PowerGrid { ... }
      ```

2.  **Prototype** 🐑
    - **Concept**: _Sabko apna milega_.
    - **In Code**:
      ```java
      @MyService(scope = "prototype") // New one every time
      public class SmartBulb { ... }
      ```

---

## 🧪 The "Tadka" (Code Snippet)

Here is the recursive logic for building the dependencies.

```java
// Inside MiniSpring.java
private void injectDependencies(Object bean) {
    for (Field field : bean.getClass().getDeclaredFields()) {
        if (field.isAnnotationPresent(MyAutowired.class)) {
            // "Arre dependency mil gayi!" 😲

            // 1. Find the matching dependency
            Object dependency = getBean(field.getType());

            // 2. Make sure reflection is allowed to access the field
            field.setAccessible(true);

            // 3. Set the dependency!
            field.set(bean, dependency);
        }
    }
}
```

---

## 🎬 The Full Picture

Let's walk through the execution flow.

### 1. The Opening Ceremony 🕯️

It all starts in `Main.java`. We create an instance of our container.

```java
MiniSpring context = new MiniSpring();
context.start("org.learn.service"); // Looks for all classes within this package
```

### 2. The Door-to-Door Survey (Scanning) 📝

The `start()` method calls `getPackagePathFiles()`.

- It converts the package name `org.learn.service` to a file path.
- It knocks on every file in that directory.
- "Hello, are you a `@MyService`?"
- If **YES**, it creates a `BeanDefinition` (a profile card) for that class and saves it in a Map.

  ```java
  // Scanning Logic (Simplified)
  File[] files = directory.listFiles();
  for (File file : files) {
      Class<?> clazz = Class.forName(package + "." + fileName);
      if (clazz.isAnnotationPresent(MyService.class)) {
          BeanDefinition def = new BeanDefinition();
          def.setBeanClass(clazz);
          definitions.put(clazz, def);
      }
  }
  ```

### 3. The Grand Assembly (Instantiation) 🏗️

Once the scanning is done, `MiniSpring` looks at its list of definitions.

- For every **Singleton** bean, it says: _"Main abhi banaunga!"_ (I'll make it now).
- It calls `getBean(Class)`.

  ```java
  // Pre-instantiating Singletons
  for (Class<?> clazz : definitions.keySet()) {
      if (isSingleton(clazz)) {
           getBean(clazz);
      }
  }
  ```

### 4. The Relationship Management (Injection) 💍

Inside `getBean()`, the real drama happens:

1.  **Create Object**: `new SmartBulb()`.
2.  **Inject Dependencies**: It sees `@MyAutowired` on `PowerGrid grid`.
3.  **Recursive Call**: It calls `getBean(PowerGrid.class)`.
4.  **Set Field**: Once it gets the `PowerGrid` object, it forcibly pushes it into the private field of `SmartBulb`.

### 5. Happy Ending 🎉

Back in `Main.java`, when you ask for:

```java
LivingRoom room = context.getBean(LivingRoom.class);
room.turnOnLights();
```

All dependencies are already resolved. The lights turn on.

---

## Output

![Terminal Output](/images/spring-output.png)

### 📝 Decoding the Output:

1.  **"Starting Scan..."**: The container wakes up and starts looking for files.
2.  **"Found X bean definitions"**: It found our `@MyService` classes (`LivingRoom`, `SmartBulb`, `PowerGrid`).
3.  **"Context initialized"**: All Singletons are ready to serve.
4.  **"[LivingRoom] Lights are shining..."**: This proves that `LivingRoom` successfully got `SmartBulb` and `PowerGrid` injected into it! 💡
5.  **Prototype Test**: You can see `Bulb 1` and `Bulb 2` have different memory addresses (e.g., `@6acbcfc0` vs `@5f184fc6`). This means they are **different objects**.
6.  **Singleton Test**: `PowerGrid` has the _same_ address both times. It’s the **same object**.

## ⚠️ Disclaimer: Picture Abhi Baaki Hai

While our **Mini-Spring** runs, comparing it to the actual **Spring Framework** is like comparing your home-made **Maggie** to a **5-Course Meal at Taj**. 🍝 vs 🍲

Real Spring handles:

- Circular dependencies.
- AOP Proxies (The middle-men/dalals)
- Web servers
- And 100 other things.

## Source Code

[MiniSpring](https://github.com/parthpanchal123/Mini-Spring)

## 设计模式
### 策略模式
在策略模式（Strategy Pattern）中，一个类的行为或其算法可以在运行时更改。这种类型的设计模式属于行为型模式。可以省略代码中大量的if-else代码，下面是使用java的枚举实现策略模式
一般的实现方法
```java
if (strategy.equals("fast")) {
  // 快速执行
} else if (strategy.equals("normal")) {
  // 正常执行
} else if (strategy.equals("smooth")) {
  // 平滑执行
} else if (strategy.equals("slow")) {
  // 慢慢执行
}
```
使用策略模式实现的方法如下
```java
public enum Strategy {
    FAST {
      @Override
      void run() {
        //do something
      }
    },
    NORMAL {
      @Override
       void run() {
         //do something
      }
    },
    SMOOTH {
      @Override
       void run() {
         //do something
      }
    },
    SLOW {
      @Override
       void run() {
         //do something
      }
    };
    abstract void run();
}
```
客户端的代码如下
```java
Strategy strategy = Strategy.valueOf(param);
strategy.run();
```

#CS/Exception/Runtime #Status/진행

# RunTimeException
[공식 문서](https://docs.oracle.com/javase/8/docs/api/java/lang/RuntimeException.html)

## ArithmeticException
정수를 0으로 나눌 때 발생하는 런타임 예외예요.

```java
int result = 10 / 0; // ArithmeticException: / by zero
```
### 상속 구조

```
Throwable
└── Exception
    └── RuntimeException
        └── ArithmeticException
```

`RuntimeException` 계열이라 **unchecked exception**이에요. 즉 try-catch를 강제하지 않아요.

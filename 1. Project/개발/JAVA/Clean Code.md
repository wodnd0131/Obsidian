#Clean_Code #Status/진행 

[읽기 좋은 코드](https://github.com/cho-log/java-learning-test/blob/main/clean-code/initial/src/test/java/cholog/goodcode/ReadableCodeTest.java)
[예측 가능한 코드](https://github.com/cho-log/java-learning-test/blob/main/clean-code/initial/src/test/java/cholog/goodcode/PredictableCodeTest.java)
[실수를 방지하는 코드](https://github.com/cho-log/java-learning-test/blob/main/clean-code/initial/src/test/java/cholog/goodcode/AvoidMistakeCodeTest.java)

# 1. **Early Return (Guard Clause)** 패턴
```
if (condA) {
    if (condB) {
        if (condC) {
            doSomething();
        }
    }
}
```
```
if (!condA) return;
if (!condB) return;
if (!condC) return;

doSomething();
```
>전자의 경우에는 'A 조건이고, B 조건이고, C 조건이면 일케한다!'로 사고되고,  
후자의 경우에는 '예외상황 A 컷, 예외상황 B 컷, 예외상황 C 컷, 그러면 일케한다!'로 사고되는데,



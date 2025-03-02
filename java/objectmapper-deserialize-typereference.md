# ObjectMapper로 제네릭 Deserialize 하기(TypeReference)

### Problem

이해없이 Jackson ObjectMapper로 Deserialize를 하다보면, 아래와 같은 Cast error와 마주칠 수 있다.

```
class java.lang.String cannot be cast to class java.util.List
class java.util.LinkedHashMap cannot be cast to class User
```

나름대로 코드를 추상화 한다며 넣어둔 코드는 대충 이런식이었다.

```java
private final ObjectMapper objectMapper;

public <T> T getData(String jsonString){
    …
    return objectMapper.readValue(jsonString, new TypeReference<>() {});
    // readValue: Json String -> Class Type
    // convertValue: Object Type -> Class Type
}

public User getUser(String jsonString){
    User user = getData(jsonString);
}
```

jsonString이 User로 Deserialize 되길 기대했지만 TypeReference는 User로 치환되지 않았다..ㅜ\


### Cause

TypeReference에 제네릭이 적용되지 않았다. 제네릭은 언제, 어떤 타입을 가지게 될까?

#### Generic Type Erasure (제네릭 타입 소거)

* Compile time
  * 제네릭의 Type 검사
* Runtime
  * `<T>`, `<?>`: Object로 변환
  * `<T extends Comparable>`: Bounded type은 Comparable로 변환

\
그러니 Runtime에는 이미 이렇게 getData 함수가 Object로 치환이 되어있었을 것이다.

```java
public Object getData(String jsonString){
    ...
    return objectMapper.readValue(jsonString, new TypeReference<>() {});
}
```

\


### Solution

#### 1. Type도 함께 넘겨주기

getData() 호출할 때 Type도 함께 넘겨주어 제네릭의 영향을 받지 않고 Deserialize를 해줄 수 있다.

```java
public <T> T getData(String jsonString, JavaType javaType){
    ...
    return objectMapper.readValue(jsonString, javaType);
}

public List<User> getUser(String jsonString){
    JavaType javaType = objectMapper.getTypeFactory()
    	.constructParametricType(List.class, User.class);
    List<User> user = getData(jsonString, javaType);
}
```

\


#### 2. TypeReference으로 넘기기

* TypeReference: 제네릭 타입 시스템에서 타입 정보를 런타임에 유지하기 위해 사용
* Super Type Token 패턴 사용
  * 실제 타입 정보가 서브클래스 바이트코드에 남아있어 이를 이용

```java
// Super Type Token Pattern
Type type = ((ParameterizedType) user.getClass().getGenericSuperclass())
	.getActualTypeArguments()[0]
```

그래서 이 typeReference를 사용하면 GenericSuperclass Parameter 정보를 가지고 올 수 있기 때문에, 문제 없이 Mapping이 가능하다!

```java
TypeReference<List<User>> typeReference = new TypeReference<List<User>>() {};
```

* 이 TypeReference를 1번처럼 넘겨주면 된다!
* 아무래도 가독성이 1번보다는 좋은 편인 것 같다!

\


#### 3. Object로 넘겨서, 받아오는 함수에서 타입 매핑

getData()에서 그대로 Object로 리턴하고, getUser()에서 convertValue를 한다. (ObjectMapper가 꽤 무겁기 때문에, 두 번이나 선언해서 사용하는 것은 좋아보이지 않는다!)

```java
public <T> T getData(String jsonString){
    ...
    return objectMapper.readValue(jsonString, Object.class);
}

public List<User> getUser(String jsonString){
    List<User> user = objectMapper
    	.convertValue(getData(jsonString), new TypeReference<>() {});
}
```

\


### Conclusion

되는대로 코드를 짜다가(ㅠㅠ이러면 안돼!!!) 안되니까 찾아보기 시작했는데, 알아가다보니 Generic에 대한 이해가 부족해서 생긴 문제였다. 제네릭을 사용할 때는 Generic Type Erasure 때문에 Runtime 시점에는 제네릭 타입이 바보!가 된다는 사실을 잊지 말자!

\
\


### References

https://stackoverflow.com/questions/6846244/jackson-and-generic-type-reference https://sungminhong.github.io/spring/superTypeToken/ https://velog.io/@happyjamy/Java-%EC%A0%9C%EB%84%A4%EB%A6%AD%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC-TypeReference https://thecodinglog.github.io/java/2021/10/12/java-generic-type-reference.html https://www.baeldung.com/java-super-type-tokens https://stackoverflow.com/questions/11936620/jackson-deserialising-json-string-typereference-vs-typefactory-constructcoll

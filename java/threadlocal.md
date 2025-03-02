# ThreadLocal로 유저 정보 처리하기

### Problem

CRUD를 구현하다보면, api call을 한 유저 정보가 필요하다. 기존에 구현된 방식은 token이 들어오면 filter에서 토큰 정보를 parsing해서 api의 parameter에 조작해서 넣어주게 되어있었다.

이 방법으로 Controller에서 필요한 param을 Service method들로 넘겨주어 사용할 수 있었지만, 거의 모든 Service method에 parameter로 유저정보를 넘겨줘야해서 코드가 지저분해졌다.

{% code overflow="wrap" %}
```java
@GetMapping
public PageImpl<ResponseDto> getPosts(
    @Valid @RequestParam(value = "code", required = true) String code,
    @Valid @RequestParam(value = "id", required = true) String id,
    @Valid @RequestParam(value = "name", required = true) String name,
    @Valid @RequestParam(value = "blahblah", required = false, defaultValue = "") String blahblah,
    PageRequest pageRequest, @RequestBody RequestDto requestDto) {
        return postService.getPosts(code, id, name, blahblah, pageRequest, requestDto);
    }
```
{% endcode %}

그리고 이렇게 되면,, 머리로는 알지만 코드 자체로는 어떤 값을 받는지 명세가 불분명해진다 ㅜ

### Solution

Spring Security를 사용하면 ThreadLocal을 사용한 Context를 제공하기 때문에, ThreadLocal을 직접 사용해서 유저 정보를 저장해서 사용하기로 했다.

```
ThreadLocal : 현재 Thread에서만 사용할 수 있는 Local variable 제공!!
```

Filter에서 token으로 parsing한 유저 정보를 setUserInfo()로 ThreadLocal에 저장해두고, Service method에서는 getUserInfo()로 ThreadLocal에서 정보를 가져와서 사용하면 parameter로 넘겨주어야하는 번거로움을 해결할 수 있다:)

```java
public class UserContext {

    private static final ThreadLocal<UserInfo> userHolder = new ThreadLocal<>();

    public static void setUserInfo(UserInfo userInfo) {
        userHolder.set(userInfo);
    }

    public static UserInfo getUserInfo() {
        return userHolder.get();
    }

    public static void clear() {
        userHolder.remove();
    }
}
```

### Conclusion

기존에는 Detail한 유저정보가 필요하면 그때마다 조회해서 사용했는데, filter에서 넣을 때 한번에 조회해서 UserInfo에 정보를 채워두면 언제든지 가져다 쓸 수 있어서 좋았다.

그리고 권한처리도 Custom Annotation으로 Controller에 붙여서 사용하고 있어서 Controller에서 불러오는 Param의 순서도 영향을 받았었는데(아래처럼 args로 받아서 사용)

```java
@Before("@annotation(serviceAuthority) && args(typeCode,..)")
```

이젠 ThreadLocal로 가지고 오면 되기 때문에, 이 annotation을 사용하기 위해서 어떤 셋팅이 필요한지 모르는 다른 개발자가 순서를 바꾸거나 누락하는 등 실수하는 위험을 막을 수 있었다!

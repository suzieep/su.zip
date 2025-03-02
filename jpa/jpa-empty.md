---
description: >-
  [ERROR] JpaSystemException : A collection with cascade="all-delete-orphan" was
  no longer referenced by the owning entity instance: User.posts
---

# JPA는 어떻게 빈(empty) 객체의 영속성을 유지할까?

### Problem

{% code overflow="wrap" %}
```
[ERROR] JpaSystemException : A collection with cascade="all-delete-orphan" was no longer referenced by the owning entity instance: User.posts
```
{% endcode %}

### As-is

User에서 OneToMany로 Posts가 매핑되어 있다고 가정해보자, 아래 코드에서는 user.posts가 비어있으면 new list 를 선언해서 채워주고 다시 이 만들어준 하위 collection을 user에 set하는 방식으로 되어있다.

```java
...
users.forEach(user -> {
    List<Post> posts = ObjectUtils.isNotEmpty(user.getPosts()) ?
        originUser.getPosts() : Lists.newArrayList();
    
    // Delete Posts
    posts.removeAll(deletedPosts);
    ...    
    // Create Posts
    posts.add(Post.builder()
        .user(user)
        ...
        .build());

    user.setPosts(posts);
});
```

이렇게 넣어두면 JPA에서 던지는 Exception으로 이 에러를 만나게 된다

{% code overflow="wrap" %}
```
A collection with cascade="all-delete-orphan" was no longer referenced by the owning entity instance: User.posts
```
{% endcode %}

### Cause

Entity 객체에 OneToMany 매핑이 있을 때, 매핑된 Collection에 대해 새로운 값을 넣는 방식으로 갱신하면 참조가 끊어진다!! 새로운 Collection을 넣으면서 JPA 참조가 끊어지기 때문이다.

그럼 빈 Collection은 어떻게 저장되어있길래 Hibernate가 추적할 수 있는 걸까?

![image](https://github.com/user-attachments/assets/c8bb3b91-0993-4f28-af27-ee40d3156fd4)

Repository에서 가지고 온 객체에 대해서 Collection이 비어 있다고 해도 이렇게 PersistentBag이라는 Collection Wrappper로 영속성을 유지한다.

### To-be

기존에 가지고 있던 Collection에 직접 갱신하는 방식으로 참조를 유지해주면서 수정해주면 문제를 해결할 수 있다

```java
users.forEach(user -> {
    // Delete Posts
    user.getPosts().removeAll(deletedPosts);
    ...    
    // Create Posts
    user.getPosts().add(Post.builder()
        .user(user)
        ...
        .build());
});
```

### References

* https://velog.io/@gnoesnooj/ERROR-A-collection-with-cascadeall-delete-orphan-was-no-longer-referenced-by-the-owning-entity-instance
* https://ttl-blog.tistory.com/138

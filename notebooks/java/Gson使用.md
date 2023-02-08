Gson



## fromJson转带泛型的对象

```java
List<People> peopleList = new Gson().fromJson(publishJsonArray, new TypeToken<List<People>>() {}.getType());
```


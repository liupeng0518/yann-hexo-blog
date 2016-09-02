---
title: java_tips
tags:
---


1. 空间换时间的小把戏
	
```java
enum Type {
        NAME("姓名"), AGE("年龄"), ADDRESS("住址");
        
        private String typeName;
        
        Type(String typeName) {
            this.typeName = typeName;
        }
        
        public static Type fromTypeName(String name) {
            for (Type t : Type.values()) {
                if (t.typeName.equals(name)) return t;
            }
            return null;
        }
    }
```

这样写的坏处大概就是这个 fromTypeName 里面的for循环，虽然语义很清晰，但是我们每次都要去遍历整个枚举对象。


```java
enum Type {
        NAME("姓名"), AGE("年龄"), ADDRESS("住址");

        private static Map<String, Type> all = new HashMap<String, Type>() {
            {
                for (Type t : Type.values()) {
                    all.put(t.typeName, t);
                }
            }
        };
        private String typeName;

        Type(String typeName) {
            this.typeName = typeName;
        }


        public static B.Type fromTypeName(String name) {
            return all.get(name);
        }
    }

```

在运行之初，我们将所有的对象都放置于Map中，这样直接通过HaskMap去获取对象的效率更高，从O(n) -> O(1) 很标准的空间换时间的小把戏。
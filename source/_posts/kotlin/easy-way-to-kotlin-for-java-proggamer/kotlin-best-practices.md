---
title:  写给 Java 开发者的 Kotlin 教程 (13) - 最佳实践
date: 2018-08-17 15:40:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![](https://s1.ax2x.com/2018/08/17/59jvsG.jpg)
<!-- more -->

## 表达式
{% tabs Expressions %}
<!-- tab ✔✔✔ -->
{% codeblock lang:kotlin %}
fun getDefaultLocale2(deliveryArea: String) = when (deliveryArea.toLowerCase()) {
    "germany", "austria" -> Locale.GERMAN
    "usa", "great britain" -> Locale.ENGLISH
    "france" -> Locale.FRENCH
    else -> Locale.ENGLISH
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab ✘✘✘ -->
{% codeblock lang:kotlin %}
fun getDefaultLocale(deliveryArea: String): Locale {
    val deliverAreaLower = deliveryArea.toLowerCase()
    if (deliverAreaLower == "germany" || deliverAreaLower == "austria") {
        return Locale.GERMAN
    }
    if (deliverAreaLower == "usa" || deliverAreaLower == "great britain") {
        return Locale.ENGLISH
    }
    if (deliverAreaLower == "france") {
        return Locale.FRENCH
    }
    return Locale.ENGLISH
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}


## Try
```kotlin
val json = """{"message":"HELLO"}"""
val message = try {
    JSONObject(json).getString("message")
} catch (ex: JSONException) {
    json
}
```

## 对象工具类
{% tabs Utility Functions %}
<!-- tab ✔✔✔ -->
{% codeblock lang:kotlin %}
fun String.countAmountOfX(): Int {
    return length - replace("x", "").length
}
"xFunxWithxKotlinx".countAmountOfX()
{% endcodeblock %}
<!-- endtab -->
<!-- tab ✘✘✘ -->
{% codeblock lang:kotlin %}
object StringUtil {
    fun countAmountOfX(string: String): Int{
        return string.length - string.replace("x", "").length
    }
}
StringUtil.countAmountOfX("xFunxWithxKotlinx")
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}


## 优先使用命名参数
{% tabs Utility Functions %}
<!-- tab ✔✔✔ -->
{% codeblock lang:kotlin %}
val config2 = SearchConfig2(
       root = "~/folder",
       term = "kotlin",
       recursive = true,
       followSymlinks = true
)
{% endcodeblock %}
<!-- endtab -->
<!-- tab ✘✘✘ -->
{% codeblock lang:kotlin %}
val config = SearchConfig()
       .setRoot("~/folder")
       .setTerm("kotlin")
       .setRecursive(true)
       .setFollowSymlinks(true)
StringUtil.countAmountOfX("xFunxWithxKotlinx")
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

## 不要重载默认参数
{% tabs Overload for Default Arguments %}
<!-- tab ✔✔✔ -->
{% codeblock lang:kotlin %}
fun find(name: String, recursive: Boolean = true){
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab ✘✘✘ -->
{% codeblock lang:kotlin %}
fun find(name: String){
    find(name, true)
}
fun find(name: String, recursive: Boolean){
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

## 考虑使用 let
{% tabs lets %}
<!-- tab ✔✔✔ -->
{% codeblock lang:kotlin %}
findOrder()?.let { dun(it.customer) }
//or
findOrder()?.customer?.let(::dun)
{% endcodeblock %}
<!-- endtab -->
<!-- tab ✘✘✘ -->
{% codeblock lang:kotlin %}
val order: Order? = findOrder()
if (order != null){
    dun(order.customer)
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

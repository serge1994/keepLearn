# springboot

## SpringBoot 中 @RestController 和 @Controller 的区别

```java
1 - 在springboot中，@RestController 相当于 @Controller + @ResponseBody;
2 - 即在Controller类中，若想返回jsp或html页面，则不能用@RestController，只能使用@Controller；
3 - 若返回的是json或xml数据，可以有两种写法：

1. @RestController注解，然后直接return json数据即可；
2. @Controller注解放类之前，然后若类中某个方法需要返回json数据，则需在该方法前添加@ResponseBody注解；
```






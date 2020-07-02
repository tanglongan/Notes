- ### @Transactional标记的方法是非public方法

    失效的原理是：@Transactional是基于动态代理的，非public的方法，使得@Transactional的目标对象为空，所以不能回滚

- 
# Spring单例Bean是否线程安全？

**不是线程安全！！！**

> Spring框架中有一个@Scope注解，默认值为singleton，单例的
>
> 因为一般在Spring的bena中都是注入无状态对象，没有线程安全问题，如果在Bean中定义了可修改的成员变量，是要考虑线程安全问题的，可以采用多例或加锁来解决

**无状态的Bean——线程安全**

~~~java
@Service
public class UserService {
    // 没有实例变量，只有局部变量和参数
    public User getUserById(Long id){
        User user=userDao.findById(id); // 局部变量
        return user;
    }
}
~~~

**有状态的Bean——线程不安全**

~~~java
@Service
public class CounterService {
    private int count=0; //共享的可变状态
    public void increment() {
        count++; // 非原子操作，线程不安全
    }
}
~~~



**保证线程安全的常用方案之一——ThreadLocal方案**

~~~java
// 此方案适用于线程隔离的场景，每个线程拥有独立的副本
@Service
public class UserContextService {
    private ThreadLocal<USer> currentUser=new ThreadLocal<>();
    public void setUser(User user){
        currentUser.set(user);
    }
    public User getUser(){
        return currentUser.get();
    }
}
~~~



在实际开发中，推荐将Service/DAO层设计为无状态，需要状态时使用局部变量或ThreadLcoal，尽量避免在单例Bean中定义可变的实例变量。






















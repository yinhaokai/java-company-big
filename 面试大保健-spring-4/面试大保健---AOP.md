
---
大鱼哥最近去面试发现，面试官ioc都不问了，就喜欢问aop，还喜欢连着aop问下设计模式或者是日志系统设计等等。好的，面试官，来都来了你就别走了。
进入正题

什么是AOP，大家也都清楚是面向切面编程，既然是一种编程模式肯定和面向对象编程，面向函数编程都是一样的，其本身就是一种编程的思考方式，当然这种思考方式肯定和语言是要相结合的。

那么回归到正题，既然是一种编程方式，在java里如何实现。这个首先还是得明确的，既然是一种编程方式就不可能直接和spring挂钩，理论上spring可以完成的，我们也可以完成，只是完成度有所区别，所以一定要把spring和aop分离开来，spring只是整合了一些原有的一些类库和一些设计模式完成的spring aop而非aop本身。

好，既然说到需要通过java完成aop，那么首先需要一个aop到底如何实现，这里就需要引入今天的幕后王者--代理。为什么aop就需要代理辅助呢。我们看下简单的静态代理的代码.

```
public interface Student {
     void say();
}
```


```
public class XiaoMing implements Student {
    @Override
    public void say() {
        System.out.println("老师好！！");
    }
}
```

```
public class StudentProxy {
    private Student student;

    public StudentProxy(Student student) {
        this.student = student;
    }
    public void say(){
        System.out.println("起立");
        student.say();
        System.out.println("坐下");
    }
    public static void main(String[] args) {
        XiaoMing xiaoMing=new XiaoMing();
        StudentProxy studentProxy=new StudentProxy(xiaoMing);
        studentProxy.say();
    }
}
```
可以发现一个现象，就是我们可以对某个类中的方法进行增强，但是每增强一个方法都需要在代理类重新实现这份代码。想像一个场景，我们在上课的时候会说老师好，下课的时候会说老师再见，那班长能否只实现一个方法即可，因为班长都是起立坐下。此外对于某些场景我们需要不需要代理，比如下课玩耍，这个时候按照静态代理仍需实现这个下课玩耍的方法。这就很有问题，我下课玩我需要你代理吗，但是又不能完全解耦，毕竟上下课还得靠你。

```
public class StudentProxyHander implements InvocationHandler {
    private Object object;
    public Object bind(Object o){
        this.object=o;
        return Proxy.newProxyInstance(o.getClass().getClassLoader(),o.getClass().getInterfaces(),this);}
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("起立");
        Object r=method.invoke(object,args);
        System.out.println("坐下");
        return r;
    }

    public static void main(String[] args) {
        StudentProxyHander studentProxyHander=new StudentProxyHander();
        Student student= (Student) studentProxyHander.bind(new XiaoMing());
        student.say();
        System.out.println("--------------上课半小时------------------");
        student.saybey();
    }
}
```
至此完成了动态代理，可能有同学要问了，代理都学完了，你aop都没讲啊。
其实一切尽在不言中，我们的"起立"就是前置通知，"坐下"就是后置通知。
然后同学们又要问了，老鱼，我们听过这个aop有切点，切面，增强，连接点，目标对象，织入，代理类等概念。那么在实际通过动态代理实现springaop的时候，如何保证一一对应。

首先在得分清，哪些是概念，哪些是实际开发中需要实现的。其实在以上几个概念中的真正需要掌握的很少，需要明确的关键点只有三个分别是：切面，切点，增强。另外切面就是连接点和增强的合集。因此实际概念只有切点和增强。
连接点是指切点实现后，java程序再运行期间动态织入的点位，切面是切点和增强合集，代理类是指被aop代理的类都叫代理类。目标对象是指被代理的类。

接着，增强在上面的动态代理的实现中，实现了一个代理类对应一种增强方法，从上面可以看出我们只能实现"起立""坐下"，如果需要实现"站起""卧倒"需要重新实现一个代理类。这是问题1.

然后，连接点在动态代理其实不明显，连接点是绑定在类的，而在这绑定是对象，即springaop是只有使用的aop，该类生成的所有对象都会被代理，而在动态代理中只有指定的对象才能使用aop。这是问题2.给个小提示:这里其实和ioc是互相绑定的，通过ioc生成bean，对bean进行代理。

好，那我们来手撕一个springaop

在手撕之前，我们先实现一个原生的springaop，还是以上面的学生的例子为基础，我们看下最后我们自己实现的aop和实际的springaop有什么不同点和相同点。


```
@Aspect
@Component
public class SpringAopAspect {
    @Pointcut("execution(* com.myself.demo.springboot.Student.*(..))")
    public void webLog(){ }

    @Before("webLog()")
    public void log(){
        System.out.println("起立");
        //webLog();
        System.out.println("坐下");
    }
}
  @Scheduled(cron = "* * * * * ?")
    public  void say(){
        System.out.println("aaaa");
        //直接new 是不能进aop代理的
       //  XiaoMing xiaoMing=new XiaoMing();
        xiaoMing.say();
```
这就是一个简单的aop。
我们来直接实现一个手写的aop
https://blog.csdn.net/litianxiang_kaola/article/details/86647057
















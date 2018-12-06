# 策略模式和Spring的结合使用

## 策略模式

策略模式的定义：
>策略模式是对算法的封装，把使用算法的责任和算法本身分隔开，委派给不同的对象管理。策略模式通常把一系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类。

![这里写图片描述](http://pj9b1v2hm.bkt.clouddn.com/strategy.png)

## 解决了我们的什么问题？


在实际的项目中，完成一项任务，可以有多种不同的方式，对于新手来说我们是怎么来用的，很经典的 if...else if...else...，正如下面的这段代码，每一个平台都有自己的**HuiTieTicketNumber**

```
if (vendeeId==1){   //携程
        TicketContext ticketContextCtrip=CtripTicketContext(reportDetail,orderDetail);
        if (ticketContextCtrip!=null){
            huitiePiaoHao=new CtripHuiTieTicketNumber();
            huitiePiaoHao.handle(ticketContextCtrip);
        }
    }else if (vendeeId==2){  //去哪
        TicketContext ticketContextQunar=QunarTicketContext(reportDetail,orderDetail);
        if (ticketContextQunar!=null){
            huitiePiaoHao=new QunarHuiTieTicketNumber();
            huitiePiaoHao.handle(ticketContextQunar);
        }
    }else if (vendeeId==3){  //阿里
        TicketContext ticketContextQua=QuaTicketContext(reportDetail,orderDetail);
        if (ticketContextQua!=null){
            huitiePiaoHao=new QuaHuiTieTicketNumber();
            huitiePiaoHao.handle(ticketContextQua);
        }
    }else if (vendeeId==4){  //同程
        TicketContext ticketContextTongCheng=QunarTicketContext(reportDetail,orderDetail);
        if (ticketContextTongCheng!=null){
            huitiePiaoHao=new TongChengHuiTieTicketNumber();
            TicketContext ticketContext=huitiePiaoHao.handle(ticketContextTongCheng);
            if (ticketContext.getStatus()==2){

            }
        }
    }
}
```

当然，如果只是为了完成功能，这样做确实是可行的，但是，这样也会被局限住，如果新增一个平台呢，就需要再来修改代码，这就违背了对内修改关闭，对外扩展开放的原则。所以在这个地方应用了策略模式。把每一个平台的**HuiTie**当成一种策略封装到独立的类中进行处理。


## 代码实现

策略工厂类，具体的策略类是在spring注入的。

```
/**
 * 平台策略模式工厂类
 * Created by ling.zhang on 2017/1/21.
 */
public class OrderIntegrateReadFactory {
    private final static Logger logger= LoggerFactory.getLogger(OrderIntegrateReadFactory.class);

    private Map<String,IVendeeContextStrategy> vendeeContextStrategyMap=new HashMap<String, IVendeeContextStrategy>();

    public Map<String, IVendeeContextStrategy> getVendeeContextStrategyMap() {
        return vendeeContextStrategyMap;
    }

    public void setVendeeContextStrategyMap(Map<String, IVendeeContextStrategy> vendeeContextStrategyMap) {
        this.vendeeContextStrategyMap = vendeeContextStrategyMap;
    }

    public boolean doAction(String strType, ReportDetail reportDetail, OrderDetail orderDetail){
        return this.vendeeContextStrategyMap.get(strType).getTicketContext(reportDetail,orderDetail);
    }
}

```
spring配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
  http://www.springframework.org/schema/context
  http://www.springframework.org/schema/context/spring-context-3.2.xsd">


    <bean id="orderIntegrateReadFactory" class="com.flight.inter.otaadapter.factory.OrderIntegrateReadFactory">
       <property name="vendeeContextStrategyMap">
           <map>
               <entry key="1" value-ref="ctripContextStrategy"/>
               <entry key="2" value-ref="qunarContextStrategy"/>
               <entry key="3" value-ref="aliquaContextStrategy"/>
               <entry key="4" value-ref="tongChengContextStrategy"/>
           </map>
       </property>
    </bean>

    <bean id="ctripContextStrategy" class="com.flight.inter.otaadapter.manage.cloudticket.CtripContextStrategy">
        <property name="huitiePiaoHao" ref="ctripHuiTieTicketNumber"/>
        <property name="redisManage" ref="policyRedis"/>
    </bean>

    <bean id="qunarContextStrategy" class="com.flight.inter.otaadapter.manage.cloudticket.QunarContextStrategy">
        <property name="huitiePiaoHao" ref="qunarHuiTieTicketNumber"/>
        <property name="redisManage" ref="policyRedis"/>
    </bean>
    <bean id="aliquaContextStrategy" class="com.flight.inter.otaadapter.manage.cloudticket.AliquaContextStrategy">
        <property name="huitiePiaoHao" ref="quaHuiTieTicketNumber"/>
        <property name="redisManage" ref="policyRedis"/>
    </bean>
    <bean id="tongChengContextStrategy" class="com.flight.inter.otaadapter.manage.cloudticket.TongChengContextStrategy">
        <property name="huitiePiaoHao" ref="TongChengHuiTieTicketNumber"/>
        <property name="redisManage" ref="policyRedis"/>
    </bean>
</beans>
```

接口类：

```
/**
 * Created by ling.zhang on 2017/1/21.
 */
public interface IVendeeContextStrategy {

    boolean getTicketContext(ReportDetail reportDetail, OrderDetail orderDetail);
}

```

具体的实现类：

```
public class QunarContextStrategy implements IVendeeContextStrategy{
    private final static Logger logger= LoggerFactory.getLogger(QunarContextStrategy.class);
    HandleManage<TicketContext> huitiePiaoHao;
    RedisManager redisManage;
    public RedisManager getRedisManage() {
        return redisManage;
    }

    public void setRedisManage(RedisManager redisManage) {
        this.redisManage = redisManage;
    }
    public HandleManage<TicketContext> getHuitiePiaoHao() {
        return huitiePiaoHao;
    }

    public void setHuitiePiaoHao(HandleManage<TicketContext> huitiePiaoHao) {
        this.huitiePiaoHao = huitiePiaoHao;
    }
    @Override
    public boolean getTicketContext(ReportDetail reportDetail, OrderDetail orderDetail){
       try {
           TicketContext ticketContextQunar = QunarTicketContext(reportDetail, orderDetail);
           if (ticketContextQunar != null) {
               TicketContext ticketContext=huitiePiaoHao.handle(ticketContextQunar);
               if (ticketContext.isHuitieresult()){
                   redisManage.add("vendee-"+ticketContext.getTtsorderno(),"true");
                   redisManage.expire("vendee-"+ticketContext.getTtsorderno(),600);
                   logger.info("orderintegrate huitie success data {}",JSONObject.toJSONString(ticketContext));
                   return true;
               }
           }
       }catch (Exception e){
           logger.error("orerintegrate read ticketcontext error {}",e);
            return false;
       }
        return false;
    }
 }

```


每个平台都会有上面的这个Strategy类，就不一一列举了。这样的话，当你的需求中在来一个新的平台的时候，只需要新建一个类，并在spring中注入上，那么就可以用了。也符合设计原则。当你的代码中大量的出现if else的话，就说明你的代码需要进行重构了，不然的话你只会被局限在这个一个小的空间里面，当你把它解耦出来后，你面对的视野和高度就不一样了，你所能做的事情就会超出原来的很多倍。




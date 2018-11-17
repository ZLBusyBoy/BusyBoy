# 观察者模式和Spring的结合使用

这周给分了一个任务，就是对查询回来的数据进行各种各样的过滤,有七种不同的过滤条件。过滤条件是在数据库中存着的。在我们项目中有一个热发，就是定时的从数据库中把数据取出来进行分类保存到Property中或者Map中。所以一开始想的一个笨的方法就是把七种不同的过滤条件热发到七个不同的Map中去。然后再定义一个过滤的类，所有的查询回来的数据都要经过这个类的处理。
后来想了想，这样做的话，不利于扩展，要是后期还有其他的过滤的话，耦合性太强了。所以这个时候就想到了设计模式中的观察者模式。应用在这个场景下再合适不过了。定义了七个Filter，当热发执行之后，通知所有的观察者来我这拿最新的数据。而且当新添一个新的过滤的过滤条件的话，只需要新加一个过滤的类，并在spring的监听器中配置上该类就可以了，其实这就实现了对内修改关闭，对外扩展。 下面试具体的代码实现。先贴一张图，很经典。

![这里写图片描述](/png/DesignPatterns/observise.png)

## 被观察者


```
package com.flight.inter.otaadapter.filters;

import java.util.ArrayList;
import java.util.List;

/**
 * 观察者模式抽象通知者类--报价拦截
 * Created by ling.zhang on 2016/12/2.
 */
public abstract class AbstractPriceFilter {
    private List<IPriceFilterListener> listeners=new ArrayList<IPriceFilterListener>();

    public void registerListener(List<IPriceFilterListener> listener){
        this.listeners=listener;
    }

    public void notifyListenner(){
        for(int i=0;i<listeners.size();i++){
            IPriceFilterListener priceFilterListener=listeners.get(i);
            priceFilterListener.actionPerformed();
        }
    }
}
```
在这个地方需要注意的是，`registerListener`是需要有注册者的，你总不用说有通知者但是没有被通知者，那时间长了通知者不得倒闭啦，哈哈。在这是通过spring来注册的，就不用在代码中写了，要不然每来一个观察者都写注册方法。我是注册了多个，所以用的是List，如果只有一个的话，不用List，具体看你怎么注册了。当在来一个新的观察者的时候，只需要在list下在新加一个ref就可以注册进去了。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
  http://www.springframework.org/schema/context
  http://www.springframework.org/schema/context/spring-context-3.2.xsd">


    <bean id="divCabinFilter" class="com.flight.inter.otaadapter.filters.DivCabinFilter"/>
    <bean id="divFilter" class="com.flight.inter.otaadapter.filters.DivFilter"/>
    <bean id="flightFilter" class="com.flight.inter.otaadapter.filters.FlightFilter"/>
    <bean id="flightGroupFilter" class="com.flight.inter.otaadapter.filters.FlightGroupFilter"/>
    <bean id="operatingCarrierFilter" class="com.flight.inter.otaadapter.filters.OperatingCarrierFilter"/>
    <bean id="shareCarriersFilter" class="com.flight.inter.otaadapter.filters.ShareCarriersFilter"/>
    <bean id="validatingCarrierFilter" class="com.flight.inter.otaadapter.filters.ValidatingCarrierFilter"/>

    <bean id="hotDeployManager" class="com.flight.inter.otaadapter.manage.HotDeployManager"/>

    <bean id="priceFilterListener" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="targetObject">
            <ref local="hotDeployManager"/>
        </property>
        <property name="targetMethod">
            <value>registerListener</value>
        </property>

        <property name="arguments">
            <list>
                <ref bean="divCabinFilter"/>
                <ref bean="divFilter"/>
                <ref bean="flightFilter"/>
                <ref bean="flightGroupFilter"/>
                <ref bean="operatingCarrierFilter"/>
                <ref bean="shareCarriersFilter"/>
                <ref bean="validatingCarrierFilter"/>
            </list>

        </property>
    </bean>
</beans>
```


上边的是抽象通知者，具体的可以看下子类。子类会去调用通知的方法。下面是个样例。

```
public class HotDeployManager extends AbstractPriceFilter
{
    //满足你的条件时，去通知观察者们
	notifyListenner();
}
```

## 观察者 

定义了一个接口先。

```
public interface IPriceFilterListener {
    void actionPerformed();
}
```

然后是具体的实现类。用一个来做个示范。通知者通知到该类的时候，这个类就回去得到自己想要的数据。

```
public class DivCabinFilter implements IPriceFilterListener{
    private static final Logger logger = LoggerFactory.getLogger(DivCabinFilter.class);

    static Map<String,Map<String,String>> divCabinMap=new HashMap<>();

    @Override
    public void actionPerformed(){
        Map<String,String> map=HotDeployManager.getPriceFilter("divCabin");

        if (map!=null){
            Set set=map.entrySet();
            Map<String,Map<String,String>> divCabinGetMap=new HashMap<>();
            for (Iterator iterator=set.iterator();iterator.hasNext();){
                Map.Entry entry=(Map.Entry)iterator.next();
                String configKey=(String)entry.getKey();
                String divCabin=(String)entry.getValue();
                Map<String,String> forbidresult=forbidDivCabin(divCabin);
                divCabinGetMap.put(configKey,forbidresult);
            }
            divCabinMap=divCabinGetMap;
        }
    }
}
```

很多的知识不是会了就会了，在自己的脑子里存着是一回事，能在特定的业务场景下能用上是另一回事。有些知识没用时觉得难，但是用过了之后就觉得真的不是很难。多实践。








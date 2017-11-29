---
layout:     post
title:      SSO单用户登录
summary:    所谓“单用户单账户登录”是指：在同一系统中，一个用户名不能在两个地方同时登录
tags:       [Blog]
---

# CAS单点登录-单用户登录（十九）

# 简介

所谓“单用户单账户登录”是指：在同一系统中，一个用户名不能在两个地方同时登录。

如：
当某账号在 A 处登录后，在未退出的情况下，如果再到 B 处登录，那么，系统会挤下 A 处登录的账号

# 程序逻辑
我们一路学习cas过来应该知道如下知识
1. 维持一个用户状态是用`tgt`
2. 用户登录成功后`tgt`会创建
3. 业务系统验证成功是采用`st`的校验
4. 用户注销相当于删除`tgt`
5. 删除tgt采用`CentralAuthenticationService.destroyTicketGrantingTicket`

---
有以上的知识我们即可对其他用户的提出，程序应该满足以下逻辑：
1. 监听`tgt`创建事件
2. 获取用户id，以及tgt
3. 根据用户id，认证方式`clientName`寻找所有的tgt
4. 过滤非当前用户的tgt的所有tgt
5. 删除过滤后的tgt（正确的逻辑过滤后一般情况剩下一个，因为已经单用户登录了）

**第三点详解：**
为什么要采用clientName进行过滤呢，因为认证平台可能通过restful认证，qq、github、微信的OAuth2认证等等，所以认证方式不同，最后的用户id以及clientName会不同，所以要根据用户认证方式以及id，找到所有该用户的认证方式进行删除tgt，否则会出现，oauth2登录的用户用账号登录无法强制注销


# 实战

## TGT创建监听
这个监听是为了用户登录成功后对其他用户进行剔除

TGTCreateEventListener.java
```java
/*
 * 版权所有.(c)2008-2017. 卡尔科技工作室
 */

package com.carl.sso.support.single.listener;

import com.carl.sso.support.single.service.IUserIdObtainService;
import com.carl.sso.support.single.service.TriggerLogoutService;
import org.apereo.cas.support.events.ticket.CasTicketGrantingTicketCreatedEvent;
import org.apereo.cas.ticket.TicketGrantingTicket;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;

import javax.validation.constraints.NotNull;
import java.util.List;

/**
 * 识别事件然后删除
 *
 * @author Carl
 * @version 创建时间：2017/11/29
 */
public class TGTCreateEventListener {
    private TriggerLogoutService logoutService;
    private IUserIdObtainService service;

    public TGTCreateEventListener(@NotNull TriggerLogoutService logoutService, @NotNull IUserIdObtainService service) {
        this.logoutService = logoutService;
        this.service = service;
    }

    @EventListener
    @Async
    public void onTgtCreateEvent(CasTicketGrantingTicketCreatedEvent event) {
        TicketGrantingTicket ticketGrantingTicket = event.getTicketGrantingTicket();
        String id = ticketGrantingTicket.getAuthentication().getPrincipal().getId();
        String tgt = ticketGrantingTicket.getId();
        String clientName = (String) ticketGrantingTicket.getAuthentication().getAttributes().get("clientName");
        //获取可以认证的id
        List<String> authIds = service.obtain(clientName, id);
        if (authIds != null) {
            //循环触发登出
            authIds.forEach(authId -> logoutService.triggerLogout(authId, tgt));
        }
    }
}

```

## 剔除过滤用户

根据用户id，tgt，筛选出用户，并剔除

TriggerLogoutService.java
```java
/*
 * 版权所有.(c)2008-2017. 卡尔科技工作室
 */



package com.carl.sso.support.single.service;

import org.apereo.cas.CentralAuthenticationService;
import org.apereo.cas.authentication.Authentication;
import org.apereo.cas.ticket.Ticket;
import org.apereo.cas.ticket.TicketGrantingTicket;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Collection;

/**
 * 登出触发器
 *
 * @author Carl
 * @date 2017/11/29
 */
public class TriggerLogoutService {
    private static final Logger LOGGER = LoggerFactory.getLogger(TriggerLogoutService.class);
    private CentralAuthenticationService service;

    public TriggerLogoutService(CentralAuthenticationService service) {
        this.service = service;
    }

    /**
     * 触发其他用户退出
     *
     * @param id  用户id
     * @param tgt 当前登录的tgt
     */
    public void triggerLogout(String id, String tgt) {
        //找出用户id，并且不为当前tgt的，这里应当考虑数据性能，直接筛选用户再筛选tgt
        Collection<Ticket> tickets = this.service.getTickets(ticket -> {
            if(ticket instanceof TicketGrantingTicket) {
                TicketGrantingTicket t = ((TicketGrantingTicket)ticket).getRoot();
                Authentication authentication = t.getAuthentication();
                return t != null && authentication != null
                        && authentication.getPrincipal() != null && id.equals(authentication.getPrincipal().getId())
                        && !tgt.equals(t.getId());
            } else {
                return false;
            }

        });

        if (tickets != null && tickets.size() > 0) {
            LOGGER.info(String.format("[%s]强制强制注销%s", id, tickets.size()));
        }

        //发出注销
        for (Ticket ticket : tickets) {
            service.destroyTicketGrantingTicket(ticket.getId());
        }
    }
}

```

## 获取用户id

UserIdObtainServiceImpl.java
```java
/*
 * 版权所有.(c)2008-2017. 卡尔科技工作室
 */

package com.carl.sso.support.single.service;


import java.util.ArrayList;
import java.util.List;

/**
 * @author Carl
 * @version 创建时间：2017/11/29
 */
public class UserIdObtainServiceImpl implements IUserIdObtainService {

    public UserIdObtainServiceImpl() {

    }

    @Override
    public List<String> obtain(String clientName, String id) {
        //由于这里目前只做测试所以只返回当前的id，在正常的情况逻辑应该如下

        //根据校验client以及登录的id找到其他同一个用户的所有校验id返回，如通过邮箱登录的id，通过github登录的id等等
        List<String> ids = new ArrayList<>();
        ids.add(id);
        return ids;
    }
}

```


## spring配置注册
SingleLogoutTriggerConfiguration.java
```java
/*
 * 版权所有.(c)2008-2017. 卡尔科技工作室
 */


package com.carl.sso.support.single.config;

import com.carl.sso.support.single.listener.TGTCreateEventListener;
import com.carl.sso.support.single.service.TriggerLogoutService;
import com.carl.sso.support.single.service.UserIdObtainServiceImpl;
import org.apereo.cas.CentralAuthenticationService;
import org.apereo.cas.configuration.CasConfigurationProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 登出配置
 *
 * @author Carl
 * @date 2017/11/29
 */
@Configuration("singleLogoutTriggerConfiguration")
@EnableConfigurationProperties(CasConfigurationProperties.class)
public class SingleLogoutTriggerConfiguration {
    @Autowired
    private CentralAuthenticationService centralAuthenticationService;

    /**
     * 触发登出服务
     *
     * @return 触发登出服务
     */
    @Bean
    protected TriggerLogoutService triggerLogoutService() {
        return new TriggerLogoutService(centralAuthenticationService);
    }

    @Bean
    //注册事件监听tgt的创建
    protected TGTCreateEventListener tgtCreateEventListener() {
        TGTCreateEventListener listener = new TGTCreateEventListener(triggerLogoutService(), new UserIdObtainServiceImpl());
        return listener;
    }
}

```

# 测试

代码提交可以参考[github提交](https://github.com/kawhii/sso/commit/437272b6dc64428c4b9e7f53369616b81182b7e7)

测试流程：
>在chrome浏览器登陆，然后在IE浏览器登陆同样的账号,chrome浏览器的用户已登出

下载代码尝试：[![GitHub](https://img.shields.io/badge/downloads-v1.7.0=RC1-brightgreen.svg)](https://github.com/kawhii/sso/releases/tag/1.7.0-RC1) 其他版本可以到[GitHub](https://github.com/kawhii/sso/releases/tag/1.7.0-RC1)或者[码云](https://gitee.com/Kawhi-Carl/sso/tree/1.6.7-RC1)查看


`发现一些意外的事情可以考虑翻翻前面的博客进行学习哦`

# 作者联系方式

如果技术的交流或者疑问可以联系或者提出issue。

邮箱：huang.wenbin@foxmail.com

QQ: 756884434 (请注明：SSO-CSDN)

[//]: <> (> 如果项目对你有技术上的提升、工作上的帮助或者一些启示，不妨请小编喝杯咖啡，小编更会满怀激情的为大家讲解和输出博文哦。)

[//]: <> (微信<img src="http://img.blog.csdn.net/20170908092906735?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDQ3NTA0MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="230" height="230"></img>支付宝<img src="http://img.blog.csdn.net/20170908100804669?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDQ3NTA0MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="230" height="230"></img>)


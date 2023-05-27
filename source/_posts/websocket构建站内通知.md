---
title: websoket构建站内通知   
date: 2019-04-10 20:05:32   
categories: "websocket"  
tags: [websocket]    
description: websoket构建站内通知

---

****背景****     
传统的报表或者统计展示都是统计一段时间的数据，然后由网页发起接口调用进而展示数据，对于一些不需要实时展示的内容来说是可以，但对于一些需要即时看到并处理的信息则可能会延后。


****原因****    

网页端定时获取数据可能会造成信息的滞后，对于一些需要立即处理的事件可能会延误。


****解决思路****

不再使用前端定时调用接口的形式获取数据，使用websocket来即时从后台推送数据到前端。



1. 使用主流长连接框架netty实现websocket功能。
2. 使用jJavaEE7支持的@ServerEndpoint注解实现websocket。


若使用netty实现，性能会较好，但需要新启端口用于netty的连接与端口监听（默认为8888），使用注解实现则可以共用web 8080端口，减轻了实施的工作量，企业用户也无需新开防火墙申请变更，因此使用@ServerEndpoint实现websocket达到实时发送信息的目的。（在之前的章节提到了如何使用netty来做心跳检测以及信息传递）


****详细实现****    
	    

	
1.在类上使用注解@ServerEndpoint，并指定编解码类，因为实际场景的数据不只是字符串那么简单，一般都是json对象。还需实现  onOpen onClose onError三个方法，用于处理打开，关闭，异常时的处理场景。 同时onMessage方法用户处理客户端向服务端发送信息时的处理逻辑。       
 
   
    import net.sf.json.JSONObject;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import java.io.IOException;
    
    import javax.websocket.*;
    import javax.websocket.server.ServerEndpoint;
    
    @ServerEndpoint(value = "/websocket", encoders = {MessageEncoder.class })
    public class SystemRemindSocketServer {

	    private Logger logger = LoggerFactory.getLogger(SystemRemindSocketServer.class);
	
	    @OnMessage
	    public void onMessage(String message, Session session) throws IOException, InterruptedException, EncodeException {
	        //暂时无需处理网页端发送的信息
	        //test
	        //        for(int i = 0;i<5;i++){
	        //            JSONObject data = new JSONObject();
	        //            data.put("icon","ts-component");
	        //            data.put("taskId",1386);
	        //            data.put("time","2019-03-26 14:31:30");
	        //            data.put("title","当时明月在,曾照彩云归.");
	        //            data.put("link","/");
	        //            data.put("template","balantflow.systemremind.message.flow");
	        //            session.getBasicRemote().sendText(data.toString());
	        //        }
	    }
	
	    @OnOpen
	    public void onOpen(Session session) {
	        String userId = session.getUserPrincipal().getName();
	        String sessionId = session.getId();
	        SystemRemindMessageManager.sessionMap.put(userId, session);
	
	    }
	
	    @OnClose
	    public void onClose(Session session) {
	        String userId = session.getUserPrincipal().getName();
	        SystemRemindMessageManager.sessionMap.remove(userId);
	    }
	
	    @OnError
	    public void onError(Throwable t) {
	        logger.error("web socket error ," + t.getMessage());
	    }
    }


  2.编解码类的实现，直接用jsonobject格式化即可   

    public class MessageEncoder implements Encoder.Text<SystemRemindMessageVo> {

	    @Override
	    public void destroy() {
	
	
	    }
	
	    @Override
	    public void init(EndpointConfig arg0) {
	
	
	    }
	
	    @Override
	    public String encode(SystemRemindMessageVo messagepojo) throws EncodeException {
	        try {
	            return JSONObject.fromObject(messagepojo).toString();
	        } catch (Exception e) {
	            return null;
	        }
	    }

    }

3.前端建立连接与数据处理    

	$(function(){
        //启动时建立连接
		var notifyws = new WebSocket("ws://localhost:8080/balantflow/websocket");
		notifyws.onopen = function(evt) { 
			 //console.log(evt);
		};

        //后台发送消息时的处理逻辑
		notifyws.onmessage = function(evt) {
			var newnotify = [];
			newnotify.push(JSON.parse(evt.data));
			updateInstant(newnotify);
		};

        //断开后重新连接
		notifyws.onclose = function(evt) {
			notifyws = new WebSocket("ws://localhost:8080/balantflow/websocket");
		};
    })

****补充说明****   
部署环境时要注意修改websocket配置，因为在用户机器上连接localhost:8080是没用的，要改为具体的websocket地址
        







​ 
​
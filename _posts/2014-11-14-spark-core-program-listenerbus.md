---
layout: post
title: Spark源码之LiveListenerBus学习
category: Spark
tags: [Spark,源码,LiveListenerBus,事件]
keywords: Spark,源码,LiveListenerBus
description: Spark中LiveListenerBus学习总结
---
> Spark通过监听事件来回调相应的方法来对作业的运行情况进行跟踪与显示。`LiveListenerBus`类是一个简单的异步处理事件类，设置极为精练。通过使用生产者-消费者模型，所有向LiveListenerBus注册的监听者获取event并进行处理。

### 1、初始化LiveListenerBus

LiveListenerBus在`SparkContext`中初始化时创建: `private[spark] val listenerBus = new LiveListenerBus`，实例化时会初始化消息队列的长度为`1000`，最主要是的创建了消费者线程，代码如下：

		private val listenerThread = new Thread("SparkListenerBus") {
     	  setDaemon(true) //守护线程
     	  override def run(): Unit = Utils.logUncaughtExceptions { //工具方法，打印传入方法的异常消息
       	 while (true) {
             eventLock.acquire() //判断是否队列中是否有消息
         	   // Atomically remove and process this event
         	   LiveListenerBus.this.synchronized {
           	  val event = eventQueue.poll
           	  if (event == SparkListenerShutdown) { //关闭消息事件
             	     // Get out of the while loop and shutdown the daemon thread
             	     return
           	  }
               Option(event).foreach(postToAll) //将event传给父类的postToAll方法         
         	  }
			}
     	  }
     	}

###2、SparkListenerBus接口

SparkListenerBus接口是LiveListenerBus的父类，监听者通过此类中的`addListener方法`向SparkListenerBus注册，上面代码中通过postToAll方法将取得的event传递给所有向listenerBus注册的监听者进行处理，postToAll方法：

		def postToAll(event: SparkListenerEvent) {
    		event match {
      		case stageSubmitted: SparkListenerStageSubmitted =>
      		  foreachListener(_.onStageSubmitted(stageSubmitted)) //通过foreachListener来调用所有监听者的onStageSubmitted方法来处理stageSubmitted事件
      							.
      							.
      							.
      		case SparkListenerShutdown => //所有监听者对关闭SparkListener的毒丸事件不做任何处理
    		}
  		}
	
`foreachListener方法`对所有注册的监听者调用相应的方法进行了处理，
	
	  private def foreachListener(f: SparkListener => Unit): Unit = {
   		 sparkListeners.foreach { listener => //sparkListeners包含所有监听者的集合
    	   try {
        	  f(listener)// 执行所有监听者的f方法来处理event
      	   } catch {
            case e: Exception =>
                logError(s"Listener ${Utils.getFormattedClassName(listener)} threw an exception", e)
          }
       }
     }



Spark中所有的的事件包括如下几种，全部继承SparkListenerEvent类：

		- SparkListenerStageSubmitted
		- SparkListenerStageCompleted
		- SparkListenerJobStart
		- SparkListenerJobEnd
		- SparkListenerTaskStart
		- SparkListenerTaskGettingResult
		- SparkListenerTaskEnd
		- SparkListenerEnvironmentUpdate
		- SparkListenerBlockManagerAdded
		- SparkListenerBlockManagerRemoved
		- SparkListenerUnpersistRDD
		- SparkListenerApplicationStart
		- SparkListenerApplicationEnd
		- SparkListenerExecutorMetricsUpdate
		- SparkListenerShutdown
		
根据事件的名称很容易知道每个event代表的意义，这里不在介绍。

###3、Spark中所有的监听者

在这里将所有的监听者及其所实现的方法列出，方便今后学习时查找，

- 所有的监听者都继承自`SparkListener`特质，SparkListener中的方法没有做任何处理，所有继承该特质的监听者，只需要实现自己所关心的事件处理方法即可，无需将所有的方法都实现。现将所有的方法列出如下：

		-   def onStageCompleted(stageCompleted: SparkListenerStageCompleted) { }
		-   def onStageSubmitted(stageSubmitted: SparkListenerStageSubmitted) { }
		-   def onTaskStart(taskStart: SparkListenerTaskStart) { }
		-   def onTaskGettingResult(taskGettingResult: SparkListenerTaskGettingResult) { }
		-   def onTaskEnd(taskEnd: SparkListenerTaskEnd) { }
		-   def onJobStart(jobStart: SparkListenerJobStart) { }
  		-   def onJobEnd(jobEnd: SparkListenerJobEnd) { }
		-   def onEnvironmentUpdate(environmentUpdate: SparkListenerEnvironmentUpdate) { }
		-   def onBlockManagerAdded(blockManagerAdded: SparkListenerBlockManagerAdded) { }
		-   def onBlockManagerRemoved(blockManagerRemoved: SparkListenerBlockManagerRemoved) { }
		-   def onUnpersistRDD(unpersistRDD: SparkListenerUnpersistRDD) { }
		-   def onApplicationStart(applicationStart: SparkListenerApplicationStart) { }
		-   def onApplicationEnd(applicationEnd: SparkListenerApplicationEnd) { }
		-   def onExecutorMetricsUpdate(executorMetricsUpdate: SparkListenerExecutorMetricsUpdate) { }
- Spark中所有的监听者
	
	- **ApplicationEventListener**
		
			* onApplicationStart   
 			* onApplicationEnd
 			* onEnvironmentUpdate

	- **JobLogger**
  			
   			* onStageSubmitted
   			* onStageCompleted
   			* onTaskEnd
   			* onJobEnd
   			* onJobStart

	- **StatsReportListener**
   
   			* onTaskEnd
   			* onStageCompleted

 	- **StorageStatusListener**

 			* onTaskEnd
   			* onUnpersistRDD
   			* onBlockManagerAdded
   			* onBlockManagerRemoved

	- **EnvironmentListener**

		    * onEnvironmentUpdate

	- **ExecutorsListener**

			* onTaskStart
   			* onTaskEnd

	- **JobProgressListener**

			* onStageCompleted
   		 	* onStageSubmitted
		  	* onTaskStart
		  	* onTaskGettingResult
  		  	* onTaskEnd
		  	* onExecutorMetricsUpdate
		  	* onEnvironmentUpdate
		  	* onBlockManagerAdded
		  	* onBlockManagerRemoved

	- **StorageListener**

		  	* onTaskEnd
		  	* onStageSubmitted
		  	* onStageCompleted
		  	* onUnpersistRDD
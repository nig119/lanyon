---
layout: post 
title: "Async Tasks at Instagram"
---

These are notes from pycon video about how Instagram usages Asynctask queues 

[slide](https://speakerdeck.com/pyconslides/messaging-at-scale-at-instagram-by-rick-branson)
[video](https://www.youtube.com/watch?v=E708csv4XgY)

## the problem of showing media in feed

> **Naive approach**

(O(∞)) : fetch all media from accounts you follow and sort them by creation time Limit 10
	
> **Fanout-on-write approach**

- maintained a list of media ids for each account in redis (for there feeds)
- every time a new photo is posted, its media id is added to all of its followers list 
- O(1) read cost, O(N) write (# of followers) ,  read:write  :: 100:1
- very heavy write cost but writes are way less than reads 
	
> **reliability problems of fanout-on-write:**
	
- database server fail 
- faster query response on web
- justin Bieber (millions of followers )

>**Async background task**

- task is submitted to a broker 
- many workers take task one by one, if worker fail, task gets resubmitted to broker 

>**Chained Task**

- each task work on 1 media item and add to one follower, task yield success tasks 
- benefits : 
	- each worker takes only few thousands and finishes in few seconds 
	- fine grained load balancing 
	- low penalty on worker failure 
	
>**Other Async tasks**

- cross posting, search indexing, spam analysis, account deletion, API hooks

`USED Gearman & Python ( simple task queue, horrible, bad persistence, single core, based on mysql) `
	
>**Celery task queue(python, Django support)**

- which broker ? Redis(memory limit) / Beanstalk (no replication) / RabbitMQ

>**RabbitMQ**

- 2 broker, mirrored 
- scaled out by adding more brokers 
- overprovisioned for load adjustment 

>Alerting (use Sensu) :alert on queue length threshold, ```rabitmqctl list_queue``` to sensu

>Graphing (use graphite and statsd) : celery hooks to get per-task sent/fail/success/retry 

>Round robin approach to selecting worker 

mean time : ~5ms , 4k tasks/sec
25k app threads publishing to these brokers 

2 types of workers, gevent and normal db processes 
gevent task - FB API, spam check 
normal task - account delete 

normal operation : workers is assigned a task, it ack about it to the broker, broker forgets about it, now worker run it 

> More Problems : 

1 broker assigns tasks in batches to some workers in that total batch completion task is monopolized by slow task
 higher concurrency and lower batch size didn't worked, 
 
		grouping slow and fast task worked 
		
2 task fail 

 		retry (60 sec) 
 
3 worker crashes still lose task 

		celery sends ack packet to broker only after completing the task 
 		on task timeout, should retry or not, (no way to choose, deal with it)
 
4 mirroring system creates redelivery , tasks runs twice 
 
 
>**different types of task in celery**

```
CELERY_QUEUE_CONFIG = {
	"default":{
		"slow_task",
	},
	"gevent":{
		"evented_task",
	},
	"fast" :{
		"fast_task",
	},
	"feed" : {
		"feed_delivery"
	}
}
```
	
data in these queues are serialized using pickle 



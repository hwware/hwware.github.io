---
layout: post
title:  "Redis Sentinel (3) Sentinel Network Connection and Tilt Mode"
date:   2020-03-03 12:58:29
categories: Redis
---

# Sentinel Nework Connection
Each Sentinel will have two connections with each masters and slaves, the Command Connection and Pub/Sub Connection. However it only have one Command connection with other Sentinels. The initialization of the command and pub/sub links as follows:

```
1.	/* Commands connection. */  
2.	    if (link->cc == NULL) {  
3.	        link->cc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);  
4.	        if (!link->cc->err && server.tls_replication &&  
5.	                (instanceLinkNegotiateTLS(link->cc) == C_ERR)) {  
6.	            sentinelEvent(LL_DEBUG,"-cmd-link-reconnection",ri,"%@ #Failed to initialize TLS");  
7.	            instanceLinkCloseConnection(link,link->cc);  
8.	        } else if (link->cc->err) {  
9.	            sentinelEvent(LL_DEBUG,"-cmd-link-reconnection",ri,"%@ #%s",  
10.	                link->cc->errstr);  
11.	            instanceLinkCloseConnection(link,link->cc);  
12.	        } else {  
13.	            link->pending_commands = 0;  
14.	            link->cc_conn_time = mstime();  
15.	            link->cc->data = link;  
16.	            redisAeAttach(server.el,link->cc);  
17.	            redisAsyncSetConnectCallback(link->cc,  
18.	                    sentinelLinkEstablishedCallback);  
19.	            redisAsyncSetDisconnectCallback(link->cc,  
20.	                    sentinelDisconnectCallback);  
21.	            sentinelSendAuthIfNeeded(ri,link->cc);  
22.	            sentinelSetClientName(ri,link->cc,"cmd");  
23.	  
24.	            /* Send a PING ASAP when reconnecting. */  
25.	            sentinelSendPing(ri);  
26.	        }  
27.	    }  
```
```
1.	/* Pub / Sub */  
2.	   if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && link->pc == NULL) {  
3.	       link->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);  
4.	       if (!link->pc->err && server.tls_replication &&  
5.	               (instanceLinkNegotiateTLS(link->pc) == C_ERR)) {  
6.	           sentinelEvent(LL_DEBUG,"-pubsub-link-reconnection",ri,"%@ #Failed to initialize TLS");  
7.	       } else if (link->pc->err) {  
8.	           sentinelEvent(LL_DEBUG,"-pubsub-link-reconnection",ri,"%@ #%s",  
9.	               link->pc->errstr);  
10.	           instanceLinkCloseConnection(link,link->pc);  
11.	       } else {  
12.	           int retval;  
13.	  
14.	           link->pc_conn_time = mstime();  
15.	           link->pc->data = link;  
16.	           redisAeAttach(server.el,link->pc);  
17.	           redisAsyncSetConnectCallback(link->pc,  
18.	                   sentinelLinkEstablishedCallback);  
19.	           redisAsyncSetDisconnectCallback(link->pc,  
20.	                   sentinelDisconnectCallback);  
21.	           sentinelSendAuthIfNeeded(ri,link->pc);  
22.	           sentinelSetClientName(ri,link->pc,"pubsub");  
23.	           /* Now we subscribe to the Sentinels "Hello" channel. */  
24.	           retval = redisAsyncCommand(link->pc,  
25.	               sentinelReceiveHelloMessages, ri, "%s %s",  
26.	               sentinelInstanceMapCommand(ri,"SUBSCRIBE"),  
27.	               SENTINEL_HELLO_CHANNEL);  
28.	           if (retval != C_OK) {  
29.	               /* If we can't subscribe, the Pub/Sub connection is useless 
30.	                * and we can simply disconnect it and try again. */  
31.	               instanceLinkCloseConnection(link,link->pc);  
32.	               return;  
33.	           }  
34.	       }  
35.	   } 
```

### The Purpose of Command Connection:
Used to send command to Master/Slaves/Sentinels, such as INFO, PING, SLAVEOF, is-master-down-by-addr etc.
### The Purpose of Pub/Sub Connection:
Used to refresh the acknowledge from other sentinels (since all other sentinels will receive message from current sentinel about master ip/port, other sentinels etc)


By default for every second, sentinel will send ping command to the monitored masters, slaves, and other sentinels, for every 10 secs it will send INFO command to monitored masters, slaves and sentinels. Then it will refresh the knowledge for the masters, slaves and sentinels, the function sentinelRefreshInstanceInfo defines the parsing and updating information for receiving the INFO command reply from other Redis instances.

For every 2 secs it will publish the Hello message into the monitored master and slave __sentinel__:hello channel. The format of the pub/sub message as shown below:

```
              __sentinel_:hello <sentinel_ip> <sentinel_port> <sentinel_running_id>              <sentinel_epoch> <master_name> <master_ip> <master_port> <master_epoch>
```

The Sentinel will refresh the knowledge of current master status from other sentinels monitoring the master, this part of code was defined in sentinelProcessHelloMessage. 

At the same time Sentinels will also send of is-master-down-by-addr to other sentinels, in order to ask other sentinel¡¯s opinion about the current master status and asking vote from other sentinel during leader election process (will be mentioned in Leader Election part) . Default time is 1 sec (not counting the failover process call for leader election).

```
1.	    if ((master->flags & SRI_S_DOWN) == 0) continue;  
2.	    if (ri->link->disconnected) continue;  
3.	    if (!(flags & SENTINEL_ASK_FORCED) &&  
4.	        mstime() - ri->last_master_down_reply_time < SENTINEL_ASK_PERIOD)  
5.	        continue;  
6.	  
7.	    /* Ask */  
8.	    ll2string(port,sizeof(port),master->addr->port);  
9.	    retval = redisAsyncCommand(ri->link->cc,  
10.	                sentinelReceiveIsMasterDownReply, ri,  
11.	                "%s is-master-down-by-addr %s %s %llu %s",  
12.	                sentinelInstanceMapCommand(ri,"SENTINEL"),  
13.	                master->addr->ip, port,  
14.	                sentinel.current_epoch,  
15.	                (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?  
16.	                sentinel.myid : "*");  
17.	    if (retval == C_OK) ri->link->pending_commands++;  
```

# Tilt Mode

Tilt mode was triggered because of two reasons:
1.	Server too busy eg. caused by Huge I/O load (memory, network, disk)
2.	Clock changed significantly to a previous time.
Sentinel will not do anything initiatively (no failover, no check master subjectively/objectively down), mentioned in the main eventloop part. But it still receives the information, refresh the info of the outside world and reply to the other sentinels. We can think it is a passive mode for sentinel.

```
1.	void sentinelCheckTiltCondition(void) {  
2.	    mstime_t now = mstime();  
3.	    mstime_t delta = now - sentinel.previous_time;  
4.	  
5.	    if (delta < 0 || delta > SENTINEL_TILT_TRIGGER) {  
6.	        sentinel.tilt = 1;  
7.	        sentinel.tilt_start_time = mstime();  
8.	        sentinelEvent(LL_WARNING,"+tilt",NULL,"#tilt mode entered");  
9.	    }  
10.	    sentinel.previous_time = mstime();  
11.	}  
```





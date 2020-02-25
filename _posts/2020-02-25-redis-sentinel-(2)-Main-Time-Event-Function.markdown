---
layout: post
title:  "Redis Sentinel (2) Main Time Event Function"
date:   2020-02-25 12:58:29
categories: Redis
---

# Main Time Event Function

In serverCron function, which is the main time event loop function running default in every 100ms, it calls sentinelTimer function, to handle all the sentinel operations, however, in sentinel mode, the server frequency will be added a specific randomization number in order to avoid the conflict of the time to vote with other sentinels during sentinel leader election process.

```
1.	void sentinelTimer(void) {  
2.	    sentinelCheckTiltCondition();  
3.	    sentinelHandleDictOfRedisInstances(sentinel.masters);  
4.	    sentinelRunPendingScripts();  
5.	    sentinelCollectTerminatedScripts();  
6.	    sentinelKillTimedoutScripts();  
7.	  
8.	    /* We continuously change the frequency of the Redis "timer interrupt" 
9.	     * in order to desynchronize every Sentinel from every other. 
10.	     * This non-determinism avoids that Sentinels started at the same time 
11.	     * exactly continue to stay synchronized asking to be voted at the 
12.	     * same time again and again (resulting in nobody likely winning the 
13.	     * election because of split brain voting). */  
14.	    server.hz = CONFIG_DEFAULT_HZ + rand() % CONFIG_DEFAULT_HZ;  
15.	} 
```
Sentinel Timer will do the following works:
1.	Check whether the sentinel server is in tilt mode(tilt mode will be explained later)
2.	Check the connection and share information with Masters, Slaves, other Sentinels, if Failover is needed, do the failover process.
3.	Checking the script status and run certain operations.
Here are the details about the work for each of the functions:
SentinelCheckTiltCondition function checks the sentinel whether is in tilt mode, tilt mode will be explained in later chapter.
SentinelHandleDictOfRedisInstances is a recursive function call doing the following jobs in sequence:
1.	It calls sentinelHandleDictOfRedisInstance to handle specific work on each type of Sentinel connection Instance. (Masters, Slaves, Sentinels)
2.	It recursively calls the SentinelHandleDictOfRedisInstances function to handle Slaves, Sentinels under the monitored Masters. 
3.	It reset the specific master instance with the promoted slave if the master fail over process is done.
```
4.	/* Perform scheduled operations for all the instances in the dictionary. 
5.	 * Recursively call the function against dictionaries of slaves. */  
6.	void sentinelHandleDictOfRedisInstances(dict *instances) {  
7.	    dictIterator *di;  
8.	    dictEntry *de;  
9.	    sentinelRedisInstance *switch_to_promoted = NULL;  
10.	  
11.	    /* There are a number of things we need to perform against every master. */  
12.	    di = dictGetIterator(instances);  
13.	    while((de = dictNext(di)) != NULL) {  
14.	        sentinelRedisInstance *ri = dictGetVal(de);  
15.	  
16.	        sentinelHandleRedisInstance(ri);  
17.	        if (ri->flags & SRI_MASTER) {  
18.	            sentinelHandleDictOfRedisInstances(ri->slaves);  
19.	            sentinelHandleDictOfRedisInstances(ri->sentinels);  
20.	            if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {  
21.	                switch_to_promoted = ri;  
22.	            }  
23.	        }  
24.	    }  
25.	    if (switch_to_promoted)  
26.	        sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);  
27.	    dictReleaseIterator(di);  
28.	}  
```
In sentinelHandleRedisInstance function, which defines the cron operation of sentinel, doing mainly two parts of jobs:
1.	It continuously checks the connection between masters, slaves, and other sentinels, and trying to reconnect if needed. Each Sentinel will have two connections with each masters and slaves, the Command Connection and Pub/Sub Connection. However it only have one Command connection with other Sentinels. The details will be explained in Network Connection Section.

2.	The second part of operation will be omitted when the server is in tilt mode, this part of operation, focusing on the initiatively check the status of masters and slaves, and doing the failover process if the master is down.

```
1.	/* Perform scheduled operations for the specified Redis instance. */  
2.	void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {  
3.	    /* ========== MONITORING HALF ============ */  
4.	    /* Every kind of instance */  
5.	    sentinelReconnectInstance(ri);  
6.	    sentinelSendPeriodicCommands(ri);  
7.	  
8.	    /* ============== ACTING HALF ============= */  
9.	    /* We don't proceed with the acting half if we are in TILT mode. 
10.	     * TILT happens when we find something odd with the time, like a 
11.	     * sudden change in the clock. */  
12.	    if (sentinel.tilt) {  
13.	        if (mstime()-sentinel.tilt_start_time < SENTINEL_TILT_PERIOD) return;  
14.	        sentinel.tilt = 0;  
15.	        sentinelEvent(LL_WARNING,"-tilt",NULL,"#tilt mode exited");  
16.	    }  
17.	  
18.	    /* Every kind of instance */  
19.	    sentinelCheckSubjectivelyDown(ri);  
20.	  
21.	    /* Masters and slaves */  
22.	    if (ri->flags & (SRI_MASTER|SRI_SLAVE)) {  
23.	        /* Nothing so far. */  
24.	    }  
25.	  
26.	    /* Only masters */  
27.	    if (ri->flags & SRI_MASTER) {  
28.	        sentinelCheckObjectivelyDown(ri);  
29.	        if (sentinelStartFailoverIfNeeded(ri))  
30.	            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);  
31.	        sentinelFailoverStateMachine(ri);  
32.	        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);  
33.	    }  
34.	}  
```

The function sentinelCheckSubjectivelyDown will check whether a specific redis instance (master, slave, sentinel) is subjectively down. Subjective down means one of following condition was met:
1.	If the instance did not get the actively ping reply for more than down\_after\_milliseconds time.
2.	If the instance report itself to be slave and the sentinel think it is master, also the role report time for this instance is longer than down after milliseconds plus two times info report time.
If one of these two conditions was meet, sentinel will set the Redis instance into S_DOWN state. 
S_down state means the Sentinel thinks this Redis instance is down, it doesn't ask other sentinels opinion which monitoring the same Redis instance yet.
``` 
1.	/* Is this instance down from our point of view? */  
2.	void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {  
3.	    mstime_t elapsed = 0;  
4.	  
5.	    if (ri->link->act_ping_time)  
6.	        elapsed = mstime() - ri->link->act_ping_time;  
7.	    else if (ri->link->disconnected)  
8.	        elapsed = mstime() - ri->link->last_avail_time;  
9.	  
10.	    .......  
11.	  
12.	    /* Update the SDOWN flag. We believe the instance is SDOWN if: 
13.	     * 
14.	     * 1) It is not replying. 
15.	     * 2) We believe it is a master, it reports to be a slave for enough time 
16.	     *    to meet the down_after_period, plus enough time to get two times 
17.	     *    INFO report from the instance. */  
18.	    if (elapsed > ri->down_after_period ||  
19.	        (ri->flags & SRI_MASTER &&  
20.	         ri->role_reported == SRI_SLAVE &&  
21.	         mstime() - ri->role_reported_time >  
22.	          (ri->down_after_period+SENTINEL_INFO_PERIOD*2)))  
23.	    {  
24.	        /* Is subjectively down */  
25.	        if ((ri->flags & SRI_S_DOWN) == 0) {  
26.	            sentinelEvent(LL_WARNING,"+sdown",ri,"%@");  
27.	            ri->s_down_since_time = mstime();  
28.	            ri->flags |= SRI_S_DOWN;  
29.	        }  
30.	    } else {  
31.	        /* Is subjectively up */  
32.	        if (ri->flags & SRI_S_DOWN) {  
33.	            sentinelEvent(LL_WARNING,"-sdown",ri,"%@");  
34.	            ri->flags &= ~(SRI_S_DOWN|SRI_SCRIPT_KILL_SENT);  
35.	        }  
36.	    }  
37.	} 
```
The function sentinelCheckObjectivelyDown function checks the vote for a specific master from other sentinel instances monitoring same master. It counts the vote and if it is larger or equal than the quorum number configured for this specific master, it will mark this master into O\_DOWN state. 
```
1.	/* Is this instance down according to the configured quorum? 
2.	 * 
3.	 * Note that ODOWN is a weak quorum, it only means that enough Sentinels 
4.	 * reported in a given time range that the instance was not reachable. 
5.	 * However messages can be delayed so there are no strong guarantees about 
6.	 * N instances agreeing at the same time about the down state. */  
7.	void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {  
8.	    dictIterator *di;  
9.	    dictEntry *de;  
10.	    unsigned int quorum = 0, odown = 0;  
11.	  
12.	    if (master->flags & SRI_S_DOWN) {  
13.	        /* Is down for enough sentinels? */  
14.	        quorum = 1; /* the current sentinel. */  
15.	        /* Count all the other sentinels. */  
16.	        di = dictGetIterator(master->sentinels);  
17.	        while((de = dictNext(di)) != NULL) {  
18.	            sentinelRedisInstance *ri = dictGetVal(de);  
19.	  
20.	            if (ri->flags & SRI_MASTER_DOWN) quorum++;  
21.	        }  
22.	        dictReleaseIterator(di);  
23.	        if (quorum >= master->quorum) odown = 1;  
24.	    }  
25.	  
26.	    /* Set the flag accordingly to the outcome. */  
27.	    if (odown) {  
28.	        if ((master->flags & SRI_O_DOWN) == 0) {  
29.	            sentinelEvent(LL_WARNING,"+odown",master,"%@ #quorum %d/%d",  
30.	                quorum, master->quorum);  
31.	            master->flags |= SRI_O_DOWN;  
32.	            master->o_down_since_time = mstime();  
33.	        }  
34.	    } else {  
35.	        if (master->flags & SRI_O_DOWN) {  
36.	            sentinelEvent(LL_WARNING,"-odown",master,"%@");  
37.	            master->flags &= ~SRI_O_DOWN;  
38.	        }  
39.	    }  
40.	} 
```

In function sentinelStartFailoverIfNeeded it checks If the master is in o\_down state, and it didn't do the failover again in 2*failover timeout time, it will open the flag SRI\_FAILOVER\_IN\_PROGRES and SENTINEL\_FAILOVER\_STATE\_WAIT\_START for starting the failover process.
```
1.	int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {  
2.	    /* We can't failover if the master is not in O_DOWN state. */  
3.	    if (!(master->flags & SRI_O_DOWN)) return 0;  
4.	  
5.	    /* Failover already in progress? */  
6.	    if (master->flags & SRI_FAILOVER_IN_PROGRESS) return 0;  
7.	  
8.	    /* Last failover attempt started too little time ago? */  
9.	    if (mstime() - master->failover_start_time <  
10.	        master->failover_timeout*2)  
11.	    {  
12.	        if (master->failover_delay_logged != master->failover_start_time) {  
13.	            time_t clock = (master->failover_start_time +  
14.	                            master->failover_timeout*2) / 1000;  
15.	            char ctimebuf[26];  
16.	  
17.	            ctime_r(&clock,ctimebuf);  
18.	            ctimebuf[24] = '\0'; /* Remove newline. */  
19.	            master->failover_delay_logged = master->failover_start_time;  
20.	            serverLog(LL_WARNING,  
21.	                "Next failover delay: I will not start a failover before %s",  
22.	                ctimebuf);  
23.	        }  
24.	        return 0;  
25.	    }  
26.	  
27.	    sentinelStartFailover(master);  
28.	    return 1;  
29.	} 
```


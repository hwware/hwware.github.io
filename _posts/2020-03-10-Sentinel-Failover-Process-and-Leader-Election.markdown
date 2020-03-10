---
layout: post
title:  "Redis Sentinel (4) Leader Election and Failover Process"
date:   2020-03-10 12:58:29
categories: Redis
---
# Leader Election


Sentinel will use is-master-down-by-addr for getting votes from other sentinels, when the leader election is happening. Each sentinel will have chance to become leader to lead the failover process.

```
18.	    if ((master->flags & SRI_S_DOWN) == 0) continue;  
19.	    if (ri->link->disconnected) continue;  
20.	    if (!(flags & SENTINEL_ASK_FORCED) &&  
21.	        mstime() - ri->last_master_down_reply_time < SENTINEL_ASK_PERIOD)  
22.	        continue;  
23.	  
24.	    /* Ask */  
25.	    ll2string(port,sizeof(port),master->addr->port);  
26.	    retval = redisAsyncCommand(ri->link->cc,  
27.	                sentinelReceiveIsMasterDownReply, ri,  
28.	                "%s is-master-down-by-addr %s %s %llu %s",  
29.	                sentinelInstanceMapCommand(ri,"SENTINEL"),  
30.	                master->addr->ip, port,  
31.	                sentinel.current_epoch,  
32.	                (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?  
33.	                sentinel.myid : "*");  
34.	    if (retval == C_OK) ri->link->pending_commands++;  
35.	}  
```
The call: is-master-down-by-addr:  <master_ip> <master_port> <current_epoch> <runid>the runid will be sentinel id if it is seeking for votes.

If Sentinel receive other Sentinel requests, it will vote if the following condition is true, 
1.	Sentinel¡¯s epoch is smaller than or equal to the new vote¡¯s sentinel epoch
2.	This is the first Sentinel request it received in current epoch 

```
3.	/* Vote for the master (or fetch the previous vote) if the request 
4.	        * includes a runid, otherwise the sender is not seeking for a vote. */  
5.	       if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*")) {  
6.	           leader = sentinelVoteLeader(ri,(uint64_t)req_epoch,  
7.	                                           c->argv[5]->ptr,  
8.	                                           &leader_epoch);  
9.	       }  
```
The sentinel receives other votes from other sentinel, it will parse and get the field of the is-master-down-by-addr and remember the vote.

```
1.	              /* Ignore every error or unexpected reply. 
2.	    * Note that if the command returns an error for any reason we'll 
3.	    * end clearing the SRI_MASTER_DOWN flag for timeout anyway. */  
4.	   if (r->type == REDIS_REPLY_ARRAY && r->elements == 3 &&  
5.	       r->element[0]->type == REDIS_REPLY_INTEGER &&  
6.	       r->element[1]->type == REDIS_REPLY_STRING &&  
7.	       r->element[2]->type == REDIS_REPLY_INTEGER)  
8.	   {  
9.	       ri->last_master_down_reply_time = mstime();  
10.	       if (r->element[0]->integer == 1) {  
11.	           ri->flags |= SRI_MASTER_DOWN;  
12.	       } else {  
13.	           ri->flags &= ~SRI_MASTER_DOWN;  
14.	       }  
15.	       if (strcmp(r->element[1]->str,"*")) {  
16.	           /* If the runid in the reply is not "*" the Sentinel actually 
17.	            * replied with a vote. */  
18.	           sdsfree(ri->leader);  
19.	           if ((long long)ri->leader_epoch != r->element[2]->integer)  
20.	               serverLog(LL_WARNING,  
21.	                   "%s voted for %s %llu", ri->name,  
22.	                   r->element[1]->str,  
23.	                   (unsigned long long) r->element[2]->integer);  
24.	           ri->leader = sdsnew(r->element[1]->str);  
25.	           ri->leader_epoch = r->element[2]->integer;  
26.	       }  
27.	   } 
```
The sentinel will check the vote result and if the following is true, it will be elected as leader to do the failover.
1.	Majority of sentinel votes.
2.	The vote number reach quorum number. (if quorum number is smaller than 50%+1, it will use %50+1 as the limit)
If this doesn't happen, the vote will start again.

```
1.	char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch) {  
2.	    dict *counters;   
3.	    dictIterator *di;  
4.	    dictEntry *de;  
5.	    unsigned int voters = 0, voters_quorum;  
6.	    char *myvote;  
7.	    char *winner = NULL;  
8.	    uint64_t leader_epoch;  
9.	    uint64_t max_votes = 0;  
10.	  
11.	    serverAssert(master->flags & (SRI_O_DOWN|SRI_FAILOVER_IN_PROGRESS));  
12.	    counters = dictCreate(&leaderVotesDictType,NULL);  
13.	  
14.	    voters = dictSize(master->sentinels)+1; /* All the other sentinels and me.*/  
15.	  
16.	    /* Count other sentinels votes */  
17.	    di = dictGetIterator(master->sentinels);  
18.	    while((de = dictNext(di)) != NULL) {  
19.	        sentinelRedisInstance *ri = dictGetVal(de);  
20.	        if (ri->leader != NULL && ri->leader_epoch == sentinel.current_epoch)  
21.	            sentinelLeaderIncr(counters,ri->leader);  
22.	    }  
23.	    dictReleaseIterator(di);  
24.	  
25.	    /* Check what's the winner. For the winner to win, it needs two conditions: 
26.	     * 1) Absolute majority between voters (50% + 1). 
27.	     * 2) And anyway at least master->quorum votes. */  
28.	    di = dictGetIterator(counters);  
29.	    while((de = dictNext(di)) != NULL) {  
30.	        uint64_t votes = dictGetUnsignedIntegerVal(de);  
31.	  
32.	        if (votes > max_votes) {  
33.	            max_votes = votes;  
34.	            winner = dictGetKey(de);  
35.	        }  
36.	    }  
37.	    dictReleaseIterator(di);  
38.	  
39.	    /* Count this Sentinel vote: 
40.	     * if this Sentinel did not voted yet, either vote for the most 
41.	     * common voted sentinel, or for itself if no vote exists at all. */  
42.	    if (winner)  
43.	        myvote = sentinelVoteLeader(master,epoch,winner,&leader_epoch);  
44.	    else  
45.	        myvote = sentinelVoteLeader(master,epoch,sentinel.myid,&leader_epoch);  
46.	  
47.	    if (myvote && leader_epoch == epoch) {  
48.	        uint64_t votes = sentinelLeaderIncr(counters,myvote);  
49.	  
50.	        if (votes > max_votes) {  
51.	            max_votes = votes;  
52.	            winner = myvote;  
53.	        }  
54.	    }  
55.	  
56.	    voters_quorum = voters/2+1;  
57.	    if (winner && (max_votes < voters_quorum || max_votes < master->quorum))  
58.	        winner = NULL;  
59.	  
60.	    winner = winner ? sdsnew(winner) : NULL;  
61.	    sdsfree(myvote);  
62.	    dictRelease(counters);  
63.	    return winner;  
64.	}
```

# Failover Process

Master fail over process was defined in function sentinelFailoverStateMachine. 

```
1.	void sentinelFailoverStateMachine(sentinelRedisInstance *ri) {  
2.	    serverAssert(ri->flags & SRI_MASTER);  
3.	  
4.	    if (!(ri->flags & SRI_FAILOVER_IN_PROGRESS)) return;  
5.	  
6.	    switch(ri->failover_state) {  
7.	        case SENTINEL_FAILOVER_STATE_WAIT_START:  
8.	            sentinelFailoverWaitStart(ri);  
9.	            break;  
10.	        case SENTINEL_FAILOVER_STATE_SELECT_SLAVE:  
11.	            sentinelFailoverSelectSlave(ri);  
12.	            break;  
13.	        case SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE:  
14.	            sentinelFailoverSendSlaveOfNoOne(ri);  
15.	            break;  
16.	        case SENTINEL_FAILOVER_STATE_WAIT_PROMOTION:  
17.	            sentinelFailoverWaitPromotion(ri);  
18.	            break;  
19.	        case SENTINEL_FAILOVER_STATE_RECONF_SLAVES:  
20.	            sentinelFailoverReconfNextSlave(ri);  
21.	            break;  
22.	    }  
23.	} 
```
The sequence of the failover process as describe follows:

In SENTINEL_FAILOVER_STATE_WAIT_START state, Sentinels will count for the votes for leader election, if it was selected as the leader, it will proceed and change the status into SENTINEL_FAILOVER_STATE_SELECT_SLAVE, otherwise, it will abort failover after the election timeout.

```
1.	void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {  
2.	    char *leader;  
3.	    int isleader;  
4.	  
5.	    /* Check if we are the leader for the failover epoch. */  
6.	    leader = sentinelGetLeader(ri, ri->failover_epoch);  
7.	    isleader = leader && strcasecmp(leader,sentinel.myid) == 0;  
8.	    sdsfree(leader);  
9.	  
10.	    /* If I'm not the leader, and it is not a forced failover via 
11.	     * SENTINEL FAILOVER, then I can't continue with the failover. */  
12.	    if (!isleader && !(ri->flags & SRI_FORCE_FAILOVER)) {  
13.	        int election_timeout = SENTINEL_ELECTION_TIMEOUT;  
14.	  
15.	        /* The election timeout is the MIN between SENTINEL_ELECTION_TIMEOUT 
16.	         * and the configured failover timeout. */  
17.	        if (election_timeout > ri->failover_timeout)  
18.	            election_timeout = ri->failover_timeout;  
19.	        /* Abort the failover if I'm not the leader after some time. */  
20.	        if (mstime() - ri->failover_start_time > election_timeout) {  
21.	            sentinelEvent(LL_WARNING,"-failover-abort-not-elected",ri,"%@");  
22.	            sentinelAbortFailover(ri);  
23.	        }  
24.	        return;  
25.	    }  
26.	    sentinelEvent(LL_WARNING,"+elected-leader",ri,"%@");  
27.	    if (sentinel.simfailure_flags & SENTINEL_SIMFAILURE_CRASH_AFTER_ELECTION)  
28.	        sentinelSimFailureCrash();  
29.	    ri->failover_state = SENTINEL_FAILOVER_STATE_SELECT_SLAVE;  
30.	    ri->failover_state_change_time = mstime();  
31.	    sentinelEvent(LL_WARNING,"+failover-state-select-slave",ri,"%@");  
32.	}  
```
In SENTINEL_FAILOVER_STATE_SELECT_SLAVE state, the leader sentinel will select the best slaves to promote, the detailed algorithm was explained in the following function sentinelSelectSlave. 
```
1.	sentinelRedisInstance *sentinelSelectSlave(sentinelRedisInstance *master) {  
2.	    sentinelRedisInstance **instance =  
3.	        zmalloc(sizeof(instance[0])*dictSize(master->slaves));  
4.	    sentinelRedisInstance *selected = NULL;  
5.	    int instances = 0;  
6.	    dictIterator *di;  
7.	    dictEntry *de;  
8.	    mstime_t max_master_down_time = 0;  
9.	  
10.	    if (master->flags & SRI_S_DOWN)  
11.	        max_master_down_time += mstime() - master->s_down_since_time;  
12.	    max_master_down_time += master->down_after_period * 10;  
13.	  
14.	    di = dictGetIterator(master->slaves);  
15.	    while((de = dictNext(di)) != NULL) {  
16.	        sentinelRedisInstance *slave = dictGetVal(de);  
17.	        mstime_t info_validity_time;  
18.	  
19.	        if (slave->flags & (SRI_S_DOWN|SRI_O_DOWN)) continue;  
20.	        if (slave->link->disconnected) continue;  
21.	        if (mstime() - slave->link->last_avail_time > SENTINEL_PING_PERIOD*5) continue;  
22.	        if (slave->slave_priority == 0) continue;  
23.	  
24.	        /* If the master is in SDOWN state we get INFO for slaves every second. 
25.	         * Otherwise we get it with the usual period so we need to account for 
26.	         * a larger delay. */  
27.	        if (master->flags & SRI_S_DOWN)  
28.	            info_validity_time = SENTINEL_PING_PERIOD*5;  
29.	        else  
30.	            info_validity_time = SENTINEL_INFO_PERIOD*3;  
31.	        if (mstime() - slave->info_refresh > info_validity_time) continue;  
32.	        if (slave->master_link_down_time > max_master_down_time) continue;  
33.	        instance[instances++] = slave;  
34.	    }  
35.	    dictReleaseIterator(di);  
36.	    if (instances) {  
37.	        qsort(instance,instances,sizeof(sentinelRedisInstance*),  
38.	            compareSlavesForPromotion);  
39.	        selected = instance[0];  
40.	    }  
41.	    zfree(instance);  
42.	    return selected;  
43.	} 
```
The sequence of selecting good slave is:
1.	It filtered out all the slaves which is in S_DOWN, O_DOWN state.
2.	It filtered out all slaves with disconnected link.
3.	It filtered out all slaves which doesn¡¯t get back to Sentinel ping in 5 secs by default.
4.	It filtered out all slaves with 0 priority.
5.	It filtered out slaves with more than 5 sec info_refresh time when master is in S_DOWN state.
6.	It filtered out slaves which has master link down time more than master down time+10*down_after_period.

     Then all the slaves will be sorted for the following sequence:
1.	Slave Priority,
2.	Slave Replication offset.
3.	Slave Id.


The first slave will be promoted to the master, and the failover will proceed into      SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE status, if there is no slave available, the failover will be aborted.

In SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE state, the leader sentinel will send slaveof no one command to the promoted master, and the master status will change into SENTINEL_FAILOVER_STATE_WAIT_PROMOTION.


```
1.	void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {  
2.	    int retval;  
3.	  
4.	    /* We can't send the command to the promoted slave if it is now 
5.	     * disconnected. Retry again and again with this state until the timeout 
6.	     * is reached, then abort the failover. */  
7.	    if (ri->promoted_slave->link->disconnected) {  
8.	        if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {  
9.	            sentinelEvent(LL_WARNING,"-failover-abort-slave-timeout",ri,"%@");  
10.	            sentinelAbortFailover(ri);  
11.	        }  
12.	        return;  
13.	    }  
14.	  
15.	    /* Send SLAVEOF NO ONE command to turn the slave into a master. 
16.	     * We actually register a generic callback for this command as we don't 
17.	     * really care about the reply. We check if it worked indirectly observing 
18.	     * if INFO returns a different role (master instead of slave). */  
19.	    retval = sentinelSendSlaveOf(ri->promoted_slave,NULL,0);  
20.	    if (retval != C_OK) return;  
21.	    sentinelEvent(LL_NOTICE, "+failover-state-wait-promotion",  
22.	        ri->promoted_slave,"%@");  
23.	    ri->failover_state = SENTINEL_FAILOVER_STATE_WAIT_PROMOTION;  
24.	    ri->failover_state_change_time = mstime();  
25.	}  
```

In SENTINEL_FAILOVER_STATE_WAIT_PROMOTION state, the sentinel will continuously waiting for the promoted Slave report the role change through Info command reply, if this doesn't happen after failover-timeout, the failover process will be aborted.
```
4.	/* We actually wait for promotion indirectly checking with INFO when the 
5.	 * slave turns into a master. */  
6.	void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri) {  
7.	    /* Just handle the timeout. Switching to the next state is handled 
8.	     * by the function parsing the INFO command of the promoted slave. */  
9.	    if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {  
10.	        sentinelEvent(LL_WARNING,"-failover-abort-slave-timeout",ri,"%@");  
11.	        sentinelAbortFailover(ri);  
12.	    }  
13.	}  
```

 However if during the wait time, it receives the Info reply about the slave role change into master, it will change its master failover status into SENTINEL_FAILOVER_STATE_RECONF_SLAVES. 

```
1.	/* Handle slave -> master role switch. */  
2.	   if ((ri->flags & SRI_SLAVE) && role == SRI_MASTER) {  
3.	       /* If this is a promoted slave we can change state to the 
4.	        * failover state machine. */  
5.	       if ((ri->flags & SRI_PROMOTED) &&  
6.	           (ri->master->flags & SRI_FAILOVER_IN_PROGRESS) &&  
7.	           (ri->master->failover_state ==  
8.	               SENTINEL_FAILOVER_STATE_WAIT_PROMOTION))  
9.	       {  
10.	           /* Now that we are sure the slave was reconfigured as a master 
11.	            * set the master configuration epoch to the epoch we won the 
12.	            * election to perform this failover. This will force the other 
13.	            * Sentinels to update their config (assuming there is not 
14.	            * a newer one already available). */  
15.	           ri->master->config_epoch = ri->master->failover_epoch;  
16.	           ri->master->failover_state = SENTINEL_FAILOVER_STATE_RECONF_SLAVES;  
17.	           ri->master->failover_state_change_time = mstime();  
18.	           sentinelFlushConfig();  
19.	           sentinelEvent(LL_WARNING,"+promoted-slave",ri,"%@");  
20.	           if (sentinel.simfailure_flags &  
21.	               SENTINEL_SIMFAILURE_CRASH_AFTER_PROMOTION)  
22.	               sentinelSimFailureCrash();  
23.	           sentinelEvent(LL_WARNING,"+failover-state-reconf-slaves",  
24.	               ri->master,"%@");  
25.	           sentinelCallClientReconfScript(ri->master,SENTINEL_LEADER,  
26.	               "start",ri->master->addr,ri->addr);  
27.	           sentinelForceHelloUpdateForMaster(ri->master);  
28.	       } else {  
```
In SENTINEL_FAILOVER_STATE_RECONF_SLAVES state, leader sentinels will send Slaveof <promoted-slave-ip> <promoted-slave-port> to other slaves:
```
1.	/* Send SLAVE OF <new master address> to all the remaining slaves that 
2.	 * still don't appear to have the configuration updated. */  
3.	void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {  
4.	    dictIterator *di;  
5.	    dictEntry *de;  
6.	    int in_progress = 0;  
7.	  
8.	    di = dictGetIterator(master->slaves);  
9.	    while((de = dictNext(di)) != NULL) {  
10.	        sentinelRedisInstance *slave = dictGetVal(de);  
11.	  
12.	        if (slave->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG))  
13.	            in_progress++;  
14.	    }  
15.	    dictReleaseIterator(di);  
16.	  
17.	    di = dictGetIterator(master->slaves);  
18.	    while(in_progress < master->parallel_syncs &&  
19.	          (de = dictNext(di)) != NULL)  
20.	    {  
21.	        sentinelRedisInstance *slave = dictGetVal(de);  
22.	        int retval;  
23.	  
24.	        /* Skip the promoted slave, and already configured slaves. */  
25.	        if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE)) continue;  
26.	  
27.	        /* If too much time elapsed without the slave moving forward to 
28.	         * the next state, consider it reconfigured even if it is not. 
29.	         * Sentinels will detect the slave as misconfigured and fix its 
30.	         * configuration later. */  
31.	        if ((slave->flags & SRI_RECONF_SENT) &&  
32.	            (mstime() - slave->slave_reconf_sent_time) >  
33.	            SENTINEL_SLAVE_RECONF_TIMEOUT)  
34.	        {  
35.	            sentinelEvent(LL_NOTICE,"-slave-reconf-sent-timeout",slave,"%@");  
36.	            slave->flags &= ~SRI_RECONF_SENT;  
37.	            slave->flags |= SRI_RECONF_DONE;  
38.	        }  
39.	  
40.	        /* Nothing to do for instances that are disconnected or already 
41.	         * in RECONF_SENT state. */  
42.	        if (slave->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG)) continue;  
43.	        if (slave->link->disconnected) continue;  
44.	  
45.	        /* Send SLAVEOF <new master>. */  
46.	        retval = sentinelSendSlaveOf(slave,  
47.	                master->promoted_slave->addr->ip,  
48.	                master->promoted_slave->addr->port);  
49.	        if (retval == C_OK) {  
50.	            slave->flags |= SRI_RECONF_SENT;  
51.	            slave->slave_reconf_sent_time = mstime();  
52.	            sentinelEvent(LL_NOTICE,"+slave-reconf-sent",slave,"%@");  
53.	            in_progress++;  
54.	        }  
55.	    }  
56.	    dictReleaseIterator(di);  
57.	  
58.	    /* Check if all the slaves are reconfigured and handle timeout. */  
59.	    sentinelFailoverDetectEnd(master);  
60.	} 
```
The failover end only if all the other healthy slave successfully received and configured the new promoted slave as replicated master, then the failover master will be in SENTINEL_FAILOVER_STATE_UPDATE_CONFIG state.
```
1.	void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {  
2.	    int not_reconfigured = 0, timeout = 0;  
3.	    dictIterator *di;  
4.	    dictEntry *de;  
5.	    mstime_t elapsed = mstime() - master->failover_state_change_time;  
6.	  
7.	    /* We can't consider failover finished if the promoted slave is 
8.	     * not reachable. */  
9.	    if (master->promoted_slave == NULL ||  
10.	        master->promoted_slave->flags & SRI_S_DOWN) return;  
11.	  
12.	    /* The failover terminates once all the reachable slaves are properly 
13.	     * configured. */  
14.	    di = dictGetIterator(master->slaves);  
15.	    while((de = dictNext(di)) != NULL) {  
16.	        sentinelRedisInstance *slave = dictGetVal(de);  
17.	  
18.	        if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE)) continue;  
19.	        if (slave->flags & SRI_S_DOWN) continue;  
20.	        not_reconfigured++;  
21.	    }  
22.	    dictReleaseIterator(di);  
23.	  
24.	    /* Force end of failover on timeout. */  
25.	    if (elapsed > master->failover_timeout) {  
26.	        not_reconfigured = 0;  
27.	        timeout = 1;  
28.	        sentinelEvent(LL_WARNING,"+failover-end-for-timeout",master,"%@");  
29.	    }  
30.	  
31.	    if (not_reconfigured == 0) {  
32.	        sentinelEvent(LL_WARNING,"+failover-end",master,"%@");  
33.	        master->failover_state = SENTINEL_FAILOVER_STATE_UPDATE_CONFIG;  
34.	        master->failover_state_change_time = mstime();  
35.	    }  
36.	  
37.	    /* If I'm the leader it is a good idea to send a best effort SLAVEOF 
38.	     * command to all the slaves still not reconfigured to replicate with 
39.	     * the new master. */  
40.	    if (timeout) {  
41.	        dictIterator *di;  
42.	        dictEntry *de;  
43.	  
44.	        di = dictGetIterator(master->slaves);  
45.	        while((de = dictNext(di)) != NULL) {  
46.	            sentinelRedisInstance *slave = dictGetVal(de);  
47.	            int retval;  
48.	  
49.	            if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE|SRI_RECONF_SENT)) continue;  
50.	            if (slave->link->disconnected) continue;  
51.	  
52.	            retval = sentinelSendSlaveOf(slave,  
53.	                    master->promoted_slave->addr->ip,  
54.	                    master->promoted_slave->addr->port);  
55.	            if (retval == C_OK) {  
56.	                sentinelEvent(LL_NOTICE,"+slave-reconf-sent-be",slave,"%@");  
57.	                slave->flags |= SRI_RECONF_SENT;  
58.	            }  
59.	        }  
60.	        dictReleaseIterator(di);  
61.	    }  
62.	} 
```
At last, when monitored master is in SENTINEL_FAILOVER_STATE_UPDATE_CONFIG state, Sentinel will reconfig master with promoted slave and make failed master as a slave of the new master. Then the failover process ends. 
```
1.	   if (ri->flags & SRI_MASTER) {  
2.	        sentinelHandleDictOfRedisInstances(ri->slaves);  
3.	        sentinelHandleDictOfRedisInstances(ri->sentinels);  
4.	        if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {  
5.	            switch_to_promoted = ri;  
6.	        }  
7.	    }  
8.	}  
9.	if (switch_to_promoted)  
10.	    sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);  
11.	dictReleaseIterator(di);  
```
The following graph describes the whole failover process.













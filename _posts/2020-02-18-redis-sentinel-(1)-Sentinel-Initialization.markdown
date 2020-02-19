---
layout: post
title:  "Redis Sentinel (1) Sentinel Initialization"
date:   2020-02-18 12:58:29
categories: Redis
---

# Initialization

Sentinel shares the same event loop structure with Redis Core, however, it uses its unique initialization step, in the main function of Redis in server.c, and we can find the following code snippets:

```
1.	int main(int argc, char **argv) {  
2.	    struct timeval tv;  
3.	    int j;      
4.	  
5.	    ............  
6.	/* We need to init sentinel right now as parsing the configuration file 
7.	     * in sentinel mode will have the effect of populating the sentinel 
8.	     * data structures with master nodes to monitor. */  
9.	    if (server.sentinel_mode) {  
10.	        initSentinelConfig();  
11.	        initSentinel();  
12.	    }  

```

The initSentinelConfig function initializes the server configuration with sentinel specific port, which default is 26379, also it set the server protection mode to false, since if Redis server running in sentinel mode it must be exposed. 

```
1.	/* This function overwrites a few normal Redis config default with Sentinel 
2.	 * specific defaults. */  
3.	void initSentinelConfig(void) {  
4.	    server.port = REDIS_SENTINEL_PORT;  
5.	    server.protected_mode = 0; /* Sentinel must be exposed. */  
6.	}  
```
At the meantime, the initSentinel function did the following two things: 
1.	It tries to replace the sever support command with sentinel specific one, which defines in sentinel.c. The command that sentinel supports are ping, sentinel, subscribe, unsubscribe, psubscribe, punsubscribe, publish, info, role, client, shutdown, auth and hello, as describe in the following code sentinelcmds list of redisCommand struct. 
2.	It initializes the sentinel main state instance.

```
1.	/* Main state. */  
2.	struct sentinelState {  
3.	    char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */  
4.	    uint64_t current_epoch;         /* Current epoch. */  
5.	    dict *masters;      /* Dictionary of master sentinelRedisInstances. 
6.	                           Key is the instance name, value is the 
7.	                           sentinelRedisInstance structure pointer. */  
8.	    int tilt;           /* Are we in TILT mode? */  
9.	    ...........  
10.	} sentinel;  
11.	  
12.	struct redisCommand sentinelcmds[] = {  
13.	    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},  
14.	    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},  
15.	    {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},  
16.	    {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},  
17.	    {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},  
18.	    {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},  
19.	    {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},  
20.	    {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},  
21.	    {"role",sentinelRoleCommand,1,"ok-loading",0,NULL,0,0,0,0,0},  
22.	    {"client",clientCommand,-2,"read-only no-script",0,NULL,0,0,0,0,0},  
23.	    {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0},  
24.	    {"auth",authCommand,2,"no-auth no-script ok-loading ok-stale fast",0,NULL,0,0,0,0,0},  
25.	    {"hello",helloCommand,-2,"no-auth no-script fast",0,NULL,0,0,0,0,0}  
26.	};  
27.	.......  
28.	  
29.	/* Perform the Sentinel mode initialization. */  
30.	void initSentinel(void) {  
31.	    unsigned int j;  
32.	  
33.	    /* Remove usual Redis commands from the command table, then just add 
34.	     * the SENTINEL command. */  
35.	    dictEmpty(server.commands,NULL);  
36.	    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {  
37.	        int retval;  
38.	        struct redisCommand *cmd = sentinelcmds+j;  
39.	  
40.	        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);  
41.	        serverAssert(retval == DICT_OK);  
42.	  
43.	        /* Translate the command string flags description into an actual 
44.	         * set of flags. */  
45.	        if (populateCommandTableParseFlags(cmd,cmd->sflags) == C_ERR)  
46.	            serverPanic("Unsupported command flag");  
47.	    }  
48.	  
49.	    /* Initialize various data structures. */  
50.	    sentinel.current_epoch = 0;  
51.	    sentinel.masters = dictCreate(&instancesDictType,NULL);  
52.	    sentinel.tilt = 0;  
53.	    sentinel.tilt_start_time = 0;  
54.	    sentinel.previous_time = mstime();  
55.	    .............  
56.	}  
```
After reading and updating the config information, the main function of Redis server calls sentinelIsRunning function, which does the following things in sequence:
1.	It checks whether the config file was set to the server, also checks whether it has proper write permission since Sentinel needs to continually write the state updates into the config file. 
2.	 Assign Sentinel Running ID if it doesn¡¯t set in the config file.
3.	Generate all the monitoring events for the master sentinel set to monitor in the config file.

```
4.	/* This function gets called when the server is in Sentinel mode, started, 
5.	 * loaded the configuration, and is ready for normal operations. */  
6.	void sentinelIsRunning(void) {  
7.	    int j;  
8.	  
9.	    if (server.configfile == NULL) {  
10.	        serverLog(LL_WARNING,  
11.	            "Sentinel started without a config file. Exiting...");  
12.	        exit(1);  
13.	    } else if (access(server.configfile,W_OK) == -1) {  
14.	        serverLog(LL_WARNING,  
15.	            "Sentinel config file %s is not writable: %s. Exiting...",  
16.	            server.configfile,strerror(errno));  
17.	        exit(1);  
18.	    }  
19.	  
20.	    /* If this Sentinel has yet no ID set in the configuration file, we 
21.	     * pick a random one and persist the config on disk. From now on this 
22.	     * will be this Sentinel ID across restarts. */  
23.	    for (j = 0; j < CONFIG_RUN_ID_SIZE; j++)  
24.	        if (sentinel.myid[j] != 0) break;  
25.	  
26.	    if (j == CONFIG_RUN_ID_SIZE) {  
27.	        /* Pick ID and persist the config. */  
28.	        getRandomHexChars(sentinel.myid,CONFIG_RUN_ID_SIZE);  
29.	        sentinelFlushConfig();  
30.	    }  
31.	  
32.	    /* Log its ID to make debugging of issues simpler. */  
33.	    serverLog(LL_WARNING,"Sentinel ID is %s", sentinel.myid);  
34.	  
35.	    /* We want to generate a +monitor event for every configured master 
36.	     * at startup. */  
37.	    sentinelGenerateInitialMonitorEvents();  
38.	}  
```
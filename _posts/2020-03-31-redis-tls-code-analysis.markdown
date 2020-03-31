---
layout: post
title:  "Redis tls code analysis"
date:   2020-03-31 12:58:29
categories: Redis
---

# Redis TLS Code Analysis

TLS support is a new feature supported by Redis 6.0. In this article, the basic parts of implementation of how Redis support TLS will be explained.

## Prerequisite

Currently Redis internally using OpenSSL development libraries for TLS/SSL support implementation. Therefore, OpenSSL suite libraries need to be preinstalled before Redis compilation time, meanwhile, when compiling Redis source code linking OpenSSL libraries is needed.
Initialization

In config.c configs struct array, it defines the tls related configuration:

```
1.	#ifdef USE_OPENSSL  
2.	    createIntConfig("tls-port", NULL, IMMUTABLE_CONFIG, 0, 65535, server.tls_port, 0, INTEGER_CONFIG, NULL, NULL), /* TCP port. */  
3.	    createBoolConfig("tls-cluster", NULL, MODIFIABLE_CONFIG, server.tls_cluster, 0, NULL, NULL),  
4.	    createBoolConfig("tls-replication", NULL, MODIFIABLE_CONFIG, server.tls_replication, 0, NULL, NULL),  
5.	    createBoolConfig("tls-auth-clients", NULL, MODIFIABLE_CONFIG, server.tls_auth_clients, 1, NULL, NULL),  
6.	    createBoolConfig("tls-prefer-server-ciphers", NULL, MODIFIABLE_CONFIG, server.tls_ctx_config.prefer_server_ciphers, 0, NULL, updateTlsCfgBool),  
7.	    createStringConfig("tls-cert-file", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.cert_file, NULL, NULL, updateTlsCfg),  
8.	    createStringConfig("tls-key-file", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.key_file, NULL, NULL, updateTlsCfg),  
9.	    createStringConfig("tls-dh-params-file", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.dh_params_file, NULL, NULL, updateTlsCfg),  
10.	    createStringConfig("tls-ca-cert-file", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.ca_cert_file, NULL, NULL, updateTlsCfg),  
11.	    createStringConfig("tls-ca-cert-dir", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.ca_cert_dir, NULL, NULL, updateTlsCfg),  
12.	    createStringConfig("tls-protocols", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.protocols, NULL, NULL, updateTlsCfg),  
13.	    createStringConfig("tls-ciphers", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.ciphers, NULL, NULL, updateTlsCfg),  
14.	    createStringConfig("tls-ciphersuites", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL, server.tls_ctx_config.ciphersuites, NULL, NULL, updateTlsCfg),  
15.	#endif 
```

When server loading configs, tls related params will be loaded into corresponding fields in server struct. For example, tls-port will be loaded into server.tls\_port field.

In InitServer function of server.c, the provided tls params will be used for configuring params for OpenSSL library:

```
1.	if (server.tls_port && tlsConfigure(&server.tls_ctx_config) == C_ERR) {  
2.	    serverLog(LL_WARNING, "Failed to configure TLS. Check logs for more info.");  
3.	    exit(1);  
4.	}  
```

the detail of tlsConfigure function was defined in tls.c. The main job this function doing is creating new SSL context and set context configuration based on tls configuration parameters when server starts. Then saving SSL context into redis\_tls\_ctx global variable.

```
1.	int tlsConfigure(redisTLSContextConfig *ctx_config) {  
2.	    char errbuf[256];  
3.	    SSL_CTX *ctx = NULL;  
4.	  
5.	    if (!ctx_config->cert_file) {  
6.	        serverLog(LL_WARNING, "No tls-cert-file configured!");  
7.	        goto error;  
8.	    }  
9.	  
10.	    if (!ctx_config->key_file) {  
11.	        serverLog(LL_WARNING, "No tls-key-file configured!");  
12.	        goto error;  
13.	    }  
14.	  
15.	    if (!ctx_config->ca_cert_file && !ctx_config->ca_cert_dir) {  
16.	        serverLog(LL_WARNING, "Either tls-ca-cert-file or tls-ca-cert-dir must be configured!");  
17.	        goto error;  
18.	    }  
19.	  
20.	    ctx = SSL_CTX_new(SSLv23_method());  
21.	  
22.	    SSL_CTX_set_options(ctx, SSL_OP_NO_SSLv2|SSL_OP_NO_SSLv3);  
23.	    SSL_CTX_set_options(ctx, SSL_OP_SINGLE_DH_USE);  
24.	  
25.	#ifdef SSL_OP_DONT_INSERT_EMPTY_FRAGMENTS  
26.	    SSL_CTX_set_options(ctx, SSL_OP_DONT_INSERT_EMPTY_FRAGMENTS);  
27.	#endif  
28.	  
29.	 ............  
30.	  
31.	    SSL_CTX_set_mode(ctx, SSL_MODE_ENABLE_PARTIAL_WRITE|SSL_MODE_ACCEPT_MOVING_WRITE_BUFFER);  
32.	    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER|SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);  
33.	    SSL_CTX_set_ecdh_auto(ctx, 1);  
34.	  
35.	    if (SSL_CTX_use_certificate_file(ctx, ctx_config->cert_file, SSL_FILETYPE_PEM) <= 0) {  
36.	        ERR_error_string_n(ERR_get_error(), errbuf, sizeof(errbuf));  
37.	        serverLog(LL_WARNING, "Failed to load certificate: %s: %s", ctx_config->cert_file, errbuf);  
38.	        goto error;  
39.	    }  
40.	          
41.	    if (SSL_CTX_use_PrivateKey_file(ctx, ctx_config->key_file, SSL_FILETYPE_PEM) <= 0) {  
42.	        ERR_error_string_n(ERR_get_error(), errbuf, sizeof(errbuf));  
43.	        serverLog(LL_WARNING, "Failed to load private key: %s: %s", ctx_config->key_file, errbuf);  
44.	        goto error;  
45.	    }  
46.	      
47.	    if (SSL_CTX_load_verify_locations(ctx, ctx_config->ca_cert_file, ctx_config->ca_cert_dir) <= 0) {  
48.	        ERR_error_string_n(ERR_get_error(), errbuf, sizeof(errbuf));  
49.	        serverLog(LL_WARNING, "Failed to configure CA certificate(s) file/directory: %s", errbuf);  
50.	        goto error;  
51.	    }  
52.	  
53.	.................  
54.	  
55.	    SSL_CTX_free(redis_tls_ctx);  
56.	    redis_tls_ctx = ctx;  
57.	  
58.	    return C_OK;  
59.	  
60.	error:  
61.	    if (ctx) SSL_CTX_free(ctx);  
62.	    return C_ERR;  
63.	} 
```

## Connection Handling

### Accept

Redis using multiplexing IO architecture to handle file events. 
In initServer function, we can find following part for socket binding and listening on TLS port for Redis server.

```
1.	if (server.tls_port != 0 &&  
2.	    listenToPort(server.tls_port,server.tlsfd,&server.tlsfd_count) == C_ERR)  
3.	    exit(1); 
```

the listenToPort function is an encapsulation of socket bind and listen. After successfully calling it, it saves the file descriptor in server.tlsfd file descriptor array. After that, the AE\_READABLE file event was registered for the file descriptor in server.tlsfd. and setup the callback to acceptTLSHandler.

```
1.	for (j = 0; j < server.tlsfd_count; j++) {  
2.	    if (aeCreateFileEvent(server.el, server.tlsfd[j], AE_READABLE,  
3.	        acceptTLSHandler,NULL) == AE_ERR)  
4.	        {  
5.	            serverPanic(  
6.	                "Unrecoverable error creating server.tlsfd file event.");  
7.	        }  
8.	}  
```

The acceptTLSHHandler function as shows below:The acceptTLSHHandler function as shows below:

```
1.	void acceptTLSHandler(aeEventLoop *el, int fd, void *privdata, int mask) {  
2.	    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;  
3.	    char cip[NET_IP_STR_LEN];  
4.	    UNUSED(el);  
5.	    UNUSED(mask);  
6.	    UNUSED(privdata);  
7.	  
8.	    while(max--) {  
9.	        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);  
10.	        if (cfd == ANET_ERR) {  
11.	            if (errno != EWOULDBLOCK)  
12.	                serverLog(LL_WARNING,  
13.	                    "Accepting client connection: %s", server.neterr);  
14.	            return;  
15.	        }  
16.	        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);  
17.	        acceptCommonHandler(connCreateAcceptedTLS(cfd, server.tls_auth_clients),0,cip);  
18.	    }  
19.	}  
```

This function calls anetTcpAccept function to accept the socket connection, and use new created socket file descriptor pass into connCreateAcceptedTLS function. This function defines the critical part for TLS handling in Redis:

```
1.	connection *connCreateAcceptedTLS(int fd, int require_auth) {  
2.	    tls_connection *conn = (tls_connection *) connCreateTLS();  
3.	    conn->c.fd = fd;  
4.	    conn->c.state = CONN_STATE_ACCEPTING;  
5.	  
6.	    if (!require_auth) {  
7.	        /* We still verify certificates if provided, but don't require them. 
8.	         */  
9.	        SSL_set_verify(conn->ssl, SSL_VERIFY_PEER, NULL);  
10.	    }  
11.	  
12.	    SSL_set_fd(conn->ssl, conn->c.fd);  
13.	    SSL_set_accept_state(conn->ssl);  
14.	  
15.	    return (connection *) conn;  
16.	}  
```

Firstly, it initializes a tls\_connection struct reference. In tls.c, tls\_connection struct was defined as follows:

```
1.	typedef struct tls_connection {  
2.	    connection c;  
3.	    int flags;  
4.	    SSL *ssl;  
5.	    char *ssl_error;  
6.	    listNode *pending_list_node;  
7.	} tls_connection;  
```

The connection type defines the common socket connection params:

```
1.	struct connection {  
2.	    ConnectionType *type;  
3.	    ConnectionState state;  
4.	    short int flags;  
5.	    short int refs;  
6.	    int last_errno;  
7.	    void *private_data;  
8.	    ConnectionCallbackFunc conn_handler;  
9.	    ConnectionCallbackFunc write_handler;  
10.	    ConnectionCallbackFunc read_handler;  
11.	    int fd;  
12.	};  
```

Then the connCreateAcceptedTLS function saves the file descriptor into conn.fd field,  it also connect the SSL context with this file descriptor, the SSL\_set\_accept_state function initializes and examines the server mode of the ssl object. At last, the function returns the connection object. This connection object was used for passing into acceptCommonHandler function in order to create redis client object and calling connTLSAccept function to accepting the tls client connections:

```
1.	static int connTLSAccept(connection *_conn, ConnectionCallbackFunc accept_handler) {  
2.	    tls_connection *conn = (tls_connection *) _conn;  
3.	    int ret;  
4.	  
5.	    if (conn->c.state != CONN_STATE_ACCEPTING) return C_ERR;  
6.	    ERR_clear_error();  
7.	  
8.	    /* Try to accept */  
9.	    conn->c.conn_handler = accept_handler;  
10.	    ret = SSL_accept(conn->ssl);  
11.	  
12.	    if (ret <= 0) {  
13.	        WantIOType want = 0;  
14.	        if (!handleSSLReturnCode(conn, ret, &want)) {  
15.	            registerSSLEvent(conn, want);   /* We'll fire back */  
16.	            return C_OK;  
17.	        } else {  
18.	            conn->c.state = CONN_STATE_ERROR;  
19.	            return C_ERR;  
20.	        }  
21.	    }  
22.	  
23.	    conn->c.state = CONN_STATE_CONNECTED;  
24.	    if (!callHandler((connection *) conn, conn->c.conn_handler)) return C_OK;  
25.	    conn->c.conn_handler = NULL;  
26.	  
27.	    return C_OK;  
28.	}  
```

The main part of this function is it calls SSL_accept function to wait for client to initialite the tls handshake, if returns successfully, it will set connection state to CONN\_STATE\_CONNECTED.

### READ/WRITE

In tls.c file, connTLSRead function was registered as a call back function when there is a AE_READABLE file event triggered in multiplexing io. The function was defined as follows:

```
1.	static int connTLSRead(connection *conn_, void *buf, size_t buf_len) {  
2.	    tls_connection *conn = (tls_connection *) conn_;  
3.	    int ret;  
4.	    int ssl_err;  
5.	  
6.	    if (conn->c.state != CONN_STATE_CONNECTED) return -1;  
7.	    ERR_clear_error();  
8.	    ret = SSL_read(conn->ssl, buf, buf_len);  
9.	    if (ret <= 0) {  
10.	        WantIOType want = 0;  
11.	        if (!(ssl_err = handleSSLReturnCode(conn, ret, &want))) {  
12.	            if (want == WANT_WRITE) conn->flags |= TLS_CONN_FLAG_READ_WANT_WRITE;  
13.	            updateSSLEvent(conn);  
14.	  
15.	            errno = EAGAIN;  
16.	            return -1;  
17.	        } else {  
18.	            if (ssl_err == SSL_ERROR_ZERO_RETURN ||  
19.	                    ((ssl_err == SSL_ERROR_SYSCALL) && !errno)) {  
20.	                conn->c.state = CONN_STATE_CLOSED;  
21.	                return 0;  
22.	            } else {  
23.	                conn->c.state = CONN_STATE_ERROR;  
24.	                return -1;  
25.	            }  
26.	        }  
27.	    }  
28.	  
29.	    return ret;  
30.	}  
```

It is a wrapper function for SSL\_read function, which reads buf\_len bytes from the tls connection. 

Similarly, for the write operation, the callback was defined in connTLSWrite function:

```
1.	static int connTLSWrite(connection *conn_, const void *data, size_t data_len) {  
2.	    tls_connection *conn = (tls_connection *) conn_;  
3.	    int ret, ssl_err;  
4.	  
5.	    if (conn->c.state != CONN_STATE_CONNECTED) return -1;  
6.	    ERR_clear_error();  
7.	    ret = SSL_write(conn->ssl, data, data_len);  
8.	  
9.	    if (ret <= 0) {  
10.	        WantIOType want = 0;  
11.	        if (!(ssl_err = handleSSLReturnCode(conn, ret, &want))) {  
12.	            if (want == WANT_READ) conn->flags |= TLS_CONN_FLAG_WRITE_WANT_READ;  
13.	            updateSSLEvent(conn);  
14.	            errno = EAGAIN;  
15.	            return -1;  
16.	        } else {  
17.	            if (ssl_err == SSL_ERROR_ZERO_RETURN ||  
18.	                    ((ssl_err == SSL_ERROR_SYSCALL && !errno))) {  
19.	                conn->c.state = CONN_STATE_CLOSED;  
20.	                return 0;  
21.	            } else {  
22.	                conn->c.state = CONN_STATE_ERROR;  
23.	                return -1;  
24.	            }  
25.	        }  
26.	    }  
27.	  
28.	    return ret;  
29.	}  
```

It calls SSL_write function for writing data_len bytes into the socket.

Reference:

[openssl](https://www.openssl.org/docs/man1.1.1/man3/）
[redis](https://github.com/antirez/redis)






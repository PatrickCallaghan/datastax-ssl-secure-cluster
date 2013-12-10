## Securing a DSE Cluster with SSL
This demo gives a walkthrough of setting a secure cluster over ssl. It shows how to create node-to-node ssl connections and how to connect to the cluster over ssl from a cqlsh client. This walkthrough is based on the standard cassandra and dse docs around securing a cluster with ssl. These should be read first for a better understanding of what we are trying to achieve.
http://www.datastax.com/docs/datastax_enterprise3.2/security/ssl_node_to_node#ssl-node-to-node
and
http://www.datastax.com/docs/datastax_enterprise3.2/security/ssl_transport



#### Pre-requisites
It is required that you have a running cluster and the relevant tools like keytool, openssl and ssh already installed and configured. Using a tool like cluster ssh it extremely handy for talking to mulitple servers at the same time. https://code.google.com/p/csshx/

To enable the hosts to talk directly to each other through hostnames, the /etc/hosts file should be updated to map all IPs to hostnames e.g.  

    192.168.25.149	cassandra-secure1
    192.168.25.150	cassandra-secure2
    192.168.25.151	cassandra-secure3

## Server to Server over ssl.

For each server we need to create a private/public keys to allow the servers to communicate to each other over ssl. In the below example I have filling in the relevant info for my hostnames but this will need to changed if you use other hostnames. For that reason, it might be better to use the hostnames I have used just to get a working version and to understand the process. 

Note : when we generate a key with use a Common Name (CN=) as the server name we are creating the key for. 

These can be run together on cassandra-secure1

    keytool -genkey -alias cassandra-secure1 -keyalg RSA -keysize 1024 -dname "CN=cassandra-secure1, OU=TEST, O=TEST, C=UK"  -keystore .cassandra-secure1-keystore -storepass cassandra -keypass cassandra
    keytool -export -alias cassandra-secure1 -file cassandra-secure1.cer -keystore .cassandra-secure1-keystore -storepass cassandra -keypass cassandra

These can be run together on cassandra-secure2

    keytool -genkey -alias cassandra-secure2 -keyalg RSA -keysize 1024 -dname "CN=cassandra-secure2, OU=TEST, O=TEST, C=UK"  -keystore .cassandra-secure2-keystore -storepass cassandra -keypass cassandra
    keytool -export -alias cassandra-secure2 -file cassandra-secure2.cer -keystore .cassandra-secure2-keystore -storepass cassandra -keypass cassandra

These can be run together on cassandra-secure3

    keytool -genkey -alias cassandra-secure3 -keyalg RSA -keysize 1024 -dname "CN=cassandra-secure3, OU=TEST, O=TEST, C=UK"  -keystore .cassandra-secure3-keystore -storepass cassandra -keypass cassandra
    keytool -export -alias cassandra-secure3 -file cassandra-secure3.cer -keystore .cassandra-secure3-keystore -storepass cassandra -keypass cassandra


To create a user based cert for accessing through cqlsh

    keytool -importkeystore -srckeystore .cassandra-secure1-keystore -destkeystore cassandra-secure1-user.p12 -deststoretype PKCS12
    openssl pkcs12 -in cassandra-secure1-user.p12 -out cassandra-secure1-user.pem -nodes 


We have created public and private keys for all nodes in our cluster. We have also created user pem files to be able to connect over cqlsh with ssl to the cassandra-secure1 node.

We need to server copy all cer files to each of the other certs directory.
eg
From cassandra-secure1

    scp cassandra-secure1.cer root@cassandra-secure2:/etc/dse/certs
    scp cassandra-secure1.cer root@cassandra-secure3:/etc/dse/certs

From cassandra-secure2

    scp cassandra-secure2.cer root@cassandra-secure1:/etc/dse/certs
    scp cassandra-secure2.cer root@cassandra-secure3:/etc/dse/certs

From cassandra-secure3

    scp cassandra-secure3.cer root@cassandra-secure1:/etc/dse/certs
    scp cassandra-secure3.cer root@cassandra-secure2:/etc/dse/certs

At this point we have each of the servers public keys (.cer files) on each of the nodes so they should be able to talk to each other over ssl.

For each node, we need to import the .cer files into the keystore so that the each server has the certificates for every other server.

On Server cassandra-secure1

    keytool -import -v -trustcacerts -alias cassandra-secure1 -file cassandra-secure1.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure2 -file cassandra-secure2.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure3 -file cassandra-secure3.cer -keystore .truststore -storepass cassandra -keypass cassandra

On Server cassandra-secure2

    keytool -import -v -trustcacerts -alias cassandra-secure1 -file cassandra-secure1.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure2 -file cassandra-secure2.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure3 -file cassandra-secure3.cer -keystore .truststore -storepass cassandra -keypass cassandra

On Server cassandra-secure3

    keytool -import -v -trustcacerts -alias cassandra-secure1 -file cassandra-secure1.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure2 -file cassandra-secure2.cer -keystore .truststore -storepass cassandra -keypass cassandra
    keytool -import -v -trustcacerts -alias cassandra-secure3 -file cassandra-secure3.cer -keystore .truststore -storepass cassandra -keypass cassandra


You can check that the .truststore has the relevant entries, each .truststore should have the 3 servers in it. 

    keytool -list -keystore .truststore 

We need to enable the node-to-node in the cassandra.yaml file

    server_encryption_options:  
        internode_encryption: all
        keystore: /etc/dse/certs/.cassandra-secure1-keystore
        keystore_password: cassandra
        truststore: /etc/dse/certs/.truststore
        truststore_password: cassandra
        
If we now restart all of the nodes, we should see a message in the logs stating
'Starting Encrypted Messaging Service on SSL port 7001'        

All servers should know be communicating over ssl.

Any problems see Troubleshooting

## Client to Server

We need to create a  ~/.cqlshrc file which holds the properties used when we are connecting to a cluster using cqlsh. NOTE this gets moved into ~/.cassandra/.cqlshrc in more recent versions. 

We have only created a pem file for cassandra-secure1 so that it where we will try to connect. If you require access to other nodes, you will need to follow the steps to create a pem file for that server. The ~/.cqlshrc should be 
  
    [authentication]
    username = 
    password = 

    [connection]
    hostname = cassandra-secure1
    port = 9160
    factory = cqlshlib.ssl.ssl_transport_factory

    [ssl]
    certfile = /Users/username/certs/cassandra-secure1-user.pem
    validate = true ## Optional, true by default.

Note - the certfile can be placed anywhere.

In the cassandra.yaml update the client encryption properties 

    client_encryption_options:
        enabled: true
        keystore: /etc/dse/certs/.cassandra-secure1-keystore
        keystore_password: cassandra

We should know we able to restart cassandra and connect securely through ssl using cqlsh. 

## Troubleshooting
NOTE : 
If you get the following error, 

ERROR 12:11:27,671 Unable to start DSE server.
java.lang.RuntimeException: Failed to setup secure pipeline
	at org.apache.cassandra.transport.Server$SecurePipelineFactory.<init>(Server.java:276)
	at org.apache.cassandra.transport.Server.run(Server.java:137)
	at org.apache.cassandra.transport.Server.start(Server.java:97)
	at org.apache.cassandra.service.CassandraDaemon.start(CassandraDaemon.java:394)
	at com.datastax.bdp.server.DseDaemon.start(DseDaemon.java:435)
	at org.apache.cassandra.service.CassandraDaemon.activate(CassandraDaemon.java:460)
	at com.datastax.bdp.server.DseDaemon.main(DseDaemon.java:550)
Caused by: java.io.IOException: Error creating the initializing the SSL Context
	at org.apache.cassandra.security.SSLFactory.createSSLContext(SSLFactory.java:124)
	at org.apache.cassandra.transport.Server$SecurePipelineFactory.<init>(Server.java:272)
	... 6 more
Caused by: java.io.FileNotFoundException: /etc/dse/certs/.cassandra-secure2keystore (No such file or directory)
	at java.io.FileInputStream.open(Native Method)
	at java.io.FileInputStream.<init>(FileInputStream.java:146)
	at java.io.FileInputStream.<init>(FileInputStream.java:101)
	at org.apache.cassandra.security.SSLFactory.createSSLContext(SSLFactory.java:113)
	... 7 more


I think you can get round it by overriding the cipher suites for both node-to-node and client-node properties
e.g.

    cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA]

    
This is because of the following problem in Oracle Java.
http://www.pathin.org/tutorials/java-cassandra-cannot-support-tls_rsa_with_aes_256_cbc_sha-with-currently-installed-providers/

You can follow the directions and download the updated security libs.

Once downloaded you can copy the files to the correct library on your server. 
e.g.

    scp * root@cassandra-secure1:/usr/lib/jvm/java-7-oracle/jre/lib/security/




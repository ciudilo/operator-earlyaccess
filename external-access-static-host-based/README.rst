Deploy Confluent Platform with Static Host-based Routing
========================================================

In this scenario workflow, you'll set up Confluent Platform component clusters
with the static host-based routing to enable external clients to access
Kafka.

Before you begin this tutorial:

* `Set up the prerequisites <https://github.com/confluentinc/operator-earlyaccess#pre-requisites>`__.

* `Create the namespace for the tutorials <https://github.com/confluentinc/operator-earlyaccess#set-up-the-kubernetes-cluster>`__.

* `Configure the Early Access credentials <https://github.com/confluentinc/operator-earlyaccess#configure-early-access-credentials>`__.

* `Clone the tutorial repo <https://github.com/confluentinc/operator-earlyaccess#download-confluent-operator-tutorial-package>`__.

* This tutorial assumes your local machine has a Java Runtime Environment (JRE) and ``kafka-console-producer`` installed. The command line producer tool comes with `Confluent Platform <https://docs.confluent.io/platform/current/installation/installing_cp/zip-tar.html#manual-install-using-zip-and-tar-archives>`__ TARball under the ``bin/`` directory.
 
To complete this tutorial, you'll follow these steps:

#. Set up the Kubernetes cluster for this tutorial.

#. Set the current tutorial directory.

#. Deploy Confluent Operator.

#. Deploy Confluent Platform.

#. Deploy the producer application.

#. Tear down Confluent Platform.

===========================
Set up a Kubernetes cluster
===========================

Set up a Kubernetes cluster for this tutorial, and save the Kubernetes cluster
domain name. 
 
In this document, ``$DOMAIN`` will be used to denote your Kubernetes cluster
domain name.
  
::

  export DOMAIN=<Your Kubernetes cluster domain name>

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=<Tutorial directory>/external-access-static-host-based

=========================
Deploy Confluent Operator
=========================

#. Install Confluent Operator using Helm:

   ::
   
     helm upgrade --install operator confluentinc_earlyaccess/confluent-operator \
       --set image.registry=confluent-docker-internal-early-access-operator-2.jfrog.io

#. Check that the Confluent Operator pod comes up and is running:

   ::
   
     kubectl get pods
      
===========================================
Generate and Deploy TLS Secrets for Servers
===========================================

This tutorial uses mutual TLS (mTLS) for encryption and authentication between
Confluent Servers (inside the Kubernetes cluster) and Kafka clients (outside the Kubernetes cluster).
In production, you would need to provide your own valid certificates,
but for this tutorial, you will generate your own.
Here are the important files you will create:

* ``cacerts.pem`` -- Certificate authority (CA) certificate
* ``privkey.pem`` -- Private key for Confluent Servers
* ``fullchain.pem`` -- Certificate for Confluent Servers
* ``privkey-client.pem`` -- Private key for Kafka client
* ``fullchain-client.pem`` -- Certificate for Kafka client
* ``client.truststore.p12`` -- Truststore for Kafka client
* ``client.keystore.p12`` -- Keystore for Kafka client

Notice that you will not generate the keystore or truststore for Confluent Servers.
Confluent Operator will generate those for Confluent Server from
``cacerts.pem``, ``privkey.pem``, and ``fullchain.pem``.

#. Generate a private key called ``rootCAkey.pem`` for the root certificate authority.

   ::

     openssl genrsa -out $TUTORIAL_HOME/certs/rootCAkey.pem 2048

#. Generate the CA certificate.

   ::

     openssl req -x509  -new -nodes \
       -key $TUTORIAL_HOME/certs/rootCAkey.pem \
       -days 3650 \
       -out $TUTORIAL_HOME/certs/cacerts.pem \
       -subj "/C=US/ST=CA/L=MVT/O=TestOrg/OU=Cloud/CN=TestCA"

#. Generate a private key called ``privkey.pem`` for Confluent Servers.

   ::

     openssl genrsa -out $TUTORIAL_HOME/certs/privkey.pem 2048

#. Create a certificate signing request (CSR) called ``server.csr`` for Confluent Servers. Note the ``*`` wildcard for the ``CN`` common name. Later, this will allow for Subject Alternate Names (SANs) for all of the brokers.

   ::

     openssl req -new -key $TUTORIAL_HOME/certs/privkey.pem \
       -out $TUTORIAL_HOME/certs/server.csr \
       -subj "/C=US/ST=CA/L=MVT/O=TestOrg/OU=Cloud/CN=*.$DOMAIN"

#. Create the ``fullchain.pem`` certificate for Confluent Servers. Notice the extension file includes a ``*`` wildcard for SANs.

   ::

     openssl x509 -req \
       -in $TUTORIAL_HOME/certs/server.csr \
       -extensions server_ext \
       -CA $TUTORIAL_HOME/certs/cacerts.pem \
       -CAkey $TUTORIAL_HOME/certs/rootCAkey.pem \
       -CAcreateserial \
       -out $TUTORIAL_HOME/certs/fullchain.pem \
       -days 365 \
       -extfile \
       <(echo "[server_ext]"; echo "extendedKeyUsage=serverAuth,clientAuth"; echo "subjectAltName=DNS:*.$DOMAIN")

#. 

#. Create a Kubernetes secret using the provided PEM files:
 
   ::

     kubectl create secret generic tls-group1 \
       --from-file=fullchain.pem=$TUTORIAL_HOME/certs/fullchain.pem \
       --from-file=cacerts.pem=$TUTORIAL_HOME/certs/cacerts.pem \
       --from-file=privkey.pem=$TUTORIAL_HOME/certs/privkey.pem
       
============================
Configure Confluent Platform
============================

You install Confluent Platform components as custom resources (CRs). 

In this tutorial, you will configure Zookeeper, Kafka, and Control Center in a
single file and deploy the components with one ``kubectl apply`` command.

The CR configuration file contains a custom resource specification for each
Confluent Platform component, including replicas, image to use, resource
allocations.

Edit the Confluent Platform CR file: ``$TUTORIAL_HOME/confluent-platform.yaml``

Specifically, note that external accesses to Confluent Platform components are
configured using host-based static routing.

The Kafka section of the file is set as follow for external access:

:: 

  Spec:
    listeners:
      external:
        externalAccess:
          type: staticForHostBasedRouting
          staticForHostBasedRouting:
            domain:                              ----- [1]
            port: 443
        tls:
          enabled: true

* [1] Set this to the value of ``$DOMAIN``, your Kubernetes cluster domain.

* The prefixes are used for external DNS hostnames. In this tutorial, Kafka bootstrap server will use the default prefix, ``kafka``, and the brokers will use the default prefix, ``b``. 

  As Kafka is configured with 3 replicas in this tutorial, the access endpoints of
  Kafka will be:
  
  * kafka.$DOMAIN for the bootstrap server
  * b0.$DOMAIN for the broker #1
  * b1.$DOMAIN for the broker #2
  * b2.$DOMAIN for the broker #3

=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe kafka
     
=========================
Deploy Ingress controller
=========================

An Ingress controller is required to access Kafka using the static host-based
routing. In this tutorial, we will use Nginx Ingress controller.

SSL passthrough is the action of passing data through a load balancer to a server without decrypting it. 
In many load balancer use cases, the decryption or SSL termination happens at the load balancer and data is passed along to the endpoint. 
But SSL passthrough keeps the data encrypted as it travels through the load balancer - and this is what Kafka expects.

#. Add the Kubernetes NginX Helm repo and update the repo.

   ::
   
     helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
     helm repo update

#. Install the Ngix controller:

   ::
   
     helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx \
       --set controller.extraArgs.enable-ssl-passthrough="true"
       
================================
Create a Kafka bootstrap service
================================

When using staticForHostBasedRouting as externalAccess type, the bootstrap
endpoint is not configured to access Kafka. 

If you want to have a bootstrap endpoint to access Kafka instead of using each
broker's endpoint, you need to provide the bootstrap endpoint, create a
DNS record pointing to Ingress controller load balancer's external IP, and
define the ingress rule for it.

Create the Kafka bootstrap service to access Kafka:

::

  kubectl apply -f $TUTORIAL_HOME/kafka-bootstrap-service.yaml

======================
Create Ingress service
======================

Create an Ingress resource that includes a collection of rules that the Ingress
control uses to route the inbound traffic to Kafka:

#. In the resource file, ``ingress-service-hostbased.yaml``, replace ``$DOMAIN`` 
   with the value of your ``$DOMAIN``.

#. Create the Ingress resource:

   ::

     kubectl apply -f $TUTORIAL_HOME/ingress-service-hostbased.yaml

===============
Add DNS records
===============

Create DNS records for Kafka brokers using the ingress controller load balancer externalIP.

#. Retrieve the external IP addresses of Nginx load balancer:

   ::
   
     kubectl get svc
     
#. Add DNS records for the Kafka brokers using the IP address and
   replacing ``$DOMAIN`` with the actual domain name of your Kubernetes cluster.

   In this tutorial, we are using the default prefixes for components and brokers as shown below:
   
   ====================== ===============================================================
   DNS name               IP address
   ====================== ===============================================================
   kafka.$DOMAIN          The ``EXTERNAL-IP`` value of the Nginx load balancer service
   b0.$DOMAIN             The ``EXTERNAL-IP`` value of the Nginx load balancer service
   b1.$DOMAIN             The ``EXTERNAL-IP`` value of the Nginx load balancer service
   b2.$DOMAIN             The ``EXTERNAL-IP`` value of the Nginx load balancer service
   ====================== ===============================================================
  
========
Validate
========

To validate, you will produce data to a topic from a Kafka client
outside of the Kubernetes cluster.

Generate Kafka Client Certificates, Keystore, and Truststore
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Confluent Servers are configured to authenticate clients with mTLS, 
so you must create a keystore **and** truststore for the Kafka client.
Here are the important files you will create:

* ``privkey-client.pem`` -- Private key for Kafka client
* ``fullchain-client.pem`` -- Certificate for Kafka client
* ``client.keystore.p12`` -- Keystore for Kafka client
* ``client.truststore.p12`` -- Truststore for Kafka client

You made the CA certificate earlier when generating the certificates for the Confluent Servers.
You will use the same CA certificate to create a certificate for the Kafka client, 
as well as a keystore and a truststore.


#. Generate a private key called ``privkey-client.pem`` for the Kafka client.

   ::

     openssl genrsa -out $TUTORIAL_HOME/certs/privkey-client.pem 2048

#. Create a certificate signing request (CSR) called ``client.csr`` for the Kafka client.

   ::

     openssl req -new -key $TUTORIAL_HOME/certs/privkey-client.pem \
       -out $TUTORIAL_HOME/certs/client.csr \
       -subj "/C=US/ST=CA/L=MVT/O=TestOrg/OU=Cloud/CN=kafka-client"

#. Create the ``fullchain-client.pem`` certificate for the Kafka client.

   ::

     openssl x509 -req \
       -in $TUTORIAL_HOME/certs/client.csr \
       -CA $TUTORIAL_HOME/certs/cacerts.pem \
       -CAkey $TUTORIAL_HOME/certs/rootCAkey.pem \
       -CAcreateserial \
       -out $TUTORIAL_HOME/certs/fullchain-client.pem \
       -days 365

#. Create the Kafka client's keystore. This keystore is used to authenticate to brokers.

   ::

     mkdir $TUTORIAL_HOME/client && \
     openssl pkcs12 -export \
       -in $TUTORIAL_HOME/certs/fullchain-client.pem \
       -inkey $TUTORIAL_HOME/certs/privkey-client.pem \
       -out $TUTORIAL_HOME/client/client.keystore.p12 \
       -name kafka-client \
       -passout pass:mystorepassword

#. Create the Kafka client's truststore from the CA. This truststore allows the client to trust the broker's certificate, which is necessary for transport encryption.

   ::

     keytool -import -trustcacerts -noprompt \
       -alias rootCA \
       -file $TUTORIAL_HOME/certs/cacerts.pem \
       -keystore $TUTORIAL_HOME/client/client.truststore.p12 \
       -deststorepass mystorepassword \
       -deststoretype pkcs12

Create the Topic
^^^^^^^^^^^^^^^^

#. Inspect the ``$TUTORIAL_HOME/topic.yaml`` file, which defines the ``elastic-0`` topic as follows:

   ::
  
     apiVersion: platform.confluent.io/v1beta1
     kind: KafkaTopic
     metadata:
       name: elastic-0
       namespace: confluent
     spec:
       replicas: 1
       partitionCount: 1
       configs:
         cleanup.policy: "delete"

#. Create the topic called ``elastic-0`` for the Kafka producer to write to.

   ::

     kubectl apply -f $TUTORIAL_HOME/topic.yaml

Create Client Properties File
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a configuration file for the client called ``kafka.properties``.

  ::

    cat <<-EOF > $TUTORIAL_HOME/client/kafka.properties
    bootstrap.servers=kafka.$DOMAIN:443
    security.protocol=SSL
    ssl.truststore.location=$TUTORIAL_HOME/client/client.truststore.p12
    ssl.truststore.password=mystorepassword
    ssl.truststore.type=PKCS12
    ssl.keystore.location=$TUTORIAL_HOME/client/client.keystore.p12
    ssl.keystore.password=mystorepassword
    ssl.keystore.type=PKCS12
    EOF

Remember that in production, all properties files with sensitive credentials 
should be locked down with elevated permissions and encrypted with 
Confluent Secret Protection.

Produce to the Topic and View the Results in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Start the ``kafka-console-producer`` command line utility. Don't enter any messages at the ``>`` prompt yet.

   ::

     kafka-console-producer --bootstrap-server kafka.$DOMAIN:443 \
       --topic elastic-0 \
       --producer.config $TUTORIAL_HOME/client/kafka.properties

#. In a new terminal window, set up port forwarding to the Confluent Control Center web UI from the local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021
     
#. Browse to `Control Center <https://localhost:9021>`__. Your browser will complain about the self-signed certificate. Bypass this in whatever way your browser requires. For example, in Google Chrome, go to Advanced -> Proceed to localhost (unsafe).

#. Navigate to Cluster 1 -> Topics -> elastic-0 -> Messages in Control Center.
     

#. Back at the console producer ``>`` prompt, enter some messages. Look in Confluent Control Center to see those messages show up.

#. Celebrate! Your Confluent deployment can securely serve Kafka clients outside of your Kubernetes cluster!

=========
Tear Down
=========

Shut down Confluent Platform and the data:

``Ctrl+C`` on the ``kafka-console-producer`` command.

::

  kubectl delete -f $TUTORIAL_HOME/producer-app-data.yaml
  
::

  kubectl delete -f $TUTORIAL_HOME/ingress-service-hostbased.yaml
  
::

  kubectl delete -f $TUTORIAL_HOME/kafka-bootstrap-service.yaml

::

  kubectl delete -f $TUTORIAL_HOME/confluent-platform.yaml
  
::

  kubectl delete secret tls-group1
  
::

  helm delete nginx-operator

::

  helm delete operator
  

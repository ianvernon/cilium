*****************************************
Getting Started Using Istio on Kubernetes
*****************************************

This document serves as an introduction to using Cilium to enforce
security policies in Kubernetes micro-services running with Istio.  It
is a detailed walk-through of getting a single-node Cilium + Istio
environment running on your machine. It is designed to take 15-30
minutes.

If you haven't read the :ref:`intro` yet, we'd encourage you to do
that first.

The best way to get help if you get stuck is to ask a question on the
`Cilium Slack channel <https://cilium.herokuapp.com>`_.  With Cilium
contributors across the globe, there is almost always someone
available to help.

Step 0: Install kubectl & minikube
==================================

This guide uses `minikube
<https://kubernetes.io/docs/getting-started-guides/minikube/>`_ to
demonstrate deployment and operation of Cilium in combination with
Istio in a single-node Kubernetes cluster.  The minikube VM requires
approximately 2 GB of RAM and supports hypervisors like VirtualBox
that run on Linux, macOS, and Windows.

If you instead want to understand the details of deploying Cilium on a
full-fledged Kubernetes cluster, then go straight to
:ref:`admin_install_daemonset`.

Install ``kubectl`` version ``>= 1.6.3`` as described in the
`Kubernetes Docs
<https://kubernetes.io/docs/tasks/tools/install-kubectl/>`_.

Install one of the `hypervisors supported by minikube
<https://kubernetes.io/docs/tasks/tools/install-minikube/>`_.

Install ``minikube`` ``>= 0.22.3`` as described on `minikube's github
page <https://github.com/kubernetes/minikube/releases>`_.

Then, boot a minikube cluster with the Container Network Interface
(CNI) network plugin as well as the `RBAC authorization module
<https://kubernetes.io/docs/admin/authorization/rbac/>`_ enabled:

::

    $ minikube addons disable dashboard
    $ minikube addons enable kube-dns
    $ minikube start --memory=4096 --network-plugin=cni --kubernetes-version=v1.8.0 --extra-config=apiserver.Authorization.Mode=RBAC

After minikube has finished setting up your new Kubernetes cluster,
you can check the status of the cluster by running ``kubectl get cs``:

::

    $ kubectl get cs
    NAME                 STATUS    MESSAGE              ERROR
    controller-manager   Healthy   ok
    scheduler            Healthy   ok
    etcd-0               Healthy   {"health": "true"}

If you're using minikube's default ``localkube`` bootstrapper, to
enable ``kube-dns`` to run with RBAC enabled, the Kubernetes system
account must be bound to the ``cluster-admin`` role:

::

    $ kubectl create clusterrolebinding kube-system-default-binding-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default

To check that all Kubernetes pods are ``Running`` and 100% ready,
including ``kube-dns``, run:

::

    $ kubectl get pods -n kube-system
    NAME                          READY     STATUS    RESTARTS   AGE
    kube-addon-manager-minikube   1/1       Running   0          59s
    kube-dns-86f6f55dd5-5xdz8     3/3       Running   0          55s
    storage-provisioner           1/1       Running   0          56s


Step 1: Installing Cilium
=========================

The next step is to install Cilium into your Kubernetes cluster.
Cilium installation leverages the `Kubernetes Daemon Set
<https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/>`_
abstraction, which will deploy one Cilium pod per cluster node.  This
Cilium pod will run in the ``kube-system`` namespace along with all
other system relevant daemons and services.  The Cilium pod will run
both the Cilium agent and the Cilium CNI plugin.

To deploy Cilium, run:

.. parsed-literal::

    $ kubectl create -f \ |SCM_WEB|\/examples/kubernetes/cilium.yaml
    configmap "cilium-config" created
    secret "cilium-etcd-secrets" created
    serviceaccount "cilium" created
    clusterrolebinding "cilium" created
    daemonset "cilium" created
    clusterrole "cilium" created

Kubernetes is now deploying Cilium with its RBAC, ConfigMap and Daemon
Set as a pod on minkube. This operation is performed in the
background.

Run the following command to check the progress of the deployment:

::

    $ kubectl get daemonsets -n kube-system
    NAME      DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE-SELECTOR   AGE
    cilium    1         1         0         1            0           <none>          6s
    
Wait until the cilium Deployment shows a ``CURRENT`` count of ``1``
like above (a ``READY`` value of ``0`` is OK for this tutorial).

Step 2: Installing Istio
========================

To download Istio version 0.2.12, run:

::

    $ export ISTIO_VERSION=0.2.12
    $ curl -L https://git.io/getLatestIstio | sh -
    $ export ISTIO_HOME=\`pwd\`/istio-${ISTIO_VERSION}
    $ export PATH="$PATH:${ISTIO_HOME}/bin"

To deploy Istio on Kubernetes, run:

::

    $ kubectl create -f ${ISTIO_HOME}/install/kubernetes/istio.yaml

Run the following command to check the progress of the deployment:

::

    $ kubectl get deployments -n istio-system
    NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    istio-ca        1         1         1            1           2m
    istio-egress    1         1         1            1           2m
    istio-ingress   1         1         1            1           2m
    istio-mixer     1         1         1            1           2m
    istio-pilot     1         1         1            1           2m

To check whether all Istio pods are ready, run:

::

    $ kubectl get pods -n istio-system
    NAME                             READY     STATUS    RESTARTS   AGE
    istio-ca-57f9bd7ddb-6mxl8        1/1       Running   0          3m
    istio-egress-6c6b84cd5d-8kjst    1/1       Running   0          3m
    istio-ingress-79cf84458b-n77vk   1/1       Running   0          3m
    istio-mixer-6fdc6784c7-kbbqk     2/2       Running   0          3m
    istio-pilot-5f4865659b-kqn6d     1/1       Running   0          3m

Once all Istio pods are ready, we are ready to install the demo
application.

Step 3: Deploy the Bookinfo Application v1
==========================================

Now that we have Cilium and Istio deployed and ``kube-dns`` operating
correctly we can deploy the v1 services of the `Istio Bookinfo sample
application <https://istio.io/docs/guides/bookinfo.html>`_.

The BookInfo application is broken into four separate microservices:

- *productpage*. The productpage microservice calls the details and
  reviews microservices to populate the page.
- *details*. The details microservice contains book information.
- *reviews*. The reviews microservice contains book reviews. It also
  calls the ratings microservice.
- *ratings*. The ratings microservice contains book ranking
  information that accompanies a book review.

In this demo, each version of each microservice is deployed into
Kubernetes using a separate YAML specification which defines:

- A Kubernetes Service.
- A Kubernetes Deployment specifying the microservice's pods.
- A Cilium Network Policy limiting the traffic to the microservice.

Each Deployment must be packaged with Istio's Envoy sidecar proxy in
order to be managed by Istio, by running ``istioctl kube-inject``
command on each YAML file.

When used in conjunction with Cilium, the Istio sidecar can be further
modified to bypass the proxy for all inbound traffic, since all
inbound traffic filtering can be performed by Cilium.  This can be
done by modifying the container image used for the ``istio-init``
container.

To package the Istio sidecar proxy and generate final YAML
specifications, run:

.. parsed-literal::

    $ for service in productpage-service productpage-v1 details-v1 reviews-v1; do \\
          curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-${service}.yaml | \\
          istioctl kube-inject -f - | \\
          sed -e 's,istio/proxy_init:0.2.12,cilium/istio_proxy_init:0.2.12,' | \\
          kubectl create -f - ; done
    service "productpage" created
    deployment "productpage-v1" created
    ciliumnetworkpolicy "productpage-v1" created
    service "details" created
    deployment "details-v1" created
    ciliumnetworkpolicy "details-v1" created
    service "reviews" created
    deployment "reviews-v1" created
    ciliumnetworkpolicy "reviews-v1" created

Run the following command to check the progress of the deployment:

::
    $ kubectl get deployments -n default
    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    details-v1       1         1         1            1           6m
    productpage-v1   1         1         1            1           6m
    ratings-v1       1         1         1            1           6m
    reviews-v1       1         1         1            1           6m

Wait until all ``AVAILABLE`` counts are ``1``.
    
To obtain the URL to the frontend productpage service, run:

::

    $ export PRODUCTPAGE=`minikube service productpage -n default --url`
    $ echo "You can now access ${PRODUCTPAGE}/productpage"

Check that the application is working, by accessing the
``/productpage`` path at that URL in your web browser.

Step 4: Canary and Deploy the Reviews Service V2
================================================

In addition to providing reviews from readers, ``reviews`` ``v2``
service calls a new ``ratings`` service, and displays each rating as 1
to 5 black stars.

As a precaution, we will deploy ``v2`` using canarying to prevent
breaking the end-to-end application completely if it is faulty.

To prevent any traffic from being routed to ``v2`` for now, create a
set of Istio route rules to route 100% of the ``reviews`` traffic to
``v1``:

.. parsed-literal::

    $ kubectl apply -f \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v1.yaml
    routerule "reviews-default" created

To deploy ``ratings v1`` and ``reviews v2``, run:

.. parsed-literal::

    $ for service in ratings-v1 reviews-v2; do \\
          curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-${service}.yaml | \\
          istioctl kube-inject -f - | \\
          sed -e 's,istio/proxy_init:0.2.12,cilium/istio_proxy_init:0.2.12,' | \\
          kubectl create -f - ; done
    service "ratings" created
    deployment "ratings-v1" created
    ciliumnetworkpolicy "ratings-v1" created
    deployment "reviews-v2" created
    ciliumnetworkpolicy "reviews-v2" created   

Check in your web browser that no stars are appearing in the Book
Reviews, even after refreshing the page several times.  All reviews
are retrieved from ``reviews v1`` and none from ``reviews v2``.

The ``ratings-v1`` Cilium Network Policy explicitly whitelists access
to the ``ratings`` API only from ``productpage`` and ``reviews v2``.
Check that ``reviews v1`` may not be able to access the ``ratings``
service, even if it were compromised or suffered from a bug:

::

    $ export POD_REVIEWS_V1=`kubectl get pods -n default -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl exec ${POD_REVIEWS_V1} -c istio-proxy -- curl --connect-timeout 5 http://ratings:9080/ratings/0
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0
    curl: (28) Connection timed out after 5000 milliseconds

Update the route rule to send 50% of ``reviews`` traffic to ``v1`` and 50% to ``v2``:

.. parsed-literal::

    $ kubectl apply -f \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v1-v2.yaml
    routerule "reviews-default" configured

Check in your web browser that stars are appearing in the Book Reviews
roughly 50% of the time (this may require refreshing the page many
times to observe).

Finally, update the route rule to send 100% of ``reviews`` traffic to ``v2``:

.. parsed-literal::

    $ kubectl apply -f \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v2.yaml
    routerule "reviews-default" configured

Refresh the product page in your web browser several times to verify
that stars are now appearing in the Book Reviews on every page
refresh.  All the reviews are now retrieved from ``reviews v2`` and
none from ``reviews v1``.

Step 5: Deploy the Product Page Service V2
==========================================

The ``productpage`` service's ``v2`` version implements a new user
authentication audit log.  On every user login or logout, it sends a
JSON-formatted message which contains the following information:
- event: ``login`` or ``logout``
- username
- client IP address
- timestamp

In addition, ``productpage v2`` has a more restrictive Cilium Network
Policy.  The policy for ``v1`` currently allows read access to the
``productpage`` REST API under the ``/api/v1`` path.  To check that
the REST API is currently accessible and returns valid JSON data, run:

::

    $ export PRODUCTPAGE=`minikube service productpage -n default --url`
    $ for APIPATH in /api/v1/products /api/v1/products/0 /api/v1/products/0/reviews /api/v1/products/0/ratings; do echo ; curl -s -S "${PRODUCTPAGE}${APIPATH}" ; echo ; done
    
    [{"descriptionHtml": "<a href=\"https://en.wikipedia.org/wiki/The_Comedy_of_Errors\">Wikipedia Summary</a>: The Comedy of Errors is one of <b>William Shakespeare's</b> early plays. It is his shortest and one of his most farcical comedies, with a major part of the humour coming from slapstick and mistaken identity, in addition to puns and word play.", "id": 0, "title": "The Comedy of Errors"}]

    {"publisher": "PublisherA", "language": "English", "author": "William Shakespeare", "id": 0, "ISBN-10": "1234567890", "ISBN-13": "123-1234567890", "year": 1595, "type": "paperback", "pages": 200}

    {"reviews": [{"reviewer": "Reviewer1", "rating": {"color": "black", "stars": 5}, "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"}, {"reviewer": "Reviewer2", "rating": {"color": "black", "stars": 4}, "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}], "id": "0"}

    {"ratings": {"Reviewer2": 4, "Reviewer1": 5}, "id": 0}

This REST API is meant only for consumption by other internal
services, and will be blocked from external clients using the updated
Cilium Network Policy in ``productpage v2``.

To deploy the Kafka broker, run:

::
    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/kafka-v1.yaml | \\
          istioctl kube-inject -f - | \\
          sed -e 's,istio/proxy_init:0.2.12,cilium/istio_proxy_init:0.2.12,' | \\
          kubectl create -f -
    service "kafka" created
    statefulset "kafka-v1" created
    ciliumnetworkpolicy "kafka-authaudit" created

Wait until the ``kafka`` pod is ready, i.e. until ``READY`` becomes ``2/2``:

::

    $ kubectl get pods -n default -l app=kafka
    NAME         READY     STATUS    RESTARTS   AGE
    kafka-v1-0   2/2       Running   0          21m

To create the ``authaudit`` topic, run:

::

    $ kubectl exec kafka-v1-0 -c kafka -- bash -c '/opt/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic authaudit --partitions 1 --replication-factor 1'
    Created topic "authaudit".

Then, to deploy the ``productpage v2`` service and its updated Cilium
Network Policy, run:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-productpage-v2.yaml | \\
          istioctl kube-inject -f - | \\
          sed -e 's,istio/proxy_init:0.2.12,cilium/istio_proxy_init:0.2.12,' | \\
          kubectl create -f -
    deployment "productpage-v2" created
    ciliumnetworkpolicy "productpage-v2" created

    $ kubectl delete -f \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-productpage-v1.yaml

Check that the ``/productpage`` is still accessible in your web
browser.  To also check that Cilium now denies access to the REST API,
run:

::

    $ export PRODUCTPAGE=`minikube service productpage -n default --url`
    $ for APIPATH in /api/v1/products /api/v1/products/0 /api/v1/products/0/reviews /api/v1/products/0/ratings; do curl -s -S "${PRODUCTPAGE}${APIPATH}" ; done
    Access denied
    Access denied
    Access denied
    Access denied

To observe the Kafka messages sent by ``productpage`` on every login
and logout into the ``authaudit`` Kafka topic, we will run an
additional ``authaudit-logger`` service.  This service fetches and
prints out all messages from that Kafka topic.  To start the service,
run:

::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/authaudit-logger-v1.yaml | \\
          istioctl kube-inject -f - | \\
          sed -e 's,istio/proxy_init:0.2.12,cilium/istio_proxy_init:0.2.12,' | \\
          kubectl create -f -

To display the logs while you login and logout of the product page,
run:

::

    $ export POD_LOGGER_V1=`kubectl get pods -n default -l app=authaudit-logger,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl logs ${POD_LOGGER_V1} -c authaudit-logger
    ...
    {"timestamp": "2017-12-04T09:34:24.341668", "remote_addr": "10.15.28.238", "event": "login", "user": "richard"}
    {"timestamp": "2017-12-04T09:34:40.943772", "remote_addr": "10.15.28.238", "event": "logout", "user": "richard"}
    {"timestamp": "2017-12-04T09:35:03.096497", "remote_addr": "10.15.28.238", "event": "login", "user": "gilfoyle"}
    {"timestamp": "2017-12-04T09:35:08.777389", "remote_addr": "10.15.28.238", "event": "logout", "user": "gilfoyle"}

As you can see, the user-identifiable information sent by
``productpage`` in every Kafka message is sensitive, so all accesses
to this Kafka topic must be protected using Cilium.  The Cilium
Network Policy configured on the Kafka broker enforces that:
- only ``productpage v2`` is allowed to produce messages into the
  ``authaudit`` topic;
- only ``authaudit-logger`` can fetch those messages;
- no service can access any other topic.

FIXME: show that a compromised ``authaudit-logger`` cannot access any
other topic.
  
Step 6: Clean-Up
=================

You have now installed Cilium, deployed a demo app, and tested both
L7 Kafka-aware network security policies.   To clean-up, run:

::

   $ minikube delete

After this, you can re-run the tutorial from Step 0.


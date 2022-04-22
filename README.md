# K8S cluster Tomcat configuration and session replication
This repository is used to document the configuration process for clustering with tomcat using k8s

# Explanation
There are several modes to make clustering and session replication with Tomcat: 
- Multicast
- StaticMembership (TCP)
- CloudMembership (new)

In my test I tried the first two, but without success or compromising the security and simplicity: 

For Multicast the main problem was that K8S by default doesn't allow udp comunication that is used with that type of communication 

For StaticMembership the complexity was high because when I used Probes for the pods the name resolution doesn't work right, and if you want to scale the things get more complicated. 

## CloudMemberShip
This option is not well documented so I'm going to explain a little so you can understand the configurations: 

## CloudMembershipProvider
This class connects with the cluster api and gets the nodes that are configured in the namespace provided by the K8S admin, but to connect you have to create and api token for the service account that you want to allow to.

Also you have to create a cluster role binding to allow that particular SA to connect to the cluster. 

So these are the steps: 

## Create an Api token for Service account : 
Save the following text in a file service-account-token.yaml
```
apiVersion: v1
kind: Secret
metadata:
  namespace: test-namespace
  name: api-access-token #Name of the api token that is used to connect to the k8s cluster
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
```

Create SA:
```
kubectl apply -f service-account-token.yaml -n {namespace}
```

## Create a cluster Role Biding: 
Save the following text in a file cluster-role-tomcat.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-tomcat
subjects:
- kind: ServiceAccount
  name: default # name of your service account
  namespace: test-namespace # this is the namespace your service account is in
roleRef: # referring to your ClusterRole
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
Create CRB:
```
kubectl apply -f cluster-role-tomcat.yaml -n {namespace}
```

## Create a ConfigMap: 
The configMap is used to control the configuration of tomcat, in this we added the cluster configuration, save the following text in a file configmap-tomcat.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: testconfig
data:
  server.xml: |
    <?xml version="1.0" encoding="UTF-8"?>

    <Server port="8005" shutdown="SHUTDOWN">
      <Listener className="org.apache.catalina.startup.VersionLoggerListener" />

      <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />

      <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
      <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
      <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

      <GlobalNamingResources>

        <Resource name="UserDatabase" auth="Container"
                  type="org.apache.catalina.UserDatabase"
                  description="User database that can be updated and saved"
                  factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                  pathname="conf/tomcat-users.xml" />
      </GlobalNamingResources>

      <Service name="Catalina">

        <Connector port="8080" protocol="HTTP/1.1"
                  connectionTimeout="20000"
                  redirectPort="8443" />

        <Engine name="Catalina" defaultHost="localhost">

          <Realm className="org.apache.catalina.realm.LockOutRealm">

            <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                  resourceName="UserDatabase"/>
          </Realm>

          <Host name="localhost"  appBase="webapps"
                unpackWARs="true" autoDeploy="true">
            <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"> 
              <Channel className="org.apache.catalina.tribes.group.GroupChannel">
                <Membership className="org.apache.catalina.tribes.membership.cloud.CloudMembershipService"/>
              </Channel>
            </Cluster>

            <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                  prefix="localhost_access_log" suffix=".txt"
                  pattern="%h %l %u %t &quot;%r&quot; %s %b" />

          </Host>
        </Engine>
      </Service>
    </Server>
```
Create the configMap:
```
kubectl apply -f configmap-tomcat.yaml -n {namespace}
```

## Create a StatefulSet: 
The statefulset create name pod, save the following text in a file statefulset-tomcat.yaml
```
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: backend
spec:
  serviceName: tomcat-service
  podManagementPolicy: "Parallel"
  selector:
    matchLabels:
      app: tomcat-test
      tier: backend
      track: stable
  replicas: 3
  template:
    metadata:
      labels:
        app: tomcat-test
        tier: backend
        track: stable
    spec:
      containers:
        - name: tomcat-9
          image: tomcat:9.0
          command: ["/bin/sh","-c"]
          args: ["printf '%s' $API_TOKEN > $SA_TOKEN_FILE  && catalina.sh run"] ## This store the token in a file
          volumeMounts:
            - mountPath: /usr/local/tomcat/conf/server.xml #Use of configMap
              subPath: server.xml
              readOnly: true
              name: serverxml
          ports:
            - name: http
              protocol: TCP
              containerPort: 8080
          env:
          - name: SA_TOKEN_FILE
            value: /usr/local/tokenFile   
          - name: KUBERNETES_NAMESPACE
            value: test-namespace ## Change this to your namespace  
          - name: API_TOKEN
            valueFrom:
              secretKeyRef:
                name: "api-access-token" #This is the name of the create secret that store the token
                key: "token"
      volumes:
        - name: serverxml
          configMap:
            name: testconfig #This used the configMap

```
Create the statefulSet:
```
kubectl apply -f statefulset-tomcat.yaml -n {namespace}
```

## Test

When the creation of the statefulset is done you have to search in the logs: 
```
kubectl logs backend-0 -n {namespace} --tail 300 -f 
```

Something like this: 
```
INFO [CloudMembership-memberDisappeared] org.apache.catalina.tribes.group.interceptors.TcpFailureDetector.memberDisappeared Verification complete. Member still alive[org.apache.catalina.tribes.membership.MemberImpl[tcp://10.244.3.130:4000,10.244.3.130,4000, alive=3144, securePort=-1, UDP Port=-1, id={86 -72 28 -118 -8777 76 69 4 781 -121 99 103 877 86 -1 87 }, payload={}, command={}, domain={}]]
```

And your done :)

For testing the application you could use the Oracle application ShoppingCart 
you have to set the <distributable /> tag in the WEB-INF/web.xml of the war

Access to application
https://www.oracle.com/webfolder/technetwork/tutorials/obe/fmw/wls/10g/r3/cluster/session_state/files/shoppingcart.zip


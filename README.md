# mongodb-community-installation-openshift
this guide shows how to install the mongodb community operator and create an instance on Openshift Container Platform (OCP).
To install the operator, follow these steps on a computer that can access the openshift 
cluster you wish to install MongoDB on.

Prerequisites:
    - An openshift cluster
    - storage classes are available e.g. Openshift Container Storage (OCS), Portworx, aws.
    - Certificate Manager installed on the cluster

1. Clone the repository and apply the custom resource definition for mongoDB:

        git clone https://github.com/mongodb/mongodb-kubernetes-operator.git
        oc apply -f mongodb-kubernetes-operator/config/crd/bases/mongodbcommunity.mongodb.com_mongodbcommunity.yaml  


2. Create the project to use for MongoDB, and apply the Role Based Access Controls (RBACs):


    
        oc new-project mongodb
        oc apply -k mongodb-kubernetes-operator/config/rbac/ --namespace mongodb
     

3. Verify that the RBACs have been installed

        oc get role mongodb-kubernetes-operator --namespace mongodb

        oc get rolebinding mongodb-kubernetes-operator --namespace mongodb

        oc get serviceaccount mongodb-kubernetes-operator --namespace mongodb


4. Install the operator using the YAML shown below:

        cat <<EOF |oc apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          annotations:
            email: support@mongodb.com
          labels:
            owner: mongodb
          name: mongodb-kubernetes-operator
          namespace: mongodb
        spec:
          replicas: 1
          selector:
            matchLabels:
              name: mongodb-kubernetes-operator
          strategy:
            rollingUpdate:
              maxUnavailable: 1
            type: RollingUpdate
          template:
            metadata:
              labels:
                name: mongodb-kubernetes-operator
            spec:
              affinity:
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchExpressions:
                      - key: name
                        operator: In
                        values:
                        - mongodb-kubernetes-operator
                    topologyKey: kubernetes.io/hostname
              containers:
              - command:
                - /usr/local/bin/entrypoint
                env:
                - name: WATCH_NAMESPACE
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
                - name: POD_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.name
                - name: MANAGED_SECURITY_CONTEXT
                  value: 'true'              
                - name: OPERATOR_NAME
                  value: mongodb-kubernetes-operator
                - name: AGENT_IMAGE
                  value: quay.io/mongodb/mongodb-agent:11.0.5.6963-1
                - name: VERSION_UPGRADE_HOOK_IMAGE
                  value: quay.io/mongodb/mongodb-kubernetes-operator-version-upgrade-post-start-hook:1.0.2
                - name: READINESS_PROBE_IMAGE
                  value: quay.io/mongodb/mongodb-kubernetes-readinessprobe:1.0.4
                - name: MONGODB_IMAGE
                  value: mongo
                - name: MONGODB_REPO_URL
                  value: docker.io
                image: quay.io/mongodb/mongodb-kubernetes-operator:0.7.0
                imagePullPolicy: Always
                name: mongodb-kubernetes-operator
                resources:
                  limits:
                    cpu: 1100m
                    memory: 1Gi
                  requests:
                    cpu: 500m
                    memory: 200Mi
              serviceAccountName: mongodb-kubernetes-operator
        EOF
 
4. Create a certificate authority and issue certificates for the services used to access the mongodb:

        TODO:

4. Create a MongoDB community version 4.2.6 instance with 3 replicas using the code shown below:

        cat <<EOF |oc apply -f -
        ---
        apiVersion: mongodbcommunity.mongodb.com/v1
        kind: MongoDBCommunity
        metadata:
          name: my-mongodb
          namespace: mongodb  
        spec:
          members: 3
          type: ReplicaSet
          version: "4.2.6"
          featureCompatibilityVersion: "4.0" 
          security:
            authentication:
              modes: ["SCRAM"]     
          users:
            - name: mongodbuser
              db: admin
              passwordSecretRef:
                name: my-user-password
              roles:
                - name: root
                  db: admin
                - name: clusterAdmin
                  db: admin
                - name: userAdminAnyDatabase
                  db: admin
              scramCredentialsSecretName: my-scram
          persistent: true
          statefulSet:
            spec:
              serviceName: my-mongodb-svc
              selector: {}    
              template:
                spec:
                  containers:
                    - name: mongod
                      env:
                      - name: MANAGED_SECURITY_CONTEXT
                        value: "true"
                      resources:
                        requests:
                          cpu: 1
                          memory: 1Gi
                        limits:
                          cpu: 3
                          memory: 5Gi
                    - name: mongodb-agent
                      env:
                      - name: MANAGED_SECURITY_CONTEXT
                        value: "true"            
                      resources:
                        requests:
                          memory: 50Mi
                        limits:
                          cpu: 500m
                          memory: 256Mi
              volumeClaimTemplates:
                - metadata:
                    name: data-volume
                  spec:
                    accessModes: ["ReadWriteOnce"]
                    storageClassName: ocs-storagecluster-ceph-rbd
                    resources:
                      requests:
                        storage: 1Gi
                - metadata:
                    name: logs-volume
                  spec:
                    accessModes: ["ReadWriteOnce"]
                    storageClassName: ocs-storagecluster-ceph-rbd
                    resources:
                      requests:
                        storage: 1Gi
        ---                
        apiVersion: v1
        kind: Secret
        metadata:
          name: my-user-password
        type: Opaque
        stringData:
          password: Password123.
        EOF            

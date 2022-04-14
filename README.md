# mongodb-community-installation-openshift
this guide shows how to install the mongodb community operator and create an instance on Openshift Container Platform (OCP).
To install the operator, follow these steps on a computer that can access the openshift 
cluster you wish to install MongoDB on.



1. Clone the repository and apply the custom resource definition for mongoDB:

    git clone https://github.com/mongodb/mongodb-kubernetes-operator.git

    oc apply -f mongodb-kubernetes-operator/config/crd/bases/mongodbcommunity.mongodb.com_mongodbcommunity.yaml
    oc get crd/mongodbcommunity.mongodbcommunity.mongodb.com


2. Create the project to use for MongoDB, and apply the Role Based Access Controls:
    
    oc new-project mongodb
    oc apply -k mongodb-kubernetes-operator/config/rbac/ --namespace mongodb
     

3. 

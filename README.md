## Create the entire infrastructure #EMPTY DATABASE
```console
oc apply -f mysql-all.yaml
```
## Set to newly created project
```console
oc project mysql-persistent
```
## Add some data
```console
oc exec dc/mysql -i -- mysql -u dbops -ppassword123 dbItems <<EOF
CREATE TABLE Student(
          Student_id INT AUTO_INCREMENT,  
          Student_name VARCHAR(100) NOT NULL,
          Student_Class VARCHAR(20) NOT NULL,
          TotalExamGiven INT   NOT NULL,
          PRIMARY KEY(Student_id )
);
INSERT INTO  
          Student(Student_name, Student_Class, TotalExamGiven)
VALUES
          ('Sayan', 'IX', 8),
          ('Nitin', 'X', 5),
          ('Aniket', 'XI', 6),
          ('Abdur', 'X', 7),
          ('Riya', 'IX', 4),
          ('Jony', 'X', 10),
          ('Deepak', 'X', 7),
          ('Ankana', 'XII', 5),
          ('Shreya', 'X', 8);
EOF
```

## Remove the DeploymentConfig and the pvc
```console
oc delete dc/mysql pvc/mysql
```

## Remove the bind from the only existing PV
```console
export DATA_PV=$(oc get pv -o name)
oc patch pv $DATA_PV \
    --type=json -p '[{"op": "remove", "path":"/spec/storageClassName"},{"op": "remove", "path":"/spec/claimRef"}]'
```

## Create PVC --will remain Pending
```console
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
  name: mysql-restored
  namespace: mysql-persistent
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 1Gi
EOF
```

## Associate the PV with the existintg PVC
```console
oc patch $DATA_PV -p '{"spec":{"claimRef":{"kind":"PersistentVolumeClaim","name":"mysql-restored","namespace":"mysql-persistent"}}}'
```

## Create a DeploymentConfig using that PV
```console
cat <<EOF | oc apply -f -
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  annotations:
    template.alpha.openshift.io/wait-for-ready: 'true'
  name: mysql-restored
  namespace: mysql-persistent
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    name: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: mysql-data
      labels:
        name: mysql
        app: mysql
    spec:
      containers:
      - env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              key: database-user
              name: mysql
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-password
              name: mysql
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-root-password
              name: mysql
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              key: database-name
              name: mysql
        image: "mysql"
        imagePullPolicy: IfNotPresent
        name: mysql
        ports:
        - containerPort: 3306
        resources:
          limits:
            memory: 512Mi
        volumeMounts:
        - mountPath: "/var/lib/mysql/data"
          name: mysql-data
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-restored
  triggers:
  - imageChangeParams:
      automatic: true
      containerNames:
      - mysql
      from:
        kind: ImageStreamTag
        name: mysql:8.0
        namespace: openshift
    type: ImageChange
  - type: ConfigChange
EOF
```

## Show restored data
```console
oc exec dc/mysql-restored -i -- mysql -u dbops -ppassword123 dbItems <<EOF
SELECT * FROM Student;
EOF
```

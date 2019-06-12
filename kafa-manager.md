apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-manager
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-manager
  template:
    metadata:
      labels:
        app: kafka-manager
    spec:
      containers:
      - name: kafka-manager
        image: solsson/kafka-manager@sha256:da602758fd542139284d1d106e7dfed78898e0f1eb9756597ab9aee8cfccd763
        ports:
        - containerPort: 80
        env:
        - name: ZK_HOSTS
          value: zk-cs:2181
        command:
        - ./bin/kafka-manager
        - -Dhttp.port=80
--
apiVersion: v1
kind: Service
metadata:
  name: kafka-manager
spec:
  selector:
    app: kafka-manager
  ports:    
    - port: 80
      targetPort: 80
      nodePort: 30100        
  type: NodePort

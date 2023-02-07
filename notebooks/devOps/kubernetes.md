# Kubernetes

[toc]

## 常用指令

### 重启组件

```shell
kubectl rollout restart deployment/ds/rs xxx
```

###  动态看状态

```shell
Kubectl get pod -w 
```



## Examples

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    author: Jerry
    date: 2021-03-11
  labels:
    tier: backend
    app: one-app
  name: one-app
spec:
  ports:
    - name: "one-app"
      port: 8080
      targetPort: 8080
  sessionAffinity: ClientIP
  selector:
    app: one-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    author: Jerry
    date: 2021-03-11
  labels:
    tier: backend
    app: one-app
  name: one-app
spec:
  selector:
    matchLabels:
      app: one-app
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: one-app
      labels:
        app: one-app
        tier: backend
    spec:
      containers:
        - env:
            - name: SPRING_PROFILES_ACTIVE
              value: docker
          image: colorlightwzg/one-app:beta-1.0
          name: one-app
          resources:
            requests:
              memory: "600Mi"
            limits:
              memory: "1000Mi"
          command:
            - java
            - -Xmx512m
            - -Xms512m
            - org.springframework.boot.loader.JarLauncher
          args:
            - --spring.cache.type=simple
            #- --spring.config.location=/var/lib/app/application.yml
            - --management.endpoint.health.probes.enabled=true
            - --management.health.livenessstate.enabled=true
            - --management.health.readinessstate.enabled=true
            - --file.static-resource-prefix=http://192.168.1.19:30080
            - --file.uploadFolder=/usr/share/nginx/html/backup/wp-content/uploads/
            - --spring.datasource.url=jdbc:mysql://one-mysql:3306/spring?zeroDateTimeBehavior=convertToNull&characterEncoding=utf-8&useSSL=false&allowPublicKeyRetrieval=true
#          lifecycle:
#            preStop:
#              exec:
#                command: [ "sh", "-c", "sleep 30" ]
          livenessProbe:
            initialDelaySeconds: 30
            timeoutSeconds: 5
            periodSeconds: 5
            httpGet:
              port: 8080
              path: /actuator/health/liveness
          readinessProbe:
            initialDelaySeconds: 40
            timeoutSeconds: 5
            periodSeconds: 5
            httpGet:
              port: 8080
              path: /actuator/health/readiness
          volumeMounts:
            - mountPath: /usr/share/nginx/html/backup/wp-content/uploads
              name: media-upload-data
            - mountPath: /logs
              name: one-app-log
            - mountPath: /var/lib/app
              name: one-app-config
      restartPolicy: Always
      volumes:
        - name: media-upload-data
          persistentVolumeClaim:
            claimName: media-upload-data
        - name: one-app-config
          configMap:
            name: one-app-springboot-config
        - name: one-app-log
          emptyDir: {}
---
```


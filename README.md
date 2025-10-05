خیلی عالی 👌
الان تو می‌خوای یک README کامل داشته باشی که هم شامل **راه‌اندازی Keycloak با Docker Compose** باشه، هم **روی Kubernetes** با سرویس، StatefulSet، دیتابیس Postgres و Ingress. من برات یک README حرفه‌ای درست می‌کنم که هر دو حالت رو پوشش بده.

---

# Keycloak with PostgreSQL

این پروژه نمونه‌ای از اجرای **Keycloak** به همراه دیتابیس **PostgreSQL** را در دو محیط فراهم می‌کند:

* **Docker Compose** برای تست سریع روی لوکال
* **Kubernetes (مانند Minikube / K8s Cluster)** برای محیط‌های ابری و توسعه پیشرفته

---

## 📂 فایل‌ها

### 1. `.env`

```env
POSTGRES_DB=keycloak_db
POSTGRES_USER=keycloak_db_user
POSTGRES_PASSWORD=keycloak_db_user_password
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=password
```

### 2. `docker-compose.yaml`

```yaml
version: '3.7'

services:
  postgres:
    image: postgres:16.2
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - keycloak_network

  keycloak:
    image: quay.io/keycloak/keycloak:23.0.6
    command: start
    environment:
      KC_HOSTNAME: localhost
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN}
      KC_HOSTNAME_PORT: 8080
      KC_HOSTNAME_STRICT_BACKCHANNEL: "false"
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT_HTTPS: "false"
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - 8080:8080
    restart: always
    depends_on:
      - postgres
    networks:
      - keycloak_network

volumes:
  postgres_data:
    driver: local

networks:
  keycloak_network:
    driver: bridge
```

---

## 🚀 اجرای Keycloak با Docker Compose

```bash
docker-compose up -d
```

سپس روی مرورگر باز کنید:

👉 [http://localhost:8080](http://localhost:8080)

* **Username:** مقدار `KEYCLOAK_ADMIN`
* **Password:** مقدار `KEYCLOAK_ADMIN_PASSWORD`

برای توقف سرویس‌ها:

```bash
docker-compose down
```

---

## ☸️ اجرای Keycloak روی Kubernetes

### 1. Service + StatefulSet + Postgres + Ingress

فایل `k8s-keycloak.yaml` شامل همه منابع است:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  labels:
    app: keycloak
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: http
      name: http
  selector:
    app: keycloak
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: keycloak
  name: keycloak-discovery
spec:
  selector:
    app: keycloak
  publishNotReadyAddresses: true
  ClusterIP: None
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
  labels:
    app: keycloak
spec:
  serviceName: keycloak-discovery
  replicas: 2
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:26.3.2
          args: ["start"]
          env:
            - name: KC_BOOTSTRAP_ADMIN_USERNAME
              value: "admin"
            - name: KC_BOOTSTRAP_ADMIN_PASSWORD
              value: "admin"
            - name: KC_PROXY_HEADERS
              value: "xforwarded"
            - name: KC_HTTP_ENABLED
              value: "true"
            - name: KC_HOSTNAME_STRICT
              value: "false"
            - name: KC_HEALTH_ENABLED
              value: "true"
            - name: KC_CACHE
              value: ispn
            - name: KC_CACHE_STACK
              value: kubernetes
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: JAVA_OPTS_APPEND
              value: '-Djgroups.dns.query="keycloak-discovery" -Djgroups.bind.address=$(POD_IP)'
            - name: KC_DB_URL_DATABASE
              value: keycloak
            - name: KC_DB_URL_HOST
              value: postgres
            - name: KC_DB
              value: postgres
            - name: KC_DB_PASSWORD
              value: keycloak
            - name: KC_DB_USERNAME
              value: keycloak
          ports:
            - name: http
              containerPort: 8080
          startupProbe:
            httpGet:
              path: /health/started
              port: 9000
            periodSeconds: 1
            failureThreshold: 600
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 9000
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 9000
            periodSeconds: 10
            failureThreshold: 3
          resources:
            limits:
              cpu: 2000m
              memory: 2000Mi
            requests:
              cpu: 500m
              memory: 1700Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: mirror.gcr.io/postgres:17
          env:
            - name: POSTGRES_USER
              value: "keycloak"
            - name: POSTGRES_PASSWORD
              value: "keycloak"
            - name: POSTGRES_DB
              value: "keycloak"
          ports:
            - name: postgres
              containerPort: 5432
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql
      volumes:
        - name: postgres-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: postgres
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
spec:
  tls:
    - hosts:
        - keycloak.192.168.49.2.nip.io
  rules:
    - host: keycloak.192.168.49.2.nip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak
                port:
                  number: 8080
```

---

## 🚀 اجرای Keycloak روی Kubernetes

1. اگر Minikube استفاده می‌کنید:

```bash
minikube addons enable ingress
```

2. سپس منابع را اعمال کنید:

```bash
kubectl apply -f k8s-keycloak.yaml
```

3. آدرس Keycloak از طریق Ingress در دسترس است:

👉 [https://keycloak.192.168.49.2.nip.io](https://keycloak.192.168.49.2.nip.io)

---

## 📌 نکات Production

* برای دیتابیس Postgres حتماً از **PersistentVolume** استفاده کنید.
* برای مدیریت پسوردها و کاربرها از **Kubernetes Secret** استفاده کنید.
* حتماً Keycloak را پشت یک **Reverse Proxy (NGINX / Traefik)** با TLS واقعی قرار دهید.

---

می‌خوای من برات یک **دیاگرام معماری** هم بکشم (Docker Compose vs Kubernetes) تا دید بهتری داشته باشی؟


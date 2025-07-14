اجرای **Blue-Green Deployment** با **Argo CD** کاملاً شدنیه و در فضای GitOps بسیار مناسب هست چون به شما اجازه می‌ده بدون downtime و با کنترل دقیق، نسخه جدید رو مستقر کنی و در صورت نیاز سریع rollback بزنی.

در ادامه گام‌به‌گام راهکار **اجرای Blue-Green Deployment در Argo CD** رو توضیح می‌دم، با دو سناریوی ممکن:

---

## 🚀 سناریوی پیشنهادی برای Blue-Green Deployment در Argo CD

### 📌 معماری پایه:

* دو نسخه از برنامه (Blue و Green) هم‌زمان Deploy می‌شن.
* فقط یکی از اون‌ها در هر لحظه ترافیک واقعی می‌گیره (با استفاده از سرویس).
* وقتی نسخه جدید Deploy شد (مثلاً Green)، با تغییر Service، ترافیک از Blue به Green منتقل می‌شه.
* اگه مشکلی پیش اومد، Service به راحتی به Blue برگردونده می‌شه.

---

## ✅ روش 1: استفاده از دو Deployment + یک Service

### 💡 ساختار فایل‌ها در Git

```
k8s/
├── blue/
│   └── deployment.yaml
├── green/
│   └── deployment.yaml
└── service.yaml
```

### 🔹 blue/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: your-image:1.0
```

### 🔹 green/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: your-image:1.1
```

### 🔹 service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: green   # این‌جا تعیین می‌کنی ترافیک کجا بره
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## 🧠 گردش کار (در GitOps):

1. Argo CD برنامه Blue را Deploy می‌کنه.
2. `service.yaml` رو تنظیم می‌کنی که به `version: blue` اشاره کنه.
3. وقتی آماده‌ی انتشار نسخه جدید هستی:

   * فایل green/deployment.yaml رو اضافه می‌کنی (با نسخه جدید image).
   * فقط یک commit نیاز هست تا Argo CD sync کنه.
4. وقتی مطمئن شدی Green خوب کار می‌کنه:

   * در `service.yaml` مقدار `version` رو از `blue` به `green` تغییر می‌دی.
5. اگر مشکلی بود:

   * فقط `service.yaml` رو برمی‌گردونی به `blue`.

این دقیقاً فلسفه‌ی GitOps و Blue-Green با هم هست ✅

---

## ✅ روش 2: با Helm chart (ساده‌تر با Templateها)

اگر از Helm استفاده می‌کنی، می‌تونی یک `values.yaml` داشته باشی:

```yaml
appVersion: green
image:
  tag: 1.1
```

و در `deployment.yaml` داخل chart:

```yaml
labels:
  version: {{ .Values.appVersion }}

selector:
  matchLabels:
    version: {{ .Values.appVersion }}
```

و همینطور در `service.yaml`:

```yaml
selector:
  version: {{ .Values.appVersion }}
```

✅ حالا فقط با تغییر مقدار `appVersion` در فایل `values.yaml`، نسخه‌ی active رو تغییر می‌دی. عالی برای Argo CD و GitOps.

---

## 🔄 اتصال این سناریو به Argo CD

در Argo CD یک اپلیکیشن تعریف می‌کنی:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: HEAD
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

---

## 📋 مزایا این روش

* ✅ مدیریت کامل از طریق Git (همه چیز declarative)
* ✅ rollback با یک commit ساده
* ✅ بدون downtime
* ✅ قابل مانیتورینگ از طریق Argo UI و diff

---

اگر خواستی من می‌تونم تمام فایل‌های YAML یا Helm chart رو برات بنویسم. فقط بگو:

* اسم برنامه
* آدرس imageها
* از Helm استفاده می‌کنی یا نه؟
* namespace مورد نظر

تا در چند دقیقه برات آماده کنم.

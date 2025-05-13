

## 🎯 هدف نهایی ما چیه؟

می‌خوایم:

1. یه اپلیکیشن ساده داشته باشیم (مثلاً یه Flask app).
2. کانفیگ Kubernetes این اپ رو توی یه ریپوی Git بذاریم.
3. هر وقت این فایل‌ها در Git تغییر کردن، Flux به صورت خودکار اون تغییرات رو روی کلاستر K8s اجرا کنه (deploy/update).

---

## ⚙️ ابزارهایی که استفاده می‌کنیم:

| ابزار          | وظیفه                                                 |
| -------------- | ----------------------------------------------------- |
| **Flux**       | اجرای خودکار deploy از روی Git                        |
| **Git**        | محلی برای ذخیره فایل‌های Kubernetes (source of truth) |
| **Kubernetes** | جایی که اپ ما اجرا می‌شه                              |

---

## 🔰 مرحله 1: اپلیکیشن و فایل Kubernetes

فرض کن یه پروژه ساده داریم:

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: yourregistry/flask-app:latest
        ports:
        - containerPort: 5000
```

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

---

## 🧩 مرحله 2: آماده‌سازی GitOps ریپو

ساختار ریپو به شکل زیره:

```
gitops-configs/
└── flask-app/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

### `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

---

## 🚀 مرحله 3: نصب Flux روی کلاستر

با ابزار [`flux`](https://fluxcd.io/flux/installation/) CLI این کار رو می‌کنی:

```bash
flux install
```

این دستور کنترلرهای لازم رو توی namespace `flux-system` نصب می‌کنه.

---

## 🔗 مرحله 4: اتصال Flux به Git

حالا باید به Flux بگیم که از کدوم ریپو فایل‌های K8s رو بخونه.

### 1. تعریف `GitRepository`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flask-repo
  namespace: flux-system
spec:
  interval: 1m0s
  url: https://gitlab.com/youruser/gitops-configs.git
  ref:
    branch: main
```

### 2. تعریف `Kustomization`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  path: ./flask-app
  prune: true
  sourceRef:
    kind: GitRepository
    name: flask-repo
  targetNamespace: default
```

---

## 📦 مرحله 5: apply کردن manifest‌ها

فایل‌های بالا رو با دستور زیر روی کلاستر apply کن:

```bash
kubectl apply -f gitrepository.yaml
kubectl apply -f kustomization.yaml
```

---

## ✅ نتیجه نهایی

بعد از 1 دقیقه، Flux:

1. فایل‌های `deployment.yaml` و `service.yaml` رو از Git می‌خونه.
2. اگر جدید هستن یا تغییر کردن → deploy/update می‌کنه.
3. شما فقط کافیه فایل‌های K8s رو توی Git تغییر بدی، Flux خودش بقیه رو انجام می‌ده.

---

## 🔁 اگر چیزی تغییر کنه چی میشه؟

مثلاً شما توی Git تصویر container رو تغییر می‌دی از:

```yaml
image: yourregistry/flask-app:latest
```

به:

```yaml
image: yourregistry/flask-app:v2
```

Flux تغییر رو می‌بینه و خودش به صورت خودکار `kubectl apply` انجام می‌ده.

---

✅ **همین بود!**
ساده‌ترین مدل GitOps با Flux بدون automation اضافی.


عالی! حالا که با Flux و GitOps ساده آشنا شدی، بریم سراغ **مرحله بعدی: Image Automation** ✨

---

## 🎯 هدف ما در این مرحله:

> وقتی یک image جدید (مثلاً `v1.2.0`) توی ریجستری منتشر می‌شه، Flux خودش:
>
> * tag جدید رو پیدا کنه
> * فایل Kustomization (یا Deployment) رو توی Git آپدیت کنه
> * commit بزنه و push کنه
> * و چون فایل Git عوض شده، Flux دوباره deploy کنه ✅

---

## 🧩 اجزای مورد نیاز:

برای image automation توی Flux به ۳ جزء اصلی نیاز داریم:

| CRD                     | وظیفه                                            |
| ----------------------- | ------------------------------------------------ |
| `ImageRepository`       | بررسی tagهای جدید در container registry          |
| `ImagePolicy`           | انتخاب بهترین tag طبق قانون (مثلاً آخرین semver) |
| `ImageUpdateAutomation` | آپدیت کردن فایل Kustomize و commit/push به Git   |

---

## ⚙️ مرحله به مرحله

فرض کن image توی GitLab Registry هست:
`registry.gitlab.com/youruser/flask-app`

---

### 1️⃣ `ImageRepository`

می‌گه کدوم image رو چک کن و هر چند وقت یک‌بار.

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: flask-app
  namespace: flux-system
spec:
  image: registry.gitlab.com/youruser/flask-app
  interval: 1m
```

---

### 2️⃣ `ImagePolicy`

قانونی برای انتخاب tag مناسب. مثلاً آخرین نسخه‌ای که با SemVer شروع می‌شه:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: flask-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: flask-app
  policy:
    semver:
      range: ">=1.0.0"
```

---

### 3️⃣ به‌روزرسانی `kustomization.yaml`

تو فایل `apps/flask-app/kustomization.yaml` این خط باید باشه:

```yaml
images:
  - name: registry.gitlab.com/youruser/flask-app
    newTag: "1.0.0"  # این مقدار توسط Flux آپدیت می‌شه
```

Flux این tag رو تغییر می‌ده و commit می‌کنه.

---

### 4️⃣ `ImageUpdateAutomation`

این آخرین قطعه‌ست که اجازه می‌ده Flux فایل رو تغییر بده و commit کنه:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flask-repo
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: Flux Bot
        email: flux@example.com
      messageTemplate: "chore: update image to {{ .NewTag }}"
    push:
      branch: main
```

---

## 🧾 نکته مهم: دسترسی نوشتن به Git

برای اینکه Flux بتونه روی ریپوی Git شما commit بزنه و push کنه، باید:

1. یک deploy key یا personal access token بسازی.
2. توی secret (داخل `flux-system`) ذخیره‌اش کنی.
3. Flux رو با اون key تنظیم کنی (موقع bootstrap یا دستی).

اگر خواستی، می‌تونم برات بگم چطور این secret رو بسازی و وصلش کنی.

---

## 🧪 تست نهایی

1. یه image جدید push کن (مثلاً `v1.3.0`)
2. Flux tag جدید رو شناسایی می‌کنه (با ImageRepository)
3. بررسی می‌کنه که آیا tag معتبر هست (با ImagePolicy)
4. فایل kustomization رو در Git تغییر می‌ده (با ImageUpdateAutomation)
5. commit و push می‌کنه
6. چون فایل جدید شده، Flux خودش deploy می‌کنه 🎉

---

✅ این شد یک سیستم **Full GitOps + Image Automation** با Flux.




خیلی خب، بریم یه پروژه کامل بسازیم که شامل:

✅ فایل‌های Kubernetes برای یه اپ ساده
✅ ساختار GitOps
✅ Flux manifests شامل:

* GitRepository
* Kustomization
* ImageRepository
* ImagePolicy
* ImageUpdateAutomation
  ✅ آماده برای تست توی یه کلاستر Flux نصب‌شده

---

## 📁 ساختار پوشه ریپوی GitOps

```
gitops-configs/
├── apps/
│   └── flask-app/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── clusters/
│   └── prod/
│       └── kustomization.yaml
└── flux-system/
    ├── gitrepository.yaml
    ├── imagerepository.yaml
    ├── imagepolicy.yaml
    └── imageupdate.yaml
```

---

### 🔧 1. `apps/flask-app/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: registry.gitlab.com/youruser/flask-app:1.0.0
          ports:
            - containerPort: 5000
```

---

### 🔧 2. `apps/flask-app/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 5000
```

---

### 🔧 3. `apps/flask-app/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

images:
  - name: registry.gitlab.com/youruser/flask-app
    newTag: 1.0.0  # این توسط Flux به‌روز می‌شه
```

---

### 🔧 4. `clusters/prod/kustomization.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/flask-app
  prune: true
  sourceRef:
    kind: GitRepository
    name: flask-repo
  targetNamespace: default
```

---

### 🔧 5. `flux-system/gitrepository.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flask-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/youruser/gitops-configs.git
  ref:
    branch: main
```

---

### 🔧 6. `flux-system/imagerepository.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: flask-app
  namespace: flux-system
spec:
  image: registry.gitlab.com/youruser/flask-app
  interval: 1m
```

---

### 🔧 7. `flux-system/imagepolicy.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: flask-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: flask-app
  policy:
    semver:
      range: ">=1.0.0"
```

---

### 🔧 8. `flux-system/imageupdate.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flask-repo
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: Flux Bot
        email: flux@example.com
      messageTemplate: "chore: update image to {{ .NewTag }}"
    push:
      branch: main
```

---

## 🪪 مرحله آخر: دسترسی نوشتن به Git

برای اینکه Flux بتونه فایل رو commit و push کنه:

1. یه **deploy key یا personal access token** برای GitLab بساز.
2. اون رو به شکل secret به flux بدی (مثلاً با `flux create secret git`).
3. موقع bootstrap یا دستی این secret رو به `ImageUpdateAutomation` مربوط کن.

اگر خواستی دستور ساخت secret مخصوص GitLab رو هم می‌نویسم.

---

## ✅ نتیجه

هر بار که image جدید (مثلاً `1.0.1`) push بشه:

1. Flux اونو از registry می‌خونه
2. بررسی می‌کنه که این tag مطابق policy هست
3. فایل `kustomization.yaml` در Git به‌روزرسانی می‌شه
4. commit → push → و بعد deploy انجام می‌شه 🎉

---

اگر می‌خوای این فایل‌ها رو یه جا توی یه ریپو واقعی داشته باشی، می‌تونم یه قالب ریپو برات بسازم یا به شکل فایل فشرده (`.zip`) برات درست کنم. کدومش رو ترجیح می‌دی؟





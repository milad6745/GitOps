خیلی خوب! حالا که گفتی سرور Argo CD شما روی IP **`192.168.1.30`** قرار داره، می‌تونیم تنظیمات رو به شکل دقیق‌تر انجام بدیم. سناریوی شما به این صورت خواهد بود:

---

## 🎯 سناریو کلی:

* ✅ Argo CD نصب شده روی کلاستر اول (مثلاً 192.168.1.10)، و IP سرورش **`192.168.1.30`**
* ✅ یک کلاستر دوم با IP **192.168.1.20** وجود دارد.
* ✅ می‌خواهی برای هر کلاستر، یک اپلیکیشن جداگانه بسازی.
* ✅ سورس اپ‌ها در GitHub هستند.

---

## 🧱 مراحل نهایی با IP مشخص:

### 🔹 مرحله 1: ورود به Argo CD CLI

```bash
argocd login 192.168.1.30 --username admin --password <your-password> --insecure
```

> `--insecure` در صورتی نیاز است که Argo CD با TLS معتبر اجرا نشده باشد.

---

### 🔹 مرحله 2: اضافه کردن کلاستر دوم به Argo CD

اول باید روی سیستمی که Argo CLI نصب است، kubeconfig دسترسی به هر دو کلاستر را داشته باشی.

#### 1. چک کردن contextها:

```bash
kubectl config get-contexts
```

فرض کنیم context کلاستر دوم به صورت `cluster2` هست.

#### 2. اضافه کردن کلاستر دوم:

```bash
argocd cluster add cluster2
```

بعد از این مرحله، با دستور زیر لیست کلاسترهای متصل را ببین:

```bash
argocd cluster list
```

---

### 🔹 مرحله 3: ساخت دو اپلیکیشن (برای هر کلاستر)

#### 🟢 اپلیکیشن برای کلاستر اول (که Argo CD روش نصبه):

```bash
argocd app create app-on-cluster1 \
  --repo https://github.com/your/repo.git \
  --path path/to/app1 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

#### 🔵 اپلیکیشن برای کلاستر دوم (192.168.1.20):

```bash
argocd app create app-on-cluster2 \
  --repo https://github.com/your/repo.git \
  --path path/to/app2 \
  --dest-server https://192.168.1.20:6443 \
  --dest-namespace default \
  --sync-policy automated
```

> توجه: آدرس `https://192.168.1.20:6443` مربوط به API سرور کلاستر دوم است. اگر `argocd cluster add` موفق بوده، می‌تونی مقدار دقیق `--dest-server` را از خروجی `argocd cluster list` برداری.

---

### 🔹 مرحله 4: Sync کردن برنامه‌ها

```bash
argocd app sync app-on-cluster1
argocd app sync app-on-cluster2
```

---

## ✅ نکته پایانی

اگر GitHub private است، باید repo credentials را در Argo CD اضافه کنی:

```bash
argocd repo add https://github.com/your/repo.git \
  --username <your-username> --password <your-password>
```

---

اگر بخوای من فایل YAML از نوع `Application` برای هر کلاستر درست کنم (برای GitOps واقعی)، فقط این اطلاعات رو بده:

* آدرس GitHub repo
* مسیر دایرکتوری app1 و app2 داخل اون repo
* namespace دلخواه برای deploy
* آیا از Helm استفاده می‌کنی یا فایل YAML خالص؟

تا برات کامل آماده کنم ✅

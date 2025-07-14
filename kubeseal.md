ابزار **`kubeseal`** یکی از ابزارهای متن‌باز ارائه‌شده توسط پروژه [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) از Bitnami است. این ابزار به منظور **مدیریت امن Secretsها در Kubernetes** در کنار GitOps (مثل ArgoCD یا FluxCD) طراحی شده است.

---

### 🔐 **مشکل اصلی که kubeseal حل می‌کند**

در محیط‌هایی مثل GitOps، شما فایل‌های Kubernetes را در مخازن Git نگه‌داری می‌کنید. اما اگر Secrets (مثل توکن‌ها، رمزها، کلیدها و ...) را به صورت *متن واضح (plaintext)* در Git قرار دهید، ریسک امنیتی بسیار بالا خواهد بود.

---

### ✅ **راه‌حل: kubeseal + Sealed Secrets**

`kubeseal` فایل‌های Kubernetes از نوع `Secret` را به فرم **رمزنگاری‌شده (sealed)** تبدیل می‌کند که می‌توان با خیال راحت در Git قرار داد. این فایل‌ها فقط توسط **Sealed Secrets Controller** در داخل خوشه (cluster) رمزگشایی و استفاده می‌شوند.

---

### ⚙️ نحوه عملکرد کلی:

1. شما یک فایل `Secret` می‌نویسید:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: my-secret
     namespace: default
   type: Opaque
   data:
     password: cGFzc3dvcmQ=   # base64(password)
   ```

2. با `kubeseal` این را رمزنگاری می‌کنید:

   ```bash
   kubeseal < secret.yaml > sealedsecret.yaml --controller-name=sealed-secrets --controller-namespace=kube-system --format=yaml
   ```

3. فایل `sealedsecret.yaml` به شکل زیر است (قابل ذخیره در Git بدون نگرانی):

   ```yaml
   apiVersion: bitnami.com/v1alpha1
   kind: SealedSecret
   metadata:
     name: my-secret
     namespace: default
   spec:
     encryptedData:
       password: Ag...long-encrypted-string...
   ```

4. در خوشه، Sealed Secrets Controller آن را به Secret واقعی رمزگشایی و Deploy می‌کند.

---

### 🎯 مزایای استفاده از `kubeseal`

| مزیت                   | توضیح                                                                                     |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| **امنیت بالا**         | داده‌ها رمزنگاری شده‌اند و فقط داخل همان کلاستر قابل رمزگشایی هستند.                      |
| **سازگار با GitOps**   | می‌توان فایل‌های رمزنگاری‌شده را در Git نگه داشت و با ابزارهایی مانند ArgoCD استفاده کرد. |
| **کنترل ساده‌تر نسخه** | تغییرات روی Secrets مثل سایر منابع قابل بررسی و version control است.                      |
| **پشتیبانی از CI/CD**  | قابل استفاده در pipelineهای اتوماتیک برای deploy امن.                                     |

---

### 🛠️ پیش‌نیازها

* نصب `kubeseal` روی سیستم شما
* نصب `Sealed Secrets Controller` در کلاستر Kubernetes

---

اگر خواستی مثال عملی یا راه‌اندازی گام‌به‌گامش رو هم برات بنویسم، خوشحال می‌شم کمک کنم.

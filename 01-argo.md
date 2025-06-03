سؤال عالیه! در ادامه، قدم‌به‌قدم نصب **Argo CD** و اتصال اون به یک **مخزن Git** رو برات توضیح می‌دم.

---

## 🎯 هدف:

نصب Argo CD روی یک کلاستر Kubernetes و اتصال اون به یک repository گیت برای پیاده‌سازی **GitOps**.

---

## 🛠️ پیش‌نیازها:

* یک **کلاستر Kubernetes** (مثلاً Minikube، Kind یا کلاستر ابری مثل GKE، EKS، AKS)
* نصب بودن `kubectl` و دسترسی به کلاستر
* نصب بودن `argocd` CLI (اختیاری ولی مفید)
* یک **repository گیت** که manifestهای Kubernetes شما (به فرمت YAML) داخلش باشه

---

## 🚀 مراحل نصب Argo CD

### 1. نصب Argo CD در کلاستر Kubernetes:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

این دستور Argo CD رو در namespace `argocd` نصب می‌کنه.

---

### 2. دسترسی به رابط گرافیکی (UI):

می‌تونید Argo CD UI رو با `port-forward` کردن ببینید:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

سپس در مرورگر باز کنید:

```
https://localhost:8080
```

---

### 3. دریافت رمز عبور اولیه:

نام کاربری پیش‌فرض: `admin`

رمز عبور پیش‌فرض:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## 🔗 اتصال به Git Repository

### 1. اضافه کردن Git Repo:

با استفاده از `argocd` CLI یا از طریق UI می‌تونی ریپو رو اضافه کنی.

#### روش CLI:

```bash
argocd login localhost:8080
argocd repo add https://github.com/USERNAME/REPO_NAME.git --username YOUR_USERNAME --password YOUR_PASSWORD
```

> اگر repo پابلیک باشه، نیازی به username/password نیست.

---

## 🎯 ساخت یک برنامه از روی Git

بله، کاملاً درست می‌گی! Argo CD از **SSH key** برای اتصال به Git repository (مثل GitLab یا GitHub) پشتیبانی می‌کنه، مخصوصاً برای repoهای خصوصی. استفاده از **SSH key authentication** به جای username/password یا token هم **امن‌تر** هست و هم در محیط‌های CI/CD رایج‌تر.

در ادامه روش اتصال Argo CD به GitLab با استفاده از **SSH key** رو برات قدم‌به‌قدم می‌نویسم:

---

## 🔐 مرحله‌به‌مرحله اتصال Argo CD به GitLab با SSH Key

### 🧱 ۱. ایجاد SSH Key (اگه نداری)

روی سیستم خودت یا یه سرور:

```bash
ssh-keygen -t ed25519 -C "argo@yourdomain.com"
```

بعد از اجرا، دو فایل ساخته می‌شه:

* کلید خصوصی (مثلاً: `~/.ssh/id_ed25519`)
* کلید عمومی (مثلاً: `~/.ssh/id_ed25519.pub`)

---

### 🔗 ۲. اضافه کردن کلید عمومی به GitLab

1. وارد GitLab شو
2. برو به: `User Settings` > `SSH Keys`
3. محتوای فایل `id_ed25519.pub` رو کپی کن
4. داخل GitLab Paste کن و Save بزن

> حالا GitLab آماده هست تا از طریق SSH اجازه دسترسی بده.

---

### ⚙️ ۳. اضافه کردن Git Repo به Argo CD با SSH

ابتدا namespace `argocd` رو تنظیم کن:

```bash
kubectl config set-context --current --namespace=argocd
```

سپس کلید خصوصی رو به عنوان Secret اضافه کن:

```bash
argocd repo add git@gitlab.com:USERNAME/REPO_NAME.git \
  --ssh-private-key-path ~/.ssh/id_ed25519
```

> توجه: آدرس `git@...` باید **SSH URL** باشه نه HTTPS.

اگر `argocd` CLI نداری، می‌تونی این Secret رو به صورت دستی با YAML هم بسازی، بگو تا برات مثالش رو بیارم.

---

### 🧪 بررسی صحت اتصال

```bash
argocd repo list
```

اگر همه‌چی درست باشه، repo شما باید توی لیست بیاد و `STATUS: Successful` باشه.

---

### ⛔ اگر با خطای fingerprint مواجه شدی

Argo باید fingerprint سرور GitLab رو بشناسه. برای اضافه کردنش:

```bash
ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```

یا اگر از داخل کلاستر نیاز داری:

```bash
kubectl exec -n argocd deployment/argocd-repo-server -c argocd-repo-server -- \
  ssh-keyscan gitlab.com >> /etc/ssh/ssh_known_hosts
```

---

اگر خواستی با GitLab CI یا GitOps کامل راه بندازیم، می‌تونم یه فایل `.gitlab-ci.yml` و پیکربندی کامل GitLab و Argo CD رو هم برات تنظیم کنم.





### روش CLI:

```bash
argocd app create my-app \
  --repo https://github.com/USERNAME/REPO_NAME.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

* `--path`: پوشه‌ای در repo که manifestها داخلش هست.
* `--dest-namespace`: جایی که برنامه deploy می‌شه.

### بعدش:

```bash
argocd app sync my-app
```

یا از طریق UI روی "Sync" کلیک کن.

---

## ✅ نتیجه:

بعد از این مراحل، Argo CD به repo شما متصل می‌شه، تغییرات جدید رو بررسی می‌کنه، و به صورت خودکار یا دستی اون‌ها رو در Kubernetes deploy می‌کنه.

---

اگه خواستی برات یه ریپوی ساده تستی بسازم که بتونی سریع امتحانش کنی، فقط بگو!

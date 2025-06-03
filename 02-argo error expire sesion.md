## Error token expire
 argocd repo list
{"level":"fatal","msg":"rpc error: code = Unauthenticated desc = invalid session: token has invalid claims: token is expired","time":"2025-06-03T15:58:45+03:30"}


این خطا نشون می‌ده که **توکن ورود به Argo CD منقضی شده** و باید دوباره **وارد Argo CD بشی (Login)** تا بتونی از `argocd` CLI استفاده کنی.

---

## ✅ راه‌حل:

### 1. دوباره لاگین کن به Argo CD:

```bash
argocd login localhost:8080 --username admin --password YOUR_PASSWORD --insecure
```

توضیحات:

* اگر داری `port-forward` استفاده می‌کنی، از `localhost:8080` استفاده کن.
* از `--insecure` استفاده کن چون داریم از HTTPS روی لوکال استفاده می‌کنیم ولی گواهینامه (TLS cert) معتبر نیست.
* `YOUR_PASSWORD` رو از secret اولیه می‌تونی دوباره بگیری (در پایین آوردم).

---

### 2. گرفتن پسورد ادمین:

اگه رمز عبور ادمین رو نداری یا فراموش کردی، این دستور رو بزن:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> بعد از لاگین موفق، توکن جدید تولید می‌شه و دستوراتی مثل `argocd repo list` بدون خطا کار می‌کنن.

---

### 💡 پیشنهاد امنیتی:

برای جلوگیری از این مشکل در آینده، می‌تونی:

* از توکن سرویس‌اکانت (`ServiceAccount`) پایدار استفاده کنی.
* یا توکن‌های session رو با مدت طولانی‌تری تنظیم کنی (در کانفیگ Argo CD قابل تغییر هست).



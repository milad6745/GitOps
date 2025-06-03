برای تغییر رمز عبور کاربر (مثلاً `admin`) در Argo CD، می‌تونی یکی از این دو روش رو استفاده کنی:

---

## 🔐 روش 1: تغییر رمز عبور از طریق `argocd` CLI

اگر به سیستم لاگین کرده‌ای، خیلی ساده:

```bash
argocd account update-password
```

این دستور ازت می‌پرسه:

```
Enter current password:
Enter new password:
Confirm new password:
```

✔️ بعد از وارد کردن درست، رمز جدید تنظیم می‌شه.

---

## 🔧 روش 2: تغییر رمز عبور از طریق Kubernetes (برای `admin`)

اگه نمی‌تونی وارد بشی (مثلاً رمز قبلی رو نداری)، می‌تونی رمز `admin` رو به صورت دستی از طریق secret تغییر بدی:

### 1. ساخت رمز جدید (base64 encoded):

```bash
echo -n 'your-new-password' | base64
```

فرض کن خروجی شد: `eW91ci1uZXctcGFzc3dvcmQ=`

### 2. ویرایش secret:

```bash
kubectl -n argocd edit secret argocd-secret
```

در YAML باز شده دنبال این بخش بگرد:

```yaml
admin.password: ...
admin.passwordMtime: ...
```

و این دو مقدار رو با مقادیر جدید جایگزین کن:

```yaml
admin.password: eW91ci1uZXctcGFzc3dvcmQ=
admin.passwordMtime: "2025-06-03T13:00:00Z"  # زمان فعلی به فرمت ISO
```

✔️ حتماً `admin.passwordMtime` رو هم تغییر بده وگرنه Argo CD متوجه تغییر نمی‌شه.

### 3. ریستارت کردن podها:

```bash
kubectl -n argocd delete pod -l app.kubernetes.io/name=argocd-server
```

بعد از بالا اومدن مجدد، می‌تونی با رمز جدید وارد بشی.

---

اگر خواستی رمز رو از طریق UI هم بتونی تغییر بدی، بگو تا فعال‌سازی مدیریت کاربران از طریق Dex یا SSO رو هم برات توضیح بدم.

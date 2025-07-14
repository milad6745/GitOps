اگر از **Argo CD** برای CD (Continuous Deployment) استفاده می‌کنی و برنامه‌ات به GitHub وصل شده، برای **rollback** یا بازگرداندن به نسخه قبلی چند راه‌حل متفاوت وجود داره، بسته به اینکه دقیقاً هدفت چیه.

در ادامه، مراحل و روش‌های اصلی Rollback در Argo CD رو می‌بینی:

---

## ✅ روش 1: استفاده از History / Rollback در UI

Argo CD خودش تاریخچه Deploy ها رو نگه می‌داره:

1. وارد **Argo CD UI** شو.
2. روی اپلیکیشن مورد نظر کلیک کن.
3. در بالا، روی تب `History and Rollback` کلیک کن.
4. لیستی از Deployهای قبلی رو می‌بینی.
5. روی نسخه‌ای که می‌خوای برگردی کلیک کن.
6. روی دکمه `Rollback` کلیک کن.
7. تایید کن.

> ⚠️ این Rollback بر اساس **manifestهایی** هست که از Git گرفته شده؛ یعنی ممکنه به نسخه قبلی manifest برگرده اما **Git repo** همچنان در HEAD باقی بمونه (یعنی شما در Git برنگشتی).

---

## ✅ روش 2: Git-based Rollback (تغییر Git و Sync مجدد)

اگه می‌خوای از طریق Git برگردی (مثلاً به commit قبلی برگردی و بعد Argo CD اون رو Sync کنه):

1. برو داخل GitHub repo.
2. `git revert` یا `git checkout` روی کامیت قبلی انجام بده.

   ```bash
   git checkout <previous-commit>
   git push origin HEAD:main
   ```

   یا مثلاً:

   ```bash
   git revert <bad-commit>
   git push origin main
   ```
3. بعد از اینکه Git تغییر کرد، برگرد به Argo CD.
4. روی دکمه `Sync` کلیک کن یا از CLI:

   ```bash
   argocd app sync <app-name>
   ```

---

## ✅ روش 3: استفاده از CLI برای Rollback

Argo CD CLI هم اجازه rollback می‌ده:

1. تاریخچه نسخه‌ها رو ببین:

   ```bash
   argocd app history <app-name>
   ```

2. Rollback کن به یک revision خاص:

   ```bash
   argocd app rollback <app-name> <revision-id>
   ```

---

## ❗ نکات مهم

* اگه manifestها در Git تغییر نکردن و فقط از Helm استفاده می‌کنی، ممکنه rollback روی Helm release انجام بشه (در این صورت بهتره Helm chart repo هم versioned باشه).
* Rollback در Argo CD به صورت **declarative** هست: یعنی وضعیت فعلی رو با وضعیت تعریف‌شده در Git مقایسه و همگام‌سازی می‌کنه.

---

اگر برات مهمه که rollback دقیقاً به چه چیزی انجام بشه (image version؟ Helm release؟ env vars؟)، لطفاً بگو تا راهنمایی دقیق‌تری بدم.

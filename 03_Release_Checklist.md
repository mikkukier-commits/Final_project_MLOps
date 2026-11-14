# 03. Чек-лист готовності до релізу (Release Readiness Checklist)

Цей чек-лист є обов'язковим робочим інструментом команди **SupportFlow AI** перед кожним production-релізом (Canary 5% → 25% → 100%) нової версії LLM-адаптера, класифікатора намірів або оновлення RAG-пайплайну. Загалом документ містить **36 вимірюваних контрольних пунктів**, розподілених по 6 обов'язкових категоріях. Пункти з міткою **`Обмеження, якщо не спрацює: Yes`** автоматично зупиняють реліз; пункти **`Обмеження, якщо не спрацює: No`** фіксуються як технічний борг із дедлайном усунення до наступного спринту.

---

## Категорія 1: Model quality gates (7 пунктів)

1. **[ ] [Model quality gates] Класифікатор намірів (25 класів) досягає macro F1-score ≥ 0.92 на ізольованій hold-out вибірці (50,000 тикетів)**
   - **Власник:** ML Lead
   - **Докази:** Посилання на MLflow run ID із таблицею метрик та 25-класовою confusion matrix
   - **Обмеження, якщо не спрацює:** Yes

2. **[ ] [Model quality gates] Log loss класифікатора становить ≤ 0.15 та не гірший за поточний production baseline (різниця ≤ +0.005)**
   - **Власник:** ML Lead
   - **Докази:** Звіт порівняння candidate vs production у MLflow UI (аналогічно підходу ДЗ 4)
   - **Обмеження, якщо не спрацює:** Yes

3. **[ ] [Model quality gates] Мінімальний F1-score по кожному з топ-10 пріоритетних бізнес-класів (включно з billing, refunds, chargebacks) ≥ 0.90**
   - **Власник:** ML Lead
   - **Докази:** Per-class evaluation звіт у MLflow Artifacts (`class_metrics.json`)
   - **Обмеження, якщо не спрацює:** Yes

4. **[ ] [Model quality gates] Частота галюцинацій LLM (Llama 3.3 70B) на golden eval-датасеті (500 фінансових та політик-кейсів) < 0.5% (baseline до оптимізації: 3.4%)**
   - **Власник:** Senior AI PM
   - **Докази:** Автоматизований звіт LLM-as-a-Judge + ручний аудит вибірки у MLflow Artifacts
   - **Обмеження, якщо не спрацює:** Yes

5. **[ ] [Model quality gates] Точність коректних відмов (Refusal Accuracy) при відсутності релевантного контексту в RAG ≥ 98%**
   - **Власник:** Head of AI Safety / Security
   - **Докази:** Лог проходження набору «Unanswerable Queries» (100 тестових питань без відповіді в базі знань)
   - **Обмеження, якщо не спрацює:** Yes

6. **[ ] [Model quality gates] Релевантність пошуку RAG (`Cosine Similarity Precision @ Top-3`) становить ≥ 0.85 на 12 мовах**
   - **Власник:** ML Lead
   - **Докази:** Результат оцінки retrieval-метрик (Recall@3, MRR, Precision@3) у Staging pipeline
   - **Обмеження, якщо не спрацює:** Yes

7. **[ ] [Model quality gates] Якість генерації на 2 другорядних мовах (<1% трафіку) не відхиляється більше ніж на 5% від середнього показника (прийнятий ризик `T-011`)**
   - **Власник:** ML Lead
   - **Докази:** Багатомовний eval-звіт по 12 мовах у MLflow
   - **Обмеження, якщо не спрацює:** No (при виявленні відхилення тикети цих мов автоматично маршрутизуються на операторів; планове покращення до Q3)

---

## Категорія 2: Data readiness (5 пунктів)

8. **[ ] [Data readiness] 100% тренувальної вибірки (12M рядків) пройшли автоматичний PII-скринінг (Presidio + AWS Comprehend) з результатом 0 відкритих персональних даних (`T-007`)**
   - **Власник:** Data Engineer
   - **Докази:** Звіт кроку `DataValidation` у AWS Step Functions (execution ARN + summary JSON)
   - **Обмеження, якщо не спрацює:** Yes

9. **[ ] [Data readiness] Валідація схеми, типів даних і допустимих значень пройдена без критичних помилок (0% null у обов'язкових полях `tenant_id`, `text`, `intent_label`)**
   - **Власник:** Data Engineer
   - **Докази:** Статус `SUCCEEDED` етапу `DataValidation` у AWS Step Functions (ДЗ 5)
   - **Обмеження, якщо не спрацює:** Yes

10. **[ ] [Data readiness] Індекс зміщення розподілу між тренувальними та поточними production-даними (PSI — Population Stability Index) < 0.15**
    - **Власник:** Data Engineer
    - **Докази:** Звіт Data Drift у Grafana / Evidently за останні 7 днів
    - **Обмеження, якщо не спрацює:** Yes

11. **[ ] [Data readiness] Версія датасету, Git Commit SHA та хеш параметрів зафіксовані в DVC і пов'язані з MLflow Run ID (`T-008`)**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Заповнені теги `git_commit`, `dvc_data_hash`, `step_function_arn` у MLflow Registry
    - **Обмеження, якщо не спрацює:** Yes

12. **[ ] [Data readiness] Санітизація HTML/Markdown у завантажених клієнтських базах знань видалила 100% прихованих тегів та скриптів перед векторизацією в Pinecone (`T-003`)**
    - **Власник:** Backend Lead
    - **Докази:** Лог інгест-пайплайну RAG із підтвердженням проходження тестів на приховані інструкції
    - **Обмеження, якщо не спрацює:** Yes

---

## Категорія 3: Infrastructure readiness (6 пунктів)

13. **[ ] [Infrastructure readiness] Docker-образи сервісів інференсу зібрані за multi-stage схемою `python:3.13-slim`, розмір CPU-сервісів ≤ 650 MB (згідно з ДЗ 1) і 0 Critical/High CVE у звіті Trivy (`T-012`)**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Лог GitLab CI pipeline з виводом розміру образу та звітом `trivy_scan.json`
    - **Обмеження, якщо не спрацює:** Yes

14. **[ ] [Infrastructure readiness] У виробничому кластері EKS для всіх подів у `prod-llm`, `prod-ml`, `prod-rag` задані `requests`/`limits` (CPU, RAM, GPU vRAM) та namespace quotas (ДЗ 2)**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Перевірений YAML-маніфест у GitOps-репозиторії Argo CD (PR approved)
    - **Обмеження, якщо не спрацює:** Yes

15. **[ ] [Infrastructure readiness] Налаштовано Horizontal Pod Autoscaler (HPA) для CPU (>70%) та GPU Utilization (>80%) із підтвердженим навантажувальним тестом на Staging**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Протокол load-тесту (симуляція 2× пікового трафіку, `p95 latency < 1.8s`)
    - **Обмеження, якщо не спрацює:** Yes

16. **[ ] [Infrastructure readiness] Всі 3 дашборди Grafana (Executive ML, EKS Infrastructure, RAG & Security) оновлені та відображають дані Staging без пропусків**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Скриншот/посилання на дашборди Grafana з активними серіями `mlflow_accuracy` та `latency_p95`
    - **Обмеження, якщо не спрацює:** Yes

17. **[ ] [Infrastructure readiness] Алерти PagerDuty/Slack підключені для порогів `mlflow_accuracy < 0.90`, `p95 > 1.8s`, `5xx > 1%`, GPU OOM та щоденного ліміту fallback API (`T-014`)**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Скриншот тестового спрацювання алерту в Slack `#mlops-alerts` та PagerDuty
    - **Обмеження, якщо не спрацює:** Yes

18. **[ ] [Infrastructure readiness] Протестовано стратегію резервного копіювання індексів Pinecone Enterprise (щоденний експорт метаданих і векторів у S3 із RTO < 2 год)**
    - **Власник:** Backend Lead
    - **Докази:** Лог тестового відновлення (restore drill) одного тестового namespace за останній місяць
    - **Обмеження, якщо не спрацює:** No (допускається запуск за наявності активних щоденних знімків S3, якщо повний restore drill перенесено не більш ніж на 14 днів)

---

## Категорія 4: Security & compliance (7 пунктів)

19. **[ ] [Security & compliance] Незмінне логування (S3 Object Lock, 12 місяців retention) для хешів промптів, RAG-джерел, MLflow run ID та відповідей увімкнено (`T-010`)**
    - **Власник:** Head of Compliance (DPO)
    - **Докази:** Конфігурація AWS S3 Bucket Policy + тестовий незмінний запис у лог аудиту
    - **Обмеження, якщо не спрацює:** Yes

20. **[ ] [Security & compliance] Перевірено RBAC у AWS IAM, EKS `prod`, MLflow Registry та Argo CD за принципом найменших привілеїв (розробники не мають прямого write-доступу в `prod`) (`T-012`, `T-013`)**
    - **Власник:** Head of AI Safety / Security
    - **Докази:** Звіт квартального рев'ю ролей IAM/RBAC, підписаний Security та Lead MLOps
    - **Обмеження, якщо не спрацює:** Yes

21. **[ ] [Security & compliance] Підтверджено шифрування даних у русі (TLS 1.3) та у спокої (AWS KMS AES-256) для S3, RDS, EBS і шлюзу fallback API**
    - **Власник:** Backend Lead
    - **Докази:** Вивід перевірки AWS Config / Terraform state на відповідність шифруванню
    - **Обмеження, якщо не спрацює:** Yes

22. **[ ] [Security & compliance] Оновлено Технічний файл EU AI Act (Стаття 11) із описом нової версії моделі, ризиків, метрик точності та процедур Human-in-the-loop**
    - **Власник:** Head of Compliance (DPO)
    - **Докази:** Оновлений документ у compliance-репозиторії з версією релізу та підписом DPO
    - **Обмеження, якщо не спрацює:** Yes

23. **[ ] [Security & compliance] Перевірено ізоляцію тенантів у Pinecone RAG: 6 інтеграційних авто-тестів підтверджують 0 результатів із `Namespace_B` при запиті з токеном `Tenant_A` (`T-002`)**
    - **Власник:** Backend Lead
    - **Докази:** Звіт CI/CD авто-тестування (`rag_tenant_isolation_test: 6/6 passed`)
    - **Обмеження, якщо не спрацює:** Yes

24. **[ ] [Security & compliance] Набір тестів NeMo Guardrails блокує ≥ 98% атак Direct/Indirect Prompt Injection та 100% запитів на фінансові компенсації поза правилами CRM (`T-001`, `T-006`)**
    - **Власник:** Head of AI Safety / Security
    - **Докази:** Звіт red-teaming прогону (150 сценаріїв на 12 мовах) перед релізом
    - **Обмеження, якщо не спрацює:** Yes

25. **[ ] [Security & compliance] Усі загрози пріоритету P1 з `02_Threat_Model.md` мають статус `Implemented` або верифікований компенсуючий контроль у staging**
    - **Власник:** Senior AI PM
    - **Докази:** Перевірена матриця статусів P1-загроз (`T-001`, `T-002`, `T-004`, `T-006`, `T-010`, `T-012`)
    - **Обмеження, якщо не спрацює:** Yes

---

## Категорія 5: Operational readiness (6 пунктів)

26. **[ ] [Operational readiness] Усі 3 сценарії в `04_Incident_Runbook.md` (Model Drift, PII/Injection, Argo CD OutOfSync) оновлені під поточний реліз і доступні в командному просторі**
    - **Власник:** Senior AI PM
    - **Докази:** Посилання на актуальну ревізію Runbook із перевіреними контактами ескалації
    - **Обмеження, якщо не спрацює:** Yes

27. **[ ] [Operational readiness] Призначено основного (Primary) та резервного (Backup) чергових інженерів (On-Call) у PagerDuty на найближчі 2 тижні після релізу**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Експорт графіка ротації PagerDuty на поточний місяць
    - **Обмеження, якщо не спрацює:** Yes

28. **[ ] [Operational readiness] Задекларовано та перевірено цільові SLO релізу: доступність ≥ 99.9%, `p95 latency < 1.8s`, `mlflow_accuracy ≥ 0.90`, похибка `5xx < 1%`**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Налаштовані об'єкти `AnalysisTemplate` в Argo Rollouts із відповідними порогами
    - **Обмеження, якщо не спрацює:** Yes

29. **[ ] [Operational readiness] У Staging проведено бойовий тест Canary-відкату (ін'єкція штучної 5xx/latency помилки) з підтвердженням авто-відкату ≤ 30 секунд (ADR-002)**
    - **Власник:** Lead MLOps Engineer
    - **Докази:** Таймлайн виконання `Argo Rollouts` abort/rollback у Staging (час реакції зафіксовано)
    - **Обмеження, якщо не спрацює:** Yes

30. **[ ] [Operational readiness] Протестовано Circuit Breaker та Rate Limiter на шлюзі комерційного fallback API (ліміт бюджету $10K/міс, автоматичне блокування спам-токенів `T-014`, `T-016`)**
    - **Власник:** Backend Lead
    - **Докази:** Результат тесту симуляції перевищення ліміту запитів у Staging
    - **Обмеження, якщо не спрацює:** Yes

31. **[ ] [Operational readiness] Підготовлені й узгоджені шаблони повідомлень про інциденти (для внутрішнього Slack `#inc-*`, статус-сторінки клієнтів та форми GDPR 72h)**
    - **Власник:** Head of Support
    - **Докази:** Затверджені макроси комунікації у системі підтримки та в DPO-репозиторії
    - **Обмеження, якщо не спрацює:** No (якщо основний шаблон статус-сторінки вже актуальний, оновлення локалізованих шаблонів допускається протягом 48 год після релізу)

---

## Категорія 6: Business readiness (5 пунктів)

32. **[ ] [Business readiness] Отримано формальне погодження бізнес-метрик та готовності релізу від Senior AI PM та CPO (особливо для модулів billing/chargebacks)**
    - **Власник:** CPO
    - **Докази:** Підписаний Release Sign-off тикет у Jira (Release Approver approval)
    - **Обмеження, якщо не спрацює:** Yes

33. **[ ] [Business readiness] Затверджено технічну та бюджетну готовність з боку CTO (підтверджено втримання unit cost у межах $0.012–$0.015/тикет та OPEX ≤ $120K/міс)**
    - **Власник:** CTO
    - **Докази:** Коментар-апрув від CTO у релізному тикеті та звірка прогнозу AWS Cost Explorer
    - **Обмеження, якщо не спрацює:** Yes

34. **[ ] [Business readiness] Проведено інструктаж L3-підтримки та агентів клієнтських команд (розуміння маркерів упевненості LLM, правила ручної ескалації high-stakes запитів)**
    - **Власник:** Head of Support
    - **Докази:** Список присутніх на тренінгу та запис сесії / тест засвоєння інструкцій (≥ 90% команди)
    - **Обмеження, якщо не спрацює:** Yes

35. **[ ] [Business readiness] Зафіксовано базовий рівень (baseline) бізнес-метрик перед запуском у BI-системі (CSAT, % auto-resolution, cost per ticket, churn rate)**
    - **Власник:** Senior AI PM
    - **Докази:** Знімок дашборду Product Analytics за попередні 30 днів до релізу
    - **Обмеження, якщо не спрацює:** No (історичні дані вже зберігаються в DWH; допускається формування порівняльного дашборду протягом першого тижня після Canary)

36. **[ ] [Business readiness] Підготовлено анонс для 1,200 enterprise-клієнтів (Release Notes про покращення якості та захист даних, опція ввімкнення нових модулів)**
    - **Власник:** Senior AI PM
    - **Докази:** Фінальний текст Release Notes, погоджений із Product Marketing та Legal
    - **Обмеження, якщо не спрацює:** No (обов'язково до переходу на 100% трафіку, але не блокує стартовий Canary 5% для пілотних клієнтів)

---

## Підсумок розподілу блокерів

| Категорія                   | Кількість пунктів | Блокери (`Yes`) | Не-блокери (`No`) | Ключові ролі-власники                                   |
| :-------------------------- | :---------------: | :-------------: | :---------------: | :------------------------------------------------------ |
| 1. Model quality gates      |         7         |        6        |         1         | ML Lead, Senior AI PM, Head of AI Safety                |
| 2. Data readiness           |         5         |        5        |         0         | Data Engineer, Lead MLOps, Backend Lead                 |
| 3. Infrastructure readiness |         6         |        5        |         1         | Lead MLOps Engineer, Backend Lead                       |
| 4. Security & compliance    |         7         |        7        |         0         | DPO, Head of AI Safety, Backend Lead, Senior AI PM      |
| 5. Operational readiness    |         6         |        5        |         1         | Lead MLOps, Backend Lead, Senior AI PM, Head of Support |
| 6. Business readiness       |         5         |        3        |         2         | CPO, CTO, Senior AI PM, Head of Support                 |
| **Разом**                   |      **36**       |     **31**      |       **5**       | **100% пунктів закріплені за ролями легенди**           |

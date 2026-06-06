# Полная рабочая инструкция
## Частичная миграция Jira 8 → Jira Data Center 10 и Confluence 7 → Confluence Data Center 9 с Keycloak (SSO/SCIM)

> Версия документа: 2026-06-06  
> Назначение: подготовка, проверка совместимости, пошаговый план и контрольные списки для **частичной миграции**:
> - отдельных **проектов Jira** из старой инсталляции в новую,
> - отдельных **пространств Confluence** из старой инсталляции в новую,
> - с сохранением работоспособности аутентификации и жизненного цикла пользователей через **Keycloak**.

---

## 1. Цель и границы документа

Этот документ покрывает **частичную**, а не полную миграцию:

- **Jira**: перенос не всего инстанса, а отдельных проектов / их конфигурации / связанных данных.
- **Confluence**: перенос не всего сайта, а отдельных пространств через XML Space Export/Import.
- **Identity**: целевая система использует Keycloak как IdP; отдельно рассматриваются варианты **native OIDC/JIT** и **полноценный SCIM provisioning**.

Документ **не** рекомендует:
- делать прямой major-upgrade Jira 8 → 10 без тестового прогона,
- импортировать Confluence site backup вместо space-by-space сценария,
- «подрисовывать» версии/Build Number как штатный способ обхода совместимости.

---

## 2. Ключевые архитектурные выводы до начала работ

### 2.1 Jira: partial import в существующую Jira требует одинаковых версий источника и цели

Для merge/partial import Atlassian прямо рекомендует сначала **обновить Jira A до версии Jira B**, затем сделать XML backup и только после этого использовать **Project Import** в Jira B. Atlassian также отдельно указывает, что для таких сценариев могут помочь **Configuration Manager for Jira** и **Project Configurator**. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/)

**Вывод:** если у вас источник = Jira 8, а цель = Jira 10, то **не стоит планировать прямой partial import “как есть” из 8 в 10**. Сначала нужно либо:
1. поднять промежуточный staging/source-upgraded Jira той же версии, что и target,  
2. либо использовать вендорский инструмент (CMJ / Project Configurator) и отдельно подтвердить совместимость этого migration path у вендора.

### 2.2 Jira 10: только manual upgrade, Java 17+

Начиная с Jira 10 поддерживается только **manual upgrade method**; бинарные installer-методы больше не являются штатным путем обновления. Jira 10 перекомпилирован на **Java 17**, а Java 8/11 для него больше не подходят. [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/) [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html)

### 2.3 Между major-версиями Jira нельзя делать ZDU

Atlassian отдельно указывает, что **Zero Downtime Upgrade между major releases не поддерживается**. Пример из документации: Jira 8 → Jira 9 требует downtime. Это правило применимо и к сценарию 8 → 10: планируйте полноценное технологическое окно. [6](https://support.atlassian.com/jira/kb/when-attempting-a-zero-down-time-upgrade-jira-will-not-start-noting-that-jira-does-not-support-zero-downtime-upgrades-between-major-releases/)

### 2.4 Confluence: space import допускается в ту же или более новую совместимую версию

Atlassian разрешает восстанавливать/импортировать пространство из XML Space Export в **ту же или более новую совместимую** версию Confluence; импорт в более старую версию не поддерживается. Для очень старых экспортов (до 5.3) нужен промежуточный временный Confluence для «подтягивания» версии экспорта. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html)

**Вывод:** для Confluence 7 → 9 blanket-требование «обязательно править build number в архиве» некорректно. Сначала нужно проверить, **принимает ли target этот экспорт штатно**. Ручная правка `exportDescriptor.properties` — это **не стандартный путь миграции**, а только edge-case workaround для специальных проблем с архивом/метаданными. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) [1](https://support.atlassian.com/confluence/kb/how-to-determine-xml-backup-confluence-version/)

### 2.5 Native OIDC у Atlassian != SCIM

Atlassian Data Center умеет **native OIDC/SAML SSO** через приложение SSO for Atlassian Data Center, а также **JIT provisioning** (создание/обновление пользователя при логине). Но Atlassian отдельно подчеркивает: **OIDC отвечает за аутентификацию**, а доступы/авторизации и пользовательские группы должны быть настроены в directory/application; JIT — это не полноценный out-of-band SCIM lifecycle provisioning. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html) [4](https://confluence.atlassian.com/enterprise/working-with-jit-provisioning-1005342571.html)

**Вывод:** если нужен полный lifecycle:
- create/update/disable users,
- group sync,
- deprovisioning,
- предварительное создание пользователей до импорта,

то нужен **отдельный SCIM/User Sync app**, а не только native OIDC.

---

## 3. Целевая архитектура

### 3.1 Что рекомендуется в target-контуре

- **Jira Data Center 10.x** (у вас: 10.6.0)
- **Confluence Data Center 9.x** (у вас: 9.4.1)
- **Keycloak** как внешний IdP
- **SCIM / User Sync app** для pre-provisioning пользователей
- отдельный **staging-контур**, максимально повторяющий production по:
  - версии продукта,
  - версии БД,
  - версии плагинов,
  - схеме reverse proxy / ingress,
  - directory и identity flow.

---

## 4. Требования к инфраструктуре и платформе

## 4.1 Java

### Jira 10
- Требует **Java 17**; Java 8/11 для Jira 10 больше не поддерживаются. [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html) [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/)

### Confluence 9.x
- Для ветки Confluence 9.x Java 17 остается актуальным рабочим вариантом; Atlassian отдельно отмечает, что **Java 17 прекращается только начиная с Confluence 10.0**, а Confluence 9.x продолжает работать с Java 17. [6](https://confluence.atlassian.com/doc/end-of-support-announcements-for-confluence-210239673.html)

**Рекомендуемое решение:** стандартизовать целевой staging/prod на **Java 17** для обоих продуктов.

---

## 4.2 База данных

### Jira
- Atlassian отдельно указывает, что поддержка **PostgreSQL 10 и 11** была удалена начиная с Jira 9.12; для Jira 10 перед запуском миграции нужно свериться с официальной матрицей совместимости БД и выбрать версию, которая для вашей конкретной target-ветки отмечена как поддерживаемая, а не устаревающая. [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html) [2](https://support.atlassian.com/jira/kb/jira-databases-compatibility-matrix-data-center/)

### Confluence
- В Confluence 9.x поддержка БД менялась по minor-версиям; Atlassian рекомендует сверяться с Supported Platforms / Upgrade Matrix и, если support window вашей БД заканчивается внутри жизненного цикла target-версии, выбирать более новую поддерживаемую версию БД. [4](https://confluence.atlassian.com/doc/confluence-upgrade-matrix-960695895.html) [6](https://confluence.atlassian.com/doc/end-of-support-announcements-for-confluence-210239673.html)

**Практическое правило:**
- не переносить старые DB-платформы “как есть”;  
- перед freeze окна миграции зафиксировать конкретную матрицу совместимости **Jira target ↔ DB version** и **Confluence target ↔ DB version**.

---

## 4.3 Кодировка и locale БД

Для Confluence site/space XML restore структура и кодировка БД должны быть корректны; на практике для PostgreSQL используйте **UTF-8** и согласованный locale, иначе импорт и индексирование могут давать трудно диагностируемые ошибки. Это обязательное инженерное требование, даже если конкретная ошибка проявится не на каждом наборе данных.

**Рекомендуемое значение:**
- encoding = `UTF8`
- locale = согласованный UTF-8 locale вашей платформы.

---

## 4.4 Диск и файловая система

### Jira
- Для XML restore / import / upgrade нужно резервировать место под:
  - backup архивы,
  - attachments,
  - index rebuild,
  - временные файлы во время импорта/апгрейда. [1](https://confluence.atlassian.com/adminjiraserver0810/upgrade_import-old-jira-data-1014673280.html) [10](https://support.atlassian.com/jira/kb/correcting-timestamps-for-jira-issues-using-xml-files/)

### Confluence
- Atlassian прямо рекомендует иметь заметный запас места: при импорте Confluence временно создает несколько копий файла; при failed import данные могут “застрять” в БД и потребовать cleanup. [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html) [4](https://support.atlassian.com/confluence/kb/after-a-space-import-fails-it-cant-be-re-imported/)

**Минимум:** staging/prod storage capacity должен быть проверен до первого dry-run.

---

## 4.5 Сеть и reverse proxy

### Для OIDC/Keycloak
OIDC в отличие от чистого SAML требует **server-to-server connectivity** между Jira/Confluence и IdP для обмена authorization code на tokens. Если есть outbound proxy — для IdP-host нужны исключения. [10](https://support.atlassian.com/jira/kb/openid-connect-authentication-fails-with-exchanging-authorization-tokens-failed/)

**Обязательные проверки:**
- Jira node → Keycloak token endpoint
- Confluence node → Keycloak token endpoint
- корректная схема reverse proxy / base URL / callback URL
- отсутствие proxy timeout/403 на token exchange

---

## 5. Требования к интеграции с Keycloak

## 5.1 Что можно сделать native, а что нет

### Native Atlassian SSO (OIDC/SAML)
Подходит для:
- SSO,
- JIT provisioning,
- обновления user profile на логине. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html)

Не покрывает полноценно:
- массовое предварительное provisioning до импорта,
- полноценный deprovisioning lifecycle,
- управляемый вне логина user/group sync.

### SCIM / User Sync app
Нужен, если вы хотите, чтобы users/groups уже существовали в target-системе **до** импорта Jira/Confluence-контента.

---

## 5.2 Совместимость вариантов SCIM apps

### Jira DC
В найденных Marketplace-источниках:
- **miniOrange SCIM Provisioning, User Sync & Group Sync for Jira** заявляет provisioning пользователей и групп из Keycloak через SCIM/REST и поддержку Jira DC вплоть до актуальных веток. [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira)
- **resolution User Sync: SCIM Provisioning, Group Sync for Jira** также заявляет sync/provisioning/deprovisioning и поддержку Keycloak. [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira)
- **LuxPlugins SCIM User Provisioning for Jira** поддерживает create/update/disable users, group sync, default groups и отдельные internal directories. [1](https://marketplace.atlassian.com/apps/1219723/scim-user-provisioning-for-jira)

### Confluence DC
- **resolution User Sync: SCIM Provisioning, Group Sync for Confluence** в найденном листинге поддерживает Confluence Data Center вплоть до **9.4.1**, что хорошо совпадает с вашей целевой версией. [7](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-user-group-sync-confluence)
- **miniOrange SCIM Provisioning for Confluence** тоже поддерживает Keycloak/SCIM, но конкретный compatibility range нужно перепроверять под вашу minor-версию перед установкой. [10](https://marketplace.atlassian.com/apps/1222394/mo-confluence-scim-user-provisioning-confluence-user-sync?tab=overview&hosting=datacenter)
- **LuxPlugins SCIM User Provisioning for Confluence** по найденному листингу ориентирован уже на Confluence 10.x; для 9.4.1 использовать только после точной проверки матрицы совместимости у вендора. [4](https://marketplace.atlassian.com/apps/1222162/scim-user-provisioning-for-confluence)

---

## 5.3 Главный жесткий requirement по идентичности пользователей

Чтобы после импорта авторы задач, комментариев, страниц и правки истории корректно смапились на существующих пользователей, значение поля, по которому целевая система матчит пользователя, должно быть **стабильным и одинаковым**.

Atlassian для OIDC/JIT отдельно подчеркивает важность корректного **Username mapping**; если mapping неверен, пользователь не будет найден или будет создан не тот пользователь. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [3](https://support.atlassian.com/confluence/kb/how-to-configure-custom-claims-in-okta-for-openid-connect-oidc/)

**Рекомендуемое правило:**
- заранее зафиксировать canonical identifier: обычно `username` или `email`,
- не менять его в Keycloak в ходе миграции,
- проверить регистр, префиксы/суффиксы, доменные части и historical aliases.

---

## 5.4 Жесткое требование по pre-provisioning

До начала import работ необходимо:
1. установить SCIM/User Sync app на target,
2. подключить Keycloak,
3. выполнить **полную первичную синхронизацию** пользователей и групп,
4. проверить, что pilot users уже существуют в target Jira/Confluence,
5. только потом импортировать проекты и пространства.

Иначе вы получите:
- “мертвых” authors/mentions,
- дубликаты пользователей,
- неправильные group grants.

---

## 6. Требования к плагинам и Marketplace apps

## 6.1 Общие правила

На target Jira/Confluence должны быть установлены **именно те версии приложений**, которые официально поддерживают target major/minor release. Это критично для:
- пользовательских макросов в Confluence,
- custom fields / post functions / validators в Jira,
- AO schema migrations,
- интерпретации app data.

Atlassian для Data Center отдельно рекомендует использовать **Data Center approved apps** и проверять поддержку app versions под нужный major release. [4](https://www.atlassian.com/blog/enterprise/data-center-approved-apps)

---

## 6.2 CMJ / Project Configurator для Jira

Atlassian прямо упоминает **Configuration Manager for Jira** и **Project Configurator** как инструменты, которые могут помочь при project import / merge scenario. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) [2](https://confluence.atlassian.com/spaces/ADMINJIRASERVER0817/pages/1167832838/Exporting+projects+from+the+source+instance)

### Практическое требование
- на source Jira 8 должен стоять **последний билд CMJ/Project Configurator, поддерживающий 8.x**;
- на target Jira 10 — **отдельный** билд того же продукта, поддерживающий Jira 10;
- перед миграцией нужно проверить vendor docs, совместим ли migration format между этими ветками.

**Важно:** это требование нельзя считать автоматически выполненным только потому, что “плагин есть на обеих сторонах”. Для app data migration нужен подтвержденный vendor-supported path.

---

## 6.3 Audit плагинов Jira

Перед миграцией Jira-проектов составьте inventory:
- ScriptRunner
- Structure
- Xporter / Xray / Zephyr / Tempo
- кастомные поля
- workflow post-functions
- validators/conditions
- listeners
- REST модули
- web fragments

Для каждого app зафиксируйте:
- версия на source,
- версия на target,
- есть ли DC 10-compatible release,
- есть ли documented migration path,
- какие custom fields / AO tables / services используются в переносимых проектах.

---

## 6.4 Macro audit для Confluence

Перед space import выполните **Macro Usage audit** на source Confluence 7 и подготовьте таблицу:
- macro name,
- источник (native / marketplace / custom user macro),
- есть ли аналог на Confluence 9,
- есть ли изменения storage format,
- нужен ли ручной remediation plan.

Это отдельный обязательный pre-check, потому что импорт пространства без совместимого набора макросов почти всегда приводит к “сломавшимся” страницам уже после формально успешного restore.

---

## 7. Специальные требования к Jira-миграции (8 → 10)

## 7.1 Какой путь является штатным

### Если нужен полный перенос whole instance
Лучший supported путь:
- поднять новый Jira 10,
- подключить **пустую DB**,
- выбрать **Import existing data**,
- импортировать корректный XML backup (`entities.xml` + `activeobjects.xml`),
- затем вернуть attachments. [1](https://confluence.atlassian.com/adminjiraserver0810/upgrade_import-old-jira-data-1014673280.html) [9](https://support.atlassian.com/jira/kb/importing-data-via-setup-wizard/)

### Если нужен partial migration into existing Jira
Штатный путь для built-in Project Import:
- сначала выровнять версию source до версии target,
- затем делать XML backup/project import. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/)

---

## 7.2 Что нельзя делать

- Нельзя рассчитывать на **direct project import из Jira 8 в Jira 10** без выравнивания версии. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/)
- Нельзя планировать **ZDU между major versions**. [6](https://support.atlassian.com/jira/kb/when-attempting-a-zero-down-time-upgrade-jira-will-not-start-noting-that-jira-does-not-support-zero-downtime-upgrades-between-major-releases/)
- Нельзя использовать старые unsupported DB/Java и надеяться “что заведется”. [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html) [2](https://support.atlassian.com/jira/kb/jira-databases-compatibility-matrix-data-center/)

---

## 7.3 Рекомендуемая практическая схема partial migration Jira

### Вариант 1 — через временный source-upgraded staging
1. Снять backup старой Jira 8.
2. Поднять staging Jira той же ветки, затем обновить staging до Jira 10 по поддерживаемому пути.
3. На staging-source установить target-compatible versions нужных apps.
4. Выполнить экспорт/partial migration через CMJ / Project Configurator / Project Import уже из **version-aligned source**.
5. Импортировать в target Jira 10 staging.
6. Провести reconciliation users, groups, workflows, custom fields.
7. Только после успешного dry-run планировать production окно.

### Вариант 2 — whole-instance upgrade + selective copy
Если старый Jira 8 нужен целиком и времени достаточно, иногда безопаснее сначала сделать **поддерживаемый апгрейд самого source-инстанса до новой ветки**, а затем уже переносить нужные проекты.

---

## 8. Специальные требования к Confluence-миграции (7 → 9)

## 8.1 Для partial migration используйте только Space Export/Import

Для переноса отдельных пространств нужен **Space XML export**, а не site backup. Atlassian описывает restore/import space backup как отдельный процесс, и XML site backup не должен использоваться как метод апгрейда Confluence. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)

### Правильный путь экспорта
На source Confluence:
- `Space tools → Content tools → Export → XML`
- при необходимости включить attachments.

---

## 8.2 Версионная совместимость space import

- Импорт в **более старую** версию невозможен. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)
- Импорт в **ту же или более новую совместимую** версию поддерживается. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)
- Для очень старых экспортов (до 5.3) нужен промежуточный Confluence той же версии, чтобы пересобрать экспорт под более новую ветку. [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html)

### Что это значит для 7 → 9
В нормальном случае сначала надо **попробовать штатный import в staging**.  
Только если target явно отклоняет архив по версии/метаданным, переходить к обходным сценариям.

---

## 8.3 Build Number: корректировка документа

Ваш исходный draft нужно **исправить** в этой части.

### Неверно как общее правило
> «Поскольку версии сильно отличаются, перед импортом в Confluence 9 потребуется вручную изменить build number в exportDescriptor.properties»

Так формулировать **нельзя**.

### Правильная формулировка
- Сначала нужно проверить, принимает ли target-version space export штатно.
- Если import не проходит, изучите `exportDescriptor.properties` и фактическую версию экспорта через `createdByBuildNumber` / `buildNumber`. [1](https://support.atlassian.com/confluence/kb/how-to-determine-xml-backup-confluence-version/)
- Для старых/проблемных экспортов предпочтителен **supported workaround через временный промежуточный Confluence** и повторный экспорт, а не «подмена build number» вслепую. [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html)
- Простая «подрисовка номера» допустима только как edge-case workaround для конкретных дефектов архива и должна использоваться **только на staging**. Atlassian community отдельно предупреждает, что подмена build number для database/site-level сценариев может привести к неконсистентным данным и поломке системы. [10](https://community.atlassian.com/forums/Confluence-questions/the-build-number-in-the-database-7202-doesn-t-match-either-the/qaq-p/1066109)

---

## 8.4 Практические требования к ZIP-архиву Confluence

При перепаковке space backup:
- в корне ZIP должны лежать как минимум `exportDescriptor.properties` и `entities.xml`;  
- нельзя zip'овать родительскую папку поверх файлов;  
- не используйте Windows ZIP compression, если архив потом не читается; лучше 7-Zip/WinZip;  
- на Mac нужно следить, чтобы ОС не разархивировала файл автоматически и не добавляла мусор вроде `__MACOSX` / `.DS_Store`. [9](https://support.atlassian.com/confluence/kb/confluence-space-import-failed-with-invalid-compression-method/) [10](https://support.atlassian.com/confluence/kb/xml-backup-restore-failed-with-the-zip-file-did-not-contain-an-entry-exportdescriptorproperties-in-confluence/) [7](https://community.atlassian.com/forums/Confluence-articles/I-see-an-error-when-importing-a-Confluence-Space-with/ba-p/2298018)

---

## 8.5 Если import space уже один раз упал

Confluence может оставить мусорные записи в БД, из-за чего тот же архив нельзя будет просто «перезалить еще раз». Atlassian описывает этот кейс отдельно и рекомендует cleanup до повторной попытки. [4](https://support.atlassian.com/confluence/kb/after-a-space-import-fails-it-cant-be-re-imported/) [7](https://support.atlassian.com/confluence/kb/unable-to-import-xml-after-first-attempt-fails/)

**Жесткое правило:** повторный импорт одного и того же space backup после failure — только после cleanup или отката staging БД к снапшоту до попытки импорта.

---

## 9. Требования к staging и dry-run

Из-за технологического разрыва между Jira 8 → 10 и Confluence 7 → 9 выполнять миграцию напрямую в production **нельзя**.

## 9.1 Staging обязателен

Staging должен повторять production по:
- версии Jira/Confluence,
- версии БД,
- версии app-пакета,
- reverse proxy/base URLs,
- Keycloak realm/client config,
- SCIM app config,
- группам доступа.

## 9.2 Обязательные dry-runs

### Jira
Минимум один dry-run на:
- 1 пилотный проект с типовыми custom fields,
- workflow с app-dependent post-functions/validators,
- реальные users/groups/roles,
- reconciliation permissions.

### Confluence
Минимум один dry-run на:
- 1 пилотное пространство с макросами и вложениями,
- импорт в clean target DB snapshot,
- smoke test страниц, вложений, ссылок, history, mentions, labels.

---

## 10. Полный Requirements Checklist

## 10.1 Платформа

- [ ] Jira 10 target работает на Java 17+. [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/) [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html)
- [ ] Confluence 9 target работает на поддерживаемой Java-платформе; для унификации выбран Java 17. [6](https://confluence.atlassian.com/doc/end-of-support-announcements-for-confluence-210239673.html)
- [ ] Версии DB для Jira и Confluence сверены с official supported matrix. [2](https://support.atlassian.com/jira/kb/jira-databases-compatibility-matrix-data-center/) [4](https://confluence.atlassian.com/doc/confluence-upgrade-matrix-960695895.html)
- [ ] Для Jira исключены PostgreSQL 10/11. [1](https://confluence.atlassian.com/adminjiraserver/end-of-support-announcements-938846831.html)
- [ ] Проверены disk space / temp space / attachments space. [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html) [4](https://support.atlassian.com/confluence/kb/after-a-space-import-fails-it-cant-be-re-imported/)
- [ ] Проверена server-to-server связь Jira/Confluence → Keycloak token endpoints. [10](https://support.atlassian.com/jira/kb/openid-connect-authentication-fails-with-exchanging-authorization-tokens-failed/)

## 10.2 Identity / Keycloak

- [ ] Выбран canonical user identifier (username/email).
- [ ] Проверено совпадение идентификаторов старых и новых пользователей.
- [ ] Выбран подход: native OIDC/JIT или полноценный SCIM. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html)
- [ ] Если нужен lifecycle provisioning — установлен SCIM/User Sync app. [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira) [7](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-user-group-sync-confluence)
- [ ] Выполнен pre-provisioning пользователей и групп до first import.

## 10.3 Jira apps

- [ ] Составлен inventory app'ов source Jira.
- [ ] Для каждого app подтверждена Jira 10 compatibility.
- [ ] Для CMJ/Project Configurator подтвержден vendor-supported migration path между source и target.
- [ ] Для partial import выбран механизм: CMJ / Project Configurator / version-aligned Project Import.

## 10.4 Confluence apps

- [ ] Выполнен macro audit source spaces.
- [ ] Все критичные макросы имеют target-compatible app equivalents.
- [ ] Проверены user macros и кастомные theme/decorator элементы.

## 10.5 Confluence архивы

- [ ] Используется именно Space XML Export, не site backup. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)
- [ ] Проверен `exportDescriptor.properties`. [1](https://support.atlassian.com/confluence/kb/how-to-determine-xml-backup-confluence-version/)
- [ ] Перепаковка делается корректно: `entities.xml` + `exportDescriptor.properties` в корне ZIP. [9](https://support.atlassian.com/confluence/kb/confluence-space-import-failed-with-invalid-compression-method/) [10](https://support.atlassian.com/confluence/kb/xml-backup-restore-failed-with-the-zip-file-did-not-contain-an-entry-exportdescriptorproperties-in-confluence/)
- [ ] Перед повторным import после failure staging БД откатывается или выполняется cleanup. [4](https://support.atlassian.com/confluence/kb/after-a-space-import-fails-it-cant-be-re-imported/)

## 10.6 Процесс

- [ ] Есть отдельный staging-контур.
- [ ] Выполнен хотя бы один Jira dry-run.
- [ ] Выполнен хотя бы один Confluence dry-run.
- [ ] Зафиксировано реальное время импорта/индексации.
- [ ] Подготовлен rollback plan.

---

## 11. Рекомендуемый пошаговый план

## Фаза A — Discovery
1. Зафиксировать точные версии source/target.
2. Собрать inventory плагинов Jira и Confluence.
3. Зафиксировать схему identity mapping из Keycloak.
4. Определить pilot project и pilot space.

## Фаза B — Identity readiness
1. Установить target-compatible SCIM/User Sync apps.
2. Подключить Keycloak.
3. Выполнить pre-provisioning.
4. Проверить pilot users и pilot groups.

## Фаза C — Jira staging migration
1. Подготовить version-aligned source staging.
2. Установить target-compatible app versions.
3. Выполнить export/import через выбранный инструмент.
4. Проверить custom fields, workflows, permissions, authorship.

## Фаза D — Confluence staging migration
1. Сделать XML export pilot space.
2. Проверить archive structure и metadata.
3. Импортировать в staging Confluence 9.
4. Проверить страницы, макросы, attachments, history, mentions.
5. Если архив проблемный — использовать intermediate workaround, а не blind build-number patching. [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html) [10](https://community.atlassian.com/forums/Confluence-questions/the-build-number-in-the-database-7202-doesn-t-match-either-the/qaq-p/1066109)

## Фаза E — Production cutover
1. Freeze window.
2. Повторный provisioning sync.
3. Экспорт/импорт согласованного набора данных.
4. Индексация.
5. Smoke tests.
6. Hypercare.

---

## 12. Что я рекомендую как следующий шаг

### Приоритет 1
Составить **точный список плагинов** на Jira 8 и Confluence 7 и проверить:
- совместимость с Jira 10 / Confluence 9,
- наличие migration path,
- что именно хранится в app data и где это используется.

### Приоритет 2
После inventory сделать **pilot dry-run**:
- 1 проект Jira,
- 1 пространство Confluence,
- уже на фоне рабочего Keycloak provisioning.

### Приоритет 3
Только если Confluence import действительно упрется в metadata/version issue, переходить к отдельной инструкции по `exportDescriptor.properties`.

---

## 13. Короткий итог по исправлению вашего исходного текста

Ваш draft в целом правильный по направлению, но я бы **обязательно скорректировал** три пункта:

1. **Build Number в Confluence**  
   Не формулировать как обязательный шаг. Это только fallback/edge-case, а не штатная процедура. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) [3](https://confluence.atlassian.com/conf719/restoring-a-space-1157467583.html)

2. **Jira partial migration 8 → 10**  
   Нужно явно добавить требование **version alignment source ↔ target** для built-in Project Import или использовать vendor tool с подтвержденным migration path. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/)

3. **Keycloak**  
   Нужно четко разделить:
   - native OIDC/JIT,
   - полноценный SCIM lifecycle provisioning. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html)

---

Если нужно, следующим шагом можно сделать **Приложение A** к этому документу:
- таблицу для инвентаризации Jira apps,
- таблицу для macro audit Confluence,
- шаблон go/no-go checklist для production окна.

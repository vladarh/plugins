# Матрица задач, плагинов, методов и импортируемости экспорта
## Jira 8 → Jira DC 10 / Confluence 7 → Confluence DC 9 / Keycloak только на новых версиях

> Контекст:
> - **DB не мигрируется**
> - **Keycloak/SCIM настраивается только на новых инстансах**
> - нужно понимать **какой инструмент закрывает какую задачу**
> - отдельно нужно видеть **какой экспорт можно потом импортировать**, а какой является только архивом/выгрузкой

---

## 1. Правила чтения матрицы

### Статус покрытия задачи
- **A** — основной инструмент / основной supported путь
- **B** — хороший вспомогательный инструмент
- **C** — узкий или дополнительный сценарий
- **Method** — плагин не нужен, используется штатный продуктовый механизм

### Как читать колонку "Что потом можно импортировать"
- **Да, в продукт/этим же app** — найден источник, из которого видно, что экспорт/пакет используется как migration/import артефакт
- **Да, штатно в продукт** — это native export/import путь продукта
- **Нет подтвержденного round-trip import в собранных источниках** — в найденных источниках app описан как exporter/archive/reporting tool, но не как supported import path обратно в Jira/Confluence

---

# 2. Core migration matrix

## 2.1 Jira partial migration

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Частичная миграция проектов Jira | A | **Configuration Manager for Jira (CMJ)** | Перенос конфигурации и deployment/migration сценарии между Jira | [CMJ history](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history) [1](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history) | **Да, в target через тот же migration workflow app** | Atlassian прямо упоминает **CMJ** как один из third-party tools, которые могут помочь мигрировать project configurations в merge/project import сценариях. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) |
| Частичная миграция проектов Jira | A | **Project Configurator for Jira** | Export/import выбранных проектов и связанной конфигурации | [PC history](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) [2](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) | **Да, в target через тот же app workflow** | Atlassian отдельно описывает export проектов с использованием **Project Configurator** и также упоминает его как migration helper tool. [2](https://confluence.atlassian.com/spaces/ADMINJIRASERVER0817/pages/1167832838/Exporting+projects+from+the+source+instance) [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) |
| Встроенный partial import проектов | Method | **Project Import + XML backup**, но только после выравнивания версий | Native import проекта из backup в существующий Jira | — | **Да, штатно в Jira**, но только в корректном сценарии | Atlassian рекомендует сначала обновить Jira A до версии Jira B, затем сделать XML backup и только потом использовать Project Import. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) |
| Полный перенос whole-instance Jira | Method | **XML backup + Import existing data** | Восстановление полной Jira в новую инсталляцию | — | **Да, штатно в Jira** | Setup Wizard импортирует backup, если ZIP содержит `activeobjects.xml` и `entities.xml`. [1](https://confluence.atlassian.com/adminjiraserver0810/upgrade_import-old-jira-data-1014673280.html) [9](https://support.atlassian.com/jira/kb/importing-data-via-setup-wizard/) |

---

## 2.2 Confluence partial migration

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Частичная миграция пространств Confluence | A | **Native Space XML export/import** | Supported partial migration для spaces | — | **Да, штатно в Confluence** | Space backup можно восстановить в ту же или более новую совместимую версию Confluence. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) |
| Автоматизация import/export spaces | B | **Scripting / UI automation** | Автоматизация загрузки и запуска import | — | **Да, импортируется штатным Confluence import** | Atlassian отдельно пишет, что REST API для Space export/import нет; автоматизация возможна через скрипты/эмуляцию UI. [5](https://support.atlassian.com/confluence/kb/confluence-space-import-via-scripting-tools/) |
| Плагин-эквивалент CMJ для Confluence | Method | **Не требуется как базовый путь** | Для partial migration spaces базовый механизм уже встроен в продукт | — | — | В собранных источниках нет обязательного "CMJ for Confluence" для вашего сценария; основной supported путь уже есть: Space XML export/import. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) |

---

# 3. Keycloak / SSO / SCIM only on target

## 3.1 Jira target

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Native SSO без SCIM | A | **Atlassian native OIDC/SAML** | Только аутентификация / SSO | — | Не относится | Atlassian native OIDC/SAML закрывает auth, но не заменяет полноценный SCIM lifecycle provisioning. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) |
| JIT provisioning on login | B | **Native JIT provisioning** | Создание/обновление пользователя во время логина | — | Не относится | JIT создаёт и обновляет пользователей на логине, но это не полноценный pre-provisioning слой перед миграцией. [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html) [4](https://confluence.atlassian.com/enterprise/working-with-jit-provisioning-1005342571.html) |
| Keycloak SSO + SCIM одним app | A | **miniOrange Jira SAML SSO + User Sync/SCIM** | SSO + User Sync + SCIM | [Jira miniOrange SAML+SCIM history](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync-scim/version-history) [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync?tab=overview) | Не относится | Плагин заявляет поддержку Keycloak и Jira Data Center 8.20.0 - 10.7.4. [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync?tab=overview) |
| SCIM / User Sync отдельно | A | **resolution User Sync for Jira** | SCIM/user/group provisioning sync | [Jira resolution User Sync history](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira/version-history) [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira) | Не относится | Хороший вариант, если auth и provisioning хотите разделить. Плагин заявляет поддержку Keycloak. [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira) |
| SCIM / provisioning отдельно | A | **miniOrange SCIM Provisioning for Jira** | SCIM/user/group provisioning | [Jira miniOrange SCIM history](https://marketplace.atlassian.com/apps/1222000/mo-jira-user-sync-group-sync-directory-sync-using-scim/version-history) [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira) | Не относится | Отдельный SCIM app, если auth хотите решать отдельно. Плагин заявляет поддержку Keycloak и Jira DC 8.20.0 - 11.3.3. [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira) |

---

## 3.2 Confluence target

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Native SSO без SCIM | A | **Atlassian native OIDC/SAML** | Только аутентификация / SSO | — | Не относится | Подходит, если нужен только auth. Для полноценного lifecycle нужен отдельный sync/provisioning app. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) |
| Keycloak SSO + SCIM одним app | A | **miniOrange Confluence SAML SSO + User Sync** | SSO + SCIM/user sync | [Confluence miniOrange SAML+SCIM history](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync/version-history) [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync) | Не относится | Product listing заявляет поддержку Confluence DC 8.5.3 - 10.2.11, то есть покрывает 9.4.1. [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync) |
| SCIM / User Sync отдельно | A | **resolution User Sync for Confluence** | SCIM/user/group provisioning sync | [Confluence resolution User Sync history](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence/version-history) [2](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence?tab=overview&hosting=datacenter) | Не относится | Очень точное попадание под ваш target: listing заявляет поддержку Confluence DC 7.1.0 - 9.4.1. [2](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence?tab=overview&hosting=datacenter) |
| OIDC SSO без SCIM | B | **miniOrange OAuth/OIDC for Confluence** | OIDC login через Keycloak | Product: [10](https://marketplace.atlassian.com/apps/1218360/mo-confluence-oauth-sso-confluence-openid-connect-oidc-sso) | Не относится | Если нужен именно OIDC login и provisioning вы решаете отдельно. [10](https://marketplace.atlassian.com/apps/1218360/mo-confluence-oauth-sso-confluence-openid-connect-oidc-sso) |

---

# 4. Data completeness / archive / audit

## 4.1 Jira

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Максимально полный machine-readable archive | A | **Owly Json Data Exporter for Jira** | JSON export: issues, projects, workflows, comments, history, plugin data | [Owly history](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira/version-history) [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira) | **Нет подтвержденного round-trip import в Jira в собранных источниках** | Listing описывает его как read-only exporter / migration helper / deterministic JSON schema для migrations, audits и continuity, но не как native Jira import package. [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira) |
| Export history + CSV/XLSX/JSON + scheduler | A | **Advanced Export** | recurring exports, standard/custom fields, full history | — | **Нет подтвержденного round-trip import в Jira в собранных источниках** | Listing описывает только export/reporting/scheduling. [3](https://marketplace.atlassian.com/apps/1217474/advanced-export-schedule-reports-any-field-and-history) |
| Human-readable audit / Excel exports | A | **Better Excel Exporter for Jira** | native XLSX, comments, worklogs, change history, report templates | [Better Excel history](https://marketplace.atlassian.com/apps/1212652/better-excel-exporter-for-jira/version-history) [2](https://marketplace.atlassian.com/apps/1212652/better-excel-exporter-for-jira/version-history) | **Нет подтвержденного round-trip import в Jira в собранных источниках** | Это exporter/reporting tool. Midori отдельно показывает comments/worklogs/history/report templates. [1](https://www.midori-global.com/products/better-excel-exporter-for-jira/data-center/) [4](https://www.midori-global.com/products/better-excel-exporter-for-jira/data-center/export-samples/) |
| Issues + attachments + comments + transitions | A | **Exporter for Jira** | export issues to Excel/CSV/PDF + metadata | [Exporter history](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv/version-history) [9](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv-pdf-more?hosting=datacenter&tab=overview) | **Нет подтвержденного round-trip import в Jira в собранных источниках** | Version history уже показывает Jira DC ветки до 10.7.4. [2](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv/version-history) |

---

## 4.2 Confluence

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Основной partial backup пространства | A | **Native Space XML export with attachments** | Контент + attachments в supported формате | — | **Да, штатно в Confluence** | Это основной partial backup/migration artifact для Confluence spaces. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) |
| Проверка полноты контента | A | **Method: Macro audit + staging import** | Проверка макросов, app coverage, storage-format проблем | — | Не относится | Для Confluence это обязательнее любого exporter app. |
| Плагин для обязательного completeness layer | Method | **Не обязателен как базовый слой** | Основной supported артефакт уже есть: space XML | — | — | В собранных источниках нет must-have DC 9.4.1 app, который был бы базовым layer для partial migration так же, как CMJ для Jira. |

---

# 5. Full backup / partial backup на новых версиях

## 5.1 Full backup

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Полный backup Jira DC | A | **Method: DB-native backup + local home + shared home + install/config backup** | Полное восстановление self-managed Jira DC | — | **Да, как инфраструктурное восстановление** | Для self-managed DC это базовый и обязательный full backup. Jira 10 manual upgrade guidance отдельно требует backup DB, home и install dir. [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/) |
| Полный backup Confluence DC | A | **Method: DB-native backup + local home + shared home + install/config backup** | Полное восстановление self-managed Confluence DC | — | **Да, как инфраструктурное восстановление** | Для self-managed DC это базовый full backup. Перед space restore Atlassian рекомендует иметь DB backup. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) |

### Важно
В собранных источниках backup-продукты типа Rewind / GitProtect / Revyz / ProBackup / HYCU относятся в основном к **Cloud backup scenarios**, поэтому для вашего **Data Center** они не должны быть primary backup strategy. [7](https://marketplace.atlassian.com/apps/1228719/gitprotect-io-for-jira-backup-restore-dr-data-management) [6](https://marketplace.atlassian.com/apps/1233056/revyz-command-center-for-confluence) [10](https://marketplace.atlassian.com/apps/1228273/probackup-for-jira) [5](https://marketplace.atlassian.com/apps/1228511/probackup-for-confluence)

---

## 5.2 Partial backup

| Задача | Покрытие | Инструмент / метод | Что закрывает | Version history | Что потом можно импортировать | Комментарий |
|---|---:|---|---|---|---|---|
| Partial backup проектов Jira | A | **CMJ / Project Configurator export package** | Migration-oriented partial backup | [CMJ history](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history) [1](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history), [PC history](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) [2](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) | **Да, тем же migration app / workflow** | Это лучший partial backup слой именно для migration use-case. |
| Partial backup Jira data completeness | A | **Owly JSON Exporter** | Архив данных/истории/plugin data | [Owly history](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira/version-history) [1](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira/version-history) | **Нет подтвержденного round-trip import** | Используйте как archive/completeness dataset. |
| Partial backup Confluence space | A | **Native Space XML export** | Supported partial backup пространства | — | **Да, штатно в Confluence** | Основной partial backup артефакт. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) |

---

# 6. Minimal practical stack

## Jira
### Обязательно
- **CMJ** или **Project Configurator**
- **один identity слой на target**:
  - miniOrange Jira SAML SSO + User Sync/SCIM, **или**
  - resolution User Sync for Jira, **или**
  - miniOrange SCIM Provisioning for Jira

### Желательно
- **Owly Json Data Exporter for Jira**
- **Better Excel Exporter for Jira**

### Опционально
- **Exporter for Jira**

---

## Confluence
### Обязательно
- **Native Space XML export/import**
- **один identity слой на target**:
  - miniOrange Confluence SAML SSO + User Sync, **или**
  - resolution User Sync for Confluence

### Желательно
- **Macro audit**
- **Staging import smoke test**

### Опционально
- scripting automation around import/export

---

# 7. Самый короткий ответ

## Если вам нужна именно final matrix decision:
- **Jira migration** → CMJ / Project Configurator
- **Confluence migration** → native Space XML export/import
- **Keycloak only on target** → miniOrange или resolution
- **Jira completeness** → Owly JSON + Better Excel
- **DC full backup** → без плагина: DB + home/shared-home + config
- **DC partial backup** → Jira: CMJ/Project Configurator + Owly, Confluence: Space XML

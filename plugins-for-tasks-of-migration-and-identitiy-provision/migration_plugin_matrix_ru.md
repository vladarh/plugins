# Матрица задач, плагинов и методов
## Jira 8 → Jira DC 10 / Confluence 7 → Confluence DC 9 / Keycloak только на новых версиях

> Условие: **базы данных не мигрируются**.  
> Значит, плагины и методы ниже нужны не для переноса целой платформы на уровне DB, а для:
> - partial migration,
> - identity onboarding на новых версиях,
> - data completeness,
> - partial/full backup safety net,
> - archive/export/audit.

---

# 1. Самое важное правило

## 1.1 Jira
Для **частичной миграции проектов Jira** в существующую новую Jira Atlassian рекомендует сначала выровнять версии источника и цели, а затем использовать XML backup/project import; также Atlassian прямо упоминает **Configuration Manager for Jira** и **Project Configurator** как инструменты, которые могут помочь в таких сценариях. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) [2](https://confluence.atlassian.com/spaces/ADMINJIRASERVER0817/pages/1167832838/Exporting+projects+from+the+source+instance)

## 1.2 Confluence
Для **частичной миграции пространств Confluence** штатный путь — **native Space XML export/import**. Atlassian также отдельно пишет, что для Space export/import нет REST API, а автоматизация возможна только через scripting / UI emulation. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) [5](https://support.atlassian.com/confluence/kb/confluence-space-import-via-scripting-tools/)

## 1.3 Keycloak
Если Keycloak будет только на новых версиях — это нормальная схема. Native Atlassian OIDC/SAML закрывает SSO, а полноценный lifecycle provisioning требует отдельного SCIM/User Sync app. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html)

---

# 2. Матрица по задачам

## Легенда покрытия
- **A — максимальное покрытие задачи**
- **B — хорошее покрытие, но не всё**
- **C — вспомогательный инструмент / узкий сценарий**
- **Method** — плагин не обязателен, используется штатный метод

---

## 2.1 Core migration

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| Частичная миграция проектов | Jira | A | **Configuration Manager for Jira (CMJ)** | Конфигурация и перенос между Jira-инстансами в migration-сценариях | History: [1](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history) | Основной кандидат для Jira partial migration. Atlassian прямо упоминает CMJ как полезный инструмент в merge/project import сценариях. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) |
| Частичная миграция проектов | Jira | A | **Project Configurator for Jira** | Export/import проектов и конфигурации | History: [2](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) | Второй основной кандидат для Jira partial migration. Тоже прямо упомянут Atlassian. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) [2](https://confluence.atlassian.com/spaces/ADMINJIRASERVER0817/pages/1167832838/Exporting+projects+from+the+source+instance) |
| Частичная миграция пространств | Confluence | A | **Method: native Space XML export/import** | Supported partial migration для spaces | Restore/import docs: [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) | Для Confluence это основной путь. Не нужен “CMJ-аналог” как обязательный инструмент. |
| Автоматизация partial migration spaces | Confluence | B | **Method: scripting / UI automation** | Автоматизация загрузки/импорта space backup | KB: [5](https://support.atlassian.com/confluence/kb/confluence-space-import-via-scripting-tools/) | REST API для space export/import нет. [5](https://support.atlassian.com/confluence/kb/confluence-space-import-via-scripting-tools/) |

---

## 2.2 Identity / Keycloak on target only

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| Native SSO без SCIM | Jira / Confluence | A | **Method: Atlassian native OIDC/SAML** | Аутентификация, SSO | OIDC docs: [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) | Хорошо, если нужен только login/JIT. Не полноценный SCIM lifecycle. [8](https://confluence.atlassian.com/enterprise/openid-connect-for-atlassian-data-center-applications-987142159.html) |
| JIT provisioning на логине | Jira / Confluence | B | **Method: native JIT provisioning** | Создание/обновление пользователей на логине | JIT docs: [1](https://confluence.atlassian.com/enterprise/jit-user-provisioning-1005342579.html) [4](https://confluence.atlassian.com/enterprise/working-with-jit-provisioning-1005342571.html) | Подходит, если users могут создаваться при first login. Не заменяет pre-provisioning перед импортом. |
| SSO + SCIM одним вендором | Jira | A | **miniOrange Jira SAML SSO + User Sync/SCIM** | Keycloak SSO + SCIM/user sync | Product: [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync?tab=overview) · History: [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync-scim/version-history) | Подходит, если хотите одним app закрыть auth + sync. Плагин заявляет поддержку Keycloak и Jira DC 8.20.0 - 10.7.4. [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync?tab=overview) |
| SCIM / group sync / pre-provisioning | Jira | A | **resolution User Sync for Jira** | SCIM/user/group sync | Product: [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira) · History: [1](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira/version-history) | Хороший вариант, если SSO и SCIM хотите разделить. Плагин заявляет Keycloak support. [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira) |
| SCIM / group sync / pre-provisioning | Jira | A | **miniOrange SCIM Provisioning for Jira** | SCIM/user/group provisioning | Product: [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira) · History: [8](https://marketplace.atlassian.com/apps/1222000/mo-jira-user-sync-group-sync-directory-sync-using-scim/version-history) | Отдельный SCIM app, если auth хотите решать независимо. Плагин заявляет Keycloak provisioning и Jira DC 8.20.0 - 11.3.3. [10](https://marketplace.atlassian.com/apps/1222000/scim-provisioning-user-sync-group-sync-for-jira) |
| SSO + SCIM одним вендором | Confluence 9.4.1 | A | **miniOrange Confluence SAML SSO + User Sync** | Keycloak SSO + SCIM/user sync | Product: [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync) · History: [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync/version-history) | Для вашего Confluence 9.4.1 этот вариант выглядит практически полезным: product listing заявляет поддержку Confluence DC 8.5.3 - 10.2.11 и Keycloak provisioning. [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync) |
| SCIM / group sync / pre-provisioning | Confluence 9.4.1 | A | **resolution User Sync for Confluence** | SCIM/user/group sync | Product: [2](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence?tab=overview&hosting=datacenter) · History: [1](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence/version-history) | Очень точное попадание в ваш target: listing заявляет поддержку Confluence DC 7.1.0 - 9.4.1. [2](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence?tab=overview&hosting=datacenter) |
| OIDC SSO без SCIM | Confluence 9.4.1 | B | **miniOrange OAuth/OIDC for Confluence** | OIDC login через Keycloak | Product: [10](https://marketplace.atlassian.com/apps/1218360/mo-confluence-oauth-sso-confluence-openid-connect-oidc-sso) | Если нужен только OIDC login. SCIM закрывать отдельно. |

---

## 2.3 Data completeness / archive / audit after migration

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| Полнота Jira-данных в структурированном виде | Jira | A | **Owly Json Data Exporter for Jira** | JSON export: issues, projects, workflows, comments, history, plugin data | Product: [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira) · History: [1](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira/version-history) | Лучший дополнительный слой для completeness/archive, если нужен machine-readable dataset и migration helper. [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira) |
| История, CSV/XLSX/JSON, schedule | Jira | A | **Advanced Export** | recurring exports, custom fields, history, JSON/CSV/XLSX | Product: [3](https://marketplace.atlassian.com/apps/1217474/advanced-export-schedule-reports-any-field-and-history) | Хороший универсальный экспортер, если нужен history + scheduler + JSON/CSV/XLSX. [3](https://marketplace.atlassian.com/apps/1217474/advanced-export-schedule-reports-any-field-and-history) |
| Excel-quality audit / human-readable exports | Jira | A | **Better Excel Exporter for Jira** | native XLSX, comments, worklogs, change history, BI/reporting | Product: [2](https://marketplace.atlassian.com/apps/1212652/better-excel-exporter-for-jira) · History: [2](https://marketplace.atlassian.com/apps/1212652/better-excel-exporter-for-jira/version-history) | Лучший вариант для бизнес-аудита и человекочитаемых выгрузок. [1](https://www.midori-global.com/products/better-excel-exporter-for-jira/data-center/) [4](https://www.midori-global.com/products/better-excel-exporter-for-jira/data-center/export-samples/) |
| Issues + comments + transitions + attachments export | Jira | A | **Exporter for Jira** | Excel/CSV/PDF export с metadata | Product: [9](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv-pdf-more?hosting=datacenter&tab=overview) · History: [2](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv/version-history) | Уже не ограничен 10.5.1: version history показывает версии для Jira DC 10.7.4 и ветки 10.6.1. [2](https://marketplace.atlassian.com/apps/1212073/exporter-for-jira-export-issues-to-excel-csv/version-history) |
| Полнота Confluence content | Confluence | A | **Method: Space XML export with attachments** | Основной migration + partial backup артефакт | Docs: [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) | Для Confluence partial migration это основной completeness-слой. |
| Проверка совместимости контента | Confluence | A | **Method: Macro audit** | Проверка, что целевой набор apps/macros покрывает source content | Guidance from process docs in your runbook | Для Confluence это обязательнее любого export plugin. |

---

## 2.4 Full backup for new DC versions

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| Full backup новой Jira DC | Jira | A | **Method: DB-native backup + local home + shared home + install/config backup** | Полное восстановление self-managed DC | Jira 10 manual upgrade guidance: [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/) | Для self-managed DC это основной и обязательный full backup. Плагин не нужен как primary strategy. |
| Full backup новой Confluence DC | Confluence | A | **Method: DB-native backup + local home + shared home + install/config backup** | Полное восстановление self-managed DC | Confluence restore guidance: [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) | Аналогично: primary backup strategy должна быть инфраструктурной, а не app-based. |

**Вывод:** среди найденных backup apps хорошие решения в основном относятся к Cloud (например Rewind / GitProtect / Revyz / ProBackup / HYCU listings в найденных результатах описывают Cloud backup scenarios), поэтому для вашего **Data Center** я **не рекомендую** строить full backup на Marketplace backup app как базовую стратегию. [7](https://marketplace.atlassian.com/apps/1228719/gitprotect-io-for-jira-backup-restore-dr-data-management) [6](https://marketplace.atlassian.com/apps/1233056/revyz-command-center-for-confluence) [10](https://marketplace.atlassian.com/apps/1228273/probackup-for-jira) [5](https://marketplace.atlassian.com/apps/1228511/probackup-for-confluence)

---

## 2.5 Partial backup for new DC versions

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| Partial backup проектов перед/после миграции | Jira | A | **CMJ / Project Configurator export package** | Конфиг + миграционный пакет | Histories: [1](https://marketplace.atlassian.com/apps/1211611/configuration-manager-for-jira-cmj/version-history), [2](https://marketplace.atlassian.com/apps/1211147/project-configurator-for-jira/version-history) | Это основной partial backup именно для migration tasks. |
| Partial backup issues/history/plugin data | Jira | A | **Owly JSON Exporter** | Архив данных и completeness | Product/history: [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira), [1](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira/version-history) | Лучший machine-readable partial backup слой. |
| Partial backup spaces | Confluence | A | **Method: Space XML export** | Partial backup пространства | Docs: [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html) | Основной supported partial backup артефакт. |
| Частичный экспорт страниц/контента в переносимом формате | Confluence | B | **No plugin confirmed as must-have for DC 9.4.1 in gathered sources** | Контентный архив лучше решать native XML + macro audit | — | На текущем наборе найденных источников я не вижу DC 9.4.1 migration app, который был бы настолько же базовым, как CMJ для Jira. |

---

## 2.6 Automation / admin helper tasks

| Задача | Продукт | Покрытие | Что использовать | Что закрывает | Version history / link | Комментарий |
|---|---|---:|---|---|---|---|
| CLI/automation around Atlassian products | Jira / Confluence | B | **Atlassian CLI / Confluence CLI** | Админские и интеграционные automation scenarios | Product: [2](https://marketplace.atlassian.com/apps/10886/atlassian-command-line-interface-cli), [8](https://marketplace.atlassian.com/apps/284/confluence-command-line-interface-cli) | Может быть полезно для automation, но это не migration engine. Для Confluence найденный CLI listing ориентирован уже на DC 10.x, а не на ваш 9.4.1. [8](https://marketplace.atlassian.com/apps/284/confluence-command-line-interface-cli) |

---

# 3. Итог: минимальный стек, который реально имеет смысл

## 3.1 Если нужен минимальный, но практичный стек

### Jira
1. **CMJ** или **Project Configurator** — ядро partial migration. [3](https://support.atlassian.com/jira/kb/merge-multiple-instances-of-jira-data-center/) [2](https://confluence.atlassian.com/spaces/ADMINJIRASERVER0817/pages/1167832838/Exporting+projects+from+the+source+instance)
2. **miniOrange Jira SAML SSO + User Sync/SCIM** *или* **resolution User Sync for Jira** — identity на новом Jira. [1](https://marketplace.atlassian.com/apps/1215430/single-sign-on-saml-sso-for-jira-sso-saml-user-sync?tab=overview) [5](https://marketplace.atlassian.com/apps/1219399/user-sync-scim-provisioning-group-sync-for-jira)
3. **Owly Json Data Exporter for Jira** — completeness/archive. [4](https://marketplace.atlassian.com/apps/1838088111/owly-json-data-exporter-for-jira)
4. **Better Excel Exporter** — если нужны human-readable audit exports. [1](https://www.midori-global.com/products/better-excel-exporter-for-jira/data-center/)

### Confluence
1. **Native Space XML export/import** — ядро partial migration. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)
2. **miniOrange Confluence SAML SSO + User Sync** *или* **resolution User Sync for Confluence** — identity на новом Confluence. [1](https://marketplace.atlassian.com/apps/1215542/single-sign-on-saml-sso-for-confluence-sso-saml-user-sync) [2](https://marketplace.atlassian.com/apps/1219400/user-sync-scim-provisioning-group-sync-for-confluence?tab=overview&hosting=datacenter)
3. **Macro audit + staging import tests** — обязательный completeness check.

---

# 4. Что не считать обязательным

- Backup apps из Marketplace для **Data Center full backup** — не базовая стратегия. Для self-managed DC опирайтесь на **DB + home/shared-home**. [8](https://support.atlassian.com/jira/kb/how-to-manually-upgrade-to-jira-10-as-the-installer-method-is-now-deprecated/) [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)
- Поиск “CMJ for Confluence” — не обязательный путь. Для Confluence partial migration основной путь уже есть: **space XML import/export**. [2](https://confluence.atlassian.com/doc/restoring-a-space-152036.html)

---

# 5. Самый короткий practical answer

## Для migration
- **Jira:** CMJ / Project Configurator  
- **Confluence:** native Space XML export/import

## Для Keycloak only on new systems
- **Jira:** miniOrange Jira SAML SSO + User Sync/SCIM **или** resolution User Sync for Jira  
- **Confluence:** miniOrange Confluence SAML SSO + User Sync **или** resolution User Sync for Confluence

## Для completeness
- **Jira:** Owly JSON Exporter (+ Better Excel Exporter, если нужен аудит в XLSX)  
- **Confluence:** native XML + macro audit

## Для full backup of new versions
- **Без плагина:** DB + local home + shared home + config backup

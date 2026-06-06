Отлично. Тогда даю **короткий decisive checklist** именно под ваш случай:

- **namespace**: `tech-confluence-jira`
- **DB**: `jira_db`
- **pod**: `jira-0`

Цель — понять, что у вас происходит с **Jira Software**:

1. package **вообще отсутствует**
2. package **есть, но disabled**
3. package **есть, но конфликтует**
4. проблема только в **license/app state**

---

# 1. Проверить, есть ли Jira Software jars

## В shared-home plugin area
```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
echo "=== shared-home installed-plugins ==="
ls -1 /var/atlassian/application-data/shared-home/plugins/installed-plugins 2>/dev/null | grep -Ei "jira-software|greenhopper" || echo "NONE"
'
```

## В install dir
```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
echo "=== install dir search ==="
find /opt/atlassian/jira -type f 2>/dev/null | grep -Ei "jira-software|greenhopper" | head -100 || echo "NONE"
'
```

---

## Как интерпретировать

### Если везде `NONE`
Тогда:
- у вас **реально нет Jira Software package**
- это уже сильный аргумент, что текущий образ/раскладка не Jira Software-based
- тогда правильный путь — **не app-fix, а product/image fix**

### Если файлы есть
Тогда идём дальше: они либо disabled, либо конфликтуют.

---

# 2. Проверить pluginstate в БД

Нужно посмотреть, не отключены ли software-модули.

## Если у вас есть доступ к psql из pod с БД
Запрос:

```sql
SELECT pluginkey, pluginenabled
FROM pluginstate
WHERE pluginkey LIKE 'com.atlassian.jira%software%'
   OR pluginkey LIKE '%greenhopper%'
ORDER BY pluginkey;
```

---

## Если вы не знаете, из какого pod стучаться в Postgres
Сначала найдите DB pod:

```bash
kubectl -n tech-confluence-jira get pods -o wide
```

Если есть postgres pod, выполнить примерно так:

```bash
kubectl -n tech-confluence-jira exec -it <postgres-pod> -- psql -d jira_db -c "
SELECT pluginkey, pluginenabled
FROM pluginstate
WHERE pluginkey LIKE 'com.atlassian.jira%software%'
   OR pluginkey LIKE '%greenhopper%'
ORDER BY pluginkey;
"
```

---

## Как интерпретировать

### Если строки есть и `pluginenabled = false`
Тогда:
- Jira Software **есть**, но его модули отключены
- это не “нужно скачать app”, а **нужно re-enable / cache cleanup**

### Если строк нет совсем
Тогда:
- Jira Software package, вероятно, не установлен/не загружен

---

# 3. Проверить, нет ли старых конфликтующих jars

```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
echo "=== all installed-plugins ==="
ls -1 /var/atlassian/application-data/shared-home/plugins/installed-plugins 2>/dev/null | sort
'
```

Смотрите:
- несколько версий `jira-software-*`
- несколько версий `jira-greenhopper-plugin-*`
- старые файлы после миграций/апгрейдов

---

# 4. Проверить, что Jira думает про system plugins

Из вашего лога уже видно:

- **User Plugins = 0**
- **System Plugins = 218**
- `Authentication for Atlassian Data Center` уже **enabled**
- Jira стартует успешно после `postgres72`

Это хорошо.  
Теперь проблема явно **не в общем старте**, а в **Jira Software application layer**.

---

# 5. Проверить через UI симптомы

Проверьте в Jira:

## 5.1 Есть ли:
- **Boards**
- **Backlog**
- возможность создать **Software project**

## 5.2 Что написано в:
- **⚙ → Applications → Versions & Licenses**

## 5.3 Project type у проектов
Если проект не `Software`, то backlog и boards не появятся даже при нормальном Jira Software. Atlassian отдельно это указывает. [1](https://support.atlassian.com/jira/kb/the-jira-software-project-sidebar-is-missing-the-backlog-and-active-sprint-links/)

---

# 6. Решение по веткам

---

## Ветка A — files есть, pluginstate = false
### Значит Jira Software installed, but disabled

### Тогда действия:
1. backup DB
2. очистить plugin cache
3. re-enable disabled plugins
4. restart

### Cache cleanup
На Jira pod:

```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
echo "Cleaning plugin cache..."
rm -rf /var/atlassian/application-data/shared-home/plugins/.bundled-plugins/*
rm -rf /var/atlassian/application-data/shared-home/plugins/.osgi-plugins/*
rm -rf /var/atlassian/application-data/jira/plugins/.bundled-plugins/*
rm -rf /var/atlassian/application-data/jira/plugins/.osgi-plugins/*
rm -rf /opt/atlassian/jira/work/*
rm -rf /opt/atlassian/jira/temp/*
echo "Done"
'
```

Atlassian рекомендует такой подход для plugin reset / cache issues. [3](https://support.atlassian.com/jira/kb/how-to-perform-an-app-plugin-reset-in-jira-data-center/)

### Потом
перезапустить pod:

```bash
kubectl -n tech-confluence-jira delete pod jira-0
```

---

## Ветка B — files есть, но pluginstate пустой/непонятный, и есть несколько версий jars
### Значит конфликт jars

Тогда:
- убрать старые `jira-software*` / `jira-greenhopper*`
- оставить только корректную совместимую версию
- очистить cache
- restart

---

## Ветка C — files нет вообще
### Значит Jira Software package реально отсутствует

Тогда уже 2 варианта:

### Вариант C1 — если UI даёт install и реально может скачать
В вашем случае маловероятно, потому что:
- `upm.pac.disable=true`
- isolated environment

### Вариант C2 — image/product fix
Это означает:
- текущий pod запущен не из полноценного Jira Software distribution
- и правильный путь — **смена базового образа / сборки**, а не лечение через обычный app upload

---

# 7. Что я думаю по вашим логам сейчас

Из того, что вы уже показали:

- `postgres72` реально починил startup
- `Authentication for Atlassian Data Center` теперь **enabled**
- Jira стартует успешно
- проблема с Jira Software осталась отдельно
- **Istio тут не первопричина**
- главная следующая проверка — **есть ли software jars вообще**

---

# 8. Что сделать прямо сейчас

Выполните **эти 2 команды** и пришлите вывод:

## Команда 1
```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
echo "=== shared-home installed-plugins ==="
ls -1 /var/atlassian/application-data/shared-home/plugins/installed-plugins 2>/dev/null | grep -Ei "jira-software|greenhopper" || echo "NONE"
echo
echo "=== install dir search ==="
find /opt/atlassian/jira -type f 2>/dev/null | grep -Ei "jira-software|greenhopper" | head -100 || echo "NONE"
'
```

## Команда 2
```bash
kubectl -n tech-confluence-jira get pods -o wide
```

---

# 9. После этого
Я вам сразу скажу, какой у вас сценарий:

- **disabled**
- **missing**
- **conflict**
- **needs image change**
- или можно лечить через **cache/plugin cleanup**

Если хотите, после вывода я дам уже **точные команды next step** без развилок.

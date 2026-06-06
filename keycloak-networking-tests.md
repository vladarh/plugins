Ниже — **быстрый и практичный checklist** для **Confluence**, чтобы проверить, нет ли у вас **аналогичной проблемы**, как была у Jira:

- приложение/модули не установлены,
- есть скрытые ошибки старта,
- плагины лежат “в образе”, но не активированы,
- есть проблемы с identity/SSO,
- есть issues с shared-home / plugin cache / attachment-like контентом.

У вас namespace тот же:

- **namespace**: `tech-confluence-jira`

Нужно только знать имя pod Confluence.  
Сначала получите его:

```bash
kubectl -n tech-confluence-jira get pods -o wide
```

Предположим, что pod называется, например, `confluence-0`. Ниже я буду использовать `<confluence-pod>`.

---

# 1. Проверить, стартует ли Confluence “чисто”

## Что искать в логах
```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=300
```

### Ищем:
- `ERROR`
- `failed to load`
- `We couldn't start`
- `Authentication`
- `plugin`
- `UPM`
- `SSO`
- `Could not`
- `Data Center`
- `cluster`

Удобнее сразу grep:

```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=1000 | grep -Ei "error|failed|plugin|authentication|sso|upm|fatal|could not"
```

---

# 2. Проверить, нет ли активного `javaagent`

Раз у Jira у вас шел с `-javaagent`, нужно проверить и Confluence.

```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
ps -ef | grep confluence | grep -v grep
'
```

Или:

```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=300 | grep -i javaagent
```

## Нормально:
- **нет** `-javaagent:/opt/atlassian/atlassian-agent/...`

Если есть — это тот же нежелательный фактор, что и в Jira.

---

# 3. Проверить, есть ли признаки “bundled / app package present but not installed”

Для Jira это было видно через:
- package в install dir,
- пустой `installed-plugins`.

Для Confluence можно проверить похожим образом.

---

## 3.1 Проверить plugin area в shared-home
```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
echo "=== shared-home plugin dirs ==="
ls -la /var/atlassian/application-data/shared-home/plugins 2>/dev/null || true
echo
echo "=== installed-plugins ==="
ls -la /var/atlassian/application-data/shared-home/plugins/installed-plugins 2>/dev/null || true
echo
echo "=== .bundled-plugins ==="
ls -la /var/atlassian/application-data/shared-home/plugins/.bundled-plugins 2>/dev/null || true
echo
echo "=== .osgi-plugins ==="
ls -la /var/atlassian/application-data/shared-home/plugins/.osgi-plugins 2>/dev/null || true
'
```

---

## 3.2 Проверить bundled/plugins в install dir
```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
echo "=== search for bundled plugins in install dir ==="
find /opt/atlassian/confluence -type f 2>/dev/null | grep -Ei "bundled|plugin|sso|oauth|openid|saml" | head -100
'
```

---

## Что это даст
Если у вас будет похожая ситуация, как в Jira, то вы увидите:

- нужные плагины/артефакты **в install dir**
- но ничего полезного **в shared-home/plugins/installed-plugins**

---

# 4. Проверить system/plugins status из UI / support info

Если UI доступен, проверьте:

- **Manage apps**
- все ли system plugins enabled
- нет ли red/yellow status рядом с:
  - SSO
  - authentication
  - user management
  - macro-related apps

---

# 5. Проверить Confluence SSO / authentication layer

У вас для Confluence важна future-интеграция с Keycloak.  
Пока можно проверить, что сам built-in auth слой жив.

В логах ищите:
- `Authentication for Atlassian Data Center`
- `OIDC`
- `SAML`
- `SSO`
- `Crowd`
- `user directory`

```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=1000 | grep -Ei "authentication|oidc|saml|sso|crowd|directory"
```

Если появятся ошибки вроде:
- plugin not started
- failed to load auth plugin
- user directory issue

это уже отдельный красный флаг.

---

# 6. Проверить user directories
В Confluence через UI:
- **General Configuration / User & Security**
- **User Directories**

Проверить:
- internal directory active?
- нет ли broken LDAP entries?
- если пока Keycloak не подключен — это нормально, что только internal directory активен

---

# 7. Проверить attachment/content storage путь
Аналогично Jira attachment check, только для Confluence важнее:
- shared-home mounted
- local-home mounted
- content/attachments реально читаются

```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
echo "=== local home ==="
ls -ld /var/atlassian/application-data/confluence
echo
echo "=== shared home ==="
ls -ld /var/atlassian/application-data/shared-home
echo
df -h /var/atlassian/application-data/confluence
df -h /var/atlassian/application-data/shared-home
'
```

---

# 8. Проверить права на Confluence homes
У вас для Confluence в values был:
- `fsGroup: 2002`

Проверьте фактические права:

```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
id
ls -ld /var/atlassian/application-data/confluence
ls -ld /var/atlassian/application-data/shared-home
'
```

## Что вы хотите видеть
- пользователь процесса Confluence может читать/писать
- группа согласована с `2002`
- нет `permission denied` в логах

---

# 9. Проверить, нет ли проблемы с Base URL / self-access
Как и у Jira, у Confluence потом будет важен self-access.

Из pod:

```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
curl -k -I -L https://<your-confluence-url> || wget -S --spider https://<your-confluence-url>
'
```

Если позже будете цеплять Keycloak по OIDC, это особенно важно, потому что OIDC требует server-to-server connectivity.

---

# 10. Проверить, нет ли известных проблем с `exportDescriptor.properties` / Space XML compatibility
Если вас интересует готовность Confluence к partial migration spaces, проверьте заранее:

- space export делается штатно?
- архив содержит:
  - `entities.xml`
  - `exportDescriptor.properties`

Но это уже не startup health check, а migration readiness check.

---

# 11. Самый короткий decisive checklist

Вот минимальный набор команд.

## 11.1 Найти pod
```bash
kubectl -n tech-confluence-jira get pods -o wide
```

## 11.2 Проверить ошибки старта
```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=1000 | grep -Ei "error|failed|plugin|authentication|sso|upm|fatal|could not"
```

## 11.3 Проверить `javaagent`
```bash
kubectl -n tech-confluence-jira logs <confluence-pod> --tail=300 | grep -i javaagent
```

## 11.4 Проверить plugin dirs
```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
ls -la /var/atlassian/application-data/shared-home/plugins 2>/dev/null || true
ls -la /var/atlassian/application-data/shared-home/plugins/installed-plugins 2>/dev/null || true
'
```

## 11.5 Проверить install dir plugin payload
```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
find /opt/atlassian/confluence -type f 2>/dev/null | grep -Ei "bundled|plugin|sso|oauth|openid|saml" | head -100
'
```

## 11.6 Проверить homes + permissions
```bash
kubectl -n tech-confluence-jira exec -it <confluence-pod> -- sh -c '
id
ls -ld /var/atlassian/application-data/confluence
ls -ld /var/atlassian/application-data/shared-home
df -h /var/atlassian/application-data/confluence
df -h /var/atlassian/application-data/shared-home
'
```

---

# Как интерпретировать результат

## Если:
- нет ERROR в логах,
- нет `javaagent`,
- shared-home/local-home доступны,
- plugin dirs не выглядят сломанными,
- system/plugins в UI green,

### значит у Confluence **нет явной проблемы уровня Jira**.

---

## Если:
- есть auth/plugin/SSO errors,
- есть `javaagent`,
- есть broken plugin state,
- есть permission/storage errors,

### значит Confluence тоже надо чистить до миграции.

---

# Мой практический совет
Самый быстрый путь сейчас:

1. дайте мне:
   - имя pod Confluence
2. выполните 4 команды:
   - logs grep
   - javaagent grep
   - plugin dirs
   - homes/permissions

И я вам сразу скажу:
- **Confluence clean**
- **Confluence suspicious**
- **Confluence has same type of issue as Jira**
- **Confluence ready for migration staging**

Если хотите, я могу сразу подготовить вам **один готовый bash block** под namespace `tech-confluence-jira`, куда вы только подставите имя Confluence pod.

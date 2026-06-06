Да — по вашему выводу уже видно главное:

## Что показала диагностика

### 1. В `shared-home/plugins/installed-plugins` **ничего нет**
```text
NONE
```

### 2. Но в install dir **Jira Software 10.6 package уже есть**
У вас есть целый каталог:

```text
/opt/atlassian/jira/atlassian-jira/WEB-INF/application-installation/jira-software-application/
```

и в нём лежат нужные jar'ы:

- `jira-software-application-10.6.0.jar`
- `jira-software-plugin-10.6.0.jar`
- `jira-greenhopper-plugin-10.6.0.jar`
- `jira-portfolio-10.6.0.jar`
- `portfolio-plugin-10.6.0.jar`
- и другие

### 3. В `pluginstate` по software нет disabled rows
```text
(0 rows)
```

---

# Вывод

## Это **не проблема Istio**
Потому что package уже **локально есть в контейнере**.  
Сеть не нужна, чтобы его поставить вручную.

## Это **не проблема отсутствия tar.gz**
Потому что нужные Jira Software jar'ы **уже распакованы внутри image**.

## Это **не проблема disabled modules в БД**
Потому что `pluginstate` пустой по software.

---

# Самый вероятный сценарий

У вас сейчас ситуация такая:

- Jira Software **package физически есть**
- но он **не установлен в plugin area**
- то есть он **не активирован как application**

Другими словами:

## Вам не нужно скачивать `tar.gz`
## Вам не нужен внешний download
## Вам нужно **развернуть уже имеющиеся Jira Software jars в shared-home plugin area**

---

# Можно ли “run this as the app”?
## Да — по сути **да**
Но не через `tar.gz`, а через уже существующие jar'ы из:

```text
/opt/atlassian/jira/atlassian-jira/WEB-INF/application-installation/jira-software-application/
```

Это и есть ваш локальный application payload.

---

# Что делать

Ниже даю **безопасный и прямой путь**.

---

## Шаг 1. Скопировать Jira Software jars в shared-home plugin area

В вашем namespace/pod:

```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
set -e

SRC=/opt/atlassian/jira/atlassian-jira/WEB-INF/application-installation/jira-software-application
DST=/var/atlassian/application-data/shared-home/plugins/installed-plugins

mkdir -p "$DST"

echo "Copying Jira Software application jars..."
cp -av "$SRC"/*.jar "$DST"/

echo "Setting permissions..."
chgrp 2001 "$DST"/*.jar || true
chmod 0664 "$DST"/*.jar

echo "Done. Current files:"
ls -1 "$DST"
'
```

---

## Шаг 2. Очистить plugin cache

```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
set -e

echo "Cleaning plugin caches..."

rm -rf /var/atlassian/application-data/shared-home/plugins/.bundled-plugins/*
rm -rf /var/atlassian/application-data/shared-home/plugins/.osgi-plugins/*

rm -rf /var/atlassian/application-data/jira/plugins/.bundled-plugins/*
rm -rf /var/atlassian/application-data/jira/plugins/.osgi-plugins/*

rm -rf /opt/atlassian/jira/work/*
rm -rf /opt/atlassian/jira/temp/*

echo "Plugin caches cleaned."
'
```

---

## Шаг 3. Перезапустить pod

Так как у вас один pod:

```bash
kubectl -n tech-confluence-jira delete pod jira-0
```

Kubernetes поднимет его заново.

---

# После старта проверить

## 1. Проверить, что software jars остались в shared-home
```bash
kubectl -n tech-confluence-jira exec -it jira-0 -- sh -c '
ls -1 /var/atlassian/application-data/shared-home/plugins/installed-plugins | grep -Ei "jira-software|greenhopper|portfolio"
'
```

## 2. Проверить UI
В Jira:
- `⚙ → Applications → Versions & Licenses`
- должно исчезнуть:
  - **“Jira Software is licensed but not currently installed”**

## 3. Проверить функциональность
- есть ли **Boards**
- есть ли **Backlog**
- можно ли создать **Software project**

---

# Если после этого сообщение останется

Тогда следующий шаг:
1. снять support zip
2. проверить application.xml
3. посмотреть, загрузились ли `jira-software-*` модули
4. уже потом смотреть на license/app registration layer

Но по вашей диагностике сейчас **самый логичный и прямой шаг — именно копирование локальных Jira Software jars в `shared-home/plugins/installed-plugins`**.

---

# Нужно ли это делать через UI?
## Не обязательно
Так как у вас:
- isolated environment,
- package уже локально есть,
- UPM/Versions & Licenses может хотеть download path, который у вас не пройдет,

### manual copy сейчас выглядит лучше и чище.

---

# Что не нужно делать
- не нужно скачивать `tar.gz`
- не нужно искать external OBR для 10.6
- не нужно связывать это с Istio
- не нужно трогать `pluginstate` по Jira Software, потому что он уже пустой
- не нужно использовать `atlassian-agent` для этого

---

Если хотите, я могу следующим сообщением дать вам **одну объединённую команду**:
- copy jars
- cleanup caches
- restart pod
- verify files

то есть прям **ready-to-run block** под ваш namespace.

Да — сделаем по-русски.

## Коротко по главному

1. **Да, можно настроить Jira и Confluence для работы в изолированном контуре**:  
   - отключить Marketplace/UPM-доступ наружу;
   - загружать плагины из файлов (`.jar` / `.obr`);
   - использовать ваши PVC в Kubernetes/Alauda.

2. **Нет, с “Atlassian Agent” я не помогу.**  
   Если вы имеете в виду обход лицензирования/взлом активации — это нелегально.  
   Для платных приложений используйте **официальную лицензию**, trial или ключ от вендора/Marketplace.

3. **`.jar` и `.obr` нужно обрабатывать по-разному**:
   - **JAR** — можно класть в файловую систему/PVC;
   - **OBR** — лучше загружать через **Upload app** в UPM;
   - для **Confluence** класть `.obr` в scan-directory **не рекомендуется**, это известная проблема. [1](https://confluence.atlassian.com/doc/managing-add-ons-or-plugins-25788666.html) [2](https://jira.atlassian.com/browse/CONFSERVER-95831)

---

# 1) Можно ли активировать плагины через Atlassian Agent?

**Нет.**  
Я не могу помогать с использованием **Atlassian Agent** или других способов обхода лицензирования.

### Что делать правильно
После установки плагина:
- если он **бесплатный** — обычно достаточно установки;
- если он **платный** — активируете его **официальным ключом** вендора / Atlassian Marketplace;
- если контур изолированный — ставите плагин **из файла**, а потом вносите **валидную лицензию вручную**.

Если хотите, я могу отдельно подсказать **легальную офлайн-схему**:
- установка из `.jar` / `.obr`,
- применение действующего license key,
- что делать, если вендор требует офлайн-активацию.



---

# 2) Что добавить в values для Jira

Для **Jira** добавьте:

```yaml
jira:
  additionalJvmArgs:
    - "-Dupm.pac.disable=true"
    - "-Dupm.plugin.upload.enabled=true"
    # если будет ошибка подписи приложений:
    # - "-Datlassian.upm.signature.check.disabled=true"
    # если плагины долго стартуют:
    # - "-Datlassian.plugins.enable.wait=300"
```

### Что это делает
- `-Dupm.pac.disable=true` — отключает доступ UPM к Atlassian Marketplace. [2](https://confluence.atlassian.com/upm/problems-connecting-to-the-atlassian-marketplace-273875718.html)
- `-Dupm.plugin.upload.enabled=true` — возвращает кнопку **Upload app** в Jira. Atlassian отдельно пишет, что для Kubernetes/Helm это делается через `additionalJvmArgs`. [3](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)

---

# 3) Что добавить в values для Confluence

Для **Confluence** добавьте:

```yaml
confluence:
  additionalJvmArgs:
    - "-Dupm.pac.disable=true"
    - "-Dupm.plugin.upload.enabled=true"
    - "-Datlassian.confluence.plugin.scan.directory=/var/atlassian/application-data/shared-home/plugins/installed-plugins"
    # если будет ошибка подписи приложений:
    # - "-Datlassian.upm.signature.check.disabled=true"
```

### Что это делает
- отключает Marketplace/UPM наружу [2](https://confluence.atlassian.com/upm/problems-connecting-to-the-atlassian-marketplace-273875718.html)
- включает **Upload app** в Confluence [1](https://confluence.atlassian.com/doc/managing-add-ons-or-plugins-25788666.html)
- заставляет Confluence сканировать каталог с плагинами при старте [1](https://confluence.atlassian.com/doc/managing-add-ons-or-plugins-25788666.html)

---

# 4) Куда класть файлы на PVC

У вас уже есть `sharedHome`, значит можно использовать его.

## Jira
Для кластера Jira ручная установка JAR делается в:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins/
```

Atlassian прямо пишет, что в cluster/DC-режиме нужно использовать **shared home/plugins/installed-plugins**. [3](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)

## Confluence
Для Confluence с `atlassian.confluence.plugin.scan.directory` используйте:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins/
```

---

# 5) Как обрабатывать `.jar` и `.obr`

## Вариант A — если у вас `.jar`

### Jira + JAR
Просто копируете `.jar` в:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins/
```

и перезапускаете pod(ы). [3](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)

### Confluence + JAR
Копируете `.jar` в:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins/
```

и перезапускаете pod(ы).  
Confluence подхватит JAR из scan-directory на старте. [1](https://confluence.atlassian.com/doc/managing-add-ons-or-plugins-25788666.html)

---

## Вариант B — если у вас `.obr`

### Jira + OBR
Для Jira **лучший путь** — **Upload app** через UI после включения:

```yaml
- "-Dupm.plugin.upload.enabled=true"
```

UPM принимает файлы **JAR и OBR**. [8](https://confluence.atlassian.com/upm/installing-marketplace-apps-273875715.html)

### Confluence + OBR
Для Confluence тоже лучше **Upload app** через UI.  
**Не кладите `.obr` в `plugin.scan.directory`**, потому что это известная проблема: Confluence scan-directory рассчитан на plugin jars, а `.obr` через этот путь работает некорректно. [1](https://confluence.atlassian.com/doc/managing-add-ons-or-plugins-25788666.html) [2](https://jira.atlassian.com/browse/CONFSERVER-95831)

---

# 6) Если нужно “конвертировать” OBR

Тут важный момент:

## OBR → это не просто “тот же JAR”
OBR — это контейнер/архив с одним или несколькими plugin bundles и зависимостями. [7](https://developer.atlassian.com/server/framework/atlassian-sdk/bundling-extra-dependencies-in-an-obr/)

### Поэтому:
- **не каждый OBR можно просто “превратить” в один JAR**;
- иногда внутри OBR лежит:
  - основной plugin JAR,
  - дополнительные JAR-зависимости,
  - `obr.xml`.

## Когда extraction допустим
Если вам нужно использовать:
- `additionalBundledPlugins`,
- или Confluence scan-directory,
- или класть артефакты вручную на PVC,

то OBR можно **распаковать и посмотреть JAR-файлы** внутри.

### Пример
```bash
unzip my-plugin.obr -d my-plugin-obr
find my-plugin-obr -name "*.jar"
```

После этого обычно видно:
- основной JAR,
- возможные dependency JAR.

## Но важно
- Для **Confluence** в scan-directory кладите **JAR**, не OBR. [2](https://jira.atlassian.com/browse/CONFSERVER-95831)
- Для **Jira** извлекать JAR из OBR можно **только если вендор это допускает**.  
  Иначе правильнее грузить **сам OBR через Upload app**.

---

# 7) Если хотите управлять JAR через Helm, а не через UI

У вас в values уже есть:
- `jira.additionalBundledPlugins`
- `confluence.additionalBundledPlugins`

Это пригодно для **JAR-only** сценария. В Atlassian chart указано, что `additionalBundledPlugins` — это путь для дополнительных bundled plugins, и **только `.jar`**, а `.obr` нужно распаковать, чтобы получить JAR. [4](https://artifacthub.io/packages/helm/atlassian-data-center/confluence)

## Пример для Jira
```yaml
jira:
  additionalBundledPlugins:
    - volumeName: shared-home
      subDirectory: helm-plugins
      fileName: my-jira-plugin.jar
```

## Пример для Confluence
```yaml
confluence:
  additionalBundledPlugins:
    - volumeName: shared-home
      subDirectory: helm-plugins
      fileName: my-confluence-plugin.jar
```

### Когда это удобно
Если хотите:
- хранить версию плагина в Git/Helm;
- чтобы после пересоздания pod плагин оставался;
- не загружать руками через UI.

### Но
Для **OBR** этот вариант неудобен.  
Для OBR я бы рекомендовал **Upload app**.

---

# 8) Практическая схема для вас

С учётом того, что:
- приложения уже работают,
- контур изолированный,
- оркестрация через **Alauda**,
- хранилище уже на **PVC**,

я бы рекомендовал так:

## Jira
- добавить:
  - `-Dupm.pac.disable=true`
  - `-Dupm.plugin.upload.enabled=true`
- **JAR** → класть в:
  ```text
  /var/atlassian/application-data/shared-home/plugins/installed-plugins/
  ```
- **OBR** → ставить через **Upload app**

## Confluence
- добавить:
  - `-Dupm.pac.disable=true`
  - `-Dupm.plugin.upload.enabled=true`
  - `-Datlassian.confluence.plugin.scan.directory=/var/atlassian/application-data/shared-home/plugins/installed-plugins`
- **JAR** → класть в scan-directory
- **OBR** → ставить через **Upload app**

---

# 9) Команды для Kubernetes / Alauda

## Jira: создать каталог
```bash
kubectl -n <namespace> exec -it <jira-pod> -- sh -c \
'mkdir -p /var/atlassian/application-data/shared-home/plugins/installed-plugins'
```

## Jira: скопировать JAR
```bash
kubectl -n <namespace> cp ./my-plugin.jar \
<jira-pod>:/var/atlassian/application-data/shared-home/plugins/installed-plugins/my-plugin.jar
```

## Confluence: создать каталог
```bash
kubectl -n <namespace> exec -it <confluence-pod> -- sh -c \
'mkdir -p /var/atlassian/application-data/shared-home/plugins/installed-plugins'
```

## Confluence: скопировать JAR
```bash
kubectl -n <namespace> cp ./my-plugin.jar \
<confluence-pod>:/var/atlassian/application-data/shared-home/plugins/installed-plugins/my-plugin.jar
```

После копирования:
- для **Jira** — rolling restart / restart pod(ов);
- для **Confluence** — restart pod(ов), чтобы scan-directory подхватился.

---

# 10) Что я советую именно по формату файлов

## Если у вас **JAR**
Используйте файловую систему/PVC.

## Если у вас **OBR**
Используйте **Upload app**.

## Если очень нужно использовать OBR без UI
Тогда:
1. распаковать OBR,
2. посмотреть состав,
3. понять, какой JAR основной,
4. проверить документацию вендора.

Но это уже зависит от конкретного плагина. Некоторые OBR состоят из нескольких частей, и “просто вынуть один JAR” недостаточно. [7](https://developer.atlassian.com/server/framework/atlassian-sdk/bundling-extra-dependencies-in-an-obr/)

---

Если хотите, я могу следующим сообщением подготовить для вас **готовые фрагменты values.yaml на русском**:

1. **точный patch для Jira**  
2. **точный patch для Confluence**  
3. **вариант только для JAR**  
4. **вариант для смешанного набора JAR + OBR**  
5. **пошаговую инструкцию именно под Alauda**.

Понял. Ниже даю **рабочий lab/PoC-сценарий** для **изолированного контура**, **без Marketplace**, с учетом того, что **в UI сейчас нет Upload App**, и отдельно — **права/owner/mode** для PVC.

## Важное сразу
С **Atlassian Agent** и обходом лицензирования я не помогаю — даже для lab/PoC.  
Для PoC используйте:
- trial/license от вендора,
- evaluation key,
- либо бесплатные плагины.

---

# Почему у вас нет Upload App в UI

Это нормальное поведение на новых версиях:
- в **Jira Data Center** загрузка плагинов через UI по умолчанию отключена, включается через `-Dupm.plugin.upload.enabled=true`, и для Helm это добавляется в `additionalJvmArgs` [1](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)
- в **Confluence Data Center** Upload App тоже по умолчанию отключен на новых версиях и включается тем же флагом `-Dupm.plugin.upload.enabled=true` [4](https://support.atlassian.com/confluence/kb/after-upgrading-confluence-upload-app-under-manage-apps-isnt-visible/)

---

# Рекомендуемый lab/PoC-сценарий

## Что делать по факту
- **JAR**:
  - Jira: можно класть на PVC в `shared-home/plugins/installed-plugins` [1](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)
  - Confluence: можно грузить из filesystem scan directory
- **OBR**:
  - Jira: лучше ставить через **Upload App**; ручной filesystem-путь у Jira ориентирован на JAR, а OBR обычно ставят через UI/UPM [5](https://support.atlassian.com/jira/kb/understand-3rd-party-app-management-in-jira-server/)
  - Confluence: **не кладите raw `.obr`** в `plugin scan directory` — это известный кейс, Confluence ругается `not a valid plugin artifact` [2](https://jira.atlassian.com/browse/CONFSERVER-95831)

---

# Готовые значения для values.yaml

## Jira
Добавьте:

```yaml
jira:
  additionalJvmArgs:
    - "-Dupm.pac.disable=true"
    - "-Dupm.plugin.upload.enabled=true"
    # опционально, если плагин долго стартует:
    # - "-Datlassian.plugins.enable.wait=300"
    # опционально, если упретесь в app signing:
    # - "-Datlassian.upm.signature.check.disabled=true"
```

### Что это даст
- `-Dupm.pac.disable=true` — отключает доступ UPM к Atlassian Marketplace [6](https://confluence.atlassian.com/upm/problems-connecting-to-the-atlassian-marketplace-273875718.html)
- `-Dupm.plugin.upload.enabled=true` — вернет **Upload App** в Jira UI после рестарта всех нод [1](https://support.atlassian.com/jira/kb/how-to-re-enable-plugin-upload-in-jira-data-center/)

---

## Confluence
Добавьте:

```yaml
confluence:
  additionalJvmArgs:
    - "-Dupm.pac.disable=true"
    - "-Dupm.plugin.upload.enabled=true"
    - "-Datlassian.confluence.plugin.scan.directory=/var/atlassian/application-data/shared-home/plugins/installed-plugins"
    # опционально, если упретесь в app signing:
    # - "-Datlassian.upm.signature.check.disabled=true"
```

### Что это даст
- отключит Marketplace [6](https://confluence.atlassian.com/upm/problems-connecting-to-the-atlassian-marketplace-273875718.html)
- вернет **Upload App** в Confluence [4](https://support.atlassian.com/confluence/kb/after-upgrading-confluence-upload-app-under-manage-apps-isnt-visible/)
- включит чтение JAR из scan directory
- но **OBR в этот каталог класть нельзя**, это ломается [2](https://jira.atlassian.com/browse/CONFSERVER-95831)

---

# Права, owner, group, mode

У вас в values уже есть:

- **Jira**: `fsGroup: 2001`
- **Confluence**: `fsGroup: 2002`

Это ключевой момент.

## Практическое правило
Если `runAsUser` явно не задан, то **не делайте ставку на точный owner UID**, а делайте ставку на:
- правильную **group ownership**
- и правильные **mode**

### Рекомендуемые права
Для каталога с плагинами:
- **директории**: `2775`
- **файлы**: `0664`

Почему:
- `2` в `2775` = **setgid**, новые файлы наследуют группу каталога
- группе приложения будет удобно писать/читать
- это хорошо сочетается с `fsGroup`

---

## Jira: права
Каталог:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins
```

Рекомендуем:
- group: `2001`
- dirs: `2775`
- files: `0664`

Если вы **точно знаете**, что процесс Jira идет от UID `2001`, и storage позволяет `chown`, можно:
- owner: `2001`
- group: `2001`

Но это **не обязательно**. Достаточно корректной группы `2001`.

### Команды
```bash
JIRA_PLUGIN_DIR=/var/atlassian/application-data/shared-home/plugins/installed-plugins

mkdir -p "${JIRA_PLUGIN_DIR}"

chgrp -R 2001 "${JIRA_PLUGIN_DIR}" || true
find "${JIRA_PLUGIN_DIR}" -type d -exec chmod 2775 {} \;
find "${JIRA_PLUGIN_DIR}" -type f -exec chmod 0664 {} \;

# только если storage позволяет и вы уверены в UID:
# chown -R 2001:2001 "${JIRA_PLUGIN_DIR}"
```

---

## Confluence: права
Каталог:

```text
/var/atlassian/application-data/shared-home/plugins/installed-plugins
```

Рекомендуем:
- group: `2002`
- dirs: `2775`
- files: `0664`

Если точно знаете UID процесса Confluence и storage разрешает:
- owner: `2002`
- group: `2002`

### Команды
```bash
CONF_PLUGIN_DIR=/var/atlassian/application-data/shared-home/plugins/installed-plugins

mkdir -p "${CONF_PLUGIN_DIR}"

chgrp -R 2002 "${CONF_PLUGIN_DIR}" || true
find "${CONF_PLUGIN_DIR}" -type d -exec chmod 2775 {} \;
find "${CONF_PLUGIN_DIR}" -type f -exec chmod 0664 {} \;

# только если storage позволяет и вы уверены в UID:
# chown -R 2002:2002 "${CONF_PLUGIN_DIR}"
```

---

# Если `chown` не работает

Это очень частый случай на:
- NFS
- CephFS
- storage с `root_squash`

Тогда:
- не пытайтесь насильно делать `chown`
- достаточно:
  - `mkdir -p`
  - `chgrp`
  - `chmod 2775/0664`

То есть **group-based access через fsGroup** — ваш основной рабочий механизм.

---

# Команды через kubectl / Alauda

## Jira — создать каталог и выставить права
```bash
kubectl -n <namespace> exec -it <jira-pod> -- sh -c '
JIRA_PLUGIN_DIR=/var/atlassian/application-data/shared-home/plugins/installed-plugins
mkdir -p "${JIRA_PLUGIN_DIR}"
chgrp -R 2001 "${JIRA_PLUGIN_DIR}" || true
find "${JIRA_PLUGIN_DIR}" -type d -exec chmod 2775 {} \;
find "${JIRA_PLUGIN_DIR}" -type f -exec chmod 0664 {} \;
'
```

## Jira — скопировать JAR
```bash
kubectl -n <namespace> cp ./my-plugin.jar \
<jira-pod>:/var/atlassian/application-data/shared-home/plugins/installed-plugins/my-plugin.jar
```

---

## Confluence — создать каталог и выставить права
```bash
kubectl -n <namespace> exec -it <confluence-pod> -- sh -c '
CONF_PLUGIN_DIR=/var/atlassian/application-data/shared-home/plugins/installed-plugins
mkdir -p "${CONF_PLUGIN_DIR}"
chgrp -R 2002 "${CONF_PLUGIN_DIR}" || true
find "${CONF_PLUGIN_DIR}" -type d -exec chmod 2775 {} \;
find "${CONF_PLUGIN_DIR}" -type f -exec chmod 0664 {} \;
'
```

## Confluence — скопировать JAR
```bash
kubectl -n <namespace> cp ./my-plugin.jar \
<confluence-pod>:/var/atlassian/application-data/shared-home/plugins/installed-plugins/my-plugin.jar
```

---

# Что делать с OBR в вашем PoC

## Вариант 1 — правильный
1. включить `Upload App`
2. перезапустить все ноды
3. загрузить **OBR** через UPM

Это рекомендованный путь для OBR:
- Jira — OBR обычно ставят через UI/UPM [5](https://support.atlassian.com/jira/kb/understand-3rd-party-app-management-in-jira-server/)
- Confluence — для OBR через filesystem scan есть известная проблема [2](https://jira.atlassian.com/browse/CONFSERVER-95831)

---

## Вариант 2 — только для lab/PoC, если хотите попробовать “конвертацию”
Можно **распаковать OBR** и посмотреть, какие JAR внутри:

```bash
mkdir obr_unpack
unzip my-plugin.obr -d obr_unpack
find obr_unpack -name "*.jar"
```

### Что дальше
- если внутри есть **основной plugin JAR**
- и, возможно, еще **дополнительные plugin bundle JAR**

тогда для **лабораторного теста** можно попробовать работать **с извлеченными JAR**, а не с самим OBR.

### Но важно
- **Confluence**: в scan directory кладите **только JAR**, не raw `.obr` [2](https://jira.atlassian.com/browse/CONFSERVER-95831)
- **Jira**: filesystem-режим безопаснее для JAR; OBR лучше грузить через UI [5](https://support.atlassian.com/jira/kb/understand-3rd-party-app-management-in-jira-server/)
- если OBR содержит несколько связанных bundle'ов, может понадобиться **не один JAR, а несколько**
- если vendor ожидает именно установку через UPM, “превращение OBR в JAR” может не дать корректный результат

---

# Оптимальная схема именно для вас

## Jira
- включить:
  - `-Dupm.pac.disable=true`
  - `-Dupm.plugin.upload.enabled=true`
- **JAR**:
  - класть в shared-home `plugins/installed-plugins`
- **OBR**:
  - ставить через Upload App
- права:
  - group `2001`
  - dirs `2775`
  - files `0664`

## Confluence
- включить:
  - `-Dupm.pac.disable=true`
  - `-Dupm.plugin.upload.enabled=true`
  - `-Datlassian.confluence.plugin.scan.directory=.../plugins/installed-plugins`
- **JAR**:
  - класть в scan directory
- **OBR**:
  - либо Upload App,
  - либо распаковать и пробовать JAR-only сценарий
- права:
  - group `2002`
  - dirs `2775`
  - files `0664`

---

# Практическая заметка
После добавления JVM-флагов:
1. сделайте redeploy/upgrade Helm release в Alauda
2. дождитесь перезапуска **всех** нод
3. потом проверяйте Manage Apps снова

Для Confluence Atlassian отдельно рекомендует держать `upm.plugin.upload.enabled=true` только на время загрузки и потом выключать обратно [4](https://support.atlassian.com/confluence/kb/after-upgrading-confluence-upload-app-under-manage-apps-isnt-visible/)

---

Если хотите, следующим сообщением я могу подготовить вам **прямо готовый patch** в формате:

- **Jira values fragment**
- **Confluence values fragment**
- **команды для Alauda/kubectl**
- **вариант “JAR-only”**
- **вариант “JAR + OBR mixed”** 

— уже в виде блока, который можно почти сразу вставить в ваш release.

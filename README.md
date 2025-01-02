# Cluster of Our Own
## Содержание
1. [Вводные](#вводные)
2. [Настройка окружения](#настройка-окружения)
3. [Настройка кластера Kubernetes с rke](#настройка-кластера-kubernetes-с-rke)
4. [Установка Gateway API (Envoy Gateway)](#установка-gateway-api-envoy-gateway)
5. [Установка Kubernetes Dashboard](#установка-kubernetes-dashboard)
6. [Установка Openproject](#установка-openproject)
7. [Установка kube-prometheus-stack](#установка-kube-prometheus-stack)
8. [Полезные команды](#полезные-команды)
9. [Полезные ссылки](#полезные-ссылки)

### Вводные
Сервер на Ubuntu server 24.04.1 с внешним IP адресом.

### Настройка окружения
Выполнить обновление apt репозиториев и пакетов:
```bash
apt-get update
apt-get upgrade -y
apt-get dist-upgrade -y
```
Установить net-tools:
```bash
apt-get install -y net-tools
```
Установить актуальную временную зону:
```bash
timedatectl set-timezone Europe/Moscow
```
Установить пакеты, необходимые для выкачки пакетов из репозиториев по https:
```bash
apt-get install curl apt-transport-https ca-certificates software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce -y
#Есть вероятность поставить версию докера, которая будет не совместима с текущей версией Kubernetes, поэтому есть смысл указывать версию явно:
#apt-get install docker-ce=5:27.2.1-1~ubuntu.24.04~noble
```
Добавить пользователя docker в группу docker:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
Установить helm:
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```
Установить kubectl:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
```
**TODO: Описать увеличение таймаута для сессии ssh:**
https://www.simplified.guide/ssh/disable-timeout

### Настройка кластера Kubernetes с rke
Скачать исполняемый файл rke:
```bash
wget https://github.com/rancher/rke/releases/download/v1.6.3/rke_linux-amd64
```
Переименовать полученный исполняемый файл, переместить его в директорию $PATH, установить права на исполнение:
```bash
mv ./rke_linux-amd64 /usr/local/bin/rke
chmod +x /usr/local/bin/rke
```
Сгенерировать самоподписной ssl сертификат для кластера:
```bash
openssl req -newkey rsa:4096 -x509 -sha256 -days 3650 -nodes -out cluster.crt -keyout cluster.key
```
**TODO: написать вариант команды генерации сертификата, в которой сразу будут прописаны все параметры сертификата**<br>
Сгенерировать ssh ключ для доступа к ноде:
```bash
ssh-keygen -t ed25519 -f /home/$USER/.ssh/id_ed25519 -P ""
```
Добавить ключ в файл ~/.ssh/authorized_keys в домашний каталог пользователя, под которым будет происходить подключение утилиты rke для развертывания Kubernetes:
```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```
**TODO: Дописать про права на директорию ~/.ssh и на authorized_keys: https://superuser.com/questions/215504/permissions-on-private-key-in-ssh-folder** <br>
Запустить утилиту rke и в интерактивном режиме заполнить конфигурацию кластера:
```bash
rke config
```
**TODO: Добавить пример готового конфига кластера и расписать, что вводить при работе с командой rke config** <br>
Запустить Kubernetes с помощью rke:
```bash
rke up --config ./cluster.yaml
```
Утилита rke создает конфигурационный файл, который необходим для работы с Kubernetes с помощью утилиты kubectl. Чтобы данный файл использовался kubectl, необходимо прописать в переменную окружения KUBECONFIG путь к данному файлу:
```bash
#Команда выполняется в директории, где расположен файл kube_config_cluster.yml
export KUBECONFIG=$(pwd)/kube_config_cluster.yml
```
Чтобы прописывание переменной окружения выполнялось при каждом подключении к серверу, указанную выше команду можно добавить в файл ~/.bashrc.

### Установка Gateway API (Envoy Gateway)
Установить Envoy Gateway:
```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -n envoy-gateway --create-namespace
#Команда для установки определенной версии:
#helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.2.1 -n envoy-gateway --create-namespace
#Команда для ожидания развертывания пода evnoy-gateway:
kubectl wait --timeout=5m -n envoy-gateway deployment/envoy-gateway --for=condition=Available
```
**TODO указать, какой командой создаются объекты с помощью kubectl** <br>
Создать объект gateway class в неймспейсе envoy-gateway:
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```
Создать секрет для gateway из сертификатов, созданных при настройке кластера:
```bash
kubectl create secret tls openproject-cert --key=cluster.key --cert=cluster.crt -n envoy-gateway
```
Создать объект gateway в неймспейсе envoy-gateway:
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-gateway
  namespace: envoy-gateway
spec:
  addresses:
  - type: IPAddress
    value: <external ip>
  gatewayClassName: envoy-gateway-class
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: custom-proxy-config
  listeners:
  - allowedRoutes:
      namespaces:
        from: All
    name: tls
    port: 443
    protocol: TLS
    tls:
      mode: Passthrough
  - allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
            - key: kubernetes.io/metadata.name
              operator: In
              values: [openproject]		
    name: https-openproject
    port: 8443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: openproject-cert
      mode: Terminate
  - allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
            - key: kubernetes.io/metadata.name
              operator: In
              values: [kube-prometheus-stack]		
    name: https-kube-prometheus-stack-prometheus
    port: 9090
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: openproject-cert
      mode: Terminate	  
  - allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
            - key: kubernetes.io/metadata.name
              operator: In
              values: [kube-prometheus-stack]		
    name: https-kube-prometheus-stack-grafana
    port: 3000
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: openproject-cert
      mode: Terminate
```
Создать объект envoy proxy в неймспейсе envoy-gateway (возможно, не потребуется после фикса [#4744](https://github.com/envoyproxy/gateway/issues/4744)):
```yaml
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name:     
spec:
  bootstrap:
    type: JSONPatch
    jsonPatches:
    - { "op": "add", "path": "/static_resources/clusters/1/dns_lookup_family", "value": "V4_ONLY" }
```
**TODO: проверить фикс** <br>

### Установка Kubernetes Dashboard
Добавить в helm репозиторий Kubernetes Dashboard:
```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```
Установить Kubernetes Dashboard:
```bash
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```
Создать service account, role binding и токен для доступа к Kubernetes Dashboard:
```bash
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
kubectl -n kubernetes-dashboard create token dashboard-admin --duration=8760h
```
Создать объект tlsroute:
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: kubernetes-dashboard-route
spec:
  parentRefs:
  - name: envoy-gateway	
    namespace: envoy-gateway
  rules:
  - backendRefs:
    - kind: Service
      name: kubernetes-dashboard-kong-proxy
      port: 443
```
**TODO: добавить шаг удаления ingress, развернутого вместе с дэшбордом. Поисследовать, есть ли опция, которая отключает этот компонент при раскатке.**

### Установка Openproject
Создать storage class для Openproject, БД Postgresql и бэкапов БД Postgresql:
```yaml
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: postgresql
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: openproject
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: postgresql-backups
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
Создать persistent volume для Openproject, БД Postgresql и бэкапов БД Postgresql:<br>
**TODO: persistent volume для бэкапов должен ссылаться на директорию на другом хосте. Нужно выделить такой хост, примонтировать его к файловой системе стенда** 
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-persistent-volume
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  capacity:
    storage: 512Gi
  hostPath:
    path: /opt/kubernetes_volumes/postgresql
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <external ip>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: postgresql
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: openproject-persistent-volume
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  capacity:
    storage: 512Gi
  hostPath:
    path: /opt/kubernetes_volumes/openproject
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - <external ip>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: openproject
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-backups-persistent-volume
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  capacity:
    storage: 512Gi
  hostPath:
    path: /opt/kubernetes_volumes/postgresql_backups
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - <external ip>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: postgresql-backups
  volumeMode: Filesystem
```
Добавить в helm репозиторий Openproject:
```bash
helm repo add openproject https://charts.openproject.org
helm repo update
```
Завести файл openproject-values.yaml со значениями для helm chart Openproject:
```yaml
postgresql:
  primary:
    pgHbaConfiguration: |-
      # TYPE  DATABASE        USER             ADDRESS                METHOD
      # "local" is for Unix domain socket connections only
      local   all             all                                     trust
      # IPv4 local connections:
      #host    all             all              127.0.0.1/32           trust
      # IPv6 local connections:
      #host    all             all              ::1/128                trust
      # Allow replication connections from localhost, by a user with the
      # replication privilege.
      local   replication     all                                     trust
      host    replication     all              127.0.0.1/32           trust
      host    replication     all              ::1/128                trust
      host    replication     postgres         all                    trust	  
      host    openproject            read_only_user ::0/0                  trust
      host    openproject            read_only_user 0.0.0.0/0              trust
      host    openproject            openproject        ::0/0                  scram-sha-256
      host    openproject            openproject       0.0.0.0/0              scram-sha-256
      host    all             all              0.0.0.0/0              reject
      host    all             all              ::0/0                  reject
      # IPv4 local connections:
      # IPv6 local connections:
      # Unix socket connections:
      local   all             postgres                                trust
      # Enable streaming replication with wal2json:
      host    replication     all             127.0.0.1/32            trust
  backup:
    cronjob:
      command:
      - /bin/sh
      - -c
      - "pg_basebackup --no-password -D ${PGDUMP_DIR}/$(date '+%Y-%m-%d-%H-%M') -Ft -z -Xs -P'"
```
Установить Openproject: <br>
**TODO: вынести все переменные в команде ниже из --set в файл openproject-values.yaml**
```bash
helm upgrade --create-namespace --namespace openproject --install openproject openproject/openproject \
--set openproject.https=true \
--set ingress.enabled=false \
--set persistence.enabled=true \
--set persistence.storageClassName=openproject \
--set openproject.useTmpVolumes=false \
--set postgresql.primary.persistence.storageClass=postgresql \
--set postgresql.global.defaultStorageClass=postgresql \
--set postgresql.backup.enabled=true \
--set postgresql.backup.cronjob.storage.enabled=true \
--set postgresql.backup.cronjob.storage.storageClass=postgresql-backups \
--set postgresql.backup.cronjob.storage.size=512Gi \
--set postgresql.backup.cronjob.storage.mountPath=/opt/kubernetes_volumes/postgresql_backups \
--set postgresql.backup.cronjob.timezone="Europe/Moscow" \
--set postgresql.auth.username=openproject \
--set postgresql.auth.password=<password> \
--set postgresql.auth.database=openproject \
--set postgresql.auth.postgresPassword=<password> \
--set postgresql.global.auth.username=openproject \
--set postgresql.global.auth.password=<password> \
--set postgresql.global.auth.database=openproject \
--set postgresql.global.auth.replicationPassword=<password> \
--set postgresql.global.auth.postgresPassword=<password> \
--set postgresql.image.tag=17.1.0-debian-12-r0 \
--set containerSecurityContext.readOnlyRootFilesystem=false \
--set containerSecurityContext.readOnlyRootFilesystem=false \
--set openproject.host=<external ip>:8443 \
--set openproject.admin_user.name=admin \
--set openproject.admin_user.password=<password> \
--set openproject.admin_user.password_reset=false \
-f openproject-values.yaml
```
В секрете openproject-core в переменной DATABASE_HOST убрать .svc.cluster.local (с этим суффиксом почему-то БД недоступна). **TODO: Разобраться, почему, и можно ли раскатывать сразу с работающим именем.**<br>
Также в ряде случаев были замечены следующие проблемы при редеплое (**TODO: исследовать проблемы**):
- если не удалить перед редеплоем volumes, то они не биндятся;
- перед редеплоем приходилось удалять persistent volume claim сервиса Postgresql и директорию data. <br>

Создать объект httproute для Openproject:
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openproject-route
spec:
  parentRefs:
  - name: envoy-gateway	
    namespace: envoy-gateway
  rules:
    - backendRefs:
        - kind: Service
          name: openproject
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /
```
Вместе с Openproject будет создана cron job, выполняющая бэкапы БД Postgresql раз в сутки. Архив с бэкапом будет размещен в директории /opt/kubernetes_volumes/postgresql_backups/<дата выполнения бэкапа>. Для восстановления БД из бэкапа нужно разархивировать файлы БД и wal-журналы в директорию data:
```bash
sudo tar xvf /opt/kubernetes_volumes/postgresql_backups/<дата выполнения бэкапа>/base.tar.gz -C  /opt/kubernetes_volumes/postgresql/data
sudo tar xvf /opt/kubernetes_volumes/postgresql_backups/<дата выполнения бэкапа>/pg_wal.tar.gz -C /opt/kubernetes_volumes/postgresql/data/pg_wal
```

### Установка kube-prometheus-stack
**TODO: добавить шаг заведения бота в telegram и получения токена и chat id** <br>
Добавить в helm репозиторий kube-prometheus-stack:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Подготовить файл kube-prometheus-stack.yaml с values для helm chart kube-prometheus-stack:
```yaml
alertmanager:
  alertmanagerSpec:
    logLevel: debug 
  config:
    global:
      http_config:
        tls_config:
          insecure_skip_verify: true
      resolve_timeout: 5m
    route:
      group_by: ["namespace"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: telegram
      routes:
        - receiver: telegram
          matchers:
            - severity = "critical"
    receivers:
      - name: telegram
        telegram_configs:
          - chat_id: "***"
            bot_token: "***"
            api_url: "https://api.telegram.org"
            send_resolved: true
            parse_mode: HTML
            message: '{{ template "telegram.simple" . }}'
            tls_config:
              insecure_skip_verify: true
    templates:
      - "/etc/alertmanager/config/*.tmpl"
  templateFiles:
    template_telegram_simple.tmpl: |-
      {{ define "telegram.simple" }}
      <b>Openproject pod is not running</b>
      <b>Pod:</b> {{ .Labels.pod }}
      <b>Instance:</b>: {{ .Labels.instance }}
      <b>Namespace</b>: {{ .Labels.namespace }}
      <b>Severity</b>: {{ .Labels.severity }}
      {{ end }}
additionalPrometheusRulesMap:
  rule-name:
    groups:
      - name: openproject
        rules:
          - alert: openproject pod not running
            expr: kube_pod_status_ready{condition="true",pod=~"openproject.*",namespace="openproject"}==0
            for:
            labels:
              severity: critical
            annotations:
              summary: some pod is not running
              description: "{{ $labels.instance }} is not running"
grafana:
  grafana.ini:
    analytics:
      check_for_updates: false
```
Установить kube-prometheus-stack:
```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --create-namespace --namespace kube-prometheus-stack \
-n kube-prometheus-stack \
-f kube-prometheus-stack.yaml
```
Создать объекты httproute для доступа к UI Grafana и Prometheus:
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kube-prometheus-stack-route-prometheus
  namespace: kube-prometheus-stack
spec:
  parentRefs:
    - name: envoy-gateway
      namespace: envoy-gateway
      sectionName: https-kube-prometheus-stack-prometheus
  rules:
    - backendRefs:
        - kind: Service
          name: kube-prometheus-stack-prometheus
          port: 9090
      matches:
        - path:
            type: PathPrefix
            value: /
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kube-prometheus-stack-route-grafana
  namespace: kube-prometheus-stack
spec:
  parentRefs:
    - name: envoy-gateway
      namespace: envoy-gateway
      sectionName: https-kube-prometheus-stack-grafana
  rules:
    - backendRefs:
        - kind: Service
          name: kube-prometheus-stack-grafana
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /
```

### Полезные команды
```bash
#Просмотр компонентов кластера Kubernetes и их статуса
kubectl get componentstatuses
kubectl cluster-info
kubectl config get-clusters
```
```bash
#Получение и просмотр секрета для Kubernetes Dashboard
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin | awk '{print $1}')
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```
**TODO: не уверена, что команда выше актуальна, вроде токен больше не пишется в секрет?** <br>
```bash
#Проверка доступности сервиса по внутреннему DNS имени с помощью вспомогательного контейнера
kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 -n openproject -- nc -zv openproject-postgresql.openproject.svc.cluster.local 5432
```
```bash
#Получение конфига Alertmanager из секрета
kubectl get secret alertmanager-kube-prometheus-stack-alertmanager -n kube-prometheus-stack -o jsonpath={".data.alertmanager\.yaml"} | base64 -d
```
```bash
# Команды для amtool - утилиты для тестирования Alertmanager. Выполняются в поде Alertmanager
# Проверка рендеринга шаблона 
amtool --alertmanager.url=http://localhost:9093 template render --template.glob='/etc/alertmanager/config/*.tmpl' --template.text='{{ template "telegram.simple" . }}'
# Просмотр списка активных алертов
amtool --alertmanager.url=http://localhost:9093 alert  -o extended
# Ручная генерация алерта
amtool --alertmanager.url=http://localhost:9093/ alert add alertname="test123" severity="critical" job="test-alert" instance="localhost" exporter="none" cluster="test"
amtool --alertmanager.url=http://localhost:9093/ alert add alertname="test" severity="critical" job="kube-prometheus-stack-alertmanagert" instance="localhost"
```
```bash
#Отправка сообщения в telegram через бота
curl -X POST \
     -H 'Content-Type: application/json' \
     -d '{"chat_id": "<chat_id>", "text": "This is a test from curl", "disable_notification": true}' \
     https://api.telegram.org/<bot_id>:<bot_token>/sendMessage
```

### Полезные ссылки
Про docker:  https://www.cherryservers.com/blog/install-docker-ubuntu <br>
Про helm: https://helm.sh/docs/intro/install/ <br>
Про kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management <br>
Про Gateway API: https://gateway-api.sigs.k8s.io/concepts/api-overview/ <br>
Про Envoy Gateway: <br>
https://gateway.envoyproxy.io/latest/install/install-helm/ <br>
https://gateway.envoyproxy.io/docs/install/gateway-helm-api/ <br>
Про настройку persistent volume на локальном диске: <br>
https://lapee79.github.io/en/article/use-a-local-disk-by-local-volume-static-provisioner-in-kubernetes/ <br>
https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/ <br>
Пример быстрой настройки кластера Kubernetes с rke: https://gist.github.com/gopalsareen/1cd88a117f9d2311c7875bba7966ad27 <br>
Инструкция от вендора по настройке кластера Kubernetes с rke: https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke1-for-rancher <br>
Инструкция от вендора по обслуживанию кластера Kubernetes с rke: https://rke.docs.rancher.com/managing-clusters <br>
Описание параметров файла cluster.yaml для rke: https://rke.docs.rancher.com/example-yamls#minimal-clusteryml-example <br>
Про балансировщик нагрузки перед Kubernetes: https://www.ibm.com/docs/en/api-connect/10.0.8?topic=deployment-load-balancer-configuration <br>
Про балансировку нагрузки внутри Kubernetes: https://www.kubecost.com/kubernetes-best-practices/load-balancer-kubernetes/ <br>
Красивая схема с внешним балансировщиком и Envoy Gateway в Kubernetes: https://docs.tetrate.io/envoy-gateway/introduction/architecture <br>
Примеры настройки отправки алертов из Alertmanager в telegram: <br>
https://selectel.ru/blog/tutorials/monitoring-in-k8s-with-prometheus/ <br>
https://selectel.ru/blog/tutorials/how-to-set-up-dbaas-monitoring-and-alerting-in-prometheus/ <br>
Примеры алертов для Alertmanager: <br>
https://www.squadcast.com/blog/prometheus-sample-alert-rules#prometheus-sample-alert-rules <br>
https://samber.github.io/awesome-prometheus-alerts/rules.html#kubernetes <br>
Настройка Prometheus и Alertmanager: https://anaisurl.com/forwarding-security-metrics-to-slack-with-prometheus-and-alertmanager/ <br>
Примеры уведомлений для Alertmanager: https://prometheus.io/docs/alerting/latest/notification_examples/ <br>
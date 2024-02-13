# Подготовка к установке раннера

Для начала давайте создадим раннер внутри проекта и возьмем его токен


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/8d14421b-b90d-4edd-8143-94d107ab4cdf)

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/432c879c-97be-4dc2-baaa-976456fa0057)

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/5b8c15ab-963b-4768-8d5a-652d3e54f96c)

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/ac03be6f-5a0e-4e90-a09a-7e033c7bbd0f)

Вводим свой тэг, по которму планируем обращаться к раннеру

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/61b6d242-0845-49ba-b70f-7e3415b9452f)

Далее листаем вниз и жмем create runner

Сохраняем себе токен и жмем Go to runners page

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/9fd999e9-244d-4d16-815d-a4ec27ab7a88)

Пока у нас раннер не доступен, установим его с помощью helm  в кластер, для этого переходим на админскую машину

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/9a959a0d-ff6f-41cc-af83-268485a2b07b)

Вводим команды

```bash

helm repo add gitlab https://charts.gitlab.io

helm repo update

```

Предварительно надо взять с dev кластера файлы values.yaml и fix-rbac.yaml для внесения переменных раннеру и создания ролей внутри кластера (находятся по пути /home/KUBE_SERVICES/runner)

```bash

helm install --create-namespace --namespace gitlab-runners gitlab-runner -f values.yaml gitlab/gitlab-runner

kubectl apply -f fix-rbac.yaml

```

Проверяме запустился ли раннер


```bash

kubectl get pods -n gitlab-runners

NAME                             READY   STATUS    RESTARTS        AGE
gitlab-runner-55b69bbc5f-5hxq4   1/1     Running   2 (2d21h ago)   4d1h

```

Раннер запущен


Теперь допишем ci/cd

В файл .gitlab-ci.yaml добавляем строчки для деплоя( имя  деплоймента в команде выбираем соответственно названию сервиса внутри кластера)


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/43be264a-6adf-443d-a405-69db4e8753e9)

Готово

























Установка стека для мониторинга на локальный K8s кластер -  Minikube

Пререквизиты:

- Установка Helm

https://github.com/helm/helm/releases 

https://get.helm.sh/helm-v3.10.1-windows-amd64.zip

- Установка kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

- Установка Minikube

https://docs.gitlab.com/charts/development/minikube/


Запуск minikube с версией kubernetes 1.24.6

В командном окне  с административными  правами выполнить команду ( VPN не должен быть запущен)

minikube start --driver=hyperv --kubernetes-version=v1.24.6 --memory=6g --bootstrapper=kubeadm --extra-config=kubelet.authentication-token-webhook=true --extra-config=kubelet.authorization-mode=Webhook --extra-config=scheduler.bind-address=0.0.0.0 --extra-config=controller-manager.bind-address=0.0.0.0

Можно проверить версию установленного kubernetes кластера и kubectl

kubectl version

![image](https://user-images.githubusercontent.com/68746298/198032922-c30b1b4b-2f61-4f27-8d0c-9865842a6d67.png)


После запуска minikube надо включить дополнение Ingress (проксирование траффика):

minikube addons enable ingress

Проверить наличие дополнения можно командой

minikube addons list

Внешний IP адрес кластера можно узнать при помощи команды

minikube service ingress-nginx-controller -n ingress-nginx

или

minikube ip

Кластер подготовлен, можно перейти к установке стека для мониторинга и самой базы данных.

Для начала надо создать namespace, в котором будет установлен стек 

kubectl create namespace monitoring 

Перед установкой стека для мониторинга следует убедиться в том, что репозиторий где хранятся сценарии установки подключен

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

и описание загружено в локальный сервер.

helm repo update 

![image](https://user-images.githubusercontent.com/68746298/199480715-90ac7677-4ce4-4ad8-a702-218f4992bc8c.png)

Далее можно переходить к установке стека, предварительно следует скопировать файл prometheus_stack.yml, где хранится описание стека на локальный хост

helm install   --namespace monitoring   -f prometheus_stack.yml   prometheus prometheus-community/kube-prometheus-stack

Для проверки состояния установленных компонент можно посмотреть на содержимое пода и список сервисов (здесь выборка происходит по всем пространствам имен). 

kubectl get pods -A 
kubectl get services -A


![image](https://user-images.githubusercontent.com/68746298/199483832-29d2b6a2-ab73-43c7-bea9-dd76ca6361ca.png)

Теперь стек готов для того, чтобы принимать и обрабатывать метрики. Осталось организовать доставку данных. Будем использовать технологию экспортеров, в нашем случае это будет prostgress_db_exporter. Prometheus может сам обнаруживать и закачиать метрики, но лучше помочь ему в этом. При установке db_exporter надо указать метку, по которой Promеtheus обнаружит источник данных, для этого следует определить метку монитора сервиса Prometheus. Получим описание в виде xml

kubectl get prometheus -n monitoring -o yaml

![image](https://user-images.githubusercontent.com/68746298/199534271-ee3b65fc-0544-442e-bae9-67f4c467aabc.png)


Для установки postgress_db_exporter следует отредактировать файл values.yml. В этом файле нужно указать метку для ServiceMonitor, для того чтобы сконфигурировать процесс передачи метрик: 


![image](https://user-images.githubusercontent.com/68746298/199539191-fe2cd6fe-f466-451f-ac7c-e190620c1dde.png)

и описание базы данных PostgreSql:

![image](https://user-images.githubusercontent.com/68746298/199539808-b3550709-f82e-483b-b7bc-0f217a1a4ea3.png)

В этом же файле находится описание самих метрик, а также процесс их получения (SQL запрос):

![image](https://user-images.githubusercontent.com/68746298/199540580-7670ed42-c915-4c77-9867-75e92fff9e3b.png)

Команда install устанавливает экспортер, update обновляет установку, например, после изменения файла values.yml.

helm install my-prometheus-postgres-exporter --namespace monitoring -f values.yml  prometheus-community/prometheus-postgres-exporter --version 3.1.3

Остается только установить и стартовать базу данных, если нет уже готовой. Образ базы определенной  версии можно взять здесь :

https://hub.docker.com/_/postgres/tags

Для этого следует выполнить команду, в качестве tag указывается необходимая версия базы данных :

docker pull postgres:14.5

После этого можно стартовать контейенер с базой   

docker run --name postgresql -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 -v e:\postgresql\data:/var/lib/postgresql/data -d postgres:14.5

В консоли после старта базы можно увидеть:

![image](https://user-images.githubusercontent.com/68746298/199548324-eab11de9-f22e-467e-8ba9-5f0ea793255b.png)

Можно переходить к настройкам стека для мониторинга. В графическом интерфейсе grafana (IP адрес полученный командой "minikube IP") надо ввести имя пользователя/пароль (admin/prom-operator)

https://IP/grafana

![image](https://user-images.githubusercontent.com/68746298/199550497-34bfac51-b0e8-44de-843b-6a6c51cafe35.png)

Необходимо добавить источник данных Prometheus:

![image](https://user-images.githubusercontent.com/68746298/199551627-1695a3c8-3f4a-4e60-b7eb-f99f42eee358.png)

В строке HTTP URL надо ввести имя сервиса (хоста), где развернут Prometheus 

http://prometheus-kube-prometheus-prometheus:9090

![image](https://user-images.githubusercontent.com/68746298/199554416-a021c2dc-3bba-4da0-82aa-8bafb50ba709.png)


Проверить соединение и сохранить настройки. 

Имя сервиса можно получить командой  

kubectl get services --namespace monitoring 

![image](https://user-images.githubusercontent.com/68746298/199555177-6f99f005-83ea-4039-a1a5-c6339681468d.png)

Осталось только импортировать соответвующий dashboard для отображения метрик,которые можно найти -

https://grafana.com/grafana/dashboards/?search=postgres

Далее неоходимо импортировать понравившийся

![image](https://user-images.githubusercontent.com/68746298/199558647-ec302bfc-bc6b-4e1a-8713-33a7fe13eb46.png)

указав соответствующий ID

![image](https://user-images.githubusercontent.com/68746298/199559247-b80a6c72-167e-423c-86c0-314457044edc.png)


![image](https://user-images.githubusercontent.com/68746298/199579282-54e2f54c-24bd-4817-ada1-82eaff2cf718.png)
























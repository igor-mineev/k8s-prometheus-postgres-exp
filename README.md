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

Теперь стек готов для того, чтобы принимать и обрабатывать метрики. Осталось организовать доставку данных. Будем использовать технологию экспортеров, в нашем случае это будет prometheus_db_exporter. Prometheus может сам обнаруживать и закачиать метрики, но лучше помочь ему в этом. При установке db_exporter надо указать метку, по которой Promеtheus обнаружит источник данных, для этого следует определить метку монитора сервиса Prometheus. Получим опмсание в виде xml

kubectl get prometheus -n monitoring -o yaml

![image](https://user-images.githubusercontent.com/68746298/199534271-ee3b65fc-0544-442e-bae9-67f4c467aabc.png)










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






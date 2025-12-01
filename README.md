# Programowanie-full-stack-w-chmurze-obliczeniowej-zadanie-8
###############################################################
LABORATORIUM 8
Autor: Michał Wójtowicz
Grupa: 2.4
###############################################################

## Instrukcja uruchomienia

### 1. Uruchomienie klastra Minikube z 4 nodami
minikube start --profile=lab8 --driver=docker --nodes=4 --cni=calico    
kubectl get nodes -o wide

### 2. Nadanie etykiet węzłom roboczym A, B, C
kubectl label node lab8-m02 node=A
kubectl label node lab8-m03 node=B
kubectl label node lab8-m04 node=C
kubectl get nodes --show-labels

### 3. Wdrożenie zasobów – deploymentów i Poda my-sql
kubectl apply -f workloads.yaml
kubectl get pods -o wide

### 4. Wystawienie usług sieciowych
kubectl apply -f services.yaml
kubectl get svc

### 5. Wdrożenie polityki sieciowej NetworkPolicy
kubectl apply -f netpol-mysql.yaml
kubectl get networkpolicy
kubectl describe networkpolicy mysql-access



## Testy poprawności działania polityki sieciowej

### 1. Test z Poda frontend (powinien być ZABLOKOWANY dostęp do MySQL)
kubectl run frontend-tester --image=busybox:1.36 --restart=Never --labels=app=frontend -- sh -c "sleep 3600"
kubectl exec -it frontend-tester -- sh -c "nc -vz mysql 3306"

**wynik:** timeout / brak możliwości połączenia.

### 2. Test z Poda backend (powinien mieć dostęp do MySQL na porcie 3306)
kubectl run backend-tester --image=busybox:1.36 --restart=Never --labels=app=backend -- sh -c "sleep 3600"
kubectl exec -it backend-tester -- sh -c "nc -vz mysql 3306"

**wynik:** połączenie zostaje nawiązane.

### 3. Test na innym porcie z backendu (powinien być ZABLOKOWANY)
kubectl exec -it backend-tester -- sh -c "nc -vz mysql 80"

**wynik:** brak możliwości połączenia.

### 4. Test dostępu do frontend przez Minikube
minikube -p lab8 service frontend-nodeport --url
curl http://$(minikube ip):30080


## Opis rozwiązania

W klastrze Kubernetes wdrożono trzy kluczowe komponenty: Deployment frontend (3 repliki, Nginx) przypisany do węzła A, Deployment backend (1 replika, Nginx) przypisany do węzła B oraz Pod my-sql (MySQL, z ustawionym hasłem dostępowym), uruchomiony na węźle C.

Komunikację sieciową zapewniono poprzez usługę NodePort dla warstwy frontend oraz usługi ClusterIP dla backendu i bazy my-sql, umożliwiając tym samym dostęp odpowiednio zewnętrzny i wewnętrzny w obrębie klastra.

Zastosowana polityka mysql-access gwarantuje pełną blokadę połączeń z frontendu do bazy oraz kontrolowany dostęp z backendu do bazy wyłącznie przez protokół TCP na porcie 3306, co zostało potwierdzone w testach przy pomocy narzędzia netcat uruchamianego w Podach testowych BusyBox.

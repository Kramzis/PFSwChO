SPRAWOZDANIE Z LABORATORIUM PROGRAMOWANIE FULL-STACK W CHMURZE OBLICZENIOWEJ
Kramek Magdalena

OBOWIĄZKOWE

Najpierw należy plik namespace.yml należy uruchomić poleceniem:
kubectl apply - f namespace.yml

Zadanie 1
Plik quota.yml należy uruchomić poleceniem:
kubectl apply -f quota.yml

Zadanie 2
Plik worker.yml należy uruchomić poleceniem:
kubectl apply -f worker.yml

Wynik polecenia get pods:


Zadanie 3
Plik php-apache.yml po zmodyfikowaniu należy uruchomić poleceniem:
kubectl apply -f php-apache.yml

Zadanie 4
Plik php-apache-hpa.yml po zmodyfikowaniu należy uruchomić poleceniem:
kubectl apply -f php-apache-hpa.yml

Dostępne zasoby pozwalają na uruchomienie większej liczby replik, jednak ograniczeniem jest quota ustawiona na 10 podów. Ponieważ jeden z nich jest już zajęty przez pod z kontenerem nginx, maksymalna liczba replik, jaką można uruchomić, wynosi 9.

Zadanie 5 & Zadanie 6
Polecenia ze sprawdzaniem wyników uruchamianych wyżej poleceń znajdują się na zdjęciach dołączonych do katalogu. 

Polecenie generujące obciążenie:
kubectl run load-generator --rm -it --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache.zad1.svc.cluster.local; done"

NIEOBOWIĄZKOWE

Zadanie 1
Czy możliwe jest dokonanie aktualizacji aplikacji (np. wersji obrazu kontenera), gdy
aplikacja jest pod kontrolą autoskalera HPA?
Tak (link: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#autoscaling-during-rolling-update)

Aby zapewnić, że:
a) Podczas aktualizacji zawsze będą aktywne 2 pody realizujące działanie przykładowej aplikacji:

Należy ustawić parametry maxSurge i maxUnavailable w strategii rollingUpdate w sposób, który zapewni minimalną liczbę dostępnych podów:

strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2  # maksymalnie 2 dodatkowe pody w trakcie aktualizacji
      maxUnavailable: 1  # zawsze musi być co najmniej 3 pody dostępne w trakcie aktualizacji (4 - 1)
 

b) Nie zostaną przekroczone parametry wcześniej zdefiniowanej quoty dla
przestrzeni zad1.

Ponieważ mamy maxSurge: 2 i maxUnavailable: 1, w sumie w momencie aktualizacji nie będzie więcej niż 2 dodatkowe pody, a liczba usuwanych podów nie przekroczy jednej jednostki. Dzięki temu nie przekroczy się zdefiniowanej kwoty zasobów, ponieważ liczba podów nie zostanie drastycznie zwiększona.

c) Jeśli należy skorelować (zmieć) ustawienia autoskalera HPA z części
obowiązkowej w związku z zaplanowaną strategią aktualizacji to należy również
przedstawić te zmiany.

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: zad1
  minReplicas: 4
  maxReplicas: 9
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

Wyjaśnienie zmian:
minReplicas: 4: Podczas aktualizacji należy mieć co najmniej 4 aktywne pody. Dostosowanie minReplicas w HPA pomaga zapewnić, że pożądana liczba podów będzie dostępna, nawet gdy rollingUpdate uruchomi nowe instancje podów.

averageUtilization: 70: Zmniejszenie progu CPU może być przydatne, ponieważ w trakcie aktualizacji aplikacja może wymagać większych zasobów. Zmniejszenie progu wykorzystania CPU do 70% pozwoli HPA lepiej reagować na zwiększone zapotrzebowanie na zasoby, jeśli pody będą bardziej obciążone.

maxReplicas: 9: Upewnienie się, że liczba podów nie przekroczy maksymalnej liczby replik w HPA. Ta wartość jest zgodna z wcześniej zdefiniowanym maxReplicas.

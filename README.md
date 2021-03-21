# Домашняя работа

Ответ на домашнее задание находится в [Kubernetes home-work2.docx](Kubernetes home-work2.docx)
Ответ на домашнюю работу:
```
Реализовать Canary развертывание приложения через Ingress. 
Трафик на canary deployment должен перенаправляться если в заголовке добавить "canary:always" в ином случае он должен идти на обычный deployment. 
Опционально можете настроить перенаправлять какой-то процент трафика на canary deployment.
```

Создаем deployment, configmap и service для двух типов приложений:
```
kubectl apply -f web-canary-v1.yaml
kubectl apply -f web-canary-v2.yaml
```

Создаем Ingress для двух типов приложений:
```
kubectl apply -f ingress-nginx-v1.yaml
kubectl apply -f ingress-nginx-v2.yaml
```
Проверяем:  
(поскольку я все запускал на моем самонастроеном кластере, на основе руководства https://github.com/mmumshad/kubernetes-the-hard-way у меня отсутствует LoadBalancer, соответсвтенно на Ingress Controller я хожу по NodePort, замените worker-2:30037 на айпи вашего LoadBalancer-а.)

```
curl -H "Host: canary.example.com"  http://worker-2:30037/
```
Output будет таким:
```
#Output:
web-canary-v1-65f77cdc96-tl2qw
 normal application
```

```
curl -H "Host: canary.example.com" -H "canary:always"  http://worker-2:30037/
```
Output будет таким:
```
#Output:
web-canary-v2-574cf96f8d-2mbdb
 CANARY application
```
Таким образом, используя в заголовке запроса "canary:always" весь трафик бует перенаправляться на второе приложение.  
Добавим перенаправление по весуб в нашем случае это будет 10%:

```
kubectl apply -f ingress-nginx-v2-weight.yaml
```

Проверяем: 
```
for i in {1..100}; do curl -H "Host: canary.example.com" http://worker-1:30037/; done | grep application | sort | uniq -c
```
На выходе получаем:
```
      8  CANARY application
     92  normal application
```
Получается, что приложение CANARY вызывалось примерно в 10% случаев

Собственно, за это отвечают следующие поля в настройках Ingress:

За принудительное перенаправление трафика если в заголовке указано "canary:always"

```
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "always"
```
За принудительное перенаправление трафика по весу, в процентном содержании:

```
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

Спасибо, за внимание!
# Домашняя работа

Ответ на домашнее задание находится в [Kubernetes-home-work2.docx](Kubernetes-home-work2.docx)   
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
(поскольку я все запускал на моем самонастроеном кластере, на основе руководства https://github.com/mmumshad/kubernetes-the-hard-way у меня отсутствует LoadBalancer, соответсвтенно на Ingress Controller я хожу по NodePort, замените `worker-1:30037` на айпи вашего LoadBalancer-а.)

```
export lb_ip=worker-1:30037
```

```
curl -H "Host: canary.example.com"  http://$lb_ip/
```
Output будет таким:
```
web-canary-v1-65f77cdc96-tl2qw
 normal application
```
Добавляем `canary:always` в заголовок запроса:
```
curl -H "Host: canary.example.com" -H "canary:always"  http://$lb_ip/
```
Output будет таким:
```
web-canary-v2-574cf96f8d-2mbdb
 CANARY application
```
Таким образом, используя в заголовке запроса `canary:always` весь трафик будет перенаправляться на второе приложение.  
Добавим перенаправление по весу, в нашем случае это будет 10%:

```
kubectl apply -f ingress-nginx-v2-weight.yaml
```

Проверяем: 
```
for i in {1..100}; do curl -H "Host: canary.example.com" http://$lb_ip/; done | grep application | sort | uniq -c
```
На выходе получаем:
```
      8  CANARY application
     92  normal application
```
Получается, что приложение CANARY вызывалось примерно в 10% случаев.   
Собственно, за это отвечают следующие поля в настройках Ingress:  

1. За принудительное перенаправление трафика если в заголовке указано "canary:always"

```
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "always"
```
2. За принудительное перенаправление трафика по весу, в процентном содержании:

```
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

Спасибо, за внимание!
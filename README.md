# «Обновление приложений» - Абрамов Сергей

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

### Решение

Для описанных условий я рекомендую использовать стратегию Rolling Update. Эта стратегия обновления не требует значительных ресурсов и является стандартной. Поскольку ресурсы ограничены, важно использовать следующие параметры:

maxSurge: 20% - Этот параметр позволяет запустить до 20% новых реплик приложения параллельно с текущими. Например, если у вас есть 5 реплик, то с этим параметром можно запустить еще одну реплику с новой версией приложения.
maxUnavailable: 20% - Данный параметр позволяет временно выключить до 20% старых реплик приложения во время обновления. Например, если у вас есть 5 реплик, то во время обновления будет выключена одна реплика, чтобы дать ресурсы новой реплике.

Однако, если обновление является мажорным и новые версии приложения не совместимы со старыми, можно установить параметр maxUnavailable: 100%. Это позволит не удалять старые реплики до тех пор, пока новые не будут проверены. Старые реплики будут использовать ресурсы, но если к ним не будет обращений, это не приведет к значительному потреблению ресурсов. После проверки новых реплик старые можно будет удалить.

Также стоит учитывать, что обновления лучше проводить в периоды, когда кластер Kubernetes наименее загружен.

Если ресурсов совсем нет, можно использовать стратегию обновления Recreate. Это приведет к остановке текущих подов и прекращению всех запросов к ним. После остановки старых подов будут созданы новые поды, а старые будут полностью удалены.

### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.
3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.
4. Откатиться после неудачного обновления.

### Решение

Создаем namespace:

```
serg@k8snode:~/git/K8s-14$ kubectl create namespace update
namespace/update created
serg@k8snode:~/git/K8s-14$ kubectl get namespace
NAME                     STATUS   AGE
default                  Active   40d
ingress                  Active   32d
kube-node-lease          Active   40d
kube-public              Active   40d
kube-system              Active   40d
nfs-server-provisioner   Active   25d
update                   Active   3m40s
```
Подготовил сразу три deployment с разными версиями:

[deployment1.19.yaml](https://github.com/smabramov/K8s-14/blob/b3867811fd4ea2681efac9d0271d7e78be00ed3c/code/deployment1.19.yaml)

[deployment1.20.yaml](https://github.com/smabramov/K8s-14/blob/b3867811fd4ea2681efac9d0271d7e78be00ed3c/code/deployment1.20.yaml)

[deployment1.28.yaml](https://github.com/smabramov/K8s-14/blob/b3867811fd4ea2681efac9d0271d7e78be00ed3c/code/deployment1.28.yaml)

Запускаем 1.19 и проверяем версию:

```
serg@k8snode:~/git/K8s-14/code$ kubectl apply -f deployment1.19.yaml 
deployment.apps/nginx-multitool created
serg@k8snode:~/git/K8s-14/code$ kubectl -n update get po
NAME                               READY   STATUS    RESTARTS   AGE
nginx-multitool-764dc864cc-bq6n8   2/2     Running   0          59s
nginx-multitool-764dc864cc-k8vh2   2/2     Running   0          59s
nginx-multitool-764dc864cc-kx8gg   2/2     Running   0          59s
nginx-multitool-764dc864cc-ldhdz   2/2     Running   0          59s
nginx-multitool-764dc864cc-mcghh   2/2     Running   0          59s
serg@k8snode:~/git/K8s-14/code$ kubectl -n update describe deployment.apps nginx-multitool | grep Image 
    Image:        nginx:1.19
    Image:      wbitt/network-multitool
```
Стратегию обновления выбираю RollingUpdate, при этом устанавливаю значение 2 для параметра maxSurge и значение 3 для параметра maxUnavailable. Это позволит сохранить работоспособность двух реплик приложения со старой версией даже в случае возникновения проблем с обновлением, обеспечивая тем самым доступность приложения.
После запуска трёх реплик с новой версией nginx, две реплики со старой версией будут отключены, и вместо них будут созданы три реплики с новой версией nginx. После запуска трёх реплик с новой версией приложения, две реплики со старой версией будут уничтожены, и вместо них будут запущены две реплики с новой версией приложения. 

```
serg@k8snode:~/git/K8s-14/code$ kubectl apply -f deployment1.20.yaml 
deployment.apps/nginx-multitool configured
serg@k8snode:~/git/K8s-14/code$ kubectl -n update get po -w
NAME                               READY   STATUS    RESTARTS   AGE
nginx-multitool-5fcb95b745-nvjnw   2/2     Running   0          2m8s
nginx-multitool-5fcb95b745-rqkzt   2/2     Running   0          2m8s
nginx-multitool-5fcb95b745-vg82r   2/2     Running   0          2m8s
nginx-multitool-5fcb95b745-xm58v   2/2     Running   0          2m8s
nginx-multitool-5fcb95b745-z88vc   2/2     Running   0          2m8s
serg@k8snode:~/git/K8s-14/code$ kubectl -n update describe deployment.apps nginx-multitool | grep Image 
    Image:        nginx:1.20
    Image:      wbitt/network-multitool
```
Посмотрим историю:

```
serg@k8snode:~/git/K8s-14/code$ kubectl -n update rollout history deployment
deployment.apps/nginx-multitool 
REVISION  CHANGE-CAUSE
1         nginx 1.19
2         update to nginx 1.20
```
Запускаем 1.28 и видим ошибки:

```
serg@k8snode:~/git/K8s-14/code$ kubectl -n update get po -w
NAME                               READY   STATUS    RESTARTS   AGE
nginx-multitool-5fcb95b745-nvjnw   2/2     Running   0          8m32s
nginx-multitool-5fcb95b745-rqkzt   2/2     Running   0          8m32s
nginx-multitool-5fcb95b745-vg82r   2/2     Running   0          8m32s
nginx-multitool-5fcb95b745-xm58v   2/2     Running   0          8m32s
nginx-multitool-5fcb95b745-z88vc   2/2     Running   0          8m32s
nginx-multitool-94fbb8d49-j6p44    0/2     Pending   0          0s
nginx-multitool-94fbb8d49-2xrdx    0/2     Pending   0          0s
nginx-multitool-94fbb8d49-j6p44    0/2     Pending   0          0s
nginx-multitool-94fbb8d49-2xrdx    0/2     Pending   0          0s
nginx-multitool-5fcb95b745-nvjnw   2/2     Terminating   0          9m37s
nginx-multitool-5fcb95b745-xm58v   2/2     Terminating   0          9m37s
nginx-multitool-94fbb8d49-2xrdx    0/2     ContainerCreating   0          0s
nginx-multitool-5fcb95b745-rqkzt   2/2     Terminating         0          9m37s
nginx-multitool-94fbb8d49-j6p44    0/2     ContainerCreating   0          0s
nginx-multitool-94fbb8d49-7lvlq    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-w458q    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-krs7d    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-7lvlq    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-krs7d    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-w458q    0/2     Pending             0          0s
nginx-multitool-94fbb8d49-7lvlq    0/2     ContainerCreating   0          0s
nginx-multitool-94fbb8d49-w458q    0/2     ContainerCreating   0          0s
nginx-multitool-94fbb8d49-krs7d    0/2     ContainerCreating   0          0s
nginx-multitool-5fcb95b745-rqkzt   2/2     Terminating         0          9m38s
nginx-multitool-5fcb95b745-xm58v   2/2     Terminating         0          9m38s
nginx-multitool-5fcb95b745-nvjnw   2/2     Terminating         0          9m38s
nginx-multitool-5fcb95b745-rqkzt   0/2     Completed           0          9m38s
nginx-multitool-5fcb95b745-xm58v   0/2     Completed           0          9m38s
nginx-multitool-94fbb8d49-2xrdx    0/2     ContainerCreating   0          1s
nginx-multitool-5fcb95b745-rqkzt   0/2     Completed           0          9m38s
nginx-multitool-5fcb95b745-rqkzt   0/2     Completed           0          9m38s
nginx-multitool-94fbb8d49-j6p44    0/2     ContainerCreating   0          1s
nginx-multitool-5fcb95b745-nvjnw   0/2     Completed           0          9m38s
nginx-multitool-94fbb8d49-7lvlq    0/2     ContainerCreating   0          1s
nginx-multitool-94fbb8d49-krs7d    0/2     ContainerCreating   0          1s
nginx-multitool-94fbb8d49-w458q    0/2     ContainerCreating   0          1s
nginx-multitool-5fcb95b745-xm58v   0/2     Completed           0          9m38s
nginx-multitool-5fcb95b745-xm58v   0/2     Completed           0          9m38s
nginx-multitool-5fcb95b745-nvjnw   0/2     Completed           0          9m39s
nginx-multitool-5fcb95b745-nvjnw   0/2     Completed           0          9m39s
nginx-multitool-94fbb8d49-krs7d    1/2     ErrImagePull        0          4s
nginx-multitool-94fbb8d49-2xrdx    1/2     ErrImagePull        0          4s
nginx-multitool-94fbb8d49-j6p44    1/2     ErrImagePull        0          4s
nginx-multitool-94fbb8d49-w458q    1/2     ErrImagePull        0          4s
nginx-multitool-94fbb8d49-7lvlq    1/2     ErrImagePull        0          4s
nginx-multitool-94fbb8d49-7lvlq    1/2     ImagePullBackOff    0          5s
nginx-multitool-94fbb8d49-w458q    1/2     ImagePullBackOff    0          5s
nginx-multitool-94fbb8d49-j6p44    1/2     ImagePullBackOff    0          5s
nginx-multitool-94fbb8d49-2xrdx    1/2     ImagePullBackOff    0          5s
nginx-multitool-94fbb8d49-krs7d    1/2     ImagePullBackOff    0          5s
nginx-multitool-94fbb8d49-j6p44    1/2     ErrImagePull        0          28s
nginx-multitool-94fbb8d49-krs7d    1/2     ErrImagePull        0          29s
nginx-multitool-94fbb8d49-w458q    1/2     ErrImagePull        0          31s
nginx-multitool-94fbb8d49-2xrdx    1/2     ErrImagePull        0          33s
nginx-multitool-94fbb8d49-7lvlq    1/2     ErrImagePull        0          36s
nginx-multitool-94fbb8d49-j6p44    1/2     ImagePullBackOff    0          40s
nginx-multitool-94fbb8d49-krs7d    1/2     ImagePullBackOff    0          42s
nginx-multitool-94fbb8d49-w458q    1/2     ImagePullBackOff    0          42s
nginx-multitool-94fbb8d49-2xrdx    1/2     ImagePullBackOff    0          44s
nginx-multitool-94fbb8d49-7lvlq    1/2     ImagePullBackOff    0          49s
nginx-multitool-94fbb8d49-w458q    1/2     ErrImagePull        0          56s
nginx-multitool-94fbb8d49-j6p44    1/2     ErrImagePull        0          56s
nginx-multitool-94fbb8d49-2xrdx    1/2     ErrImagePull        0          57s
nginx-multitool-94fbb8d49-krs7d    1/2     ErrImagePull        0          58s
nginx-multitool-94fbb8d49-7lvlq    1/2     ErrImagePull        0          62s
nginx-multitool-94fbb8d49-j6p44    1/2     ImagePullBackOff    0          68s
nginx-multitool-94fbb8d49-krs7d    1/2     ImagePullBackOff    0          69s
nginx-multitool-94fbb8d49-2xrdx    1/2     ImagePullBackOff    0          69s
nginx-multitool-94fbb8d49-w458q    1/2     ImagePullBackOff    0          71s
nginx-multitool-94fbb8d49-7lvlq    1/2     ImagePullBackOff    0          75s
```
Два пода предыдущей версии запущенно,по остальным ошибки:

```
^Cserg@k8snode:~/git/K8s-14/code$ kubectl -n update get po -w
NAME                               READY   STATUS             RESTARTS   AGE
nginx-multitool-5fcb95b745-vg82r   2/2     Running            0          11m
nginx-multitool-5fcb95b745-z88vc   2/2     Running            0          11m
nginx-multitool-94fbb8d49-2xrdx    1/2     ImagePullBackOff   0          2m15s
nginx-multitool-94fbb8d49-7lvlq    1/2     ImagePullBackOff   0          2m15s
nginx-multitool-94fbb8d49-j6p44    1/2     ImagePullBackOff   0          2m15s
nginx-multitool-94fbb8d49-krs7d    1/2     ImagePullBackOff   0          2m15s
nginx-multitool-94fbb8d49-w458q    1/2     ImagePullBackOff   0          2m15s
```
Смотрим историю и откатываемся на предыдущую версию:

```
serg@k8snode:~/git/K8s-14/code$ kubectl -n update rollout undo deployment
deployment.apps/nginx-multitool rolled back
serg@k8snode:~/git/K8s-14/code$ kubectl -n update get po -w
NAME                               READY   STATUS    RESTARTS   AGE
nginx-multitool-5fcb95b745-mwp85   2/2     Running   0          2s
nginx-multitool-5fcb95b745-r9bjm   2/2     Running   0          2s
nginx-multitool-5fcb95b745-vg82r   2/2     Running   0          17m
nginx-multitool-5fcb95b745-z88vc   2/2     Running   0          17m
nginx-multitool-5fcb95b745-zdn4s   2/2     Running   0          2s
^Cserg@k8snode:~/git/K8s-14/code$ kubectl -n update rollout history deployment
deployment.apps/nginx-multitool 
REVISION  CHANGE-CAUSE
1         nginx 1.19
3         update to nginx 1.28
4         update to nginx 1.20
```
В истории видим, что откатились на предыдущую версию и появился 4 ход.

## Дополнительные задания — со звёздочкой*

Задания дополнительные, необязательные к выполнению, они не повлияют на получение зачёта по домашнему заданию. **Но мы настоятельно рекомендуем вам выполнять все задания со звёздочкой.** Это поможет лучше разобраться в материале.   

### Задание 3*. Создать Canary deployment

1. Создать два deployment'а приложения nginx.
2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

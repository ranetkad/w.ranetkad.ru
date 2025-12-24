# Kubernetes - kubectl

## Коротко

CLI для управления кластером. Все операции: просмотр, создание, удаление, отладка.

## Команды

Просмотр ресурсов:

```bash
kubectl get pods
```

Поды в текущем namespace.

```bash
kubectl get pods -A
```

Поды во всех namespace.

```bash
kubectl get pods -o wide
```

Расширенный вывод (IP, узел).

```bash
kubectl get deploy,svc,pods
```

Несколько типов ресурсов сразу.

Детали объекта:

```bash
kubectl describe pod my-pod
```

Подробная информация: события, состояние, volumes.

Логи:

```bash
kubectl logs my-pod
```

Логи пода.

```bash
kubectl logs my-pod -f
```

Следить за логами (tail -f).

```bash
kubectl logs my-pod -c my-container
```

Логи конкретного контейнера (если несколько).

```bash
kubectl logs my-pod --previous
```

Логи предыдущего контейнера (после рестарта).

Выполнение команд:

```bash
kubectl exec my-pod -- ls /app
```

Выполнить команду в поде.

```bash
kubectl exec -it my-pod -- /bin/sh
```

Интерактивный shell.

Применение манифестов:

```bash
kubectl apply -f manifest.yaml
```

Создать/обновить ресурсы из файла.

```bash
kubectl apply -f ./manifests/
```

Применить все файлы из директории.

Удаление:

```bash
kubectl delete pod my-pod
```

Удалить под.

```bash
kubectl delete -f manifest.yaml
```

Удалить ресурсы из файла.

Масштабирование:

```bash
kubectl scale deployment my-deploy --replicas=3
```

Изменить число реплик.

Rollout (деплой):

```bash
kubectl rollout status deployment/my-deploy
```

Статус выкатки.

```bash
kubectl rollout history deployment/my-deploy
```

История ревизий.

```bash
kubectl rollout undo deployment/my-deploy
```

Откат на предыдущую версию.

```bash
kubectl rollout restart deployment/my-deploy
```

Перезапуск подов деплоймента.

## Примеры

Найти поды по лейблу:

```bash
kubectl get pods -l app=nginx
```

Смотреть изменения в реальном времени:

```bash
kubectl get pods -w
```

Вывод в YAML:

```bash
kubectl get pod my-pod -o yaml
```

Создать deployment:

```bash
kubectl create deployment nginx --image=nginx
```

Проброс порта для отладки:

```bash
kubectl port-forward pod/my-pod 8080:80
```

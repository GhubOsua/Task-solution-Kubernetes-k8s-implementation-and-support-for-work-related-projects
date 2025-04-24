# Описание задачи из вакансии и мой ответ ниже.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  config.yaml: |
    setting1: value1
    setting2: value2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 4  # Поддерживаем 4 Pod для высокой доступности
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate  # Используем постепенное обновление
    rollingUpdate:
      maxSurge: 1         # Максимум 1 дополнительный Pod во время обновления
      maxUnavailable: 1   # Максимум 1 Pod может быть недоступен
  template:
    metadata:
      labels:
        app: my-app
    spec:
      affinity:
        podAntiAffinity: # Не размещать два пода одного приложения на одной ноде.
          requiredDuringSchedulingIgnoredDuringExecution: # Жёсткое требование. Под не будет запущен, если правило нарушается
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - my-app
            topologyKey: "kubernetes.io/hostname" # Это стандартная метка каждой ноды, содержащая её имя (hostname)
      topologySpreadConstraints: # Равномерное распределение подов по зонам
      - maxSkew: 1 # Максимальная разница в количестве подов между зонами должна быть не больше 1
        topologyKey: "topology.kubernetes.io/zone" # Определяем зону размещения по этой метке
        whenUnsatisfiable: DoNotSchedule # Если нельзя равномерно распределить, лучше вообще не запускать новый под
        labelSelector:
          matchLabels:
            app: my-app
      containers:
      - name: my-app-container
        image: my-app:1.0.0
        ports:
        - containerPort: 8080  # Приложение слушает порт 8080
        readinessProbe:  # Проверка готовности для маршрутизации трафика
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10 # Это задержка перед началом проверки
          periodSeconds: 5 # Проверяем каждые n секунд
          timeoutSeconds: 2 # Таймаут ответа 2 секунд. Максимальное время ожидания ответа от проверки
          failureThreshold: 3 # Проверяем до 3 раз перед ошибкой Pod
        livenessProbe:  # Проверка "живости" для перезапуска зависших Pod
          httpGet:
            path: /livez
            port: 8080
          initialDelaySeconds: 10 # Это задержка перед началом проверки
          periodSeconds: 5 # Проверяем каждые n секунд
          timeoutSeconds: 2 # Таймаут ответа 2 секунд. Максимальное время ожидания ответа от проверки
          failureThreshold: 3 # Проверяем до 3 раз перед перезапуском Pod
        resources:  # Ограничения ресурсов для контейнера
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "130Mi"
            cpu: "0.5"
        env:  # Переменные окружения для контейнера
        - name: APP_ENV
          value: production
        - name: LOG_LEVEL
          value: debug
        volumeMounts:  # Пример подключения volume
        - name: config-volume
          mountPath: /app/config
      volumes:  # Пример volume из ConfigMap
      - name: config-volume
        configMap:
          name: my-app-config
      restartPolicy: Always
      terminationGracePeriodSeconds: 30  # Время для завершения работы Pod. Время, которое даётся Pod для завершения работы перед удалением.
      imagePullPolicy: Always  # Убедиться, что новый образ тянется всегда. Указывает Kubernetes всегда проверять наличие новой версии образа.
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef: # Указываем, к какому Deployment привязываем автоскейлинг (my-app)
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2 # Минимальное количество подов (ночью или при низкой нагрузке)
  maxReplicas: 6 # Максимальное количество подов (в пиковую нагрузку) 
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50 # HPA при превышении порога видит, что 58% > 50% (целевой уровень). Нужно увеличить количество подов, чтобы разгрузить нагрузку и вернуться к целевым 50% . 4 пода в кластере. У каждого requests.cpu = 300m. Суммарный requests для всех подов = 4 × 300m = 1200m. При реальной нагрузке: Если сейчас суммарное реальное потребление CPU = 700m, Тогда средняя утилизация по всем подам: utilization = 700m / 1200m , значит utilization ≈ 58%.

# Лабораторная работа 4 - Observability

В этой лабораторной работе мы научимся базовой работе с системами получения, хранения и отображения метрик,
настроим дашборды и алерты для нашего приложения.

# Что нужно сделать к защите

В лабораторной работе необходимо поднять Prometheus + Grafana + Alertmanager для работы с метриками, дашбордами и
алертами.

К защите необходимо:

1. Иметь поднятые Prometheus + Grafana + Alertmanager, например через docker-compose;
2. Настроить Prometheus на сбор метрик для вашего приложения:
    1. Ваше приложение должно отдавать как минимум одну кастомную метрику типа Counter;
    2. Ваше приложение должно отдавать как минимум одну кастомную метрику любого другого типа;
    3. Как минимум для одной из метрик должны быть представлены лейблы, объявленные вами (значений не меньше двух).
3. В Grafana должно быть настроено не менее 3 дашбордов.
4. Должен быть сделан сбор метрик с хотя бы одной внешней системы, к которой подключается ваше приложение (Postgres,
   Redis, RabbitMQ, любое другое).
5. Должно быть настроено как минимум три различных алерта.
6. Должен быть подключен Alertmanager и настроен на раздачу алертов (через SMTP/Telegram/что угодно еще).
> Сохраните в виде скринов список сервисов, которые опрашивает прометей, дашборды графаны, а также настроенные алерты.
> Зафиксируйте конфигурацию алертменеджера. Запомните, а лучше тоже зафиксируйте, какие именно кастомные метрики вы отдаете.

Лабораторная работа может быть выполнена на любом ЯП.

# Вопросы к защите

Я буду задавать вопросы (1-2) из списка, однако могу дополнительно спросить (исключительно) по содержанию практики.

Если ответить не получилось - в рамках того же занятия сможете освежить свои знания и подойти чуть позже.

1. Что такое Observability и зачем оно нужно? Какие есть компоненты в Observability?
2. Что такое SLA/SLO/SLI? Как метрики помогают определить SLI? Как трассировки и логирование помогает достичь SLO?
3. Что такое Prometheus? Зачем он нужен и как работает?
4. Что такое Grafana? Как она работает и зачем нужна?
5. Что такое Alertmanager? Как подключить его к Prometheus и зачем он нужен?
6. Какие модели сбора метрик поддерживает Prometheus?
7. Какие типы данных есть в Prometheus? Что такое PromQL?
8. Что такое Trace ID и Span ID? Как они помогают трассировать запросы?

# Ход работы

Работа будет выполняться из ветки, в которой была закончена лабораторная работа 3. Идеально было бы создать отдельную ветку,
назвать ее `lab4`.

Для отдачи метрик мы будем использовать pull модель, прометей будет забирать наши метрики с определенного эндпоинта.

Для включения эндпоинта будем использовать Micrometer - удобный инструмент для работы с Observability в джава приложениях.

Подключим зависимости в `shared-components/pom.xml` (секция `<dependencies></dependencies>`:

```xml
<!-- Core Actuator for metrics endpoints -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Prometheus Registry -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Теперь настроим наше приложение, добавим в `user-service/src/main/resources/application.properties` следующие параметры:

```properties
management.endpoints.web.exposure.include=health,info,prometheus
management.metrics.tags.application=${spring.application.name}
```

Теперь наше приложение будет отдавать Prometheus метрики по эндпоинту `/actuator/prometheus`.

Spring Boot (фреймворк используемый нашим приложением) по умолчанию отдает достаточно много метрик, связанных 
с нагрузкой на CPU системы, на память, а также с длительностью запросов и количеством ошибок. 

Давайте добавим в наше приложение кастомные метрики: одну на количество созданных пользователей с момента развертывания приложения,
а вторую - на длительность выполнения POST (создание) и GET (получение) запросов для пользователей.

Для этого создадим класс `shared-contract/src/main/java/com/misis/archapp/contract/metrics/Metrics.java` в котором будем
хранить параметры (имена и теги) наших метрик:

```java
package com.misis.archapp.contract.metrics;

public abstract class Metrics {

    // Counter метрика - сколько всего было создано пользователей
    public static final String USERS_CREATED_TOTAL = "users.new";

    // Summary метрика - сколько времени обрабатывался запрос на создание пользователя
    public static final String API_USER_REQ_DURATION = "api.user.request.duration";
    public static final String METHOD_TAG = "method";
    public static final String POST_TAG_VAL = "post";
    public static final String GET_TAG_VAL = "get";

}
```

Далее напишем логику учета метрик, модифицируем контроллер для пользователей (класс `user-service/src/main/java/com/misis/archapp/user/controller/UserRestApiController.java`):

```java
package com.misis.archapp.user.controller;

import com.misis.archapp.contract.metrics.Metrics;
import com.misis.archapp.user.dto.UserCreateDTO;
import com.misis.archapp.user.dto.UserDTO;
import com.misis.archapp.user.dto.UserUpdateDTO;
import com.misis.archapp.user.service.UserService;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import jakarta.validation.Valid;
import java.util.List;
import java.util.UUID;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PatchMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("users")
public class UserRestApiController {

   private final UserService userService;
   private final MeterRegistry meterRegistry;

   @Autowired
   public UserRestApiController(
           UserService userService,
           MeterRegistry meterRegistry
   ) {
      this.userService = userService;
      this.meterRegistry = meterRegistry;
   }

   @GetMapping
   public List<UserDTO> getAllUsers() {
      return userService.getAllUsers();
   }

   @GetMapping("{id}")
   public UserDTO getUserById(@PathVariable("id") UUID id) {
      // метрика времени выполнения запроса с тегом GET
      Timer timer = meterRegistry.timer(Metrics.API_USER_REQ_DURATION, Metrics.METHOD_TAG, Metrics.GET_TAG_VAL);
      return timer.record(() -> userService.getUserById(id));
   }

   @PostMapping
   public UserDTO createUser(@RequestBody @Valid UserCreateDTO userCreateDTO) {
      // метрика времени выполнения запроса с тегом POST
      Timer timer = meterRegistry.timer(Metrics.API_USER_REQ_DURATION, Metrics.METHOD_TAG, Metrics.POST_TAG_VAL);
      return timer.record(() -> userService.createUser(userCreateDTO));
   }

   @PatchMapping("{id}")
   public UserDTO updateUser(@PathVariable("id") UUID id, @RequestBody @Valid UserUpdateDTO userUpdateDTO) {
      return userService.updateUser(id, userUpdateDTO);
   }

   @DeleteMapping("{id}")
   @ResponseStatus(HttpStatus.NO_CONTENT)
   public void deleteUser(@PathVariable("id") UUID id) {
      userService.deleteUser(id);
   }

}
```

Теперь при выполнении POST и GET запросов будет собираться метрика с соответствующими лейблами. 

Micrometer автоматически создаст три метрики, преобразовав разделители . в _:

![img.png](images/img.png)

Заметьте, что к названиям нашей метркии были добавлены суффиксы. Для нашей кастомной метркии не будут генерироваться
персентили, чтобы не нагружать Prometheus. Однако это можно сделать, указав дополнительные аргументы при создании `Timer`.

Дальше давайте объявим Counter метрику на суммарное кол-во созданий пользователей с момента поднятия сервиса. Для этого модифицируем
класс `user-service/src/main/java/com/misis/archapp/user/service/UserService.java`:

```java
package com.misis.archapp.user.service;

import com.misis.archapp.contract.dto.UserCreatedEvent;
import com.misis.archapp.contract.metrics.Metrics;
import com.misis.archapp.user.db.User;
import com.misis.archapp.user.db.UserRepository;
import com.misis.archapp.user.dto.UserCreateDTO;
import com.misis.archapp.user.dto.UserDTO;
import com.misis.archapp.user.dto.UserUpdateDTO;
import com.misis.archapp.user.dto.mapper.UserMapper;
import com.misis.archapp.user.service.cache.UserCacheService;
import com.misis.archapp.user.service.publisher.UserEventPublisher;
import io.micrometer.core.instrument.MeterRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Service
public class UserService {

    private static final Logger LOGGER = LoggerFactory.getLogger(UserService.class);

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final UserCacheService userCacheService;
    private final UserEventPublisher userEventPublisher;
    private final MeterRegistry meterRegistry;

    @Autowired
    public UserService(
        UserRepository userRepository,
        UserMapper userMapper,
        UserCacheService userCacheService,
        UserEventPublisher userEventPublisher,
        MeterRegistry meterRegistry
    ) {
        this.userRepository = userRepository;
        this.userMapper = userMapper;
        this.userCacheService = userCacheService;
        this.userEventPublisher = userEventPublisher;
        this.meterRegistry = meterRegistry;
    }

    public List<UserDTO> getAllUsers() {
        return userMapper.toDTOList(userRepository.findAll());
    }

    public UserDTO getUserById(UUID id) {
        Optional<UserDTO> cachedUser = userCacheService.getFromCache(id);

        // cache hit - нашел пользователя в кэше
        //noinspection OptionalIsPresent
        if (cachedUser.isPresent()) {
            LOGGER.info("User cache hit");
            return cachedUser.get();
        }

        LOGGER.info("User cache miss");
        // cache miss - пользователя в кэше не оказалось
        UserDTO userFromDB = userRepository.findById(id).map(userMapper::toDTO)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        // актуализирует кэш значением из БД
        userCacheService.saveToCache(userFromDB);
        return userFromDB;
    }

    public UserDTO createUser(UserCreateDTO userCreateDTO) {
        User user = userMapper.toEntity(userCreateDTO);
        User savedUser = userRepository.save(user);

        // отправляет ивент с данными о пользователе
        UserCreatedEvent userCreatedEvent = new UserCreatedEvent(user.getId(), user.getEmail(), user.getName());
        userEventPublisher.publishUserEvent(userCreatedEvent);

        // после того как пользователь был создан и ивент отправлен - увеличивает значение метрики
        meterRegistry.counter(Metrics.USERS_CREATED_TOTAL).increment();
        return userMapper.toDTO(savedUser);
    }

    public UserDTO updateUser(UUID id, UserUpdateDTO userUpdateDTO) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        if (userUpdateDTO.name().isPresent()) {
            user.setName(userUpdateDTO.name().get());
        }

        if (userUpdateDTO.email().isPresent()) {
            user.setEmail(userUpdateDTO.email().get());
        }

        User savedUser = userRepository.save(user);

        // после обновления - очищает данные из кэша
        LOGGER.info("User cache evict on update");
        userCacheService.removeFromCache(user.getId());

        return userMapper.toDTO(savedUser);
    }

    public void deleteUser(UUID id) {
        userRepository.deleteById(id);
        // после удаления - очищает данные из кэша
        LOGGER.info("User cache evict on delete");
        userCacheService.removeFromCache(id);
    }

}

```

Теперь после выполнения операции по созданию пользователя каунтер метрики будет увеличен. Обратите внимание,
что Prometheus автоматчиески добавляет суффикс _total для нашей метрики:

![img_1.png](images/img_1.png)

Отлично, метрики заданы и теперь они будут собираться приложением в дополнении к дефолтным и отдаваться по настроенному
эндпоинту. Давайте подключать Prometheus!

Для этого модифицируем наш `dockercompose/docker-compose.yaml`:

```yaml
services:
   # Сервис для вашего приложения
   user-service:
      build:
         context: ../user-service  # Указывает на корень проекта, где находится директория user-service
         dockerfile: Dockerfile  # Путь к Dockerfile в директории user-service
      container_name: user-service
      environment:
         SPRING_DATA_REDIS_HOST: keydb
         SPRING_RABBITMQ_HOST: rabbitmq
         SPRING_DATASOURCE_URL: 'jdbc:postgresql://postgres:5432/apidemo-db'
      ports:
         - "8080:8080"  # Пробрасывание порта для доступа к вашему приложению

   # Сервис для вашего приложения
   notification-service:
      build:
         context: ../notification-service  # Указывает на корень проекта, где находится директория user-service
         dockerfile: Dockerfile  # Путь к Dockerfile в директории user-service
      container_name: notification-service
      environment:
         SPRING_RABBITMQ_HOST: rabbitmq

   postgres:
      image: postgres:13.3
      container_name: postgres-lab3
      environment:
         POSTGRES_DB: "apidemo-db"
         POSTGRES_USER: "postgres"
         POSTGRES_PASSWORD: "postgres"
         PGDATA: "/var/lib/postgresql/data/pgdata"
      ports:
         - "5432:5432"
      volumes:
         - .:/var/lib/postgresql/data
      restart: always

   keydb:
      image: "eqalpha/keydb:x86_64_v5.3.3"
      container_name: keydb-lab3
      command: "keydb-server /etc/keydb/redis.conf --server-threads 2"
      volumes:
         #      - "./redis.conf:/etc/keydb/redis.conf"
         - "data:/data"
      ports:
         - "6379:6379"
      restart: unless-stopped

   rabbitmq:
      image: "rabbitmq:3-management"
      container_name: rabbitmq-archapp-lab4
      ports:
         - "5672:5672"
         - "15672:15672"
         - "15692:15692"  # По этому порту Prometheus может получить метрики
      environment:
         RABBITMQ_DEFAULT_USER: guest
         RABBITMQ_DEFAULT_PASS: guest
         RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS: "-rabbitmq_prometheus true" # Включает плагин для отдачи метрик
      volumes:
         - ./rabbitmq-data:/var/lib/rabbitmq

   # Сервис Prometheus
   prometheus:
      image: prom/prometheus:v2.44.0
      container_name: prometheus-archapp-lab4
      ports:
         - "9090:9090"   # Prometheus web UI
      volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml  # Конфигурационный файл
         - ./alert_rules.yml:/etc/prometheus/alert_rules.yml  # Файл с алертами

   # Сервис Grafana
   grafana:
      image: grafana/grafana
      container_name: grafana-archapp-lab4
      ports:
         - "3000:3000"   # Grafana web UI
      environment:
         - GF_SECURITY_ADMIN_PASSWORD=admin  # Пароль администратора
      volumes:
         - grafana-data:/var/lib/grafana
      depends_on:
         - prometheus  # Дожидается прометея

   # Сервис Alertmanager
   alertmanager:
      image: prom/alertmanager:v0.24.0
      container_name: alertmanager-archapp-lab4
      ports:
         - "9093:9093"   # Alertmanager web UI
      volumes:
         - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml  # Конфигурационный файл

   # Тестовый SMTP сервер
   mailpit:
      image: axllent/mailpit
      container_name: mailpit-lab4
      ports:
         - "1025:1025"  # SMTP
         - "8025:8025"  # Web UI
      restart: unless-stopped

volumes:
   data:
      driver: local
   grafana-data:
```

Добавилось супер много разных сервисов, а также появились настройки сборки наших приложений в докере.

Это нужно чтобы до них мог достучаться Prometheus, также запущенный в контейнере! Дальше разберемся, как именно
теперь производить запуск приложений, пока давайте настроим Prometheus.

Для этого создадим конфигурационные файлы в папке `dockercompose` (рядом с ямликом докер компоуза):

1. Для конфигурации прометея используем `prometheus.yml`:
   
   ```yaml
   global:
     # интервал забора метрик
     scrape_interval: 15s
     # интервал вычисления выражений для алертов
     evaluation_interval: 15s
   
   scrape_configs:
     - job_name: 'user-service'
       metrics_path: "/actuator/prometheus"
       static_configs:
         - targets: ['user-service:8080']
   
     - job_name: 'rabbitmq'
       static_configs:
         - targets: ['rabbitmq:15692']
   
   alerting:
     alertmanagers:
       - static_configs:
           - targets:
               - "alertmanager:9093"
   
   rule_files:
     - "alert_rules.yml"  # путь до файла с алертами
   ```
   
2. Создадим пустой файл `alert_rules.yml`, в нем будут храниться алерты (это важно сделать сейчас, заполним мы его чуть позже);
3. Создадим пустой файл `alertmanager.yml`, в нем будет храниться конфигурация Alertmanager (это важно сделать сейчас, заполним мы его чуть позже).

Отлично! Теперь прометей настроен на сбор метрик с нашего приложения. Также обратите внимание, что в компоузе используется плагин для 
рэббита, который отдает метрики нашего брокера сообщений по порту 15692, по которому их и читает прометей.

Давайте создадим алерты! Алерты создаются с помощью yaml конфигурации, где мы указываем такие параметры как PromQL выражение, вызвающее срабатывание алерта,
сколько данное выражние должно действовать чтобы вызвать алерт, а также уровень алерта и его описание.

Алерты пропишем в добавленном нами файле `dockercompose/alert_rules.yml`:

```yaml
groups:
  - name: cpu-alerts
    rules:
      - alert: HighSystemCPUUsage
        expr: system_cpu_usage > 0.90
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High system CPU usage detected"
          description: "Overall CPU usage on the system is over 90% for the last 2 minutes."

  - name: user-creation-alerts
    rules:
      - alert: HighUserCreationRate
        # Указанное значение специально низкое, чтобы можно было проверить алерт
        expr: increase(users_new_total[5m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High user creation increase detected"
          description: "The total of user creation has exceeded 10 users in the last 5 minutes."

  - name: api-user-request-alerts
    rules:
      - alert: HighGETRequestDuration
        expr: rate(api_user_request_duration_seconds_sum{method="get"}[5m]) / rate(api_user_request_duration_seconds_count{method="get"}[5m]) > 2
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High duration for GET request on API"
          description: "The 95th percentile duration for GET requests exceeds 1.5 seconds for the last 1 minute."

```

Здесь описано три алерта, каждый из которых находится в своей группе.

Первый алерт сигнализирует о том, что нагрузка на ЦПУ в системе превышала допустимую в течение 2 минут.

Второй алерт сигнализирует о том, что за последние пять минут было создано более 10 пользователей. Значение специально низкое, чтобы можно было
данный алерт протестировать.

Третий алерт имеет самый сложный PromQL запрос, в нем высчитывается средняя длительность запроса получения пользователя за последние пять минут. И если 
данное значение превышает 2 секунды - алерт выстреливает.

Алерты пишутся с использованием PromQL, очень богатого и мощного языка запросов. Подробнее узнать про него можно [в документации 
Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/).

Отлично, алерты написаны, однако они не будут уходить нам, пока мы не настроим Alertmanager - компонент системы который отвечает за отправку алертов. 

В файле `dockercompose/alertmanager.yml` укажем:

```yaml
global:
  smtp_smarthost: 'mailpit:1025'
  smtp_from: 'alertmanager@example.com'
  smtp_require_tls: false

receivers:
  - name: 'email-receiver'
    email_configs:
      - to: 'test@local.dev'
        send_resolved: true

route:
  receiver: 'email-receiver'
```

Теперь алерты будут уходить на тестовый почтовый сервер, просмотреть мы их можем по урлу `localhost:8025`.

Супер, давайте поднимать нашу систему, смотреть на метрики, которые нам приходят, а также настраивать их визуальное отображение в дашбордах!

Для поднятия кода нужно сделать следующее:

1. **Соберите исходники сервисов**
   ```bash
   mvn clean install
   ```

   Или используейте вкладку maven в IntellijIDEA (кнопки clean и install во вкладке arch-app):

   ![img_2.png](images/img_2.png)

2. **Соберите контейнеры сервисов**
   ```bash
   cd dockercompose
   docker-compose build
   ```

3. **Запустите контейнеры**
   ```bash
   cd dockercompose
   docker-compose up -d
   ```

**После запуска user-service API доступно по:** `http://localhost:8080`  
**Swagger UI:** `http://localhost:8080/swagger-ui.html`
**Prometheus:** `http://localhost:9090`
**Grafana:** `http://localhost:3000`
**Тестовый SMTP сервер:** `http://localhost:8025`

**После внесения изменеий в код выполните команду `docker compose down` и повторите шаги 1-3**

Обратите внимание, что наши сервисы теперь тоже запускаются в контейнерах (сборка происходит через докер компоуз).

Это нужно для того, чтобы прометей мог до них достучаться, будучи развернутым в контейнере.

Запустим наш сетап, выполнив команды выше, и зайдем в прометея.

Увидим алерты:

![img_3.png](images/img_3.png)

Подключенные сервисы (вкладка status -> targets):

![img_4.png](images/img_4.png)

> Сохраните это!

Давайте попробуем выполнить пару запросов в консоль Prometheus и посмотрим, что они вернут:

![img_5.png](images/img_5.png)

![img_6.png](images/img_6.png)

![img_7.png](images/img_7.png)

> Сохраните это!

Обратите внимание, что добавленные нами кастомные метрики отобразятся только когда мы совершим определенные действия в сваггере.
До этого они не будут инициализированы. Также прометей забирает метрики не сразу, по нашим настройкам он делает это раз в 15 секунд. Поэтому
и значения обновятся не сразу!

Консоль прометея хороший инструмент для дебага и изучения PromQL, однако следить за метриками в ней не очень удобно.

Давайте используем Grafana чтобы создать удобные дашборды для наших метрик!

Перейдем по ссылке `localhost:3000` и залогинимся (логин admin, пароль admin):

![img_8.png](images/img_8.png)

Далее добавим датасурс во вкладке Connections (плагин Prometheus):

![img_9.png](images/img_9.png)

Нажмем на кнопку Add new datasource:

![img_10.png](images/img_10.png)

В качестве URL укажем `http://prometheus:9090`:

![img_11.png](images/img_11.png)

После чего внизу страницы нажмем на кнопку Save and Test. Наш датасурс успешно добавлен!

![img_12.png](images/img_12.png)

Отлично, теперь давайте настроим дашборды. Перейдем во вкладку Dashboards:

![img_13.png](images/img_13.png)

И нажмем на кнопку Create new dashboard:

![img_14.png](images/img_14.png)

Жмем Add visualization (выбираем в качестве сурса прометея) и попадаем в конструктор дашбордов:

![img_15.png](images/img_15.png)

Здесь в поле Metric мы задаем название метрики и фильтры по лейблам, а также операции, выполняемые над метриками.

Также можно задавать PromQL выражения, если переключить из Builder в Code.

Сохранить дашборд можно кнопкой Save dashboard.

Давайте создадим три дашборда:

1. График загрузки ЦПУ:

   ![img_16.png](images/img_16.png)

2. График по количеству регистраций за последние 5 минут:

   ![img_17.png](images/img_17.png)

3. График по количеству сообщений в реббите, ждущих консьюмера:

   ![img_18.png](images/img_18.png)

В итоге получим следующий общий дашборд:

![img_19.png](images/img_19.png)

> Сохраните это!

Отлично, теперь проверим последнее - алерты. Давайте быстро создадим 10 пользователей через Swagger:

![img_20.png](images/img_20.png)

Увидим, что наш алерт перешел в статус Pending:

![img_21.png](images/img_21.png)

И по прошествии минуты он переходит в статус Firing:

![img_22.png](images/img_22.png)

> Сохраните это!

После чего Alertmanager высылает письмо об алерте:

![img_23.png](images/img_23.png)

> Сохраните это!

Супер, мы научились базовой работе с метриками приложения, подключили Prometheus + Grafana + Alertmanager. 
Настроили кастомные метрики, дашборды и алерты.






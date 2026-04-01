# Лабораторная работа 3 – Event-driven архитектура (RabbitMQ)

В этой лабораторной работе мы научимся базовой работе с брокером сообщений RabbitMQ,
который является важной составляющей любого приложения, построенного по Event-driven архитектуре.

# Что нужно сделать к защите

В лабораторной работе необходимо поднять RabbitMQ, выполнить базовые операции с ним с помощью UI панели администрирования, а также
подключить RabbitMQ в ваше приложение для реализации event-driven взаимодействия между различными сервисами.

К защите необходимо:

1. Иметь поднятый RabbitMQ (например, через docker-compose);
2. С помощью UI панели администрирования создать:
    1. `Fanout exchange` с названием `{ваша_группа}.{номер_в_списке}.fanout`;
    2. `Direct exchange` с названием `{ваша_группа}.{номер_в_списке}.direct`;
    3. `Topic exchange` с названием `{ваша_группа}.{номер_в_списке}.topic`;
    4. `Headers exchange` с названием `{ваша_группа}.{номер_в_списке}.headers`;
    5. Очередь с названием `queue.{ваша_группа}.{номер_в_списке}`;
    6. `Binding` очереди с `fanout exchange` без `routing key`;
    7. `Binding` очереди с `direct exchange` c `routing key` = `{ваша_группа}.{номер_в_списке}.routing.key`;
    8. `Binding` очереди с `topic exchange` с `routing key` = `{ваша_группа}.*.routing.key`;
    9. `Binding` очереди с `headers exchange` без `routing key` с заголовками `group` = `{ваша_группа}`, `number` = `{номер_в_списке}`;
    10. Продемонстрировать логику роутинга сообщений в очередь для каждого биндинга;
> Сохраните в виде скринов список обменников, очередей, а также биндинги между ними.
3. Понимать и иметь возможность продемонстрировать логику перехода сообщения с определенными `routing key`/заголовками из exchange в
   очередь (логика роутинга сообщения);
4. Добавить в ваше приложение (или в то, которое указано в `Ход работы`) асинхронную коммуникацию между двумя сервисами посредством
   сообщений (event-driven communication).
5. Итог работы должен быть опубликован в GitHub/GitLab/GitFlic.

Лабораторная работа может быть выполнена на любом ЯП.

# Вопросы к защите

Я буду задавать вопросы (1-2) из списка, однако могу дополнительно спросить (исключительно) по содержанию практики.

Если ответить не получилось - в рамках того же занятия сможете освежить свои знания и подойти чуть позже.

1. Что такое Event-Driven Architecture, какие ключевые компоненты она в себя включает?
2. Какие есть плюсы и минусы от использования Event-Driven взаимодействия в рамках приложения?
3. Для чего нужен и какую роль играет RabbitMQ? Какой протокол он реализует?
4. Какие основные сущности есть в RabbitMQ? Как выполняется коммуникация между Producer и Consumer?
5. Из каких частей состоит сообщение в RabbitMQ? Что такое routing key сообщения?
6. Какие виды Exchange поддерживает RabbitMQ?
7. Чем direct exchange отличается от topic exchange?
8. Как выполняется связь между exchange и queue?
9. Для чего и как используется RabbitMQ в вашем приложении?

# Ход работы

В рамках лабораторной работы мы познакомимся с брокером сообщений RabbitMQ, который реализует протокол AMQP.
Реализуем с помощью данного брокера event-driven взаимодействие между двумя сервисами нашего приложения.

Давайте начнем с установки RabbitMQ, сделаем это через `Docker`. Используем следующий `docker-compose.yaml`:

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-standalone-lab3
    ports:
      # - "5672:5672"
      - "15671:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - ./rabbitmq-data:/var/lib/rabbitmq
```

Такая конфигурация поднимет нам RabbitMQ, однако не пробросит порт 5672 на хостовую систему (это порт для работы с AMQP, он нам пока не
нужен).
Консоль администратора будет доступна в браузере по `localhost:15671`, пользователь `guest`, пароль `guest`.

Давайте зайдем в консоль администратора:

![img.png](images/img.png)

![img_1.png](images/img_1.png)

Отлично, через данный интерфейс мы можем создавать и настраивать `Exchange`, `Queue`, `Binding`, а также посылать сообщения в любые
`Exchange`.

Вспомним, что принципиально схема роутинга сообщений в RabbitMQ выглядит следующим образом:

![img_2.png](images/img_2.png)

То есть `Producer` отправляет сообщение с определенными параметрами (`routing key` и заголовки) в обменник (`exchange`), после чего,
в зависимости от типа обменника и объявленных правил (`binding`), сообщение отправляется в одну или несколько очередей (`queue`).
После чего `Consumer` подхватывает сообщение из очереди и обрабатывает его.

> Фиксируйте выполнение этапов скриншотами, они понадобятся на защите. Скринить можно итоговые результаты.

Давайте начнем с создания очереди, сделать это мы сможем на вкладке `Queues and Streams`:

![img_3.png](images/img_3.png)

Заполним параметры создаваемой очереди, в качестве имени укажем `queue.{номер_группы}.{номер_в_списке}`:

> Здесь и далее в качестве параметров {номер_группы}.{номер_в_списке} не будут подставляться конкретные числа, вам же необходимо будет
> подставить номер своей группы и свой номер в списке.

![img_7.png](images/img_7.png)

![img_8.png](images/img_8.png)

Отлично, очередь создана, однако сама по себе она использована быть не может. Нам необходимо создать обменник и связать его с данной
очередью некоторым правилом.
Только после этого в очередь смогут поступать сообщения.

Создать обменник можно во вкладке `Exchanges`:

![img_6.png](images/img_6.png)

Давайте создадим `fanout` обменник, он будет отправлять сообщения, приходящие в него, в каждую из связанных с ним очередей.
При этом `routing key` в правилах связи указывать необязательно, для данного типа обменника он ни на что не будет влиять.

Давайте заполним форму создания обменника, укажем нужное название и тип:

![img_9.png](images/img_9.png)

![img_10.png](images/img_10.png)

Отлично, наш обменник появился в списке обменников и мы можем связать его с нашей очередью.
Сделать это можно либо на странице с информацией об обменнике, либо на странице с информацией об очереди.

Давайте перейдем на страницу с информацией об обменнике, нажав на название нашего обменника:

![img_11.png](images/img_11.png)

Теперь добавим связь с нашей очередью, без указания значения `routing key`:

![img_12.png](images/img_12.png)

![img_13.png](images/img_13.png)

Из-за того, что наш обменник - `fanout` он и так будет отправлять любые пришедшие
в него сообщения во все подключенные очереди, никаких правил роутинга по `routing key` или по заголовкам применяться не будет.

Схема роутинга будет выглядить примерно следующим образом:

![img_21.png](images/img_21.png)

Попробуем отправить сообщение, чтобы проверить это!
Давайте отправим два сообщения - одно с пустым `routing key`, в другом в качестве `routing key` укажем произвольное значение:

![img_14.png](images/img_14.png)

![img_15.png](images/img_15.png)

Оба сообщения успешно отправились в единственную подключенную к нашему обменнику очередь, убедиться в этом можно на странице самой очереди:

![img_16.png](images/img_16.png)

Видим, что в очередь пришли оба отправленных нами сообщения.

Теперь создадим `direct` обменник и проверим, как работают его правила роутинга сообщений:

![img_17.png](images/img_17.png)

Добавим связь с очередью, в качестве `routing key` в данном случае укажем `{ваша_группа}.{номер_в_списке}.routing.key`:

![img_18.png](images/img_18.png)

![img_19.png](images/img_19.png)

Для `direct` обменников правила роутинга сообщений более сложные - сообщение попадает в связанную очередь тогда и только тогда, когда
`routing key` сообщения
совпадает с тем, который указан в правиле связи (`binding`) для подключенной к `direct` обменнику очереди.

Схема роутинга выглядит примерно следующим образом:

![img_20.png](images/img_20.png)

Проверим это! Давайте отправим сообщение с `routing key`, который совпадает с тем, что указан в `binding`, а также второе сообщение с другим
`routing key`:

![img_22.png](images/img_22.png)

![img_23.png](images/img_23.png)

Первое сообщение успешно отправилось и ушло в очередь, тогда как второе было откинуто.

Теперь создадим обменник с типом `topic` - он похож на `direct`, но более гибкий.

![img_24.png](images/img_24.png)

В правилах связи с очередями для него можно указывать `wildcard` символы: `*` - мэтчит ровно одно слово; `#` - мэтчит 0 и более слов.

Давайте создадим связь с ключом `{ваша_группа}.*.routing.key`, он будет мэтчить любой номер в списке вашей группы (и не только номер, в
целом любое непустое слово на этой позиции):

![img_25.png](images/img_25.png)

![img_26.png](images/img_26.png)

Логика роутинга тут очень похожа на `direct` обменник, если в правиле связи не используются `wildcard` символы.
Если они используются - то проверяется соответствие ключа сообщения шаблонному выражению, и если соответствие есть, то сообщение
направляется в соответствующую очередь.

Схема примерно следующая:

![img_27.png](images/img_27.png)

Давайте проверим эту логику. Отправим два сообщения, одно с ключом, подходящим под регулярку, а второе - с двумя словами вместо одного между
номером группы и словом routing:

![img_28.png](images/img_28.png)

![img_29.png](images/img_29.png)

Видим, что первое сообщение попало под регулярку по своему ключу, а второе - нет.
`Topic` обменник обеспечивает большую гибкость в роутинге сообщений по сравнению с `direct`, однако сохраняет логику роутинга по
`routing key` сообщения.

До этого момента мы не проставляли заголовки нашим сообщениям по той причине, что они не влияли на роутинг (однако их можно было
использовать для хранения мета-информации).

Однако есть обменник с типом `headers` - в ней логика роутинга осуществляется путем сравнения заголовков сообщения и объявленных `binding`,
при этом `routing key` во внимание не берется.

Давайте создадим такой обменник:

![img_30.png](images/img_30.png)

Теперь объявим правила связи с очередью, в правилах также необходимо указывать заголовки.
Заголовок `x-match` позволяет управлять логикой мэтчинга.
Если установлено значение `all` (поведение по умолчанию) - для срабатывания правила необходимо, чтобы все заголовки из правила были в
сообщении.
Значение `any` вызывает срабатывание, если в сообщении присутствует хотя бы один из заголовков правила.

Укажем заголовки `group` = 'номер группы' и `number` = 'номер в группе':

![img_31.png](images/img_31.png)

![img_32.png](images/img_32.png)

Отлично, теперь только сообщения у которого указаны оба данных заголовка с данными значениями уйдет в очередь. Схема роутинга примерно
следующая:

![img_33.png](images/img_33.png)

Давайте отправим два сообщения, одно с правильными заголовками, одно только с одним из нужных заголовков:

![img_34.png](images/img_34.png)

![img_35.png](images/img_35.png)

Отлично, все сработало как надо. `Headers` - самый гибкий и самый необычный способ роутинга сообщений в RabbitMQ.

> Не забудьте на скриншотах зафиксировать списки созданных обменников, очередей и биндингов!

На этом наше знакомство с работой с RabbitMQ через менеджмент панель завершается, давайте переходить к реализации event-driven коммуникации
в нашем приложении
с помощью этого замечательного инструмента!

Добавлять логику event-driven взаимодействия будем
в [демонстрационный проект](https://github.com/tidbid-kt/archapp).

Склонируем проект или обновим форк, работать будем в ветке `master` (или в той же, где вы делали вторую лабораторную).

В нашем проекте есть два приложения - `user-service` и `notification-service`.
Первый представляет собой REST API для работы с пользователями, а второй - систему отправки уведомлений (реализовано простое логирование в
консоль).

В данный момент эти сервисы никак не связаны, однако нам необходимо, чтобы приложение `notification-service` отправляло уведомление при
создании нового пользователя.
Для этого мы создали общую модель `UserCreatedEvent`, которая теоретически должна передаваться от приложения `user-service` в
`notification-service`.

Реализовать это взаимодействие можно несколькими способами, например, используя простое синхронное взаимодействие через `HttpClient` - после
создания пользователя
приложение `user-service` через `HttpClient` вызывает эндпоинт приложения `notification-service` (например `POST` `/notify`) и передает туда
нужные данные в теле запроса.

Такой подход жестко связывает данные приложения, а также замедляет работу `user-service`, ведь ему сначала необходимо дождаться отправки
уведомления
(а это может быть долгим процессом, особенно если нагрузка на систему большая), а только потом отдать ответ пользователю.

В дополнение к этому при выходе из строя системы отправки уведомлений `user-service` также перестанет корректно отрабатывать.

Избежать этих проблем можно используя асинхронную Event-driven коммуникацию через брокера сообщений, например, RabbitMQ.

В данном случае `user-service` отправляет сообщение в обменник, из которого оно попадает в очередь, откуда ее забирает
`notification-service`.

Для начала подключим в наш проект клиентскую библиотеку RabbitMQ, которая позволит работать с рэббитом из нашего приложения:

Для начала модифицируем файл `pom.xml` для модуля `shared-contracts` (путь `shared-contract/pom.xml`), 
добавив в секцию `<dependencies>...</dependencies>` следующие зависимости:

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

Конфигурацию работы с рэббитом вместе с клиентской библиотекой вынесем в общий модуль, чтобы не дублировать ее в каждый сервис.

В конфигурационные файлы обоих сервисов (`user-service/src/main/resources/application.properties` и `notification-service/src/main/resources/application.properties`)
добавим параметры подключения к рэббиту:

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

Далее настроим `exchange` и подключим к ней очередь, сделаем это в `shared-contracts`, объявив класс `RabbitConfiguration` по пути 
`shared-contract/src/main/java/com/misis/archapp/contract/configuration/RabbitConfiguration.java`:

```java
package com.misis.archapp.contract.configuration;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfiguration {

   public static final String USER_QUEUE = "user.events";
   public static final String USER_EXCHANGE = "user.exchange";
   public static final String USER_ROUTING_KEY = "user.#";

   @Bean
   public Jackson2JsonMessageConverter converter() {
      return new Jackson2JsonMessageConverter();
   }

   @Bean
   public Queue userQueue() {
      return new Queue(USER_QUEUE, true);
   }

   @Bean
   public TopicExchange userExchange() {
      return new TopicExchange(USER_EXCHANGE);
   }

   @Bean
   public Binding userBinding(Queue userQueue, TopicExchange userExchange) {
      return BindingBuilder.bind(userQueue).to(userExchange).with(USER_ROUTING_KEY);
   }
}
```

Здесь мы объявляем `topic` обменник `user.exchange`, который связываем с очередью `user.queue` ключом `user.#`.

Инфраструктурная логика находится в общем для всех сервисов модуле, чтобы не было дублирования.

Чтобы наш класс был нормально обработан спрингом, 
модифицируем классы `user-service/src/main/java/com/misis/archapp/user/UserServiceApplication.java` и `notification-service/src/main/java/com/misis/archapp/notification/NotificationServiceApplication.java`:

```java
package com.misis.archapp.user;

import com.misis.archapp.contract.configuration.RabbitConfiguration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;

@SpringBootApplication
@Import(RabbitConfiguration.class)
public class UserServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }

}
```

```java
package com.misis.archapp.notification;

import com.misis.archapp.contract.configuration.RabbitConfiguration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;

@SpringBootApplication
@Import(RabbitConfiguration.class)
public class NotificationServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(NotificationServiceApplication.class, args);
    }

}
```

Далее давайте добавим логику отправки события создания пользователя в обменник. 
Для этого добавим `UserEventPublisher` - сервис, который позволит публиковать сообщения в RabbitMQ.

Создадим класс по пути `user-service/src/main/java/com/misis/archapp/user/service/publisher/UserEventPublisher.java`:

```java
package com.misis.archapp.user.service.publisher;

import com.misis.archapp.contract.dto.UserCreatedEvent;
import com.misis.archapp.contract.configuration.RabbitConfiguration;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class UserEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    @Autowired
    public UserEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishUserEvent(UserCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfiguration.USER_EXCHANGE,
            "user.created",
            event
        );
    }
    
}
```

Теперь добавим зависимость в `UserService.java` (путь `user-service/src/main/java/com/misis/archapp/user/service/UserService.java`) 
и используем ее, чтобы отправить ивент после создания пользователя: 

> Код, представленный ниже, будет работать только если в репозитории была реализована вторая лабораторная работа

```java
package com.misis.archapp.user.service;

import com.misis.archapp.contract.dto.UserCreatedEvent;
import com.misis.archapp.user.db.User;
import com.misis.archapp.user.db.UserRepository;
import com.misis.archapp.user.dto.UserCreateDTO;
import com.misis.archapp.user.dto.UserDTO;
import com.misis.archapp.user.dto.UserUpdateDTO;
import com.misis.archapp.user.dto.mapper.UserMapper;
import com.misis.archapp.user.service.cache.UserCacheService;
import com.misis.archapp.user.service.publisher.UserEventPublisher;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class UserService {

    private static final Logger LOGGER = LoggerFactory.getLogger(UserService.class);

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final UserCacheService userCacheService;
    private final UserEventPublisher userEventPublisher;

    @Autowired
    public UserService(
        UserRepository userRepository,
        UserMapper userMapper,
        UserCacheService userCacheService,
        UserEventPublisher userEventPublisher
    ) {
        this.userRepository = userRepository;
        this.userMapper = userMapper;
        this.userCacheService = userCacheService;
        this.userEventPublisher = userEventPublisher;
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

Отлично, логика отправки ивента (логика со стороны `Producer` - `user-service`) прописана, теперь необходимо добавить логику обработки ивента в 
`notification-service`.

Для этого объявим слушателя событий над очередью `user.events` в `notification-service`!

Создадим класс `EventConsumerListener` (путь `notification-service/src/main/java/com/misis/archapp/notification/listener/EventConsumerListener.java`):

```java
package com.misis.archapp.notification.listener;

import com.misis.archapp.contract.dto.UserCreatedEvent;
import com.misis.archapp.notification.service.NotificationService;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class EventConsumerListener {

   private final NotificationService notificationService;

   @Autowired
   public EventConsumerListener(
           NotificationService notificationService
   ) {
      this.notificationService = notificationService;
   }

   @RabbitListener(queues = "user.events")
   public void handleUserEvent(UserCreatedEvent event) {
      notificationService.sendNotification(event);
   }

}
```

Теперь при поднятии приложения оно будет слушать очередь `user.events`, забирать события оттуда и логировать их содержимое в консоль.

Давайте дополним наш `docker-compose.yaml` поднятием рэббита и проверим работоспособность получившейся схемы:

```yaml
services:
  postgres:
    image: postgres:13.3
    container_name: postgres-archapp-lab3
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
    container_name: keydb-archapp-lab3
    command: "keydb-server /etc/keydb/redis.conf --server-threads 2"
    volumes:
      #      - "./redis.conf:/etc/keydb/redis.conf"
      - "data:/data"
    ports:
      - "6379:6379"
    restart: unless-stopped
    
  rabbitmq:
    image: "rabbitmq:3-management"
    container_name: rabbitmq-archapp-lab3
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - ./rabbitmq-data:/var/lib/rabbitmq

volumes:
  data:
    driver: local
```

Запустим сервисы:

![img.png](images/img123.png)

После поднятия всех сервисов запустим `UserServiceApplication` и `NotificationServiceApplication` (инструкции см. в README проекта):

![img_1.png](images/img_1123.png)

Далее создадим пользователя через Swagger (метод `POST`) и посмотрим, пришел ли ивент во второй сервис:

![img_2.png](images/img_2123.png)

Видим, что ивенты приходят и логируются, а значит наши сервисы общаются асинхронно с помощью брокера сообщений!

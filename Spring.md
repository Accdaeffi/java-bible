# База

## Dependency Injection
@Qualifier("beanName") - чтобы указать, какой именно брать. Вешается в момент иньекции.
@Profile("test") - указывает, к какому профилю относится бин. Сам профиль меняется в .properties файле как "spring.profile.active=test".
@Primary - чтобы сделать приоритетным для выбора (если есть несколько одинаковых) и другие методы не используются.

## Жизненный цикл бина
1. Создание инстанса.
2. Заполнение полей
3. Работа с *Awarness
    1. Установка имени бина, если BeanNameAwarness
    2. Установка фабрики бинов, если BeanFactoryAwarness
    3. Установка контекста, если ApplicationContextAwarness
4. Преинициализация - вызовы @PostConstruct методов в порядке определения в класе. + отработка PreInitialization от BeanPostProcessors
5. Если имплементирован InitializingBean - вызов afterPropertiesSet() метода.
6. Вызов метода, указанного в @Bean(initMethod="foo")
7. Отработка PostInitialization от BeanPostProcessors
8. ЖИВОЙ
9. Контейнер посылает сигнал на завершение.
10. Отработка @PreDestroy методов в порядке определения в классе.
11. Если импелементирован DisposableBean - отработка destroy()
12. Вызов метода, указанного в @Bean(destroyMethod="bar")
13. Смерть.

## External Properties
Берутся из аргументов командной строки, переменных окружения и файлов с параметрами. Возможна перезапись в порядке написания (т.е. аргументы командной строки - самое важное).
Spring сам берёт значения из application.properties в ресурсах, но можно указать дополнительные файлы с аннотацией @PropertySource("classpath:foo.properties").

Если есть значение foo.bar, то задача его значения будет выглядеть так:
properties: foo.bar=test
env var: FOO_BAR=test
CL args: --foo.bar=test
Да, они все переписывают одно и то же значение.

properties файлы дополнительно могут переписывать друг друга - например, про использовании профилей. Также они могут автоматически подключаться при смене профиля.
Так, например, про использовании профиля "dev" кроме application.properties будет автоматически подключен ещё application-dev.properties.

Если все значения в одном файле имеют одинаковый префикс (например, db.foo, db.bar...), можно использовать аннотацию @ConfigurationProperties("db"), чтобы не писать каждый раз. 
Правда, это также потребует наличия @Configuration.



# Web MVC
Если надо вернуть не просто строку, а нормальную HTML-страницу, то в сигнатуре метод должен возвращать Model или же ModelAndView.

WebDataBinder - позволяет более гибко настраивать, как привязывать объект к View, например, запрещает работать с некоторыми полями.
Аннотация на метод - @InitBinder, в параметрах - WebDataBinder

Аннотация @ModelAttribute("foo") вешается на метод и позволяет указать, что записывать в атрибут каждого отправленной из контролера модели.
Также можно вешать на поля - и это позволит напрямую

Создание API: аннотация @ResponseBody означает, что объект, возвращаемый методом, надо как-то засунуть в тело ответа. 
Для строки очевидно, а вот разные объекты могут во что-то переделаться - например, в JSON.

Интерфейс Formatter<Class> позволяет удобно парсить строку, вернутую запросом, в объект - например, искать значение в Enum. 
Имплементируется, прописывается нужный класс и метод parse перегружается.

## Обработка исключений
@ExceptionHandler позволяет обрабатывать исключения на уровне контролёра. Более гибко, может работать с ModelAndView (но не с Model).

@ResponseStatus вешается на кастомное исключение и позволяет сказать Spring, какой HTTP код кидать, если это исключение вылезло.
Глобальное для приложения.

SimpleMappingExceptionResolver - позволяет при указанных исключениях перенаправлять на указанный вид. 
Данные передать нельзя, но сказать, что "всё сломалось, но мы работаем", можно.

Дефолтный просто переводит стандартные исключения Spring в коды - но ничего более.

Можно сделать своё - и даже указать приоритетность обработки. Для этого надо имплементировать HandlerExceptionResolver.
Желательно, через интерфейс Ordered, указать приоритетность.

Аннотация @ControllerAdvice позволяет отметить класс, где будут глобальные @ExcepptionHandler. 
При тестировании через MockMVC, правда, его надо указывать напрямую - через .setControllerAdvice. 

## Content Negotiation View Resolver
Определяет, к какому именно view стучаться - к html? к pdf? к xml?
Вообще и само по себе неплохо работает. 

Позволяет клиентам самим решать, выгружать ли им данные в json или же xml, например. Нужны библиотеки, чтобы подключить новый типы, например, xml.
Работает на основе Accept в HTTP Header.
Если попытаться попросить то, чего нет - кинет 406 Not Acceptable.

При тестировании придётся указывать, какой именно тип View мы хотим получить. Да и просто при запросах.


# WebFlux
Альтернатива WebMVC для упора в реактивность. Работает на Netty контейнере, вместо сервлетов - потоки.
Все сервлеты не работают, WebMVC - тоже. Контроллеры, кстати, вроде пашут.

ModelAndView не работает, а вот Model - спокойно.

Для валидации - тот же самый @Valid, только выглядит вот так
```java
public Mono<String> saveOrUpdate(@Valid @ModelAttribute("recipe") Mono<RecipeCommand> command) {
    return command
        // нормальное поведение
        .flatMap(recipeService::saveRecipeCommand)
        .map(recipe -> "redirect:/recipe/" + recipe.getId() + "/show")
        // вылезает при ошибке 
        .doOnError(thr -> log.error("Error saving recipe"))
        .onErrorResume(WebExchangeBindException.class, thr -> Mono.just(RECIPE_RECIPEFORM));
}
```

Для того, чтобы сервак принимал реактивный объект (например, для REST полезно), можно сделать:
methodName (@RequestBody Publishier<CLASS> ClassStream)
А потом, например, чтобы сохранить:
return repository.saveAll(ClassStream).then()

RouterFunction позволяет отдельно настроить, что возвращают некоторые запросы. Оформляется как @Bean в классе конфигов.
(хз вообще, зачем, но ладно) Полезно для создания api.

WebTestClient можно использовать для тестирования. 
Забиваются все параметры, потом .exchange() для осуществления запроса и куча .expect*() для проверки.
Создание: WebTestClient.bindToController(ControllerToTest.class).build();



# REST
Как-то завязано на RestTemplate, который позволяет получать по указанному URL объект указанного класса (который автоматически парсится в поля).

UriComponentsBuilder помогает конструировать URL-адрес для запроса (например, задавать значения параметров).

WebClient - позволяет делать запросы куда надо. Отличается от RestTemplate тем, что реактивный.

@Controller должен возвращать ResponseEntity<нужный класс>. @RestController может просто возвращать нужный класс, сам обернёт, как надо.
Статус указывается либо в параметрах конструктора ResponseEntity, либо в аннотации @ResponseStatus, которая вешается на метод.

## Spring REST docs
Программа для автогенерации документации.

Работает на тестах контроллеров.
Парсит тесты, генерирует сниппеты (curl-request, http-request, request-body, response-body...), собирает их (через Asciidoctor).

На классы тестов вешается @ExtendWith(RestDocumentationExtension.class) и @AutoConfigureRestDocs(по вкусу можно добавить: uriScheme = "http", uriHost = "localhost", uriPort = 80).
А ещё заменить импорт MockMVCRequestBuilders на RestDocumentationRequestBuilders.
К mockMvc после .andExpect() надо приделать .andDo() с различными параметрами, как и что описывать.

```java
.andDo(document("id запроса", 
  pathParameters(
    parameterWithName("name").description("desc")
  ),
  requestParameters(
    parameterWithName("name2").description("desc2")
  ),
  responseFields(
    fieldWithPath("ignField").description("desc4"),
    fieldWithPath("field").description("desc3")
  )
  requestFields(
    fieldWithPath("ignField").ignored(),
    fieldWithPath("field").desription("desc3butanother")
  )));
```

Можно ещё сделать автодобавление ограничений - но потребуется создавать образец в test.java.resources.org.springframework.restdocs.templates 
и свой кастомный метод, возвращающий FieldDesriptor. Но это довольно сложно. Можно похимчить с https://scacap.github.io/spring-auto-restdocs/.

Полученная документация может сама встраиваться в проект - благодаря Spring Boot.

Есть расширения аля restdocs-api-specs (автогенерация OpenApi) или spring-auto-restdocs (благодаря рефлексии показывает входные и выходные параметры).

Для итогового файла для asciidoctor надо настроить разметку в /src/main/asciidoc/index.doc

Отображение файла - с помощью maven-resource-plagin.

  
# Транзакции
Позволяют делать "всё или ничего" из куска кода. Особо часто работает в БД.
  
Как реализовывать?
1. Программно, через entityManager.getTransaction(). Скучно, сложно, легко ошибится. Но зато чётко ясно, где конец транзакции.
2. Через Spring.
    1. Программно. 
        1. TransactionTemplate.execute(new transactionCallback() {  @Override Object doInTransaction() } ). Потокобезопасны, но вот настройки (например, уровень изоляции) - общий.
        2. PlatformTransactionManager. Создаём DefaultTransactionDefinition, пишем туда имя и поведение при пропагации, передаём это в manager.getTransaction, получаем TransactionStatus. После этого стандартный try с rollback(status) и commit(status) в нужных местах. Сложнее, менее приоритетно, не надо.
    2. Декларативно. @EnableTransactionManagement на конфиг, @Transactional на ПУБЛИЧНЫЙ метод/класс (можно и интерфейс, но тогда может не распознаться). Лучший вариант.
  
Уровни изоляции (Isolation.*):
1. DEFAULT - как в БД
2. READ_UNCOMMITED - чтение незафиксированных данных. Может читать даже то, что не зафиксировали другие транзакции. (Postgres и Oracle не поддерживают)
3. READ_COMMITED - только фиксированные данные. Грязное чтение предотвращено, фантомное и неповторяющее - нет. (дефолтный для Postgres, Oracle и SQL Server)
4. REPEATABLE_READ - грязное и неповторяющееся чтение предотвращено, а вот фантомное - возможно. (дефолтный для MySQL, но Oracle не поддерживает)
5. SERIALIZABLE - всё предотвращено, но максимально бьёт по производительности.
  
Грязное чтение - чтение того, что ещё не зафиксировала другая транзакция.
Неповторяющееся чтение - считывание данных, их ИЗМЕНЕНИЕ другой транзакции, повторное считывание. Считанные данные различаются.
Фантомное чтение - считывание данных, ДОБАВЛЕНИЕ новых другой транзакции, повторное считыванние. Считанные данные различаются.

Распространение (Propagation.*):
1. REQUIRED - если транзакция уже есть, то присоединяется к ней, иначе создаёт новую.
2. SUPPORT - если транзакция уже есть, то присоединяется, иначе выполняется без транзакции. Лучший вариант для извлечения данных.
3. MANDATORY - если транзакция уже есть, то присоединяется, иначе кидает ошибку.
4. NEVER - если транзакция есть - кидает ошибку.
5. NOT_SUPPORTED - приостанавливает имеющуюся транзакцию и работает без транзакции.
6. REQUIRES_NEW - приостанавливает имеющуюся и создаёт новую.
7. NESTED - если транзакция уже есть, то создает точку сохранения, если нет - то создаёт новую.
  
Можно ещё добавить, чтобы транзакция откатывалась при определённом исключении (rollbackFor=CheckedExc.class) (дефолтно откатывается при всех Runtime, чего, обычно, хватает) или же чтобы не откатывалась при некоторых исключениях (noRollbackFor=IllegalStateException.class). 
  
Аннотации лучше всего размещать на сервисном уровне, а не на уровне DAO.
  
Для логирования: org.springframework.transaction=TRACE
  
# Security
Безопасность, все дела.
У пользователя должны быть настроены роли.
    
Включение вебовской защиты:
```java
@EnableWebSecurity
@EnableOAuth2Sso
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.requestMatchers()
            .antMatchers("/login", "/oauth/authorize")
            .and()
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .formLogin().permitAll();
    }
```
    
Получение юзеров из бд: либо автополучение из дефолтной схемы (довольно грустная), либо надо прямо указывать, что и как запрашивать. Пример:
```java
public void configureGlobal(AuthenticationManagerBuilder auth) 
  throws Exception {
    auth.jdbcAuthentication()
      .dataSource(dataSource)
      .usersByUsernameQuery("select email,password,enabled "
        + "from bael_users "
        + "where email = ?")
      .authoritiesByUsernameQuery("select email,authority "
        + "from authorities "
        + "where email = ?");
}
```
    
## Защита от CSRF
CSRF - это когда стучатся прямым запросом по API вместо нормального использования нужной формы. Опасно, особенно для любителей не выходить из сайтов.
Мера защиты - специальное секретное поле с уникальным секретным ключом, который отправляется вместе с формой, так что наш сайт легко получит. А вот вредоносный - нет. По традиции, поле с этим ключом называется _csrf.
Защищать надо всё, кроме тех запросов, которые идут напрямую от клиентов (т.е. API). Даже те, которые из JSON.
    
Защита:
1. Нормальные HTTP глаголы. Чтобы лишь PUT, POST, PATCH и DELETE могли что-то изменять.
2. Включить в фреймворке (в Spring Security 4.0) по умолчанию включена.
3. Добавить в запрос. 
    1. Форма: В Thymeleaf и Spring MVC <form:form> само, в остальных - надо добавить 
    ```html
    <input type="hidden"
	    name="${_csrf.parameterName}"
	    value="${_csrf.token}"/>
    ```
    2. JSON: прописать в head странички
    ```html    
    <meta name="_csrf" content="${_csrf.token}"/>
	<!-- default header name is X-CSRF-TOKEN -->
	<meta name="_csrf_header" content="${_csrf.headerName}"/>
    ````
    3. AJAX
    ```javascript
    var token = $("meta[name='_csrf']").attr("content");
    var header = $("meta[name='_csrf_header']").attr("content");
    $(document).ajaxSend(function(e, xhr, options) {
	    xhr.setRequestHeader(header, token);
    ```
    
С фалйлами отдельное веселье.
    
    

# i18n 
По-дефолту - берутся из Accpet-language пришедшего заголовка HTTP. 
Но, если сильно надо, можно использоваться FixedLocaleResolver (берёт из настроек JVM), а также CookieLocaleResolver (из печенек) или SessionLocaleResolver (из парамтеров сессии).

Файлы локализации берутся в порядке - полное совпадение, только язык, какое-то дефолтное (без указания локализации).

# Литература:
https://spring.io/projects/spring-restdocs - RESTdocs 
https://www.baeldung.com/spring-transactional-propagation-isolation - транзакции
https://akorsa.ru/2017/01/sovety-i-oshibki-v-spring-transactions/ - транзакции

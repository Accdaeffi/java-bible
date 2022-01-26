# Общее
Кратко - легковесная специализированная виртуалка.

Делится на слои, каждый слой (кроме последнего) - неизменяем.
Позволяет использовать один и тот же слой в нескольких виртуалках, что круто.

Образы (images) - образы для запуска.
Контейнеры (containers) - уже запущенный образ.
Volumes - примонтированные к докеру папки хоста. Например, чтобы БД не очищалась при перезапуске контейнера. 

# Команды
Запускается на docker run <image>, останавливается на docker stop <conta>. Смотреть общую ситуацию - на docker ps.
Логи - docker logs <conta>. Только конец - добавить ключ -f.
Общая информация про локальные образы - docker images. 
Удаление контейнеров - docker rm <conta>, удаление образов - docker rmi <image>

Запуск:
В фоне: -d
Интерактивно: -it <image> bash
С проброской порта: -p hostport:imageport. Возможно указывать диапазон: minport-maxport:imageport.
С маппингом в файловую систему: -v hostpath:imagepath
Переменная окружения: -e VAR=foo

Если внезапно надо влезть интеркативно в работающее: docker exec -it <conta> bash

Делать свои контейнеры на docker build -t <tag> <путь до докерфайла>, где <tag> - название версии. 
Надо долго и весело настраивать Dockerfile, так что сложно.

Очистка всего мусора, что накопился: docker system prune --volumes. Вместо system можно написать volume, image или же container.

# Docker Compose
Или когда лень прописывать все команды.

Встроено в docker, позволяет управлять сразу кучей контейнеров - тем самым must have для микросервисов.

## Как использовать
1. Создаётся docker-compose.yaml.
2. В нём прописывается текущая версия docker compose: "version: ''"
3. В нём прописываются сервисы "services:"
	3.1. Название сервиса: "myDB:"
		3.2. Вся информация по сервису: какой образ, какие порты, какие переменные окружения, какую команду запускать...
		3.3. Можно ещё прописать "depends_on: 'имена сервисов'", чтобы чётко показать, как от кого зависит (например, чтобы конфиг поднимался первым).
		3.4. docker-compose 2.0: Также ещё можно прописать "restart: on-failure", чтобы перезапускалось при каждом падении.
		
В общем, тут пипец, потому что документация противоречит примерам. Молиться, гуглить и пытаться. 		

Иногда надо указывать прямой URL доступа (например, datasource.url) и при этом уже старый-добрый локалхост не прокатит. Вместо него используют "host.docker.internal" - главное не забыть воткнуть порт в конце.

## Команды
1. docker-compose up - запуск всего 
2. docker-compose down - остановка всего
3. ps, images - как и в обычном докере.
4. logs, exec - как и в обычном докере, но надо указать имя сервиса.

# Что там с Java?
Докер работает слоями и очевидный вариант - выделить все зависимости в один слой, чтобы они не дублировалась для каждого микросервиса.
Для этого, в pom.xml надо прописать:
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <layers>
          <enabled>true</enabled>
          <includeLayerTools>true</includeLayerTools>
        </layers>
      </configuration>
    </plugin>
  </plugins>
</build>
``` 
Будет создан всё ещё 1 JAR, но внутри будут слои. Посмотреть их можно по команде ```java -Djarmode=layertools -jar my-app.jar list```.

Но просто так его не используешь, так что надо ещё и нормально использовать. Вот так, например.
```
//Создание слоёв
FROM openjdk:11-jre-slim as builder
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

// Создание нужного образа с копированием слоёв
FROM openjdk:11-jre-slim
WORKDIR application
COPY – from=builder application/dependencies/ ./
COPY – from=builder application/spring-boot-loader/ ./
COPY – from=builder application/snapshot-dependencies/ ./
COPY – from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```


# Дополнение: Запуск из Maven
А оно вообще надо?
```xml
<plugin>
         <groupId>io.fabric8</groupId>
         <artifactId>docker-maven-plugin</artifactId>
         <version>0.26.0</version>
         <executions>
             <execution>
                 <id>start</id>
                 <phase>pre-integration-test</phase>
                 <goals>
                     <goal>build</goal>
                     <goal>start</goal>
                 </goals>
             </execution>
             <execution>
                 <id>stop</id>
                 <phase>post-integration-test</phase>
                 <goals>
                     <goal>stop</goal>
                 </goals>
             </execution>
         </executions>


         <configuration>
             <images>
                 <image>
                 <name>docker.io/kkapelon/docker-maven-comparison</name>
                     <build>
                     <dockerFile>${project.basedir}/Dockerfile</dockerFile >
                         </build>
                         <run>
                             <ports>
                                 <port>8080:8080</port>
                             </ports>
                             <wait>
                             <!-- Check for this URL to return a 200 return code .... -->
                             <url>http://localhost:8080/wizard</url>
                                 <time>120000</time>
                             </wait>     
                         </run>
                     </image>

                 </images>
             </configuration>
         </plugin>
```

# Список литературы:
https://docs.docker.com/config/pruning/ - доки, а точнее - ссылка на очистку.
https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/#packaging.layers - веселье со слоями

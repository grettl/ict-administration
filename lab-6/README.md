# Лабораторная работа № 6 — GitLab, CI/CD и мониторинг nginx

## Задание

В этой лабораторной мы должны были запустить GitLab, настроить вход через Keycloak и проверить, как GitLab запускает задания на двух разных раннерах. Ещё нужно было собрать Docker-образ, загрузить его в Registry и вывести статистику nginx в Grafana.

## Ход работы

### Запуск GitLab и настройка nginx

Сначала запустили GitLab и остальные сервисы через Docker Compose. Перед тем как открыть GitLab в браузере, добавили в Windows сертификат нашего локального центра сертификации. Он нужен, чтобы браузер доверял сертификатам для адресов лабораторной.

<img src="screenshots/01.png" alt="Добавление локального CA в доверенные сертификаты" width="740">


Этот же сертификат добавили в GitLab через папку `gitlab/trusted-certs`. Он нужен самому GitLab для подключения к Keycloak по HTTPS. Дождались, пока Docker покажет статус `healthy`, и открыли `https://gitlab.lab.test`. Появилась страница входа в GitLab.

<img src="screenshots/02.png" alt="Контейнер GitLab в состоянии healthy" width="520">

<img src="screenshots/03.png" alt="Страница входа GitLab CE по HTTPS" width="650">


Дальше проверили nginx. При запросе по HTTP он отправил нас на HTTPS и вернул код `301`. На `/-/metrics` получили `403`, то есть доступ к этому адресу был закрыт. Потом несколько раз подряд обратились к странице входа `/users/sign_in`. Первые шесть запросов прошли с кодом `200`, а на следующие nginx ответил `429`. Так мы проверили, что ограничение действительно работает.

<img src="screenshots/04.png" alt="Проверка перенаправления, ограничения доступа к метрикам и частоты запросов" width="400">


### Единый вход через Keycloak

В Keycloak создали отдельную область `lab6`. Там у нас был тестовый пользователь `student` и два клиента, `gitlab` и `grafana`. Клиент в Keycloak — это настройка для конкретного приложения. Благодаря этому мы могли входить в оба приложения под одной учётной записью.

Сначала проверили, что адрес настроек OpenID Connect открывается по HTTPS. Получили код `200`. Потом попробовали войти в GitLab через Keycloak. После ввода логина и пароля вернулись в GitLab, на страницу проектов.

<img src="screenshots/05.png" alt="Проверка конфигурации OIDC по HTTPS" width="690">

<img src="screenshots/06.png" alt="Страница входа Keycloak" width="610">

<img src="screenshots/07.png" alt="Страница проектов после входа в GitLab" width="780">


Дальше проверили Grafana. По HTTPS получили код `302`, то есть Grafana перенаправила нас на страницу входа. Открыли её в браузере и увидели кнопку входа через Keycloak.

<img src="screenshots/08.png" alt="Перенаправление на страницу входа Grafana" width="760">

<img src="screenshots/09.png" alt="Страница входа Grafana" width="620">

<img src="screenshots/10.png" alt="Кнопка входа в Grafana через Keycloak" width="620">


Нажали на кнопку и вошли через Keycloak под тем же пользователем `student`. После этого открылась главная страница Grafana. Заодно ещё раз проверили, что GitLab тоже показывает профиль пользователя лаборатории.

<img src="screenshots/11.png" alt="Авторизация пользователя student" width="590">

<img src="screenshots/12.png" alt="Настройки клиента grafana в Keycloak" width="790">

<img src="screenshots/13.png" alt="Главная страница Grafana после входа" width="730">

<img src="screenshots/14.png" alt="Профиль пользователя лаборатории в GitLab" width="780">


### Статический Docker Runner

Перед настройкой раннера проверили, видят ли контейнеры GitLab по именам `gitlab` и `gitlab.lab.test`. Оба имени нашли. Ещё хотели посмотреть сертификат через `openssl`, но в контейнере nginx этой команды не оказалось. Что раннер доверяет нашему сертификату, стало понятно позже, когда он смог зарегистрироваться.

<img src="screenshots/15.png" alt="План настройки раннера и состояние контейнеров" width="760">

<img src="screenshots/16.png" alt="Проверка имён GitLab внутри контейнеров" width="800">


В GitLab создали проект `lab6-ci` и добавили раннер `lab6-static-docker` с тегом `docker`. Сразу после создания GitLab показал `Never contacted`. Это значит, что раннер уже добавили в проект, но он ещё ни разу не подключался.

<img src="screenshots/17.png" alt="Проект lab6-ci" width="550">

<img src="screenshots/18.png" alt="Форма создания проектного раннера" width="500">

<img src="screenshots/19.png" alt="Страница регистрации статического раннера" width="740">

<img src="screenshots/20.png" alt="Статус раннера до подключения" width="760">


Потом зарегистрировали раннер и выбрали Docker для запуска заданий. В GitLab появились статусы `Online` и `Idle`. То есть раннер подключился и ждал, когда ему дадут задание.

<img src="screenshots/21.png" alt="Успешная регистрация Docker Runner" width="790">

<img src="screenshots/22.png" alt="Статический раннер подключён и ожидает задания" width="800">


Для первой проверки создали `.gitlab-ci.yml` с заданием `runner-check`. Оно запускалось в Alpine, выводило версию системы и проверяло, что из репозитория скачался `README.md`. В конце в логе появилось `LAB6_RUNNER_OK`.

<img src="screenshots/23.png" alt="Настройка задания runner-check" width="650">

<img src="screenshots/24.png" alt="Успешное выполнение runner-check" width="860">


### Сборка и публикация Docker-образа

Потом добавили `build-image`. В этом задании мы собирали Docker-образ, запускали его для проверки и отправляли в GitLab Container Registry. Для сборки использовали Docker-in-Docker. В задании запускались команды Docker, а саму сборку выполнял отдельный Docker-демон.

Со сборкой пришлось повозиться. Сначала Docker долго ждал ответа от Docker Hub и так и не смог скачать `alpine:3.22`. В другой попытке образ уже запустился, но при отправке в Registry появилась ошибка `blob unknown to registry`.

<img src="screenshots/25.png" alt="Ошибка загрузки базового образа в build-image" width="880">


В итоге собрали образ через Buildx с параметрами `--provenance=false --sbom=false --load`. Первые два отключили дополнительные данные о сборке и составе образа. Параметр `--load` загрузил образ в Docker, чтобы мы могли его запустить и потом отправить в Registry. В логе сначала появилось `LAB6_IMAGE_OK`, а после отправки — `LAB6_BUILD_PUSH_OK`.

<img src="screenshots/26.png" alt="Успешный push образа в Container Registry" width="820">


В пайплайне № 5 оба задания, `runner-check` и `build-image`, завершились успешно. Потом открыли Container Registry и увидели там образ с тегом `8254a493`. Это короткий хеш коммита, из которого собрали образ. В GitLab размер отображался как 8,19 MiB.

<img src="screenshots/27.png" alt="Успешный пайплайн № 5" width="680">

<img src="screenshots/28.png" alt="Репозиторий lab6-ci в Container Registry" width="380">

<img src="screenshots/29.png" alt="Тег опубликованного образа и его размер" width="690">


### Динамический Kubernetes Runner

Для второго раннера подняли Kubernetes через kind. Посмотрели, к какому кластеру подключён `kubectl`, и проверили состояние кластера. Узел был в статусе `Ready`, а системные поды — в `Running`.

<img src="screenshots/30.png" alt="Узел и системные поды кластера kind" width="860">


В том же проекте GitLab появился второй раннер, `lab6-kubernetes-dynamic`. Теперь сборку образа можно было оставить Docker-раннеру, а проверки запускать в Kubernetes. Для каждой такой проверки создаётся свой временный под. После задания под удаляется, а раннер остаётся подключённым и может брать следующие задания.

<img src="screenshots/31.png" alt="Два подключённых раннера проекта" width="800">


Сначала запустили одно задание, `kubernetes-check`. В логе увидели, что скачался репозиторий, прошла проверка `README.md` и появилось `LAB6_K8S_JOB_OK`. После этого запустили две проверки одновременно. На скриншоте GitLab обе ещё выполняются. На другом скриншоте видно, что один под ещё работает, а второй уже удаляется.

<img src="screenshots/32.png" alt="Успешная проверка Kubernetes Runner" width="740">

<img src="screenshots/33.png" alt="Два задания Kubernetes выполняются одновременно" width="290">

<img src="screenshots/33_parallel_pods.png" alt="Временные поды параллельных заданий" width="800">


В итоге в пайплайне № 7 успешно прошли все пять заданий. В стадии тестирования прошли проверка Docker-раннера, проверка Kubernetes-раннера и две параллельные проверки. Следом выполнилась сборка образа.

<img src="screenshots/34.png" alt="Пайплайн № 7 со всеми пятью успешными заданиями" width="450">


### Настройка мониторинга

Чтобы увидеть статистику nginx в Grafana, сначала подключили nginx exporter и Prometheus. Exporter берёт статистику у nginx, а Prometheus её собирает. Для проверки открыли адрес `nginx-exporter:9113/metrics` из сети стенда. Там увидели `nginx_up` со значением `1`, четыре активных соединения и 6623 запроса.

<img src="screenshots/35.png" alt="Проверка метрик nginx внутри сети стенда" width="810">


Заодно проверили в Docker Compose, что контейнеры exporter и Prometheus запущены. Потом запустили Grafana с файлом `compose.override.yaml` и указали адрес Prometheus `http://prometheus:9090`. Нажали **Save & test**. Grafana подтвердила, что смогла получить ответ от Prometheus.

<img src="screenshots/36.png" alt="Состояние nginx exporter и Prometheus" width="820">

<img src="screenshots/37.png" alt="Состояние Grafana после применения Compose override" width="760">

<img src="screenshots/38.png" alt="Настройки источника Prometheus в Grafana" width="800">

<img src="screenshots/39.png" alt="Успешная проверка соединения с Prometheus" width="780">


### Создание графиков в Grafana

На первом графике показали запросы в секунду. Счётчик `nginx_http_requests_total` всё время растёт, поэтому для графика нам нужна скорость его изменения. Так видно, когда запросов становилось больше или меньше.

<img src="screenshots/40.png" alt="График запросов nginx в секунду" width="760">


На втором графике вывели `nginx_connections_active`, чтобы видеть, сколько соединений открыто в данный момент. На третьей панели показали `nginx_up`. На скриншоте там стоит `1`, значит exporter может получить данные от nginx.

<img src="screenshots/41.png" alt="Графики запросов и активных соединений" width="860">

<img src="screenshots/42.png" alt="Итоговый дашборд с тремя показателями nginx" width="880">


### Что посмотреть на запущенном стенде

GitLab, Keycloak, nginx и Grafana запускали с файлами `compose.yaml` и `compose.override.yaml`. Для Prometheus и exporter использовали `compose.monitoring.yaml`, а для статического раннера — `compose.runner.yaml`. Для Kubernetes-раннера отдельно подняли кластер kind.

Для проверки входа можно открыть `https://gitlab.lab.test` и `https://grafana.lab.test` и войти через Keycloak. В GitLab нужно открыть проект `lab6-ci`, затем раздел **Build** и страницу **Pipelines**. Там виден пайплайн № 7. Образ можно посмотреть на странице **Container Registry** в разделе **Deploy**, а три панели nginx — на дашборде Grafana.

## Вывод

Полезнее всего было посмотреть, что происходит с заданием после запуска пайплайна. На Docker-раннере мы собрали образ и отправили его в Registry. В Kubernetes увидели, как для двух проверок создаются отдельные поды и как один из них удаляется, пока второй ещё работает.

Больше всего времени ушло на сборку и отправку образа. Сначала не скачивался Alpine, а потом Registry выдавал ошибку уже при загрузке готового образа. После успешной попытки проверили и лог задания, и сам тег в Registry. Ещё разобрались с мониторингом. На дашборде стало видно, когда запросов и открытых соединений становилось больше, и получает ли exporter данные от nginx.

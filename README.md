# Демонстрационное микросервисное приложение с интеграцией Istio Service Mesh

## Состав проекта

| Сервис               | Описание                                                          |
|----------------------|-------------------------------------------------------------------|
| **`webapp`**         | Веб-сервис для тестирования входящих запросов                     |
| **`psql`**           | Клиент для тестирования подключения к СУБД PostgreSQL             |
| **`ingressgateway`** | Граничный шлюз, обслуживающий входящие запросы в Service Mesh     |
| **`egressgateway`**  | Граничный шлюз, обслуживающий исходящие запросы из Service Mesh   |
| **`keycloak`**       | IdP-провайдер для аутентификации входящих в Service Mesh запросов |

## Конфигурация

### Секция `images`

| Параметр       | Описание                                                              |
|----------------|-----------------------------------------------------------------------|
| **`webapp`**   | Docker-образ прикладного веб-сервиса                                  |
| **`envoy`**    | Docker-образ прокси Envoy                                             |
| **`keycloak`** | Docker-образ IdP-провайдера Keycloak для аутентификации и авторизации |
| **`postgres`** | Docker-образ СУБД PostgreSQL                                          |

### Секция `cp`
| Параметр                         | Описание                                                                             |
|----------------------------------|--------------------------------------------------------------------------------------|
| **`cp.istiod`**                  | Адрес Istiod — управляющего компонента Istio (control plane) в формате <host>:<port> |

**Примечание:** для запуска демонстрационного микросервисного приложения требуется предварительная настройка контрольной панели Istio

### Секция `keycloak`
| Параметр                | Описание                                                   |
|-------------------------|------------------------------------------------------------|
| **`host`**              | Хост IdP-провайдера Keycloak в Kubernetes                  |
| **`database.host`**     | IP-адрес или имя хоста базы данных PostgreSQL для Keycloak |
| **`database.port`**     | Порт подключения к БД PostgreSQL, используемой Keycloak    |
| **`database.user`**     | Имя пользователя для подключения к БД Keycloak             |
| **`database.password`** | Пароль пользователя БД Keycloak                            |
| **`database.schema`**   | Имя схемы в БД, которую использует Keycloak                |
| **`database.name`**     | Название базы данных, используемой Keycloak                |
| **`admin.user`**        | Имя администратора Keycloak                                |
| **`admin.password`**    | Пароль администратора Keycloak                             |
| **`realm`**             | Realm (домен) в Keycloak                                   |

### Секция `internal`

| Параметр          | Описание                                                              |
|-------------------|-----------------------------------------------------------------------|
| **`webapp.host`** | Хост прикладного веб-сервиса в Kubernetes                             |

### Секция `external`
| Параметр            | Описание                                 |
|---------------------|------------------------------------------|
| **`postgres.host`** | Доменное имя внешнего сервера PostgreSQL |
| **`postgres.ip`**   | IP-адрес сервера PostgreSQL              |
| **`postgres.port`** | Порт сервера PostgreSQL                  |
| **`web.host`**      | Хост внешнего веб-сервиса                |
| **`web.ip`**        | IP-адрес внешнего веб-сервиса            |
| **`web.port`**      | Порт внешнего веб-сервиса.               |

### Секция `idp`
| Параметр      | Описание                                                            |
|---------------|---------------------------------------------------------------------|
| **`issuer`**  | URL Issuer Identity Provider для проверки JWT-токенов.              |
| **`jwksUri`** | URI для получения публичных ключей JWKS для верификации подписи JWT |

### Секция `accessControl`
| Параметр                  | Описание                                                |
|---------------------------|---------------------------------------------------------|
| **`accessControl.regex`** | Регулярное выражение для проверки Subject в сертификате |

### Секция `certs`
| Параметр             | Описание                                                                     |
|----------------------|------------------------------------------------------------------------------|
| `egress."tls.crt"`   | Сертификат TLS граничного Egress-шлюза                                       |
| `egress."tls.key"`   | Приватный ключ для сертификата граничного Egress-шлюза                       |
| `ingress."tls.crt"`  | Сертификат TLS прикладного сервиса                                           |
| `ingress."tls.key"`  | Приватный ключ для сертификата прикладного сервиса                           |
| `keycloak."tls.crt"` | Сертификат TLS для IdP-провайдера Keycloak                                   |
| `keycloak."tls.key"` | Приватный ключ для сертификата IdP-провайдера Keycloak                       |
| `common."ca.crt"`    | Корневой и промежуточные CA-сертификаты, доверенные и используемые в проекте |

**Примечание:** сертификаты и ключи указываются в формате **PEM**.


## Установка Helm-чарта
```
helm install $RELEASE_NAME --namespace $NAMESPACE
```

## Обновление Helm-чарта
```
helm upgrade $RELEASE_NAME --namespace $NAMESPACE
```

## Удаление Helm-чарта
```
helm uninstall $RELEASE_NAME --namespace $NAMESPACE
```

## Примеры запросов

### Запрос токена JWT у IdP-провайдера Keycloak
```
TOKEN=$(curl --cacert ca.crt -X POST -d "client_id=demo&client_secret=demo&grant_type=password&username=admin&password=admin" "https://$KEYCLOAK_HOST/auth/realms/demo/protocol/openid-connect/token" | jq -r '.access_token')
```

### Запрос к прикладному веб-сервису снаружи кластера Kubernetes
```
curl -I -X GET --http1.1 -H "Authorization: Bearer $TOKEN" --cacert ca.crt --cert client.crt --key client.key https://$WEBAPP_HOST/readiness
```

### Запрос к внешнему веб-сервису из кластера Kubernetes
```
kubectl exec -ti $WEBAPP_POD -- curl -I --http1.1 http://$WEB_HOST:$WEB_PORT
```

### Запрос к внешнему сервису СУБД из кластера Kubernetes (passthrough)
```
kubectl exec -ti $PSQL_POD -- psql "host=$DB_HOST port=$DB_PORT user=$DB_USER sslrootcert=/certs/ca.crt sslmode=verify-ca"
```


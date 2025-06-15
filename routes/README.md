# Routes Module - Детальное описание

Модуль маршрутизации представляет собой центральную систему управления URL-адресами и их привязкой к соответствующим обработчикам. Давайте разберем эту архитектуру пошагово, начиная с базовых концепций и постепенно переходя к более сложным аспектам реализации.

## Концептуальная основа маршрутизации

Маршрутизация в веб-приложении работает как система навигации в здании. Представьте, что ваш веб-сервер - это большое здание с множеством комнат (обработчиков), а маршруты - это указатели, которые говорят посетителям (HTTP-запросам), в какую именно комнату им нужно попасть. Echo Framework предоставляет нам мощный инструмент для создания такой системы навигации.

## Структура файлов маршрутизации

В директории `routes/` каждый файл отвечает за определенную функциональную область системы. Это позволяет нам поддерживать принцип разделения ответственности и облегчает сопровождение кода. Рассмотрим ключевые файлы:

### `routes/web.go` - Основные веб-маршруты

```go
package routes

import (
    "github.com/labstack/echo/v4"
    "github.com/Her0x27/x-routersbc/handlers"
    "github.com/Her0x27/x-routersbc/core/middleware"
)

// RegisterWebRoutes регистрирует все основные веб-маршруты
// Эти маршруты отвечают за отображение HTML-страниц пользователю
func RegisterWebRoutes(e *echo.Echo) {
    // Группа маршрутов с аутентификацией
    // Middleware auth.Required проверяет, что пользователь авторизован
    authenticated := e.Group("", middleware.AuthRequired())
    
    // Главная страница dashboard - точка входа после авторизации
    authenticated.GET("/", handlers.DashboardHandler)
    
    // Маршруты управления сетью
    // Каждый маршрут привязан к соответствующему обработчику
    networkGroup := authenticated.Group("/network")
    {
        // Основная страница сетевых настроек
        networkGroup.GET("", handlers.NetworkIndexHandler)
        
        // Управление сетевыми интерфейсами
        // Показывает физические, VLAN и VPN интерфейсы
        networkGroup.GET("/interfaces", handlers.NetworkInterfacesHandler)
        
        // Настройки WAN соединения
        // Включает обычный WAN и Multi-WAN балансировку
        networkGroup.GET("/wan", handlers.NetworkWANHandler)
        
        // Настройки локальной сети
        // DHCP, DNS, мостовые соединения
        networkGroup.GET("/lan", handlers.NetworkLANHandler)
        
        // Беспроводные настройки
        // AP, STA, ADHOC, MONITOR режимы
        networkGroup.GET("/wireless", handlers.NetworkWirelessHandler)
        
        // Маршрутизация и UPnP
        networkGroup.GET("/routing", handlers.NetworkRoutingHandler)
        
        // Настройки файрвола
        // Динамически подгружает NFTables или IPTables интерфейс
        networkGroup.GET("/firewall", handlers.NetworkFirewallHandler)
    }
}
```

### `routes/api.go` - REST API маршруты

```go
package routes

import (
    "github.com/labstack/echo/v4"
    "github.com/Her0x27/x-routersbc/handlers/api"
    "github.com/Her0x27/x-routersbc/core/middleware"
)

// RegisterAPIRoutes регистрирует все REST API маршруты версии 1
// API используется для взаимодействия с системой без перезагрузки страниц
func RegisterAPIRoutes(e *echo.Echo) {
    // Группа API с префиксом /api/v1
    // Все API запросы проходят через JSON middleware для корректной обработки
    apiV1 := e.Group("/api/v1", middleware.JSONMiddleware())
    
    // Подгруппа с аутентификацией для защищенных операций
    protected := apiV1.Group("", middleware.AuthRequired())
    
    // API управления сетевыми интерфейсами
    interfacesAPI := protected.Group("/interfaces")
    {
        // GET /api/v1/interfaces - получить список всех интерфейсов
        interfacesAPI.GET("", api.GetInterfacesHandler)
        
        // POST /api/v1/interfaces - создать новый интерфейс
        // Тело запроса содержит конфигурацию интерфейса
        interfacesAPI.POST("", api.CreateInterfaceHandler)
        
        // GET /api/v1/interfaces/:id - получить конкретный интерфейс
        interfacesAPI.GET("/:id", api.GetInterfaceHandler)
        
        // PUT /api/v1/interfaces/:id - обновить интерфейс
        interfacesAPI.PUT("/:id", api.UpdateInterfaceHandler)
        
        // DELETE /api/v1/interfaces/:id - удалить интерфейс
        interfacesAPI.DELETE("/:id", api.DeleteInterfaceHandler)
        
        // POST /api/v1/interfaces/:id/enable - включить интерфейс
        interfacesAPI.POST("/:id/enable", api.EnableInterfaceHandler)
        
        // POST /api/v1/interfaces/:id/disable - отключить интерфейс
        interfacesAPI.POST("/:id/disable", api.DisableInterfaceHandler)
    }
    
    // API управления файрволом
    firewallAPI := protected.Group("/firewall")
    {
        // Управление правилами файрвола
        firewallAPI.GET("/rules", api.GetFirewallRulesHandler)
        firewallAPI.POST("/rules", api.CreateFirewallRuleHandler)
        firewallAPI.PUT("/rules/:id", api.UpdateFirewallRuleHandler)
        firewallAPI.DELETE("/rules/:id", api.DeleteFirewallRuleHandler)
        
        // Управление цепочками правил
        firewallAPI.GET("/chains", api.GetFirewallChainsHandler)
        firewallAPI.POST("/chains", api.CreateFirewallChainHandler)
        
        // Получение текущего статуса файрвола
        firewallAPI.GET("/status", api.GetFirewallStatusHandler)
        
        // Применение конфигурации файрвола
        firewallAPI.POST("/apply", api.ApplyFirewallConfigHandler)
    }
    
    // API управления DNS
    dnsAPI := protected.Group("/dns")
    {
        // Локальные DNS зоны
        dnsAPI.GET("/zones", api.GetDNSZonesHandler)
        dnsAPI.POST("/zones", api.CreateDNSZoneHandler)
        dnsAPI.PUT("/zones/:id", api.UpdateDNSZoneHandler)
        dnsAPI.DELETE("/zones/:id", api.DeleteDNSZoneHandler)
        
        // DNS резолверы
        dnsAPI.GET("/resolvers", api.GetDNSResolversHandler)
        dnsAPI.POST("/resolvers", api.CreateDNSResolverHandler)
        
        // DNS маршрутизация
        dnsAPI.GET("/routing", api.GetDNSRoutingHandler)
        dnsAPI.POST("/routing", api.CreateDNSRoutingHandler)
    }
}
```

### `routes/websocket.go` - WebSocket маршруты

```go
package routes

import (
    "github.com/labstack/echo/v4"
    "github.com/Her0x27/x-routersbc/handlers/ws"
    "github.com/Her0x27/x-routersbc/core/middleware"
)

// RegisterWebSocketRoutes регистрирует WebSocket маршруты для real-time обновлений
// WebSocket позволяет серверу отправлять обновления клиенту без запроса
func RegisterWebSocketRoutes(e *echo.Echo) {
    // WebSocket группа с аутентификацией
    wsGroup := e.Group("/ws", middleware.AuthRequired())
    
    // Основной WebSocket endpoint для системных обновлений
    // Клиенты подключаются сюда для получения real-time уведомлений
    wsGroup.GET("/system", ws.SystemWebSocketHandler)
    
    // WebSocket для мониторинга сетевых интерфейсов
    // Отправляет обновления статуса интерфейсов, трафика, ошибок
    wsGroup.GET("/network", ws.NetworkWebSocketHandler)
    
    // WebSocket для мониторинга файрвола
    // Уведомления о блокированных соединениях, изменениях правил
    wsGroup.GET("/firewall", ws.FirewallWebSocketHandler)
    
    // WebSocket для системных логов
    // Real-time отображение логов системы
    wsGroup.GET("/logs", ws.LogsWebSocketHandler)
}
```

### `routes/auth.go` - Маршруты аутентификации

```go
package routes

import (
    "github.com/labstack/echo/v4"
    "github.com/Her0x27/x-routersbc/handlers/auth"
)

// RegisterAuthRoutes регистрирует маршруты для аутентификации
// Эти маршруты доступны без авторизации
func RegisterAuthRoutes(e *echo.Echo) {
    // Группа аутентификации без middleware проверки авторизации
    authGroup := e.Group("/auth")
    
    // GET /auth/login - показать форму входа
    authGroup.GET("/login", auth.ShowLoginHandler)
    
    // POST /auth/login - обработать данные входа
    // Проверяет хеш пароля и создает сессию
    authGroup.POST("/login", auth.ProcessLoginHandler)
    
    // POST /auth/logout - выход из системы
    // Очищает сессию и перенаправляет на страницу входа
    authGroup.POST("/logout", auth.LogoutHandler)
    
    // GET /auth/check - проверить статус авторизации (для AJAX)
    authGroup.GET("/check", auth.CheckAuthHandler)
}
```

### `routes/static.go` - Статические ресурсы

```go
package routes

import (
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

// RegisterStaticRoutes настраивает обслуживание статических файлов
func RegisterStaticRoutes(e *echo.Echo) {
    // Middleware для сжатия статических файлов
    e.Use(middleware.Gzip())
    
    // Обслуживание CSS, JS, изображений из директории static
    e.Static("/static", "static")
    
    // Favicon обслуживается отдельно для удобства
    e.File("/favicon.ico", "static/img/favicon.ico")
    
    // Middleware для установки правильных заголовков кеширования
    e.Use(middleware.StaticWithConfig(middleware.StaticConfig{
        Root:   "static",
        Browse: false, // Запрещаем просмотр директорий
        HTML5:  true,  // Поддержка HTML5 History API
    }))
}
```

### `routes/errors.go` - Обработка ошибок

```go
package routes

import (
    "net/http"
    "github.com/labstack/echo/v4"
    "github.com/Her0x27/x-routersbc/handlers"
)

// RegisterErrorRoutes настраивает обработку HTTP ошибок
func RegisterErrorRoutes(e *echo.Echo) {
    // Кастомный обработчик HTTP ошибок
    e.HTTPErrorHandler = func(err error, c echo.Context) {
        // Получаем код ошибки из Echo error
        code := http.StatusInternalServerError
        if he, ok := err.(*echo.HTTPError); ok {
            code = he.Code
        }
        
        // В зависимости от кода ошибки показываем соответствующую страницу
        switch code {
        case http.StatusNotFound:
            // 404 - страница не найдена
            handlers.Error404Handler(c)
        case http.StatusNotImplemented:
            // 501 - функционал не реализован
            handlers.Error501Handler(c)
        case http.StatusInternalServerError:
            // 500 - внутренняя ошибка сервера
            handlers.Error500Handler(c)
        case http.StatusUnauthorized:
            // 401 - требуется авторизация
            handlers.ErrorUnauthorizedHandler(c)
        case http.StatusForbidden:
            // 403 - доступ запрещен
            handlers.ErrorForbiddenHandler(c)
        default:
            // Для всех остальных ошибок показываем общую страницу
            handlers.ErrorGenericHandler(c, code)
        }
    }
}
```

## Принципы организации маршрутов

Архитектура маршрутизации следует нескольким важным принципам, которые обеспечивают масштабируемость и удобство сопровождения системы.

### Группировка по функциональности

Маршруты группируются по логическим блокам функциональности. Это не только упрощает понимание структуры приложения, но и позволяет легко применять middleware к целым группам маршрутов. Например, все сетевые настройки находятся в группе `/network`, что логично и интуитивно понятно.

### Версионирование API

REST API использует версионирование через префикс `/api/v1`. Это критически важно для поддержания обратной совместимости при развитии системы. Когда потребуется внести breaking changes в API, можно будет создать `/api/v2`, сохранив работоспособность существующих клиентов.

### Разделение ответственности

Каждый файл маршрутов отвечает за конкретную область функциональности. Веб-маршруты обслуживают HTML-страницы, API-маршруты предоставляют данные в JSON формате, WebSocket-маршруты обеспечивают real-time коммуникацию. Такое разделение упрощает навигацию по коду и его модификацию.

## Middleware и безопасность

Система использует несколько уровней middleware для обеспечения безопасности и корректной работы приложения.

### Аутентификация

Middleware `AuthRequired()` проверяет наличие действительной сессии пользователя. Он применяется ко всем защищенным маршрутам, обеспечивая контроль доступа на уровне маршрутизации. Это первая линия защиты от несанкционированного доступа.

### Обработка JSON

Для API маршрутов используется `JSONMiddleware()`, который обеспечивает корректную обработку JSON запросов и ответов, устанавливает правильные заголовки Content-Type и обрабатывает ошибки парсинга JSON.

### Сжатие и кеширование

Статические ресурсы обслуживаются с включенным сжатием Gzip и правильными заголовками кеширования, что существенно улучшает производительность загрузки страниц.

## Интеграция с обработчиками

Каждый маршрут связан с соответствующим обработчиком в модуле `handlers/`. Эта связь является контрактом между уровнем маршрутизации и уровнем бизнес-логики. Обработчики получают контекст Echo, содержащий всю информацию о запросе, и возвращают соответствующий ответ.

## Расширяемость системы

Архитектура маршрутов спроектирована с учетом будущего расширения функциональности. Добавление новых маршрутов требует создания соответствующего файла в директории `routes/` и регистрации маршрутов в главном инициализаторе системы. Такой подход обеспечивает чистую архитектуру и упрощает добавление новых возможностей.

Эта система маршрутизации обеспечивает надежную основу для веб-интерфейса управления роутером, предоставляя четкое разделение между различными типами запросов и обеспечивая безопасность через правильно настроенные middleware.

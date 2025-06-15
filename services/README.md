# Services Module - Детальная Документация

Модуль Services представляет собой бизнес-логический слой вашего приложения, который выполняет роль моста между обработчиками запросов (handlers) и системными конфигураторами (utils/configurators). Этот слой критически важен для архитектуры, поскольку он инкапсулирует всю сложную логику управления сетевыми настройками и системными операциями.

## Архитектурная роль Services

Представьте Services как опытного системного администратора, который знает, как правильно выполнить каждую операцию. Когда веб-интерфейс получает запрос на изменение настроек, handlers передают этот запрос соответствующему сервису, который затем координирует все необходимые действия: валидацию данных, обращение к конфигураторам, применение изменений и проверку результата.

## Структура файлов Services

Каждый сервис представлен отдельным файлом Go, который содержит структуру сервиса и все связанные с ним методы. Вот основные файлы, которые должны присутствовать в директории services/:

```
services/
├── network_service.go      # Управление сетевыми интерфейсами
├── wan_service.go         # Конфигурация WAN подключений
├── lan_service.go         # Настройки локальной сети
├── wireless_service.go    # WiFi и беспроводные технологии
├── dns_service.go         # DNS серверы и зоны
├── dhcp_service.go        # DHCP сервер и пулы адресов
├── routing_service.go     # Маршрутизация и UPnP
├── firewall_service.go    # Управление межсетевым экраном
├── system_service.go      # Системные операции
└── auth_service.go        # Аутентификация и авторизация
```

## Детальное описание каждого сервиса

### NetworkService (network_service.go)

Этот сервис отвечает за управление всеми типами сетевых интерфейсов в системе. Он работает как диспетчер, который понимает различия между физическими интерфейсами, VLAN и VPN соединениями.

```go
type NetworkService struct {
    netConfigurator *configurators.NetPlanConfigurator
    interfaceConf   *configurators.InterfacesConfigurator
    db              *sql.DB
    logger          *log.Logger
}

// Основные методы сервиса
func (ns *NetworkService) GetInterfaces() ([]NetworkInterface, error)
func (ns *NetworkService) CreateInterface(req CreateInterfaceRequest) error
func (ns *NetworkService) UpdateInterface(id string, req UpdateInterfaceRequest) error
func (ns *NetworkService) DeleteInterface(id string) error
func (ns *NetworkService) EnableInterface(id string) error
func (ns *NetworkService) DisableInterface(id string) error
```

Ключевая особенность NetworkService заключается в том, что он должен понимать различные типы интерфейсов и их специфику. Например, при создании VLAN интерфейса сервис должен проверить существование родительского интерфейса, корректность VLAN ID, а затем сгенерировать соответствующую конфигурацию через netplan или interfaces файл в зависимости от операционной системы.

### WANService (wan_service.go)

WAN сервис управляет подключениями к внешним сетям и является одним из наиболее сложных компонентов системы, поскольку должен поддерживать различные типы подключений и балансировку нагрузки.

```go
type WANService struct {
    networkService  *NetworkService
    routingService  *RoutingService
    firewallService *FirewallService
    db              *sql.DB
}

func (ws *WANService) ConfigureWAN(config WANConfig) error
func (ws *WANService) SetupMultiWAN(configs []WANConfig, policy LoadBalancePolicy) error
func (ws *WANService) GetWANStatus() ([]WANStatus, error)
func (ws *WANService) TestWANConnectivity(wanID string) (ConnectivityResult, error)
```

WAN сервис должен уметь работать с различными типами подключений: статический IP, DHCP, PPPoE, а также координировать настройки маршрутизации и firewall правил для обеспечения корректной работы Multi-WAN сценариев.

### LANService (lan_service.go)

Локальная сеть требует координации между DHCP сервером, DNS настройками и bridge конфигурацией. LAN сервис управляет этими взаимосвязанными компонентами.

```go
type LANService struct {
    dhcpService    *DHCPService
    dnsService     *DNSService
    networkService *NetworkService
    db             *sql.DB
}

func (ls *LANService) ConfigureLAN(config LANConfig) error
func (ls *LANService) SetupBridge(interfaces []string, bridgeName string) error
func (ls *LANService) GetLANClients() ([]LANClient, error)
func (ls *LANService) AssignStaticLease(mac, ip string) error
```

Особенность этого сервиса в том, что он должен обеспечивать согласованность между различными компонентами локальной сети. Например, при изменении диапазона DHCP он должен проверить, что новый диапазон не конфликтует со статическими назначениями.

### WirelessService (wireless_service.go)

Беспроводные технологии требуют специализированного подхода, поскольку WiFi интерфейсы могут работать в различных режимах и требуют специфической конфигурации.

```go
type WirelessService struct {
    networkService *NetworkService
    systemService  *SystemService
    db             *sql.DB
}

func (ws *WirelessService) ScanNetworks(interface string) ([]WiFiNetwork, error)
func (ws *WirelessService) ConfigureAP(config APConfig) error
func (ws *WirelessService) ConnectToNetwork(config STAConfig) error
func (ws *WirelessService) SetMonitorMode(interface string) error
func (ws *WirelessService) CreateAdhocNetwork(config AdhocConfig) error
```

Сервис должен взаимодействовать с системными утилитами типа wpa_supplicant, hostapd, и iw для управления беспроводными соединениями.

### DNSService (dns_service.go)

DNS сервис управляет разрешением имен и является критически важным для работы сети. Он должен поддерживать различные режимы работы и современные протоколы безопасности.

```go
type DNSService struct {
    systemService *SystemService
    db            *sql.DB
    dnsCache      map[string]DNSRecord
}

func (ds *DNSService) ConfigureDNSMode(mode DNSMode) error // Direct ISP / Proxy / Forward / Server
func (ds *DNSService) AddLocalZone(zone LocalDNSZone) error
func (ds *DNSService) ConfigureDoT(servers []DoTServer) error
func (ds *DNSService) ConfigureDoH(servers []DoHServer) error
func (ds *DNSService) SetupGeoDNSRouting(rules []GeoDNSRule) error
```

DNS сервис должен уметь генерировать конфигурации для различных DNS серверов (unbound, dnsmasq, bind9) и обеспечивать переключение между ними без потери настроек.

### DHCPService (dhcp_service.go)

DHCP сервис управляет автоматическим назначением IP адресов в локальной сети и тесно интегрирован с DNS сервисом для обеспечения корректного разрешения имен.

```go
type DHCPService struct {
    dnsService *DNSService
    db         *sql.DB
}

func (ds *DHCPService) ConfigureDHCPPool(pool DHCPPool) error
func (ds *DHCPService) AddStaticLease(lease StaticLease) error
func (ds *DHCPService) SetupDHCPRelay(relayConfig DHCPRelayConfig) error
func (ds *DHCPService) GetActiveLeases() ([]DHCPLease, error)
```

### RoutingService (routing_service.go)

Маршрутизация требует глубокого понимания сетевых принципов и должна координироваться с firewall настройками для обеспечения безопасности.

```go
type RoutingService struct {
    networkService  *NetworkService
    firewallService *FirewallService
    db              *sql.DB
}

func (rs *RoutingService) AddStaticRoute(route StaticRoute) error
func (rs *RoutingService) ConfigureUPnP(config UPnPConfig) error
func (rs *RoutingService) SetupTrafficShaping(rules []TrafficShapingRule) error
func (rs *RoutingService) GetRoutingTable() ([]Route, error)
```

### FirewallService (firewall_service.go)

Межсетевой экран является критически важным компонентом безопасности и должен поддерживать как традиционные iptables, так и современные nftables.

```go
type FirewallService struct {
    systemService *SystemService
    db            *sql.DB
    firewallType  FirewallType // iptables or nftables
}

func (fs *FirewallService) AddRule(rule FirewallRule) error
func (fs *FirewallService) CreateChain(chain FirewallChain) error
func (fs *FirewallService) ConfigurePortForwarding(config PortForwardConfig) error
func (fs *FirewallService) SwitchFirewallBackend(backend FirewallType) error
func (fs *FirewallService) BackupRules() ([]byte, error)
func (fs *FirewallService) RestoreRules(backup []byte) error
```

### SystemService (system_service.go)

Системный сервис предоставляет интерфейс для управления системными процессами и параметрами ядра.

```go
type SystemService struct {
    serviceConfigurator *configurators.ServicesConfigurator
    kernelConfigurator  *configurators.KernelConfigurator
    db                  *sql.DB
}

func (ss *SystemService) RestartService(serviceName string) error
func (ss *SystemService) SetKernelParameter(param, value string) error
func (ss *SystemService) GetSystemStatus() (SystemStatus, error)
func (ss *SystemService) BackupConfiguration() error
func (ss *SystemService) RestoreConfiguration(backupPath string) error
```

### AuthService (auth_service.go)

Сервис аутентификации обеспечивает безопасный доступ к системе управления.

```go
type AuthService struct {
    db            *sql.DB
    sessionStore  map[string]Session
    passwordHasher PasswordHasher
}

func (as *AuthService) Authenticate(username, password string) (Session, error)
func (as *AuthService) ValidateSession(sessionID string) (bool, error)
func (as *AuthService) ChangePassword(username, oldPassword, newPassword string) error
func (as *AuthService) HashPassword(password string) (string, error)
```

## Принципы взаимодействия между сервисами

Сервисы не должны напрямую обращаться к другим сервисам без четкого понимания зависимостей. Например, когда WAN сервис настраивает новое подключение, он должен координировать свои действия с Routing и Firewall сервисами через определенные интерфейсы.

## Обработка ошибок и транзакционность

Каждый сервис должен обеспечивать атомарность операций. Если настройка не может быть применена полностью, все изменения должны быть откачены. Это особенно важно для сетевых настроек, где частичная конфигурация может привести к потере связи с устройством.

## WebSocket интеграция

Все сервисы должны поддерживать отправку уведомлений через WebSocket для обновления пользовательского интерфейса в реальном времени. Это позволяет пользователям видеть статус применения изменений и результаты операций без необходимости обновления страницы.

Такая архитектура обеспечивает надежную, масштабируемую и поддерживаемую систему управления сетевым оборудованием с четким разделением ответственности между компонентами.

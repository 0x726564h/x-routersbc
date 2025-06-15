# Utils - Детальное Описание

`utils` представляет собой фундаментальный компонент системы управления маршрутизатором, который служит мостом между высокоуровневой бизнес-логикой и низкоуровневыми системными операциями. Думайте о нем как о наборе специализированных инструментов, каждый из которых знает, как правильно взаимодействовать с определенными аспектами операционной системы.

## Архитектурная Роль

Модуль `utils` решает критически важную проблему в системном программировании - абстракцию различий между операционными системами. Когда ваш Go-код хочет настроить сетевой интерфейс, он не должен знать, использует ли система Netplan (Ubuntu), файл `/etc/network/interfaces` (Debian), или другой механизм конфигурации. Именно здесь вступают в игру конфигураторы.

## Структура Директорий

```
utils/
├── configurators/           # Основные конфигураторы системы
│   ├── net/                # Сетевые конфигураторы
│   │   ├── netplan.go      # Управление Netplan (Ubuntu/современный подход)
│   │   ├── interfaces.go   # Классический /etc/network/interfaces
│   │   ├── wan.go          # Конфигурация WAN соединений
│   │   ├── dns.go          # Управление DNS резолверами
│   │   ├── dhcp.go         # DHCP сервер конфигурация
│   │   ├── routing.go      # Статическая маршрутизация
│   │   └── firewall.go     # Правила файрвола (iptables/nftables)
│   └── sys/                # Системные конфигураторы
│       ├── kernel.go       # Параметры ядра (/proc/sys, sysctl)
│       └── services.go     # Управление systemd сервисами
├── helpers/                # Вспомогательные функции
│   ├── file.go            # Работа с файлами (атомарная запись, бэкапы)
│   ├── command.go         # Безопасное выполнение системных команд
│   ├── validation.go      # Валидация конфигураций
│   └── network.go         # Сетевые утилиты (проверка IP, интерфейсов)
└── types/                 # Общие типы данных
    ├── network.go         # Структуры для сетевых конфигураций
    ├── system.go          # Системные типы
    └── errors.go          # Специализированные ошибки
```

## Детальный Разбор Конфигураторов

### Сетевые Конфигураторы (`configurators/net/`)

#### `netplan.go` - Современный Подход к Сетевой Конфигурации

Netplan представляет собой декларативный способ конфигурации сети в современных Ubuntu системах. Этот конфигуратор отвечает за генерацию YAML файлов, которые Netplan затем преобразует в конфигурации для NetworkManager или systemd-networkd.

```go
type NetplanConfigurator struct {
    configPath    string           // Обычно /etc/netplan/01-network-manager-all.yaml
    backupPath    string           // Путь для бэкапов конфигураций
    renderer      string           // NetworkManager или networkd
    logger        *log.Logger      // Логирование операций
}

// Основные методы конфигуратора
func (nc *NetplanConfigurator) GenerateConfig(interfaces []NetworkInterface) error
func (nc *NetplanConfigurator) ApplyConfig() error  // Выполняет netplan apply
func (nc *NetplanConfigurator) ValidateConfig() error
func (nc *NetplanConfigurator) CreateBackup() error
func (nc *NetplanConfigurator) RestoreFromBackup(backupID string) error
```

Ключевая особенность этого конфигуратора заключается в том, что он должен понимать различные типы сетевых интерфейсов и корректно их описывать в YAML формате. Например, для настройки VLAN интерфейса, конфигуратор создает такую структуру:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: false
  vlans:
    vlan100:
      id: 100
      link: eth0
      addresses: [192.168.100.1/24]
      gateway4: 192.168.100.254
```

#### `interfaces.go` - Классический Debian Подход

Этот конфигуратор работает с традиционным файлом `/etc/network/interfaces`, который до сих пор используется во многих системах Debian и старых версиях Ubuntu. Он должен понимать синтаксис этого файла и генерировать корректные конфигурации.

```go
type InterfacesConfigurator struct {
    configFile     string    // /etc/network/interfaces
    includeDir     string    // /etc/network/interfaces.d/
    currentConfig  []InterfaceConfig
}

// Пример генерируемой конфигурации для статического IP:
// auto eth0
// iface eth0 inet static
//     address 192.168.1.100
//     netmask 255.255.255.0
//     gateway 192.168.1.1
//     dns-nameservers 8.8.8.8 8.8.4.4
```

#### `wan.go` - Управление WAN Соединениями

WAN конфигуратор представляет собой более сложный компонент, поскольку он должен обрабатывать различные типы интернет-соединений: статический IP, DHCP, PPPoE, PPTP, и даже беспроводные соединения когда WiFi адаптер используется как WAN интерфейс.

```go
type WANConfigurator struct {
    primaryWAN    WANInterface
    secondaryWAN  []WANInterface    // Для Multi-WAN конфигураций
    loadBalancer  LoadBalancingConfig
    failover      FailoverConfig
}

type WANInterface struct {
    InterfaceName string           // eth0, wlan0, ppp0
    ConnectionType string          // static, dhcp, pppoe, pptp
    StaticConfig  *StaticIPConfig  // Для статических соединений
    PPPConfig     *PPPConfig       // Для PPPoE/PPTP
    Wireless      *WirelessConfig  // Для WiFi WAN
    Priority      int              // Для Multi-WAN
    Weight        int              // Для балансировки нагрузки
}
```

Особенность WAN конфигуратора в том, что он должен координировать свою работу с другими компонентами системы. Например, при настройке Multi-WAN он должен создать соответствующие правила маршрутизации и настроить файрвол для корректного распределения трафика.

#### `dns.go` - Сложная DNS Конфигурация

DNS конфигуратор - это один из самых сложных компонентов, поскольку современные требования к DNS включают поддержку DNS-over-TLS (DoT), DNS-over-HTTPS (DoH), локальных зон, географической маршрутизации DNS запросов, и кеширования.

```go
type DNSConfig struct {
    Mode          string                 // direct, proxy, forward, server
    Resolvers     []DNSResolver         // Внешние резолверы
    LocalZones    []LocalDNSZone        // Локальные DNS зоны
    GeoRouting    []GeoRoutingRule      // Географическая маршрутизация
    Cache         CacheConfig           // Настройки кеширования
    Security      DNSSecurityConfig     // DoT, DoH, DNSSEC
}

type DNSResolver struct {
    Address   string    // IP адрес или URL
    Protocol  string    // udp, tcp, tls, https
    Port      int       // Порт резолвера
    Priority  int       // Приоритет использования
    Timeout   time.Duration
}

type LocalDNSZone struct {
    Zone     string              // .local, .lan
    Records  []DNSRecord        // A, AAAA, CNAME записи
}

type GeoRoutingRule struct {
    Domain    string    // target.com или wildcard *.target.com
    GeoZone   string    // ru, us, eu
    Action    string    // direct, block, resolver
    Target    string    // IP резолвера или "block"
}
```

DNS конфигуратор должен генерировать конфигурации для различных DNS серверов в зависимости от выбранного режима работы. Например, для режима "server" он настраивает полноценный DNS сервер (обычно dnsmasq или bind9), а для режима "proxy" - только перенаправление запросов.

#### `dhcp.go` - DHCP Сервер

DHCP конфигуратор управляет настройками DHCP сервера, включая пулы адресов, статические резервации, и дополнительные опции DHCP.

```go
type DHCPConfig struct {
    Mode           string              // server, relay, disabled
    Interface      string              // Интерфейс для прослушивания
    AddressPools   []AddressPool      // Пулы IP адресов
    StaticLeases   []StaticLease      // Статические резервации
    DHCPOptions    []DHCPOption       // Дополнительные DHCP опции
    LeaseTime      time.Duration      // Время аренды адресов
}

type AddressPool struct {
    StartIP    net.IP
    EndIP      net.IP
    Netmask    net.IPMask
    Gateway    net.IP
    DNSServers []net.IP
}
```

#### `routing.go` - Управление Маршрутизацией

Этот конфигуратор отвечает за статические маршруты и более сложные сценарии маршрутизации, включая policy-based routing для Multi-WAN конфигураций.

```go
type RoutingConfigurator struct {
    staticRoutes    []StaticRoute
    routingTables   []RoutingTable    // Для policy-based routing
    routingRules    []RoutingRule     // ip rule команды
}

type StaticRoute struct {
    Destination  *net.IPNet    // Сеть назначения
    Gateway      net.IP        // Шлюз
    Interface    string        // Исходящий интерфейс
    Metric       int           // Метрика маршрута
    Table        string        // Таблица маршрутизации (main, local, или custom)
}
```

#### `firewall.go` - Управление Файрволом

Файрвол конфигуратор должен поддерживать как классический iptables, так и современный nftables, предоставляя единый интерфейс для управления правилами безопасности.

```go
type FirewallConfigurator struct {
    backend      string                 // iptables или nftables
    chains       map[string][]Rule     // Цепочки и правила
    natRules     []NATRule             // NAT правила
    portForwards []PortForwardRule     // Проброс портов
}

type Rule struct {
    Chain       string              // INPUT, OUTPUT, FORWARD, custom
    Action      string              // ACCEPT, DROP, REJECT, LOG
    Protocol    string              // tcp, udp, icmp, all
    Source      *net.IPNet         // Источник
    Destination *net.IPNet         // Назначение
    Ports       []PortRange        // Порты
    Interface   string             // Интерфейс
    State       []string           // connection tracking states
}
```

### Системные Конфигураторы (`configurators/sys/`)

#### `kernel.go` - Параметры Ядра

Этот конфигуратор управляет параметрами ядра Linux через sysctl интерфейс, что критически важно для правильной работы маршрутизатора.

```go
type KernelConfigurator struct {
    sysctlFile    string    // /etc/sysctl.conf или /etc/sysctl.d/99-router.conf
    currentParams map[string]string
}

// Важные параметры для маршрутизатора:
var routerKernelParams = map[string]string{
    "net.ipv4.ip_forward":                "1",      // Включить маршрутизацию IPv4
    "net.ipv6.conf.all.forwarding":      "1",      // Включить маршрутизацию IPv6
    "net.ipv4.conf.all.send_redirects":  "0",      // Отключить ICMP redirects
    "net.ipv4.conf.all.accept_redirects": "0",     // Не принимать redirects
    "net.ipv4.icmp_ignore_bogus_error_responses": "1",
    "net.ipv4.tcp_syncookies":            "1",      // Защита от SYN flood
}
```

#### `services.go` - Управление Сервисами

Сервисный конфигуратор предоставляет единый интерфейс для управления systemd сервисами, что необходимо для перезапуска сетевых сервисов после изменения конфигураций.

```go
type ServiceManager struct {
    systemctl string    // Путь к systemctl
    logger    *log.Logger
}

func (sm *ServiceManager) StartService(serviceName string) error
func (sm *ServiceManager) StopService(serviceName string) error  
func (sm *ServiceManager) RestartService(serviceName string) error
func (sm *ServiceManager) ReloadService(serviceName string) error
func (sm *ServiceManager) EnableService(serviceName string) error
func (sm *ServiceManager) DisableService(serviceName string) error
func (sm *ServiceManager) GetServiceStatus(serviceName string) (ServiceStatus, error)
```

## Вспомогательные Модули

### `helpers/file.go` - Безопасная Работа с Файлами

Этот модуль критически важен для надежности системы, поскольку неправильная запись конфигурационных файлов может привести к потере сетевого соединения.

```go
// Атомарная запись файла - сначала записывается во временный файл,
// затем переименовывается, что гарантирует целостность
func AtomicWriteFile(filename string, data []byte, perm os.FileMode) error

// Создание бэкапа с timestamp
func CreateBackup(filename string) (string, error)

// Восстановление из бэкапа
func RestoreFromBackup(filename, backupPath string) error

// Валидация синтаксиса конфигурационных файлов
func ValidateConfigFile(filename string, validator ConfigValidator) error
```

### `helpers/command.go` - Безопасное Выполнение Команд

Системные команды должны выполняться безопасно с правильной обработкой ошибок и логированием.

```go
type CommandExecutor struct {
    timeout time.Duration
    logger  *log.Logger
}

func (ce *CommandExecutor) Execute(command string, args ...string) (*CommandResult, error)
func (ce *CommandExecutor) ExecuteWithInput(command string, input string, args ...string) (*CommandResult, error)
func (ce *CommandExecutor) ExecuteAsync(command string, args ...string) (*AsyncCommand, error)
```

### `helpers/validation.go` - Валидация Конфигураций

Валидация должна происходить на нескольких уровнях - синтаксическом (корректность IP адресов, диапазонов портов) и логическом (отсутствие конфликтов в конфигурации).

```go
func ValidateIPAddress(ip string) error
func ValidateIPRange(ipRange string) error
func ValidatePortRange(portRange string) error
func ValidateInterfaceName(interfaceName string) error
func ValidateNetworkConfig(config *NetworkConfig) []ValidationError
```

## Паттерны Проектирования и Лучшие Практики

### Интерфейсы для Абстракции

Каждый конфигуратор должен реализовывать общий интерфейс, что позволяет легко добавлять поддержку новых операционных систем:

```go
type Configurator interface {
    GenerateConfig(config interface{}) error
    ApplyConfig() error
    ValidateConfig() error
    CreateBackup() (string, error)
    RestoreFromBackup(backupID string) error
}
```

### Обработка Ошибок

Система должна различать типы ошибок и предоставлять понятные сообщения:

```go
type ConfigurationError struct {
    Component string    // Какой компонент вызвал ошибку
    Operation string    // Какая операция не удалась
    Cause     error     // Первичная причина
    Suggestion string   // Предложение по исправлению
}
```

### Логирование и Мониторинг

Все операции должны логироваться для последующего анализа и отладки:

```go
type ConfigurationLogger struct {
    systemLogger *log.Logger
    auditLogger  *log.Logger    // Для аудита изменений
}
```

Модуль `utils` служит фундаментом для всего функционала системы управления маршрутизатором, обеспечивая надежное и безопасное взаимодействие с операционной системой. Его правильная реализация критически важна для стабильности всей системы.

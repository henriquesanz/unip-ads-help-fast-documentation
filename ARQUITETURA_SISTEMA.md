# Arquitetura do Sistema HelpFast

## 📋 **Visão Geral**

O HelpFast é um sistema multiplataforma de gerenciamento de chamados que utiliza uma arquitetura moderna baseada em microsserviços, com componentes distribuídos entre Azure Cloud e Oracle Cloud para máxima eficiência e escalabilidade.

## 🏗️ **Arquitetura Geral**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   HelpFast      │    │   HelpFast      │    │   HelpFast      │
│   Web App       │    │   Mobile App    │    │   Desktop App   │
│ (ASP.NET Core)  │    │   (Flutter)     │    │ (WinForms.NET)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Load Balancer)│
                    └─────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   HelpFast      │    │   Java API      │    │   Azure SQL     │
│   WebApp        │    │   (Oracle       │    │   Database      │
│ (Oracle Cloud)  │    │    Cloud)       │    │ (Free Tier)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## ☁️ **Infraestrutura Cloud**

### **Azure Cloud (Free Tier)**
- **Azure SQL Database:** Banco de dados principal
- **Azure Storage:** Armazenamento de arquivos
- **Azure App Service:** Hospedagem web (futuro)
- **Azure Monitor:** Monitoramento e logs

### **Oracle Cloud (Free Tier)**
- **HelpFast WebApp:** Aplicação web principal
- **Java API:** API de integração com IA
- **Oracle Database:** Banco de dados secundário (futuro)
- **Oracle Container Engine:** Deploy de containers

## 🗄️ **Banco de Dados**

### **Azure SQL Database (Principal)**
```sql
-- Tabelas Principais
CREATE TABLE Usuarios (
    Id INT PRIMARY KEY IDENTITY(1,1),
    Nome NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100) UNIQUE NOT NULL,
    Telefone NVARCHAR(15) NOT NULL,
    Senha NVARCHAR(255) NOT NULL,
    TipoUsuario INT NOT NULL,
    DataCriacao DATETIME2 NOT NULL,
    UltimoLogin DATETIME2,
    Ativo BIT NOT NULL DEFAULT 1,
    CriadoPorId INT NULL,
    FOREIGN KEY (CriadoPorId) REFERENCES Usuarios(Id)
);

CREATE TABLE Chamados (
    Id INT PRIMARY KEY IDENTITY(1,1),
    Titulo NVARCHAR(100) NOT NULL,
    Descricao NVARCHAR(1000) NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'Aberto',
    Prioridade NVARCHAR(50) NOT NULL DEFAULT 'Media',
    DataCriacao DATETIME2 NOT NULL,
    DataAtualizacao DATETIME2,
    DataResolucao DATETIME2,
    UsuarioId INT NOT NULL,
    TecnicoId INT NULL,
    FOREIGN KEY (UsuarioId) REFERENCES Usuarios(Id),
    FOREIGN KEY (TecnicoId) REFERENCES Usuarios(Id)
);

CREATE TABLE Notificacoes (
    Id INT PRIMARY KEY IDENTITY(1,1),
    UsuarioId INT NOT NULL,
    Tipo NVARCHAR(50) NOT NULL,
    Titulo NVARCHAR(200) NOT NULL,
    Mensagem NVARCHAR(500) NOT NULL,
    Lida BIT NOT NULL DEFAULT 0,
    DataEnvio DATETIME2 NOT NULL,
    DataLeitura DATETIME2,
    FOREIGN KEY (UsuarioId) REFERENCES Usuarios(Id)
);
```

### **Estrutura de Dados**
- **Usuarios:** Gestão de usuários e perfis
- **Chamados:** Tickets de suporte
- **Comentarios:** Histórico de conversas
- **Notificacoes:** Sistema de notificações
- **FAQ:** Perguntas frequentes
- **Logs:** Auditoria do sistema

## 🔧 **Componentes Técnicos**

### **1. HelpFast WebApp (ASP.NET Core)**
```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(connectionString));
    
    services.AddScoped<IUsuarioService, UsuarioService>();
    services.AddScoped<IChamadoService, ChamadoService>();
    services.AddScoped<INotificacaoService, NotificacaoService>();
    
    services.AddControllers();
    services.AddSwaggerGen();
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseRouting();
app.MapControllers();
app.Run();
```

### **2. Java API (Oracle Cloud)**
```java
@SpringBootApplication
@RestController
@RequestMapping("/api")
public class HelpFastAIApplication {
    
    @Autowired
    private OpenAIService openAIService;
    
    @PostMapping("/chat")
    public ResponseEntity<AIResponse> chat(@RequestBody ChatRequest request) {
        // Processamento de IA
        String response = openAIService.processChat(request.getMessage());
        return ResponseEntity.ok(new AIResponse(response));
    }
}
```

### **3. Desktop App (WinForms.NET)**
```csharp
// Program.cs
static void Main()
{
    ApplicationConfiguration.Initialize();
    
    var host = CreateHostBuilder().Build();
    
    using (var scope = host.Services.CreateScope())
    {
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        context.Database.EnsureCreated();
    }
    
    var loginForm = host.Services.GetRequiredService<LoginForm>();
    Application.Run(loginForm);
}
```

## 🔌 **Integrações Externas**

### **OpenAI API**
```json
{
  "openai": {
    "apiKey": "sk-...",
    "model": "gpt-4",
    "temperature": 0.7,
    "maxTokens": 1000,
    "timeout": 30
  }
}
```

### **Serviços de Email**
```json
{
  "email": {
    "smtp": {
      "host": "smtp.gmail.com",
      "port": 587,
      "username": "helpfast@company.com",
      "password": "app-password"
    }
  }
}
```

## 🔐 **Segurança e Autenticação**

### **Autenticação JWT**
```csharp
public class JwtService
{
    public string GenerateToken(Usuario usuario)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _issuer,
            audience: _audience,
            claims: GetClaims(usuario),
            expires: DateTime.Now.AddHours(8),
            signingCredentials: creds
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### **Criptografia de Senhas**
```csharp
public class PasswordService
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password);
    }
    
    public bool VerifyPassword(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}
```

## 📊 **Monitoramento e Logs**

### **Estrutura de Logs**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "service": "HelpFast.WebApp",
  "message": "User login successful",
  "userId": 123,
  "ipAddress": "192.168.1.100",
  "userAgent": "Mozilla/5.0...",
  "correlationId": "abc-123-def"
}
```

### **Métricas de Performance**
- **Tempo de Resposta:** < 2 segundos
- **Disponibilidade:** 99.9% uptime
- **Throughput:** 100 requests/second
- **Error Rate:** < 0.1%

## 🚀 **Deploy e DevOps**

### **Pipeline CI/CD**
```yaml
# .github/workflows/deploy.yml
name: Deploy HelpFast
on:
  push:
    branches: [main]

jobs:
  deploy-web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup .NET
        uses: actions/setup-dotnet@v1
      - name: Build
        run: dotnet build
      - name: Deploy to Oracle Cloud
        run: docker build -t helpfast-web .
```

### **Containerização**
```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .
RUN dotnet build
EXPOSE 80
ENTRYPOINT ["dotnet", "run"]
```

## 📱 **Multiplataforma**

### **Web App (ASP.NET Core)**
- **Frontend:** Razor Pages + JavaScript
- **Backend:** ASP.NET Core Web API
- **Database:** Entity Framework Core
- **Deploy:** Oracle Cloud

### **Desktop App (WinForms.NET)**
- **Framework:** .NET 8.0 WinForms
- **Database:** Entity Framework Core
- **Deploy:** Executável Windows

### **Mobile App (Futuro - Flutter)**
- **Framework:** Flutter
- **Backend:** REST API
- **Database:** SQLite local + API sync

## 🔄 **Comunicação entre Componentes**

### **API REST**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ChamadosController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<List<Chamado>>> GetChamados()
    {
        var chamados = await _chamadoService.ListarTodosChamadosAsync();
        return Ok(chamados);
    }
    
    [HttpPost]
    public async Task<ActionResult<Chamado>> CriarChamado([FromBody] Chamado chamado)
    {
        var novoChamado = await _chamadoService.CriarChamadoAsync(chamado);
        return CreatedAtAction(nameof(GetChamado), new { id = novoChamado.Id }, novoChamado);
    }
}
```

### **SignalR (Tempo Real)**
```csharp
public class NotificationHub : Hub
{
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
    
    public async Task SendNotification(string userId, string message)
    {
        await Clients.Group($"user_{userId}").SendAsync("ReceiveNotification", message);
    }
}
```

## 📈 **Escalabilidade**

### **Horizontal Scaling**
- **Load Balancer:** Distribuição de carga
- **Database Sharding:** Particionamento de dados
- **Caching:** Redis para performance
- **CDN:** Distribuição de conteúdo estático

### **Vertical Scaling**
- **Resource Optimization:** Otimização de recursos
- **Database Indexing:** Índices otimizados
- **Query Optimization:** Consultas eficientes
- **Memory Management:** Gerenciamento de memória

## 🛡️ **Conformidade e Segurança**

### **LGPD Compliance**
- **Data Minimization:** Dados mínimos necessários
- **Consent Management:** Controle de consentimento
- **Data Retention:** Política de retenção
- **Right to be Forgotten:** Direito ao esquecimento

### **Security Best Practices**
- **HTTPS Everywhere:** Comunicação segura
- **Input Validation:** Validação de entrada
- **SQL Injection Prevention:** Prevenção de ataques
- **Rate Limiting:** Controle de taxa de requisições

## 📝 **Notas Importantes**

- **Cloud Agnostic:** Preparado para migração entre clouds
- **Microservices Ready:** Arquitetura preparada para microsserviços
- **API First:** Todas as funcionalidades expostas via API
- **Event Driven:** Comunicação baseada em eventos
- **Observability:** Monitoramento completo do sistema


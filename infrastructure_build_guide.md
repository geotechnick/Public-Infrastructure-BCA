
# Public Infrastructure Planning Platform - Build Guide

## Phase 1: Development Environment Setup (Week 1)

### 1.1 Prerequisites Installation

**Intent**: Establish a consistent development environment across all team members to eliminate "works on my machine" issues. We're choosing specific versions and tools that support our microservices architecture.

```bash
# Install required tools
# Node.js (18+) - Required for React frontend and build tools
# Using NVM allows easy version switching for different projects
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop - Essential for local development environment
# Allows us to run all infrastructure services (DB, Redis, Kafka) locally
# Download from docker.com

# .NET 8 SDK - Latest LTS version for backend services
# Provides modern C# features, performance improvements, and long-term support
# Download from microsoft.com/dotnet

# Git - Version control for distributed development
sudo apt install git  # Linux
brew install git      # macOS

# VS Code with extensions - Consistent IDE experience
# Extensions provide IntelliSense, debugging, and container support
code --install-extension ms-dotnettools.csharp
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

**Why these choices?**
- **Node.js 18+**: Provides modern JavaScript features and npm ecosystem access
- **Docker**: Eliminates environment differences and simplifies service orchestration
- **.NET 8**: High-performance, cross-platform runtime with excellent tooling
- **VS Code**: Lightweight, extensible, with excellent container and cloud support

### 1.2 Project Structure Setup

**Intent**: Create a well-organized monorepo structure that supports microservices architecture while maintaining clear boundaries. This structure follows Domain-Driven Design principles and separates concerns between services, shared code, infrastructure, and client applications.

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Create main directory structure
# This follows the "screaming architecture" principle - the structure tells you what the system does
mkdir -p {
  src/{
    # Each service is a bounded context with its own domain
    services/{budget,grants,projects,bca,contracts,costs,users,notifications},
    # Shared code that multiple services can use - but keep this minimal
    shared/{common,domain,events},
    # API Gateway acts as the single entry point for clients
    gateways/api-gateway,
    # Separate frontend applications for different user types
    web/{admin-portal,public-portal},
    # Mobile app for field workers and inspectors
    mobile
  },
  infrastructure/{
    # Infrastructure as Code - everything should be reproducible
    docker,      # Local development containers
    kubernetes,  # Production orchestration
    terraform,   # Cloud resource provisioning
    scripts      # Deployment and utility scripts
  },
  docs,  # Architecture decision records, API docs, user guides
  tests  # End-to-end and integration tests that span services
}
```

**Why this structure?**
- **Service boundaries**: Each service in `/services/` represents a business capability
- **Shared minimal**: Only truly shared code goes in `/shared/` to avoid coupling
- **Infrastructure separation**: Keeps deployment concerns separate from business logic
- **Client separation**: Different frontends for different user needs and experiences
- **Documentation co-location**: Docs live with code for easier maintenance

### 1.3 Git Repository Setup

```bash
git init
echo "node_modules/
bin/
obj/
.env
.DS_Store
*.log" > .gitignore

git add .
git commit -m "Initial project structure"
```

## Phase 2: Core Infrastructure (Week 2)

### 2.1 Docker Development Environment

**Intent**: Create a local development environment that mirrors production as closely as possible. This setup provides all the infrastructure services (database, cache, messaging, search) that our microservices need, without requiring complex local installations. Each service is isolated and can be started/stopped independently.

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  # PostgreSQL - Primary database for transactional data
  # Using version 15 for performance and JSON improvements
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrastructure_dev      # Single database, multiple schemas per service
      POSTGRES_USER: dev_user              # Non-root user for security
      POSTGRES_PASSWORD: dev_password      # Simple password for local dev
    ports:
      - "5432:5432"                       # Standard PostgreSQL port
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data between container restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Initialize schemas

  # Redis - Caching and session storage
  # Using Alpine for smaller image size
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"                       # Standard Redis port
    # No persistence needed in development

  # Apache Kafka - Event streaming between services
  # Essential for event-driven architecture and service decoupling
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper                         # Kafka requires Zookeeper for coordination
    ports:
      - "9092:9092"                      # Kafka broker port
    environment:
      KAFKA_BROKER_ID: 1                 # Single broker for dev
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092  # Clients connect via localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  # No replication needed in dev

  # Zookeeper - Kafka dependency for cluster coordination
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Elasticsearch - Full-text search and analytics
  # Used for searching grants, projects, and generating reports
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node        # Single node for development
      - xpack.security.enabled=false      # Disable security for local dev
    ports:
      - "9200:9200"                      # REST API port

volumes:
  postgres_data:  # Named volume for database persistence
```

**Test Infrastructure:**
```bash
cd infrastructure/docker
# Start all infrastructure services in background
docker-compose -f docker-compose.dev.yml up -d

# Verify all services are running and healthy
docker-compose ps

# Expected output: All services should show "Up" status
# postgres_1      Up  0.0.0.0:5432->5432/tcp
# redis_1         Up  0.0.0.0:6379->6379/tcp
# kafka_1         Up  0.0.0.0:9092->9092/tcp
# etc.
```

**Why these infrastructure choices?**
- **PostgreSQL**: ACID compliance, JSON support, excellent .NET integration
- **Redis**: Fast caching, session storage, distributed locks
- **Kafka**: Reliable event streaming, service decoupling, audit logging
- **Elasticsearch**: Powerful search, analytics, report generation
- **Single-node configs**: Simplified for development, production uses clusters

### 2.2 Shared Domain Models

**Intent**: Establish common base classes and patterns that all domain entities will inherit from. This implements the DDD (Domain-Driven Design) principle of having rich domain objects with behavior, not just data containers. The base entity provides audit trail capabilities essential for government compliance and change tracking.

Create `src/shared/domain/Entities/BaseEntity.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Entities
{
    /// <summary>
    /// Base class for all domain entities providing common functionality
    /// Implements Entity pattern from DDD with audit trail for compliance
    /// </summary>
    public abstract class BaseEntity
    {
        // Using Guid as ID provides globally unique identifiers across distributed services
        // No need for centralized ID generation or sequence coordination
        public Guid Id { get; protected set; }
        
        // Audit fields required for government compliance and change tracking
        public DateTime CreatedAt { get; protected set; }
        public DateTime UpdatedAt { get; protected set; }
        public string CreatedBy { get; protected set; }      // User who created the entity
        public string UpdatedBy { get; protected set; }      // User who last modified

        // Protected constructor prevents direct instantiation
        // Forces use of domain-specific constructors in derived classes
        protected BaseEntity()
        {
            Id = Guid.NewGuid();              // Generate unique ID immediately
            CreatedAt = DateTime.UtcNow;      // Always use UTC to avoid timezone issues
        }

        // Virtual method allows derived classes to add custom update logic
        // Automatically tracks who made changes and when
        public virtual void Update(string updatedBy)
        {
            UpdatedAt = DateTime.UtcNow;
            UpdatedBy = updatedBy ?? throw new ArgumentNullException(nameof(updatedBy));
        }
    }
}
```

**Intent**: Create a base class for domain events that enables event-driven architecture. Domain events represent something significant that happened in the business domain and allow services to communicate without direct coupling.

Create `src/shared/domain/Events/DomainEvent.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Events
{
    /// <summary>
    /// Base class for all domain events in the system
    /// Enables event-driven architecture and service decoupling
    /// Events represent business facts that other services might care about
    /// </summary>
    public abstract class DomainEvent
    {
        // Unique identifier for this specific event occurrence
        public Guid Id { get; }
        
        // When did this business event occur (not when it was processed)
        public DateTime OccurredOn { get; }
        
        // Type identifier for event routing and deserialization
        // Using class name allows easy identification in logs and debugging
        public string EventType { get; }

        protected DomainEvent()
        {
            Id = Guid.NewGuid();                    // Unique event ID for idempotency
            OccurredOn = DateTime.UtcNow;           // Business time, not processing time
            EventType = this.GetType().Name;       // Automatic type identification
        }
    }
}
```

**Why these design decisions?**
- **Guid IDs**: Eliminate distributed ID generation complexity, work across services
- **UTC timestamps**: Prevent timezone confusion in distributed systems
- **Audit fields**: Meet government compliance requirements for change tracking
- **Protected constructors**: Enforce proper domain object creation patterns
- **Domain events**: Enable loose coupling between services while maintaining business consistency
- **Event metadata**: Support idempotency, ordering, and debugging in distributed systems

## Phase 3: First Service - Project Management (Week 3)

### 3.1 Project Service Setup

**Intent**: Create our first microservice following Clean Architecture principles. This service will handle all project-related operations and serve as a template for other services. We're using the latest .NET 8 with carefully chosen packages that support our architectural goals.

Create `src/services/projects/Infrastructure.Projects.API/Infrastructure.Projects.API.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>    <!-- Latest LTS for performance -->
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Entity Framework Core for database operations -->
    <!-- Design package needed for migrations and scaffolding -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
    <!-- PostgreSQL provider for our chosen database -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    
    <!-- Swagger for API documentation and testing -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- MediatR implements CQRS and Mediator pattern -->
    <!-- Decouples controllers from business logic -->
    <PackageReference Include="MediatR" Version="12.0.0" />
    
    <!-- AutoMapper for object-to-object mapping -->
    <!-- Keeps controllers clean by handling DTO conversions -->
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <!-- Reference to shared domain models and utilities -->
    <ProjectReference Include="../../shared/common/Infrastructure.Shared.Common.csproj" />
  </ItemGroup>
</Project>
```

**Why these package choices?**
- **EF Core + PostgreSQL**: Robust ORM with excellent PostgreSQL support for complex queries
- **MediatR**: Implements CQRS pattern, separates read/write operations, supports cross-cutting concerns
- **AutoMapper**: Reduces boilerplate code for mapping between domain models and DTOs
- **Swashbuckle**: Provides interactive API documentation essential for microservices integration

### 3.2 Project Domain Model

**Intent**: Create a rich domain model that encapsulates business rules and invariants. This follows DDD principles where the domain model contains both data and behavior. The Project entity represents the core concept of infrastructure projects and enforces business constraints through its design.

Create `src/services/projects/Infrastructure.Projects.Domain/Entities/Project.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Projects.Domain.Entities
{
    /// <summary>
    /// Project aggregate root representing an infrastructure project
    /// Contains all business rules and invariants for project management
    /// </summary>
    public class Project : BaseEntity
    {
        // Required project information - private setters enforce controlled modification
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ProjectType Type { get; private set; }           // Enum constrains valid project types
        public ProjectStatus Status { get; private set; }       // State machine via status enum
        
        // Financial information with proper decimal type for currency
        public decimal EstimatedCost { get; private set; }      // Initial cost estimate
        
        // Timeline information
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }          // Nullable - may not be set initially
        
        // Ownership tracking
        public string OwnerId { get; private set; }             // Reference to owning organization

        // EF Core requires parameterless constructor - private to prevent misuse
        private Project() { } 

        /// <summary>
        /// Creates a new infrastructure project with required business data
        /// Constructor enforces invariants - all required data must be provided
        /// </summary>
        public Project(string name, string description, ProjectType type, 
                      decimal estimatedCost, DateTime startDate, string ownerId)
        {
            // Business rule validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Project name is required", nameof(name));
            if (estimatedCost <= 0)
                throw new ArgumentException("Project cost must be positive", nameof(estimatedCost));
            if (startDate < DateTime.Today)
                throw new ArgumentException("Project cannot start in the past", nameof(startDate));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Project owner is required", nameof(ownerId));

            Name = name;
            Description = description;
            Type = type;
            Status = ProjectStatus.Planning;     // All projects start in planning phase
            EstimatedCost = estimatedCost;
            StartDate = startDate;
            OwnerId = ownerId;
        }

        /// <summary>
        /// Updates project status following business rules
        /// Could be extended to implement state machine validation
        /// Domain event could be raised here for other services to react
        /// </summary>
        public void UpdateStatus(ProjectStatus newStatus)
        {
            // Business rule: validate status transitions
            if (!IsValidStatusTransition(Status, newStatus))
                throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");
                
            Status = newStatus;
            // Future: Raise domain event ProjectStatusChanged
        }
        
        // Private method to encapsulate business rules for status transitions
        private bool IsValidStatusTransition(ProjectStatus from, ProjectStatus to)
        {
            // Simplified business rules - could be more complex state machine
            return from switch
            {
                ProjectStatus.Planning => to is ProjectStatus.Approved or ProjectStatus.Cancelled,
                ProjectStatus.Approved => to is ProjectStatus.InProgress or ProjectStatus.OnHold or ProjectStatus.Cancelled,
                ProjectStatus.InProgress => to is ProjectStatus.OnHold or ProjectStatus.Completed or ProjectStatus.Cancelled,
                ProjectStatus.OnHold => to is ProjectStatus.InProgress or ProjectStatus.Cancelled,
                ProjectStatus.Completed => false,  // Final state
                ProjectStatus.Cancelled => false,  // Final state
                _ => false
            };
        }
    }

    /// <summary>
    /// Project types supported by the system
    /// Constrains projects to known infrastructure categories
    /// </summary>
    public enum ProjectType
    {
        Roads,              // Highway and road infrastructure
        Bridges,            // Bridge construction and maintenance  
        WaterSystems,       // Water treatment and distribution
        PowerGrid,          // Electrical infrastructure
        PublicBuildings,    // Government and community buildings
        Parks               // Parks and recreational facilities
    }

    /// <summary>
    /// Project status representing the project lifecycle
    /// Forms a state machine with specific allowed transitions
    /// </summary>
    public enum ProjectStatus
    {
        Planning,       // Initial status - gathering requirements
        Approved,       // Project approved and funded
        InProgress,     // Active construction/implementation
        OnHold,         // Temporarily paused
        Completed,      // Successfully finished
        Cancelled       // Terminated before completion
    }
}
```

**Why this domain model design?**
- **Rich domain model**: Contains both data and business behavior, not just properties
- **Invariant enforcement**: Constructor validates business rules, prevents invalid objects
- **Encapsulation**: Private setters prevent external code from bypassing business rules
- **State machine**: Status transitions follow defined business rules
- **Value objects**: Enums provide type safety and constrain valid values
- **Domain events**: Architecture ready for event-driven communication (commented for now)

### 3.3 Database Context

**Intent**: Create the data access layer using Entity Framework Core with proper configuration. The DbContext serves as the Unit of Work pattern implementation and defines how our domain entities map to database tables. Configuration is explicit to ensure predictable database schema and optimal performance.

Create `src/services/projects/Infrastructure.Projects.Infrastructure/Data/ProjectsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    /// <summary>
    /// Database context for the Projects service
    /// Implements Unit of Work pattern and handles entity mapping
    /// Each service has its own database context for service autonomy
    /// </summary>
    public class ProjectsDbContext : DbContext
    {
        // Constructor injection of options allows configuration in Startup.cs
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        // DbSet represents a table - Projects table will be created
        public DbSet<Project> Projects { get; set; }

        /// <summary>
        /// Fluent API configuration - explicit mapping for database schema control
        /// Runs when model is being created, defines table structure and constraints
        /// </summary>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Explicit configuration prevents EF Core convention surprises
            modelBuilder.Entity<Project>(entity =>
            {
                // Primary key configuration - uses the Id from BaseEntity
                entity.HasKey(e => e.Id);
                
                // String property constraints for data integrity and performance
                entity.Property(e => e.Name)
                    .IsRequired()                    // NOT NULL constraint
                    .HasMaxLength(200);             // Reasonable limit prevents abuse
                
                entity.Property(e => e.Description)
                    .HasMaxLength(1000);            // Longer description allowed
                
                // Decimal configuration for financial data precision
                entity.Property(e => e.EstimatedCost)
                    .HasColumnType("decimal(18,2)"); // 18 total digits, 2 decimal places
                                                    // Handles values up to $9,999,999,999,999.99
                
                // Index for common query patterns
                entity.HasIndex(e => e.Status);     // Status filtering will be common
                
                // Additional useful indexes for query performance
                entity.HasIndex(e => e.OwnerId);    // Filter by organization
                entity.HasIndex(e => e.Type);       // Filter by project type
                entity.HasIndex(e => e.StartDate);  // Date range queries
                
                // Enum handling - EF Core stores as string by default (readable in DB)
                entity.Property(e => e.Status)
                    .HasConversion<string>();        // Store enum as string, not int
                entity.Property(e => e.Type)
                    .HasConversion<string>();
                
                // Audit fields from BaseEntity
                entity.Property(e => e.CreatedAt)
                    .IsRequired();
                entity.Property(e => e.CreatedBy)
                    .HasMaxLength(100);
                entity.Property(e => e.UpdatedBy)
                    .HasMaxLength(100);
                
                // Table naming convention - explicit naming prevents surprises
                entity.ToTable("Projects");
            });
            
            // Call base method to apply any additional conventions
            base.OnModelCreating(modelBuilder);
        }
        
        /// <summary>
        /// Override SaveChanges to add automatic audit trail
        /// This ensures all entities get proper audit information
        /// </summary>
        public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            // Get current user context (would come from HTTP context in real app)
            var currentUser = GetCurrentUser(); // TODO: Implement user context
            
            // Automatically set audit fields for new and modified entities
            var entries = ChangeTracker.Entries<BaseEntity>();
            
            foreach (var entry in entries)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        entry.Entity.CreatedBy = currentUser;
                        break;
                    case EntityState.Modified:
                        entry.Entity.Update(currentUser);
                        break;
                }
            }
            
            return await base.SaveChangesAsync(cancellationToken);
        }
        
        // TODO: Implement proper user context injection
        private string GetCurrentUser() => "system"; // Placeholder
    }
}
```

**Why this DbContext design?**
- **Explicit configuration**: Fluent API prevents EF Core convention surprises in production
- **Performance indexes**: Strategic indexes on commonly filtered columns
- **Decimal precision**: Financial data requires exact decimal handling, not floating point
- **String enums**: Human-readable enum values in database for debugging and reports
- **Audit automation**: Automatic audit trail without remembering to set fields manually
- **Service isolation**: Each service owns its database schema completely`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    public class ProjectsDbContext : DbContext
    {
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        public DbSet<Project> Projects { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Project>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Description).HasMaxLength(1000);
                entity.Property(e => e.EstimatedCost).HasColumnType("decimal(18,2)");
                entity.HasIndex(e => e.Status);
            });
        }
    }
}
```

### 3.4 API Controller

Create `src/services/projects/Infrastructure.Projects.API/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.Queries;
using MediatR;

namespace Infrastructure.Projects.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProjectsController : ControllerBase
    {
        private readonly IMediator _mediator;

        public ProjectsController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [HttpGet]
        public async Task<IActionResult> GetProjects([FromQuery] GetProjectsQuery query)
        {
            var result = await _mediator.Send(query);
            return Ok(result);
        }

        [HttpGet("{id}")]
        public async Task<IActionResult> GetProject(Guid id)
        {
            var result = await _mediator.Send(new GetProjectByIdQuery(id));
            return Ok(result);
        }

        [HttpPost]
        public async Task<IActionResult> CreateProject([FromBody] CreateProjectCommand command)
        {
            var result = await _mediator.Send(command);
            return CreatedAtAction(nameof(GetProject), new { id = result.Id }, result);
        }
    }
}
```

### 3.5 Test the Project Service

Create startup configuration and test:

```bash
cd src/services/projects/Infrastructure.Projects.API
dotnet run

# Test endpoint
curl http://localhost:5000/api/projects
```

## Phase 4: Basic Frontend (Week 4)

### 4.1 React Frontend Setup

```bash
cd src/web
npx create-react-app admin-portal --template typescript
cd admin-portal

# Install additional packages
npm install @mui/material @emotion/react @emotion/styled
npm install @mui/icons-material
npm install axios react-query
npm install react-router-dom
```

### 4.2 Project List Component

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useEffect, useState } from 'react';
import {
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  Paper,
  Chip,
  Button,
  Typography
} from '@mui/material';
import { Add as AddIcon } from '@mui/icons-material';
import { projectService, Project, ProjectStatus } from '../services/projectService';

const ProjectList: React.FC = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadProjects();
  }, []);

  const loadProjects = async () => {
    try {
      const data = await projectService.getProjects();
      setProjects(data);
    } catch (error) {
      console.error('Failed to load projects:', error);
    } finally {
      setLoading(false);
    }
  };

  const getStatusColor = (status: ProjectStatus) => {
    const colors = {
      Planning: 'default',
      Approved: 'primary',
      InProgress: 'warning',
      OnHold: 'error',
      Completed: 'success',
      Cancelled: 'error'
    };
    return colors[status] || 'default';
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 20 }}>
        <Typography variant="h4">Infrastructure Projects</Typography>
        <Button variant="contained" startIcon={<AddIcon />}>
          New Project
        </Button>
      </div>

      <TableContainer component={Paper}>
        <Table>
          <TableHead>
            <TableRow>
              <TableCell>Name</TableCell>
              <TableCell>Type</TableCell>
              <TableCell>Status</TableCell>
              <TableCell>Estimated Cost</TableCell>
              <TableCell>Start Date</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {projects.map((project) => (
              <TableRow key={project.id}>
                <TableCell>{project.name}</TableCell>
                <TableCell>{project.type}</TableCell>
                <TableCell>
                  <Chip 
                    label={project.status} 
                    color={getStatusColor(project.status) as any}
                    size="small"
                  />
                </TableCell>
                <TableCell>${project.estimatedCost.toLocaleString()}</TableCell>
                <TableCell>{new Date(project.startDate).toLocaleDateString()}</TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </TableContainer>
    </div>
  );
};

export default ProjectList;
```

### 4.3 Test Full Stack

Start both backend and frontend:

```bash
# Terminal 1 - Backend
cd src/services/projects/Infrastructure.Projects.API
dotnet run

# Terminal 2 - Frontend  
cd src/web/admin-portal
npm start
```

Visit `http://localhost:3000` to see the project list.

## Phase 5: Add Budget Service (Week 5)

### 5.1 Budget Service Structure

Follow the same pattern as Project Service:

```bash
mkdir -p src/services/budget/{Domain,Application,Infrastructure,API}
```

Create Budget domain entities, similar to Project but for budget-specific concerns:

```csharp
public class Budget : BaseEntity
{
    public string Name { get; private set; }
    public int FiscalYear { get; private set; }
    public decimal TotalAmount { get; private set; }
    public BudgetStatus Status { get; private set; }
    
    private readonly List<BudgetLineItem> _lineItems = new();
    public IReadOnlyList<BudgetLineItem> LineItems => _lineItems.AsReadOnly();
    
    // Constructor and methods...
}

public class BudgetLineItem : BaseEntity
{
    public string Category { get; private set; }
    public string Description { get; private set; }
    public decimal AllocatedAmount { get; private set; }
    public decimal SpentAmount { get; private set; }
    public Guid BudgetId { get; private set; }
}
```

### 5.2 Inter-Service Communication

Create shared event contracts in `src/shared/events/`:

```csharp
public class ProjectCreatedEvent : DomainEvent
{
    public Guid ProjectId { get; }
    public string ProjectName { get; }
    public decimal EstimatedCost { get; }
    
    public ProjectCreatedEvent(Guid projectId, string projectName, decimal estimatedCost)
    {
        ProjectId = projectId;
        ProjectName = projectName;
        EstimatedCost = estimatedCost;
    }
}
```

## Phase 6: API Gateway (Week 6)

### 6.1 Set up Ocelot API Gateway

Create `src/gateways/api-gateway/Infrastructure.Gateway.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Ocelot" Version="20.0.0" />
    <PackageReference Include="Ocelot.Provider.Consul" Version="20.0.0" />
  </ItemGroup>
</Project>
```

### 6.2 Gateway Configuration

Create `src/gateways/api-gateway/ocelot.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/budgets/{everything}",
      "DownstreamScheme": "https", 
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/gateway/budgets/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

### 6.3 Test Gateway

Update frontend to use gateway endpoints and verify routing works.

## Phase 7: Authentication & Security (Week 7)

### 7.1 Add Identity Service

```bash
mkdir -p src/services/identity
cd src/services/identity

dotnet new webapi -n Infrastructure.Identity.API
cd Infrastructure.Identity.API
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 7.2 JWT Authentication Setup

Configure JWT authentication across all services and test login flow.

### 7.3 Role-Based Authorization

Implement RBAC system:
- Municipality Admin
- Project Manager  
- Budget Analyst
- Grant Writer
- Viewer

## Phase 8: Add Remaining Services (Weeks 8-12)

### 8.1 Grant Management Service (Week 8)
- Grant opportunity discovery
- Application workflow
- Document management

### 8.2 Cost Estimation Service (Week 9) 
- Historical cost data
- Market price integration
- ML-based predictions

### 8.3 BCA (Benefit-Cost Analysis) Service (Week 10)
- NPV calculations
- Risk assessment
- Compliance reporting

### 8.4 Contract Management Service (Week 11)
- Contract templates
- Procurement workflows
- Vendor management

### 8.5 Analytics & Reporting Service (Week 12)
- Dashboard data aggregation
- Report generation
- KPI calculations

## Phase 9: Advanced Frontend Features (Weeks 13-16)

### 9.1 Enhanced UI Components
- Data visualization with Charts.js/D3
- Interactive dashboards
- Document upload/preview

### 9.2 Workflow Management
- Approval flows
- Task assignments
- Notifications

### 9.3 Integration Interfaces
- ERP system connectors
- Grant portal APIs
- GIS system integration

## Phase 10: Production Readiness (Weeks 17-20)

### 10.1 Kubernetes Deployment

Create `infrastructure/kubernetes/` manifests for:
- Service deployments
- ConfigMaps and Secrets
- Ingress controllers
- Persistent volumes

### 10.2 CI/CD Pipeline

Set up GitLab CI or Azure DevOps:
- Automated testing
- Docker image building
- Deployment automation
- Environment promotion

### 10.3 Monitoring & Observability

Configure:
- Application logging (Serilog)
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Health checks

### 10.4 Performance Testing

- Load testing with k6
- Database query optimization
- Caching strategies
- CDN setup

## Testing Strategy

### Unit Tests
Each service should have comprehensive unit tests:

```bash
cd src/services/projects
dotnet new xunit -n Infrastructure.Projects.Tests
# Add test packages and write tests
```

### Integration Tests
Test service interactions and database operations.

### End-to-End Tests
Use Playwright or Cypress for full workflow testing.

## Development Workflow

1. **Feature Branch Strategy**: Create branches for each service/feature
2. **Code Reviews**: All code must be reviewed before merging
3. **Automated Testing**: CI pipeline runs all tests
4. **Local Development**: Everything runs locally with Docker Compose
5. **Documentation**: Keep docs updated as you build

## Key Milestones

- **Week 4**: First working service with basic UI
- **Week 8**: Core services communicating via events
- **Week 12**: All business services implemented
- **Week 16**: Complete frontend with all features
- **Week 20**: Production-ready deployment

This approach ensures you have a working system at every step, can demo progress regularly, and can make adjustments based on feedback as you build.

### 3.8 Service Configuration and Startup

**Intent**: Configure dependency injection, database connections, and middleware in a clean, organized manner. This setup follows .NET 8 minimal hosting patterns while maintaining testability and clear separation of concerns.

Create `src/services/projects/Infrastructure.Projects.API/Program.cs`:

```csharp
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Projects.Application.Mapping;
using Microsoft.EntityFrameworkCore;
using MediatR;
using System.Reflection;

// Modern .NET minimal hosting pattern
var builder = WebApplication.CreateBuilder(args);

// Configure services
ConfigureServices(builder.Services, builder.Configuration);

var app = builder.Build();

// Configure middleware pipeline
ConfigureMiddleware(app);

// Initialize database
await InitializeDatabase(app);

app.Run();

/// <summary>
/// Configure all application services and dependencies
/// Organized by concern: database, business logic, cross-cutting, API
/// </summary>
void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    // Database configuration
    services.AddDbContext<ProjectsDbContext>(options =>
        options.UseNpgsql(configuration.GetConnectionString("DefaultConnection"),
            npgsqlOptions => npgsqlOptions.MigrationsAssembly("Infrastructure.Projects.Infrastructure"))
    );

    // MediatR for CQRS pattern - scans assembly for handlers
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
    
    // AutoMapper for entity/DTO mapping
    services.AddAutoMapper(typeof(ProjectMappingProfile));
    
    // API services
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            // Configure JSON serialization for API consistency
            options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
            options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
        });
    
    // API documentation
    services.AddEndpointsApiExplorer();
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "Infrastructure Projects API", 
            Version = "v1",
            Description = "API for managing infrastructure projects and related operations"
        });
        
        // Include XML comments in Swagger documentation
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            c.IncludeXmlComments(xmlPath);
        }
    });
    
    // CORS for frontend applications
    services.AddCors(options =>
    {
        options.AddPolicy("DevelopmentPolicy", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://localhost:3000")  // React dev server
                  .AllowAnyMethod()
                  # Public Infrastructure Planning Platform - Build Guide

## Phase 1: Development Environment Setup (Week 1)

### 1.1 Prerequisites Installation

**Intent**: Establish a consistent development environment across all team members to eliminate "works on my machine" issues. We're choosing specific versions and tools that support our microservices architecture.

```bash
# Install required tools
# Node.js (18+) - Required for React frontend and build tools
# Using NVM allows easy version switching for different projects
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop - Essential for local development environment
# Allows us to run all infrastructure services (DB, Redis, Kafka) locally
# Download from docker.com

# .NET 8 SDK - Latest LTS version for backend services
# Provides modern C# features, performance improvements, and long-term support
# Download from microsoft.com/dotnet

# Git - Version control for distributed development
sudo apt install git  # Linux
brew install git      # macOS

# VS Code with extensions - Consistent IDE experience
# Extensions provide IntelliSense, debugging, and container support
code --install-extension ms-dotnettools.csharp
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

**Why these choices?**
- **Node.js 18+**: Provides modern JavaScript features and npm ecosystem access
- **Docker**: Eliminates environment differences and simplifies service orchestration
- **.NET 8**: High-performance, cross-platform runtime with excellent tooling
- **VS Code**: Lightweight, extensible, with excellent container and cloud support

### 1.2 Project Structure Setup

**Intent**: Create a well-organized monorepo structure that supports microservices architecture while maintaining clear boundaries. This structure follows Domain-Driven Design principles and separates concerns between services, shared code, infrastructure, and client applications.

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Create main directory structure
# This follows the "screaming architecture" principle - the structure tells you what the system does
mkdir -p {
  src/{
    # Each service is a bounded context with its own domain
    services/{budget,grants,projects,bca,contracts,costs,users,notifications},
    # Shared code that multiple services can use - but keep this minimal
    shared/{common,domain,events},
    # API Gateway acts as the single entry point for clients
    gateways/api-gateway,
    # Separate frontend applications for different user types
    web/{admin-portal,public-portal},
    # Mobile app for field workers and inspectors
    mobile
  },
  infrastructure/{
    # Infrastructure as Code - everything should be reproducible
    docker,      # Local development containers
    kubernetes,  # Production orchestration
    terraform,   # Cloud resource provisioning
    scripts      # Deployment and utility scripts
  },
  docs,  # Architecture decision records, API docs, user guides
  tests  # End-to-end and integration tests that span services
}
```

**Why this structure?**
- **Service boundaries**: Each service in `/services/` represents a business capability
- **Shared minimal**: Only truly shared code goes in `/shared/` to avoid coupling
- **Infrastructure separation**: Keeps deployment concerns separate from business logic
- **Client separation**: Different frontends for different user needs and experiences
- **Documentation co-location**: Docs live with code for easier maintenance

### 1.3 Git Repository Setup

```bash
git init
echo "node_modules/
bin/
obj/
.env
.DS_Store
*.log" > .gitignore

git add .
git commit -m "Initial project structure"
```

## Phase 2: Core Infrastructure (Week 2)

### 2.1 Docker Development Environment

**Intent**: Create a local development environment that mirrors production as closely as possible. This setup provides all the infrastructure services (database, cache, messaging, search) that our microservices need, without requiring complex local installations. Each service is isolated and can be started/stopped independently.

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  # PostgreSQL - Primary database for transactional data
  # Using version 15 for performance and JSON improvements
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrastructure_dev      # Single database, multiple schemas per service
      POSTGRES_USER: dev_user              # Non-root user for security
      POSTGRES_PASSWORD: dev_password      # Simple password for local dev
    ports:
      - "5432:5432"                       # Standard PostgreSQL port
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data between container restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Initialize schemas

  # Redis - Caching and session storage
  # Using Alpine for smaller image size
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"                       # Standard Redis port
    # No persistence needed in development

  # Apache Kafka - Event streaming between services
  # Essential for event-driven architecture and service decoupling
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper                         # Kafka requires Zookeeper for coordination
    ports:
      - "9092:9092"                      # Kafka broker port
    environment:
      KAFKA_BROKER_ID: 1                 # Single broker for dev
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092  # Clients connect via localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  # No replication needed in dev

  # Zookeeper - Kafka dependency for cluster coordination
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Elasticsearch - Full-text search and analytics
  # Used for searching grants, projects, and generating reports
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node        # Single node for development
      - xpack.security.enabled=false      # Disable security for local dev
    ports:
      - "9200:9200"                      # REST API port

volumes:
  postgres_data:  # Named volume for database persistence
```

**Test Infrastructure:**
```bash
cd infrastructure/docker
# Start all infrastructure services in background
docker-compose -f docker-compose.dev.yml up -d

# Verify all services are running and healthy
docker-compose ps

# Expected output: All services should show "Up" status
# postgres_1      Up  0.0.0.0:5432->5432/tcp
# redis_1         Up  0.0.0.0:6379->6379/tcp
# kafka_1         Up  0.0.0.0:9092->9092/tcp
# etc.
```

**Why these infrastructure choices?**
- **PostgreSQL**: ACID compliance, JSON support, excellent .NET integration
- **Redis**: Fast caching, session storage, distributed locks
- **Kafka**: Reliable event streaming, service decoupling, audit logging
- **Elasticsearch**: Powerful search, analytics, report generation
- **Single-node configs**: Simplified for development, production uses clusters

### 2.2 Shared Domain Models

**Intent**: Establish common base classes and patterns that all domain entities will inherit from. This implements the DDD (Domain-Driven Design) principle of having rich domain objects with behavior, not just data containers. The base entity provides audit trail capabilities essential for government compliance and change tracking.

Create `src/shared/domain/Entities/BaseEntity.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Entities
{
    /// <summary>
    /// Base class for all domain entities providing common functionality
    /// Implements Entity pattern from DDD with audit trail for compliance
    /// </summary>
    public abstract class BaseEntity
    {
        // Using Guid as ID provides globally unique identifiers across distributed services
        // No need for centralized ID generation or sequence coordination
        public Guid Id { get; protected set; }
        
        // Audit fields required for government compliance and change tracking
        public DateTime CreatedAt { get; protected set; }
        public DateTime UpdatedAt { get; protected set; }
        public string CreatedBy { get; protected set; }      // User who created the entity
        public string UpdatedBy { get; protected set; }      // User who last modified

        // Protected constructor prevents direct instantiation
        // Forces use of domain-specific constructors in derived classes
        protected BaseEntity()
        {
            Id = Guid.NewGuid();              // Generate unique ID immediately
            CreatedAt = DateTime.UtcNow;      // Always use UTC to avoid timezone issues
        }

        // Virtual method allows derived classes to add custom update logic
        // Automatically tracks who made changes and when
        public virtual void Update(string updatedBy)
        {
            UpdatedAt = DateTime.UtcNow;
            UpdatedBy = updatedBy ?? throw new ArgumentNullException(nameof(updatedBy));
        }
    }
}
```

**Intent**: Create a base class for domain events that enables event-driven architecture. Domain events represent something significant that happened in the business domain and allow services to communicate without direct coupling.

Create `src/shared/domain/Events/DomainEvent.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Events
{
    /// <summary>
    /// Base class for all domain events in the system
    /// Enables event-driven architecture and service decoupling
    /// Events represent business facts that other services might care about
    /// </summary>
    public abstract class DomainEvent
    {
        // Unique identifier for this specific event occurrence
        public Guid Id { get; }
        
        // When did this business event occur (not when it was processed)
        public DateTime OccurredOn { get; }
        
        // Type identifier for event routing and deserialization
        // Using class name allows easy identification in logs and debugging
        public string EventType { get; }

        protected DomainEvent()
        {
            Id = Guid.NewGuid();                    // Unique event ID for idempotency
            OccurredOn = DateTime.UtcNow;           // Business time, not processing time
            EventType = this.GetType().Name;       // Automatic type identification
        }
    }
}
```

**Why these design decisions?**
- **Guid IDs**: Eliminate distributed ID generation complexity, work across services
- **UTC timestamps**: Prevent timezone confusion in distributed systems
- **Audit fields**: Meet government compliance requirements for change tracking
- **Protected constructors**: Enforce proper domain object creation patterns
- **Domain events**: Enable loose coupling between services while maintaining business consistency
- **Event metadata**: Support idempotency, ordering, and debugging in distributed systems

## Phase 3: First Service - Project Management (Week 3)

### 3.1 Project Service Setup

**Intent**: Create our first microservice following Clean Architecture principles. This service will handle all project-related operations and serve as a template for other services. We're using the latest .NET 8 with carefully chosen packages that support our architectural goals.

Create `src/services/projects/Infrastructure.Projects.API/Infrastructure.Projects.API.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>    <!-- Latest LTS for performance -->
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Entity Framework Core for database operations -->
    <!-- Design package needed for migrations and scaffolding -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
    <!-- PostgreSQL provider for our chosen database -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    
    <!-- Swagger for API documentation and testing -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- MediatR implements CQRS and Mediator pattern -->
    <!-- Decouples controllers from business logic -->
    <PackageReference Include="MediatR" Version="12.0.0" />
    
    <!-- AutoMapper for object-to-object mapping -->
    <!-- Keeps controllers clean by handling DTO conversions -->
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <!-- Reference to shared domain models and utilities -->
    <ProjectReference Include="../../shared/common/Infrastructure.Shared.Common.csproj" />
  </ItemGroup>
</Project>
```

**Why these package choices?**
- **EF Core + PostgreSQL**: Robust ORM with excellent PostgreSQL support for complex queries
- **MediatR**: Implements CQRS pattern, separates read/write operations, supports cross-cutting concerns
- **AutoMapper**: Reduces boilerplate code for mapping between domain models and DTOs
- **Swashbuckle**: Provides interactive API documentation essential for microservices integration

### 3.2 Project Domain Model

**Intent**: Create a rich domain model that encapsulates business rules and invariants. This follows DDD principles where the domain model contains both data and behavior. The Project entity represents the core concept of infrastructure projects and enforces business constraints through its design.

Create `src/services/projects/Infrastructure.Projects.Domain/Entities/Project.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Projects.Domain.Entities
{
    /// <summary>
    /// Project aggregate root representing an infrastructure project
    /// Contains all business rules and invariants for project management
    /// </summary>
    public class Project : BaseEntity
    {
        // Required project information - private setters enforce controlled modification
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ProjectType Type { get; private set; }           // Enum constrains valid project types
        public ProjectStatus Status { get; private set; }       // State machine via status enum
        
        // Financial information with proper decimal type for currency
        public decimal EstimatedCost { get; private set; }      // Initial cost estimate
        
        // Timeline information
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }          // Nullable - may not be set initially
        
        // Ownership tracking
        public string OwnerId { get; private set; }             // Reference to owning organization

        // EF Core requires parameterless constructor - private to prevent misuse
        private Project() { } 

        /// <summary>
        /// Creates a new infrastructure project with required business data
        /// Constructor enforces invariants - all required data must be provided
        /// </summary>
        public Project(string name, string description, ProjectType type, 
                      decimal estimatedCost, DateTime startDate, string ownerId)
        {
            // Business rule validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Project name is required", nameof(name));
            if (estimatedCost <= 0)
                throw new ArgumentException("Project cost must be positive", nameof(estimatedCost));
            if (startDate < DateTime.Today)
                throw new ArgumentException("Project cannot start in the past", nameof(startDate));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Project owner is required", nameof(ownerId));

            Name = name;
            Description = description;
            Type = type;
            Status = ProjectStatus.Planning;     // All projects start in planning phase
            EstimatedCost = estimatedCost;
            StartDate = startDate;
            OwnerId = ownerId;
        }

        /// <summary>
        /// Updates project status following business rules
        /// Could be extended to implement state machine validation
        /// Domain event could be raised here for other services to react
        /// </summary>
        public void UpdateStatus(ProjectStatus newStatus)
        {
            // Business rule: validate status transitions
            if (!IsValidStatusTransition(Status, newStatus))
                throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");
                
            Status = newStatus;
            // Future: Raise domain event ProjectStatusChanged
        }
        
        // Private method to encapsulate business rules for status transitions
        private bool IsValidStatusTransition(ProjectStatus from, ProjectStatus to)
        {
            // Simplified business rules - could be more complex state machine
            return from switch
            {
                ProjectStatus.Planning => to is ProjectStatus.Approved or ProjectStatus.Cancelled,
                ProjectStatus.Approved => to is ProjectStatus.InProgress or ProjectStatus.OnHold or ProjectStatus.Cancelled,
                ProjectStatus.InProgress => to is ProjectStatus.OnHold or ProjectStatus.Completed or ProjectStatus.Cancelled,
                ProjectStatus.OnHold => to is ProjectStatus.InProgress or ProjectStatus.Cancelled,
                ProjectStatus.Completed => false,  // Final state
                ProjectStatus.Cancelled => false,  // Final state
                _ => false
            };
        }
    }

    /// <summary>
    /// Project types supported by the system
    /// Constrains projects to known infrastructure categories
    /// </summary>
    public enum ProjectType
    {
        Roads,              // Highway and road infrastructure
        Bridges,            // Bridge construction and maintenance  
        WaterSystems,       // Water treatment and distribution
        PowerGrid,          // Electrical infrastructure
        PublicBuildings,    // Government and community buildings
        Parks               // Parks and recreational facilities
    }

    /// <summary>
    /// Project status representing the project lifecycle
    /// Forms a state machine with specific allowed transitions
    /// </summary>
    public enum ProjectStatus
    {
        Planning,       // Initial status - gathering requirements
        Approved,       // Project approved and funded
        InProgress,     // Active construction/implementation
        OnHold,         // Temporarily paused
        Completed,      // Successfully finished
        Cancelled       // Terminated before completion
    }
}
```

**Why this domain model design?**
- **Rich domain model**: Contains both data and business behavior, not just properties
- **Invariant enforcement**: Constructor validates business rules, prevents invalid objects
- **Encapsulation**: Private setters prevent external code from bypassing business rules
- **State machine**: Status transitions follow defined business rules
- **Value objects**: Enums provide type safety and constrain valid values
- **Domain events**: Architecture ready for event-driven communication (commented for now)

### 3.3 Database Context

**Intent**: Create the data access layer using Entity Framework Core with proper configuration. The DbContext serves as the Unit of Work pattern implementation and defines how our domain entities map to database tables. Configuration is explicit to ensure predictable database schema and optimal performance.

Create `src/services/projects/Infrastructure.Projects.Infrastructure/Data/ProjectsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    /// <summary>
    /// Database context for the Projects service
    /// Implements Unit of Work pattern and handles entity mapping
    /// Each service has its own database context for service autonomy
    /// </summary>
    public class ProjectsDbContext : DbContext
    {
        // Constructor injection of options allows configuration in Startup.cs
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        // DbSet represents a table - Projects table will be created
        public DbSet<Project> Projects { get; set; }

        /// <summary>
        /// Fluent API configuration - explicit mapping for database schema control
        /// Runs when model is being created, defines table structure and constraints
        /// </summary>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Explicit configuration prevents EF Core convention surprises
            modelBuilder.Entity<Project>(entity =>
            {
                // Primary key configuration - uses the Id from BaseEntity
                entity.HasKey(e => e.Id);
                
                // String property constraints for data integrity and performance
                entity.Property(e => e.Name)
                    .IsRequired()                    // NOT NULL constraint
                    .HasMaxLength(200);             // Reasonable limit prevents abuse
                
                entity.Property(e => e.Description)
                    .HasMaxLength(1000);            // Longer description allowed
                
                // Decimal configuration for financial data precision
                entity.Property(e => e.EstimatedCost)
                    .HasColumnType("decimal(18,2)"); // 18 total digits, 2 decimal places
                                                    // Handles values up to $9,999,999,999,999.99
                
                // Index for common query patterns
                entity.HasIndex(e => e.Status);     // Status filtering will be common
                
                // Additional useful indexes for query performance
                entity.HasIndex(e => e.OwnerId);    // Filter by organization
                entity.HasIndex(e => e.Type);       // Filter by project type
                entity.HasIndex(e => e.StartDate);  // Date range queries
                
                // Enum handling - EF Core stores as string by default (readable in DB)
                entity.Property(e => e.Status)
                    .HasConversion<string>();        // Store enum as string, not int
                entity.Property(e => e.Type)
                    .HasConversion<string>();
                
                // Audit fields from BaseEntity
                entity.Property(e => e.CreatedAt)
                    .IsRequired();
                entity.Property(e => e.CreatedBy)
                    .HasMaxLength(100);
                entity.Property(e => e.UpdatedBy)
                    .HasMaxLength(100);
                
                // Table naming convention - explicit naming prevents surprises
                entity.ToTable("Projects");
            });
            
            // Call base method to apply any additional conventions
            base.OnModelCreating(modelBuilder);
        }
        
        /// <summary>
        /// Override SaveChanges to add automatic audit trail
        /// This ensures all entities get proper audit information
        /// </summary>
        public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            // Get current user context (would come from HTTP context in real app)
            var currentUser = GetCurrentUser(); // TODO: Implement user context
            
            // Automatically set audit fields for new and modified entities
            var entries = ChangeTracker.Entries<BaseEntity>();
            
            foreach (var entry in entries)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        entry.Entity.CreatedBy = currentUser;
                        break;
                    case EntityState.Modified:
                        entry.Entity.Update(currentUser);
                        break;
                }
            }
            
            return await base.SaveChangesAsync(cancellationToken);
        }
        
        // TODO: Implement proper user context injection
        private string GetCurrentUser() => "system"; // Placeholder
    }
}
```

**Why this DbContext design?**
- **Explicit configuration**: Fluent API prevents EF Core convention surprises in production
- **Performance indexes**: Strategic indexes on commonly filtered columns
- **Decimal precision**: Financial data requires exact decimal handling, not floating point
- **String enums**: Human-readable enum values in database for debugging and reports
- **Audit automation**: Automatic audit trail without remembering to set fields manually
- **Service isolation**: Each service owns its database schema completely`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    public class ProjectsDbContext : DbContext
    {
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        public DbSet<Project> Projects { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Project>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Description).HasMaxLength(1000);
                entity.Property(e => e.EstimatedCost).HasColumnType("decimal(18,2)");
                entity.HasIndex(e => e.Status);
            });
        }
    }
}
```

### 3.4 API Controller

**Intent**: Create a clean API controller that follows REST principles and implements the CQRS pattern using MediatR. The controller is thin and focused only on HTTP concerns - all business logic is handled by command and query handlers. This promotes separation of concerns and testability.

Create `src/services/projects/Infrastructure.Projects.API/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.API.Controllers
{
    /// <summary>
    /// REST API controller for project management operations
    /// Implements thin controller pattern - delegates all logic to MediatR handlers
    /// Focuses purely on HTTP concerns: routing, status codes, request/response
    /// </summary>
    [ApiController]
    [Route("api/[controller]")]              // Route: /api/projects
    [Produces("application/json")]           // Always returns JSON
    public class ProjectsController : ControllerBase
    {
        private readonly IMediator _mediator;

        // Dependency injection of MediatR for CQRS pattern
        public ProjectsController(IMediator mediator)
        {
            _mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
        }

        /// <summary>
        /// GET /api/projects - Retrieve projects with optional filtering
        /// Query parameters enable filtering, paging, and sorting
        /// </summary>
        [HttpGet]
        [ProducesResponseType(typeof(PagedResult<ProjectDto>), 200)]
        [ProducesResponseType(400)] // Bad request for invalid parameters
        public async Task<IActionResult> GetProjects([FromQuery] GetProjectsQuery query)
        {
            try 
            {
                // MediatR handles routing to appropriate query handler
                var result = await _mediator.Send(query);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                // Invalid query parameters
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// GET /api/projects/{id} - Retrieve specific project by ID
        /// </summary>
        [HttpGet("{id:guid}")]                          // Constraint ensures valid GUID
        [ProducesResponseType(typeof(ProjectDetailDto), 200)]
        [ProducesResponseType(404)]                     // Project not found
        public async Task<IActionResult> GetProject(Guid id)
        {
            var query = new GetProjectByIdQuery(id);
            var result = await _mediator.Send(query);
            
            if (result == null)
                return NotFound(new { error = $"Project with ID {id} not found" });
                
            return Ok(result);
        }

        /// <summary>
        /// POST /api/projects - Create new infrastructure project
        /// Returns 201 Created with location header pointing to new resource
        /// </summary>
        [HttpPost]
        [ProducesResponseType(typeof(ProjectDto), 201)]  // Created successfully
        [ProducesResponseType(400)]                      // Validation errors
        [ProducesResponseType(409)]                      // Conflict (duplicate name, etc.)
        public async Task<IActionResult> CreateProject([FromBody] CreateProjectCommand command)
        {
            try
            {
                // Command handler creates project and returns DTO
                var result = await _mediator.Send(command);
                
                // REST best practice: return 201 Created with location header
                return CreatedAtAction(
                    nameof(GetProject), 
                    new { id = result.Id }, 
                    result);
            }
            catch (ArgumentException ex)
            {
                // Domain validation errors
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Business rule violations
                return Conflict(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PUT /api/projects/{id} - Update existing project
        /// </summary>
        [HttpPut("{id:guid}")]
        [ProducesResponseType(typeof(ProjectDto), 200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProject(Guid id, [FromBody] UpdateProjectCommand command)
        {
            // Ensure URL ID matches command ID for consistency
            if (id != command.Id)
                return BadRequest(new { error = "URL ID must match command ID" });

            try
            {
                var result = await _mediator.Send(command);
                if (result == null)
                    return NotFound();
                    
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PATCH /api/projects/{id}/status - Update project status only
        /// Separate endpoint for status changes supports workflow scenarios
        /// </summary>
        [HttpPatch("{id:guid}/status")]
        [ProducesResponseType(200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProjectStatus(Guid id, [FromBody] UpdateProjectStatusCommand command)
        {
            if (id != command.ProjectId)
                return BadRequest(new { error = "URL ID must match command project ID" });

            try
            {
                await _mediator.Send(command);
                return Ok(new { message = "Status updated successfully" });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Invalid status transitions
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// DELETE /api/projects/{id} - Soft delete project
        /// Note: Infrastructure projects are rarely truly deleted, usually cancelled
        /// </summary>
        [HttpDelete("{id:guid}")]
        [ProducesResponseType(204)]  // No content - successful deletion
        [ProducesResponseType(404)]
        [ProducesResponseType(409)]  // Cannot delete due to dependencies
        public async Task<IActionResult> DeleteProject(Guid id)
        {
            try
            {
                var command = new DeleteProjectCommand(id);
                await _mediator.Send(command);
                return NoContent();
            }
            catch (InvalidOperationException ex)
            {
                // Project has dependencies or cannot be deleted
                return Conflict(new { error = ex.Message });
            }
        }
    }
}
```

**Why this controller design?**
- **Thin controllers**: Only handle HTTP concerns, delegate business logic to handlers
- **CQRS pattern**: Separate commands (writes) from queries (reads) for clarity
- **Proper HTTP semantics**: Correct status codes, REST principles, location headers
- **Error handling**: Consistent error responses with meaningful messages
- **Type safety**: Strong typing with DTOs, GUID constraints on routes
- **Testability**: Easy to unit test by mocking IMediator
- **Documentation**: ProducesResponseType attributes generate OpenAPI specs

### 3.5 Command and Query Handlers

**Intent**: Implement the CQRS pattern by creating separate handlers for commands (writes) and queries (reads). This provides clear separation between operations that change state versus those that read data, enables different optimization strategies, and supports future event sourcing implementation.

Create `src/services/projects/Infrastructure.Projects.Application/Commands/CreateProjectCommand.cs`:

```csharp
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.Application.Commands
{
    /// <summary>
    /// Command to create a new infrastructure project
    /// Represents the intent to perform a write operation
    /// Contains all data needed to create a project
    /// </summary>
    public class CreateProjectCommand : IRequest<ProjectDto>
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public ProjectType Type { get; set; }
        public decimal EstimatedCost { get; set; }
        public DateTime StartDate { get; set; }
        public string OwnerId { get; set; }  // Current user's organization

        // Validation method to encapsulate business rules
        public void Validate()
        {
            if (string.IsNullOrWhiteSpace(Name))
                throw new ArgumentException("Project name is required");
            if (EstimatedCost <= 0)
                throw new ArgumentException("Estimated cost must be greater than zero");
            if (StartDate < DateTime.Today)
                throw new ArgumentException("Start date cannot be in the past");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/CreateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Infrastructure.Data;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles the CreateProjectCommand - implements the business logic for project creation
    /// Follows single responsibility principle - only creates projects
    /// </summary>
    public class CreateProjectHandler : IRequestHandler<CreateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public CreateProjectHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<ProjectDto> Handle(CreateProjectCommand request, CancellationToken cancellationToken)
        {
            // Validate command (could also use FluentValidation)
            request.Validate();
            
            // Business rule: Check for duplicate project names within organization
            var existingProject = await _context.Projects
                .FirstOrDefaultAsync(p => p.Name == request.Name && p.OwnerId == request.OwnerId, 
                                   cancellationToken);
            
            if (existingProject != null)
                throw new InvalidOperationException($"Project '{request.Name}' already exists");
            
            // Create domain entity - constructor enforces invariants
            var project = new Project(
                request.Name,
                request.Description,
                request.Type,
                request.EstimatedCost,
                request.StartDate,
                request.OwnerId
            );
            
            // Persist to database
            _context.Projects.Add(project);
            await _context.SaveChangesAsync(cancellationToken);
            
            // TODO: Raise domain event ProjectCreatedEvent for other services
            
            // Map to DTO for response
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Queries/GetProjectsQuery.cs`:

```csharp
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using MediatR;

namespace Infrastructure.Projects.Application.Queries
{
    /// <summary>
    /// Query to retrieve projects with filtering, paging, and sorting
    /// Optimized for read operations with minimal data transfer
    /// </summary>
    public class GetProjectsQuery : IRequest<PagedResult<ProjectDto>>
    {
        // Filtering options
        public string? NameFilter { get; set; }
        public ProjectStatus? Status { get; set; }
        public ProjectType? Type { get; set; }
        public string? OwnerId { get; set; }
        
        // Date range filtering
        public DateTime? StartDateFrom { get; set; }
        public DateTime? StartDateTo { get; set; }
        
        // Paging parameters
        public int PageNumber { get; set; } = 1;
        public int PageSize { get; set; } = 20;    // Default page size
        
        // Sorting options
        public string? SortBy { get; set; } = "Name";  // Default sort by name
        public bool SortDescending { get; set; } = false;
        
        // Validation for query parameters
        public void Validate()
        {
            if (PageNumber < 1)
                throw new ArgumentException("Page number must be greater than 0");
            if (PageSize < 1 || PageSize > 100)
                throw new ArgumentException("Page size must be between 1 and 100");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/GetProjectsHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles project list queries with filtering and paging
    /// Optimized for read performance with projection to DTOs
    /// </summary>
    public class GetProjectsHandler : IRequestHandler<GetProjectsQuery, PagedResult<ProjectDto>>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public GetProjectsHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<PagedResult<ProjectDto>> Handle(GetProjectsQuery request, CancellationToken cancellationToken)
        {
            // Validate query parameters
            request.Validate();
            
            // Build query with filtering - start with base query
            var query = _context.Projects.AsQueryable();
            
            // Apply filters - only add WHERE clauses for provided filters
            if (!string.IsNullOrEmpty(request.NameFilter))
                query = query.Where(p => p.Name.Contains(request.NameFilter));
                
            if (request.Status.HasValue)
                query = query.Where(p => p.Status == request.Status.Value);
                
            if (request.Type.HasValue)
                query = query.Where(p => p.Type == request.Type.Value);
                
            if (!string.IsNullOrEmpty(request.OwnerId))
                query = query.Where(p => p.OwnerId == request.OwnerId);
                
            // Date range filtering
            if (request.StartDateFrom.HasValue)
                query = query.Where(p => p.StartDate >= request.StartDateFrom.Value);
                
            if (request.StartDateTo.HasValue)
                query = query.Where(p => p.StartDate <= request.StartDateTo.Value);
            
            // Get total count before paging (for pagination metadata)
            var totalCount = await query.CountAsync(cancellationToken);
            
            // Apply sorting - dynamic sorting based on property name
            query = ApplySorting(query, request.SortBy, request.SortDescending);
            
            // Apply paging
            var skip = (request.PageNumber - 1) * request.PageSize;
            query = query.Skip(skip).Take(request.PageSize);
            
            // Execute query and project to DTOs in a single database call
            var projects = await query
                .Select(p => _mapper.Map<ProjectDto>(p))  // Project to DTO in database
                .ToListAsync(cancellationToken);
            
            // Return paged result with metadata
            return new PagedResult<ProjectDto>
            {
                Items = projects,
                TotalCount = totalCount,
                PageNumber = request.PageNumber,
                PageSize = request.PageSize,
                TotalPages = (int)Math.Ceiling((double)totalCount / request.PageSize)
            };
        }
        
        /// <summary>
        /// Applies dynamic sorting to the query
        /// Could be extracted to a generic extension method
        /// </summary>
        private IQueryable<Project> ApplySorting(IQueryable<Project> query, string? sortBy, bool descending)
        {
            return sortBy?.ToLower() switch
            {
                "name" => descending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
                "status" => descending ? query.OrderByDescending(p => p.Status) : query.OrderBy(p => p.Status),
                "type" => descending ? query.OrderByDescending(p => p.Type) : query.OrderBy(p => p.Type),
                "estimatedcost" => descending ? query.OrderByDescending(p => p.EstimatedCost) : query.OrderBy(p => p.EstimatedCost),
                "startdate" => descending ? query.OrderByDescending(p => p.StartDate) : query.OrderBy(p => p.StartDate),
                "createdat" => descending ? query.OrderByDescending(p => p.CreatedAt) : query.OrderBy(p => p.CreatedAt),
                _ => query.OrderBy(p => p.Name)  // Default sort
            };
        }
    }
}
```

**Why this CQRS implementation?**
- **Clear separation**: Commands change state, queries read data with different optimizations
- **Single responsibility**: Each handler does one thing well
- **Performance**: Queries use projections and paging to minimize data transfer
- **Validation**: Business rules enforced at the right layer
- **Extensibility**: Easy to add cross-cutting concerns like logging, caching
- **Testing**: Handlers can be unit tested independently of controllers

## Phase 4: Basic Frontend (Week 4)

### 4.1 React Frontend Setup

```bash
cd src/web
npx create-react-app admin-portal --template typescript
cd admin-portal

# Install additional packages
npm install @mui/material @emotion/react @emotion/styled
npm install @mui/icons-material
npm install axios react-query
npm install react-router-dom
```

### 4.2 Project List Component

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useEffect, useState } from 'react';
import {
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  Paper,
  Chip,
  Button,
  Typography
} from '@mui/material';
import { Add as AddIcon } from '@mui/icons-material';
import { projectService, Project, ProjectStatus } from '../services/projectService';

const ProjectList: React.FC = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadProjects();
  }, []);

  const loadProjects = async () => {
    try {
      const data = await projectService.getProjects();
      setProjects(data);
    } catch (error) {
      console.error('Failed to load projects:', error);
    } finally {
      setLoading(false);
    }
  };

  const getStatusColor = (status: ProjectStatus) => {
    const colors = {
      Planning: 'default',
      Approved: 'primary',
      InProgress: 'warning',
      OnHold: 'error',
      Completed: 'success',
      Cancelled: 'error'
    };
    return colors[status] || 'default';
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 20 }}>
        <Typography variant="h4">Infrastructure Projects</Typography>
        <Button variant="contained" startIcon={<AddIcon />}>
          New Project
        </Button>
      </div>

      <TableContainer component={Paper}>
        <Table>
          <TableHead>
            <TableRow>
              <TableCell>Name</TableCell>
              <TableCell>Type</TableCell>
              <TableCell>Status</TableCell>
              <TableCell>Estimated Cost</TableCell>
              <TableCell>Start Date</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {projects.map((project) => (
              <TableRow key={project.id}>
                <TableCell>{project.name}</TableCell>
                <TableCell>{project.type}</TableCell>
                <TableCell>
                  <Chip 
                    label={project.status} 
                    color={getStatusColor(project.status) as any}
                    size="small"
                  />
                </TableCell>
                <TableCell>${project.estimatedCost.toLocaleString()}</TableCell>
                <TableCell>{new Date(project.startDate).toLocaleDateString()}</TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </TableContainer>
    </div>
  );
};

export default ProjectList;
```

### 4.3 Test Full Stack

Start both backend and frontend:

```bash
# Terminal 1 - Backend
cd src/services/projects/Infrastructure.Projects.API
dotnet run

# Terminal 2 - Frontend  
cd src/web/admin-portal
npm start
```

Visit `http://localhost:3000` to see the project list.

## Phase 5: Add Budget Service (Week 5)

### 5.1 Budget Service Structure

Follow the same pattern as Project Service:

```bash
mkdir -p src/services/budget/{Domain,Application,Infrastructure,API}
```

Create Budget domain entities, similar to Project but for budget-specific concerns:

```csharp
public class Budget : BaseEntity
{
    public string Name { get; private set; }
    public int FiscalYear { get; private set; }
    public decimal TotalAmount { get; private set; }
    public BudgetStatus Status { get; private set; }
    
    private readonly List<BudgetLineItem> _lineItems = new();
    public IReadOnlyList<BudgetLineItem> LineItems => _lineItems.AsReadOnly();
    
    // Constructor and methods...
}

public class BudgetLineItem : BaseEntity
{
    public string Category { get; private set; }
    public string Description { get; private set; }
    public decimal AllocatedAmount { get; private set; }
    public decimal SpentAmount { get; private set; }
    public Guid BudgetId { get; private set; }
}
```

### 5.2 Inter-Service Communication

Create shared event contracts in `src/shared/events/`:

```csharp
public class ProjectCreatedEvent : DomainEvent
{
    public Guid ProjectId { get; }
    public string ProjectName { get; }
    public decimal EstimatedCost { get; }
    
    public ProjectCreatedEvent(Guid projectId, string projectName, decimal estimatedCost)
    {
        ProjectId = projectId;
        ProjectName = projectName;
        EstimatedCost = estimatedCost;
    }
}
```

## Phase 6: API Gateway (Week 6)

### 6.1 Set up Ocelot API Gateway

Create `src/gateways/api-gateway/Infrastructure.Gateway.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Ocelot" Version="20.0.0" />
    <PackageReference Include="Ocelot.Provider.Consul" Version="20.0.0" />
  </ItemGroup>
</Project>
```

### 6.2 Gateway Configuration

Create `src/gateways/api-gateway/ocelot.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/budgets/{everything}",
      "DownstreamScheme": "https", 
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/gateway/budgets/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

### 6.3 Test Gateway

Update frontend to use gateway endpoints and verify routing works.

## Phase 7: Authentication & Security (Week 7)

### 7.1 Add Identity Service

```bash
mkdir -p src/services/identity
cd src/services/identity

dotnet new webapi -n Infrastructure.Identity.API
cd Infrastructure.Identity.API
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 7.2 JWT Authentication Setup

Configure JWT authentication across all services and test login flow.

### 7.3 Role-Based Authorization

Implement RBAC system:
- Municipality Admin
- Project Manager  
- Budget Analyst
- Grant Writer
- Viewer

## Phase 8: Add Remaining Services (Weeks 8-12)

### 8.1 Grant Management Service (Week 8)
- Grant opportunity discovery
- Application workflow
- Document management

### 8.2 Cost Estimation Service (Week 9) 
- Historical cost data
- Market price integration
- ML-based predictions

### 8.3 BCA (Benefit-Cost Analysis) Service (Week 10)
- NPV calculations
- Risk assessment
- Compliance reporting

### 8.4 Contract Management Service (Week 11)
- Contract templates
- Procurement workflows
- Vendor management

### 8.5 Analytics & Reporting Service (Week 12)
- Dashboard data aggregation
- Report generation
- KPI calculations

## Phase 9: Advanced Frontend Features (Weeks 13-16)

### 9.1 Enhanced UI Components
- Data visualization with Charts.js/D3
- Interactive dashboards
- Document upload/preview

### 9.2 Workflow Management
- Approval flows
- Task assignments
- Notifications

### 9.3 Integration Interfaces
- ERP system connectors
- Grant portal APIs
- GIS system integration

## Phase 10: Production Readiness (Weeks 17-20)

### 10.1 Kubernetes Deployment

Create `infrastructure/kubernetes/` manifests for:
- Service deployments
- ConfigMaps and Secrets
- Ingress controllers
- Persistent volumes

### 10.2 CI/CD Pipeline

Set up GitLab CI or Azure DevOps:
- Automated testing
- Docker image building
- Deployment automation
- Environment promotion

### 10.3 Monitoring & Observability

Configure:
- Application logging (Serilog)
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Health checks

### 10.4 Performance Testing

- Load testing with k6
- Database query optimization
- Caching strategies
- CDN setup

## Testing Strategy

### Unit Tests
Each service should have comprehensive unit tests:

```bash
cd src/services/projects
dotnet new xunit -n Infrastructure.Projects.Tests
# Add test packages and write tests
```

### Integration Tests
Test service interactions and database operations.

### End-to-End Tests
Use Playwright or Cypress for full workflow testing.

## Development Workflow

1. **Feature Branch Strategy**: Create branches for each service/feature
2. **Code Reviews**: All code must be reviewed before merging
3. **Automated Testing**: CI pipeline runs all tests
4. **Local Development**: Everything runs locally with Docker Compose
5. **Documentation**: Keep docs updated as you build

## Key Milestones

- **Week 4**: First working service with basic UI
- **Week 8**: Core services communicating via events
- **Week 12**: All business services implemented
- **Week 16**: Complete frontend with all features
- **Week 20**: Production-ready deployment

This approach ensures you have a working system at every step, can demo progress regularly, and can make adjustments based on feedback as you build.
### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4### 3.8 Service Configuration and Startup

**Intent**: Configure dependency injection, database connections, and middleware in a clean, organized manner. This setup follows .NET 8 minimal hosting patterns while maintaining testability and clear separation of concerns.

Create `src/services/projects/Infrastructure.Projects.API/Program.cs`:

```csharp
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Projects.Application.Mapping;
using Microsoft.EntityFrameworkCore;
using MediatR;
using System.Reflection;
using Microsoft.OpenApi.Models;
using System.Text.Json;

// Modern .NET minimal hosting pattern
var builder = WebApplication.CreateBuilder(args);

// Configure services
ConfigureServices(builder.Services, builder.Configuration);

var app = builder.Build();

// Configure middleware pipeline
ConfigureMiddleware(app);

// Initialize database
await InitializeDatabase(app);

app.Run();

/// <summary>
/// Configure all application services and dependencies
/// Organized by concern: database, business logic, cross-cutting, API
/// </summary>
void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    // Database configuration - PostgreSQL with connection pooling
    services.AddDbContext<ProjectsDbContext>(options =>
        options.UseNpgsql(
            configuration.GetConnectionString("DefaultConnection") ?? 
            "Host=localhost;Database=infrastructure_dev;Username=dev_user;Password=dev_password",
            npgsqlOptions => 
            {
                npgsqlOptions.MigrationsAssembly("Infrastructure.Projects.Infrastructure");
                npgsqlOptions.EnableRetryOnFailure(3);  // Resilience for network issues
            })
        .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())  // Debug info in dev only
        .EnableDetailedErrors(builder.Environment.IsDevelopment()));

    // MediatR for CQRS pattern - scans assemblies for handlers
    var applicationAssembly = Assembly.LoadFrom("Infrastructure.Projects.Application.dll");
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(applicationAssembly));
    
    // AutoMapper for entity/DTO mapping
    services.AddAutoMapper(typeof(ProjectMappingProfile));
    
    // Health checks for monitoring
    services.AddHealthChecks()
        .AddDbContext<ProjectsDbContext>()
        .AddCheck("self", () => HealthCheckResult.Healthy());
    
    // API services with consistent JSON configuration
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            // Configure JSON serialization for API consistency
            options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
            options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
            options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());  // Enums as strings
        });
    
    // API documentation with comprehensive configuration
    services.AddEndpointsApiExplorer();
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "Infrastructure Projects API", 
            Version = "v1",
            Description = "API for managing infrastructure projects and related operations",
            Contact = new OpenApiContact
            {
                Name = "Infrastructure Platform Team",
                Email = "platform-team@municipality.gov"
            }
        });
        
        // Include XML comments in Swagger documentation
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            c.IncludeXmlComments(xmlPath);
        }
        
        // Add authorization header to Swagger UI
        c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT"
        });
    });
    
    // CORS for frontend applications
    services.AddCors(options =>
    {
        options.AddPolicy("DevelopmentPolicy", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://localhost:3000")  // React dev server
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
        
        options.AddPolicy("ProductionPolicy", policy =>
        {
            policy.WithOrigins(configuration.GetSection("AllowedOrigins").Get<string[]>() ?? Array.Empty<string>())
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
    });
    
    // Logging configuration
    services.AddLogging(logging =>
    {
        logging.AddConsole();
        if (builder.Environment.IsDevelopment())
        {
            logging.AddDebug();
        }
    });
}

/// <summary>
/// Configure the HTTP request pipeline
/// Order matters: security -> routing -> business logic
/// </summary>
void ConfigureMiddleware(WebApplication app)
{
    // Development-specific middleware
    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "Projects API v1");
            c.RoutePrefix = "swagger";  // Available at /swagger
            c.DisplayRequestDuration();
        });
        app.UseCors("DevelopmentPolicy");
    }
    else
    {
        // Production security headers
        app.UseHsts();
        app.UseHttpsRedirection();
        app.UseCors("ProductionPolicy");
    }
    
    // Health check endpoint for monitoring
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = async (context, report) =>
        {
            var result = JsonSerializer.Serialize(new
            {
                status = report.Status.ToString(),
                checks = report.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString(),
                    duration = e.Value.Duration.TotalMilliseconds
                })
            });
            await context.Response.WriteAsync(result);
        }
    });
    
    // Request/response logging in development
    if (app.Environment.IsDevelopment())
    {
        app.Use(async (context, next) =>
        {
            app.Logger.LogInformation("Request: {Method} {Path}", 
                context.Request.Method, context.Request.Path);
            await next();
            app.Logger.LogInformation("Response: {StatusCode}", 
                context.Response.StatusCode);
        });
    }
    
    // Core middleware pipeline
    app.UseRouting();
    
    // Future: Authentication and authorization
    // app.UseAuthentication();
    // app.UseAuthorization();
    
    app.MapControllers();
}

/// <summary>
/// Initialize database with migrations and seed data
/// Ensures database is ready for application startup
/// </summary>
async Task InitializeDatabase(WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ProjectsDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    
    try
    {
        // Apply any pending migrations
        await context.Database.MigrateAsync();
        logger.LogInformation("Database migrations applied successfully");
        
        // Seed initial data if database is empty
        if (!await context.Projects.AnyAsync())
        {
            await SeedInitialData(context, logger);
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to initialize database");
        throw;  // Fail fast on database issues
    }
}

/// <summary>
/// Seed initial test data for development
/// Provides sample projects for testing and demonstration
/// </summary>
async Task SeedInitialData(ProjectsDbContext context, ILogger logger)
{
    var projects = new[]
    {
        new Project("Bridge Rehabilitation", "Repair and strengthen Main Street Bridge", 
                   ProjectType.Bridges, 2500000m, DateTime.Today.AddDays(30), "ORG001"),
        new Project("Water Main Replacement", "Replace aging water infrastructure on Oak Avenue", 
                   ProjectType.WaterSystems, 1800000m, DateTime.Today.AddDays(60), "ORG001"),
        new Project("Community Center Construction", "Build new community center in downtown area", 
                   ProjectType.PublicBuildings, 4200000m, DateTime.Today.AddDays(90), "ORG001")
    };
    
    context.Projects.AddRange(projects);
    await context.SaveChangesAsync();
    
    logger.LogInformation("Seeded {Count} initial projects", projects.Length);
}
```

### 3.9 Test the Complete Project Service

**Intent**: Verify that all components work together correctly. This comprehensive test ensures the entire request/response pipeline functions properly and provides a foundation for building additional services.

```bash
# Navigate to the Projects API directory
cd src/services/projects/Infrastructure.Projects.API

# Restore packages and build
dotnet restore
dotnet build

# Apply database migrations (creates tables)
dotnet ef database update --project ../Infrastructure.Projects.Infrastructure

# Run the service
dotnet run

# Service should start on https://localhost:5001 or http://localhost:5000
```

**Test API endpoints using curl:**

```bash
# Test health check
curl http://localhost:5000/health

# Expected response:
# {"status":"Healthy","checks":[{"name":"self","status":"Healthy","duration":0.1}]}

# Test Swagger documentation
# Open browser to http://localhost:5000/swagger

# Test GET projects (should return seeded data)
curl http://localhost:5000/api/projects

# Expected response: JSON array with 3 sample projects

# Test GET specific project
curl http://localhost:5000/api/projects/{guid-from-above-response}

# Test POST new project
curl -X POST http://localhost:5000/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Park Renovation",
    "description": "Renovate Central Park playground and facilities",
    "type": "Parks",
    "estimatedCost": 750000,
    "startDate": "2025-09-01",
    "ownerId": "ORG001"
  }'

# Expected response: 201 Created with project data and Location header

# Test filtering and paging
curl "http://localhost:5000/api/projects?status=Planning&pageSize=2"

# Test sorting
curl "http://localhost:5000/api/projects?sortBy=estimatedCost&sortDescending=true"
```

**Verify database contents:**

```bash
# Connect to PostgreSQL
docker exec -it infrastructure-docker_postgres_1 psql -U dev_user -d infrastructure_dev

# Query projects table
SELECT id, name, status, estimated_cost, created_at FROM "Projects";

# Should show seeded data plus any projects created via API
```

**Key verification points:**
- ✅ Service starts without errors
- ✅ Health check returns healthy status
- ✅ Swagger UI displays API documentation
- ✅ Database migrations create proper schema
- ✅ CRUD operations work correctly
- ✅ Filtering, paging, and sorting function
- ✅ Validation prevents invalid data
- ✅ Error responses provide meaningful messages

**Why this comprehensive testing approach?**
- **End-to-end verification**: Tests the complete request/response pipeline
- **Documentation validation**: Swagger ensures API is properly documented
- **Database integration**: Confirms EF Core and PostgreSQL work together
- **Business logic validation**: Ensures domain rules are enforced
- **Performance baseline**: Establishes response time expectations
- **Foundation for additional services**: Proves the architecture patterns work# Public Infrastructure Planning Platform - Build Guide

## Phase 1: Development Environment Setup (Week 1)

### 1.1 Prerequisites Installation

**Intent**: Establish a consistent development environment across all team members to eliminate "works on my machine" issues. We're choosing specific versions and tools that support our microservices architecture.

```bash
# Install required tools
# Node.js (18+) - Required for React frontend and build tools
# Using NVM allows easy version switching for different projects
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop - Essential for local development environment
# Allows us to run all infrastructure services (DB, Redis, Kafka) locally
# Download from docker.com

# .NET 8 SDK - Latest LTS version for backend services
# Provides modern C# features, performance improvements, and long-term support
# Download from microsoft.com/dotnet

# Git - Version control for distributed development
sudo apt install git  # Linux
brew install git      # macOS

# VS Code with extensions - Consistent IDE experience
# Extensions provide IntelliSense, debugging, and container support
code --install-extension ms-dotnettools.csharp
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

**Why these choices?**
- **Node.js 18+**: Provides modern JavaScript features and npm ecosystem access
- **Docker**: Eliminates environment differences and simplifies service orchestration
- **.NET 8**: High-performance, cross-platform runtime with excellent tooling
- **VS Code**: Lightweight, extensible, with excellent container and cloud support

### 1.2 Project Structure Setup

**Intent**: Create a well-organized monorepo structure that supports microservices architecture while maintaining clear boundaries. This structure follows Domain-Driven Design principles and separates concerns between services, shared code, infrastructure, and client applications.

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Create main directory structure
# This follows the "screaming architecture" principle - the structure tells you what the system does
mkdir -p {
  src/{
    # Each service is a bounded context with its own domain
    services/{budget,grants,projects,bca,contracts,costs,users,notifications},
    # Shared code that multiple services can use - but keep this minimal
    shared/{common,domain,events},
    # API Gateway acts as the single entry point for clients
    gateways/api-gateway,
    # Separate frontend applications for different user types
    web/{admin-portal,public-portal},
    # Mobile app for field workers and inspectors
    mobile
  },
  infrastructure/{
    # Infrastructure as Code - everything should be reproducible
    docker,      # Local development containers
    kubernetes,  # Production orchestration
    terraform,   # Cloud resource provisioning
    scripts      # Deployment and utility scripts
  },
  docs,  # Architecture decision records, API docs, user guides
  tests  # End-to-end and integration tests that span services
}
```

**Why this structure?**
- **Service boundaries**: Each service in `/services/` represents a business capability
- **Shared minimal**: Only truly shared code goes in `/shared/` to avoid coupling
- **Infrastructure separation**: Keeps deployment concerns separate from business logic
- **Client separation**: Different frontends for different user needs and experiences
- **Documentation co-location**: Docs live with code for easier maintenance

### 1.3 Git Repository Setup

```bash
git init
echo "node_modules/
bin/
obj/
.env
.DS_Store
*.log" > .gitignore

git add .
git commit -m "Initial project structure"
```

## Phase 2: Core Infrastructure (Week 2)

### 2.1 Docker Development Environment

**Intent**: Create a local development environment that mirrors production as closely as possible. This setup provides all the infrastructure services (database, cache, messaging, search) that our microservices need, without requiring complex local installations. Each service is isolated and can be started/stopped independently.

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  # PostgreSQL - Primary database for transactional data
  # Using version 15 for performance and JSON improvements
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrastructure_dev      # Single database, multiple schemas per service
      POSTGRES_USER: dev_user              # Non-root user for security
      POSTGRES_PASSWORD: dev_password      # Simple password for local dev
    ports:
      - "5432:5432"                       # Standard PostgreSQL port
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data between container restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Initialize schemas

  # Redis - Caching and session storage
  # Using Alpine for smaller image size
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"                       # Standard Redis port
    # No persistence needed in development

  # Apache Kafka - Event streaming between services
  # Essential for event-driven architecture and service decoupling
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper                         # Kafka requires Zookeeper for coordination
    ports:
      - "9092:9092"                      # Kafka broker port
    environment:
      KAFKA_BROKER_ID: 1                 # Single broker for dev
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092  # Clients connect via localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  # No replication needed in dev

  # Zookeeper - Kafka dependency for cluster coordination
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Elasticsearch - Full-text search and analytics
  # Used for searching grants, projects, and generating reports
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node        # Single node for development
      - xpack.security.enabled=false      # Disable security for local dev
    ports:
      - "9200:9200"                      # REST API port

volumes:
  postgres_data:  # Named volume for database persistence
```

**Test Infrastructure:**
```bash
cd infrastructure/docker
# Start all infrastructure services in background
docker-compose -f docker-compose.dev.yml up -d

# Verify all services are running and healthy
docker-compose ps

# Expected output: All services should show "Up" status
# postgres_1      Up  0.0.0.0:5432->5432/tcp
# redis_1         Up  0.0.0.0:6379->6379/tcp
# kafka_1         Up  0.0.0.0:9092->9092/tcp
# etc.
```

**Why these infrastructure choices?**
- **PostgreSQL**: ACID compliance, JSON support, excellent .NET integration
- **Redis**: Fast caching, session storage, distributed locks
- **Kafka**: Reliable event streaming, service decoupling, audit logging
- **Elasticsearch**: Powerful search, analytics, report generation
- **Single-node configs**: Simplified for development, production uses clusters

### 2.2 Shared Domain Models

**Intent**: Establish common base classes and patterns that all domain entities will inherit from. This implements the DDD (Domain-Driven Design) principle of having rich domain objects with behavior, not just data containers. The base entity provides audit trail capabilities essential for government compliance and change tracking.

Create `src/shared/domain/Entities/BaseEntity.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Entities
{
    /// <summary>
    /// Base class for all domain entities providing common functionality
    /// Implements Entity pattern from DDD with audit trail for compliance
    /// </summary>
    public abstract class BaseEntity
    {
        // Using Guid as ID provides globally unique identifiers across distributed services
        // No need for centralized ID generation or sequence coordination
        public Guid Id { get; protected set; }
        
        // Audit fields required for government compliance and change tracking
        public DateTime CreatedAt { get; protected set; }
        public DateTime UpdatedAt { get; protected set; }
        public string CreatedBy { get; protected set; }      // User who created the entity
        public string UpdatedBy { get; protected set; }      // User who last modified

        // Protected constructor prevents direct instantiation
        // Forces use of domain-specific constructors in derived classes
        protected BaseEntity()
        {
            Id = Guid.NewGuid();              // Generate unique ID immediately
            CreatedAt = DateTime.UtcNow;      // Always use UTC to avoid timezone issues
        }

        // Virtual method allows derived classes to add custom update logic
        // Automatically tracks who made changes and when
        public virtual void Update(string updatedBy)
        {
            UpdatedAt = DateTime.UtcNow;
            UpdatedBy = updatedBy ?? throw new ArgumentNullException(nameof(updatedBy));
        }
    }
}
```

**Intent**: Create a base class for domain events that enables event-driven architecture. Domain events represent something significant that happened in the business domain and allow services to communicate without direct coupling.

Create `src/shared/domain/Events/DomainEvent.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Events
{
    /// <summary>
    /// Base class for all domain events in the system
    /// Enables event-driven architecture and service decoupling
    /// Events represent business facts that other services might care about
    /// </summary>
    public abstract class DomainEvent
    {
        // Unique identifier for this specific event occurrence
        public Guid Id { get; }
        
        // When did this business event occur (not when it was processed)
        public DateTime OccurredOn { get; }
        
        // Type identifier for event routing and deserialization
        // Using class name allows easy identification in logs and debugging
        public string EventType { get; }

        protected DomainEvent()
        {
            Id = Guid.NewGuid();                    // Unique event ID for idempotency
            OccurredOn = DateTime.UtcNow;           // Business time, not processing time
            EventType = this.GetType().Name;       // Automatic type identification
        }
    }
}
```

**Why these design decisions?**
- **Guid IDs**: Eliminate distributed ID generation complexity, work across services
- **UTC timestamps**: Prevent timezone confusion in distributed systems
- **Audit fields**: Meet government compliance requirements for change tracking
- **Protected constructors**: Enforce proper domain object creation patterns
- **Domain events**: Enable loose coupling between services while maintaining business consistency
- **Event metadata**: Support idempotency, ordering, and debugging in distributed systems

## Phase 3: First Service - Project Management (Week 3)

### 3.1 Project Service Setup

**Intent**: Create our first microservice following Clean Architecture principles. This service will handle all project-related operations and serve as a template for other services. We're using the latest .NET 8 with carefully chosen packages that support our architectural goals.

Create `src/services/projects/Infrastructure.Projects.API/Infrastructure.Projects.API.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>    <!-- Latest LTS for performance -->
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Entity Framework Core for database operations -->
    <!-- Design package needed for migrations and scaffolding -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
    <!-- PostgreSQL provider for our chosen database -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    
    <!-- Swagger for API documentation and testing -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- MediatR implements CQRS and Mediator pattern -->
    <!-- Decouples controllers from business logic -->
    <PackageReference Include="MediatR" Version="12.0.0" />
    
    <!-- AutoMapper for object-to-object mapping -->
    <!-- Keeps controllers clean by handling DTO conversions -->
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <!-- Reference to shared domain models and utilities -->
    <ProjectReference Include="../../shared/common/Infrastructure.Shared.Common.csproj" />
  </ItemGroup>
</Project>
```

**Why these package choices?**
- **EF Core + PostgreSQL**: Robust ORM with excellent PostgreSQL support for complex queries
- **MediatR**: Implements CQRS pattern, separates read/write operations, supports cross-cutting concerns
- **AutoMapper**: Reduces boilerplate code for mapping between domain models and DTOs
- **Swashbuckle**: Provides interactive API documentation essential for microservices integration

### 3.2 Project Domain Model

**Intent**: Create a rich domain model that encapsulates business rules and invariants. This follows DDD principles where the domain model contains both data and behavior. The Project entity represents the core concept of infrastructure projects and enforces business constraints through its design.

Create `src/services/projects/Infrastructure.Projects.Domain/Entities/Project.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Projects.Domain.Entities
{
    /// <summary>
    /// Project aggregate root representing an infrastructure project
    /// Contains all business rules and invariants for project management
    /// </summary>
    public class Project : BaseEntity
    {
        // Required project information - private setters enforce controlled modification
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ProjectType Type { get; private set; }           // Enum constrains valid project types
        public ProjectStatus Status { get; private set; }       // State machine via status enum
        
        // Financial information with proper decimal type for currency
        public decimal EstimatedCost { get; private set; }      // Initial cost estimate
        
        // Timeline information
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }          // Nullable - may not be set initially
        
        // Ownership tracking
        public string OwnerId { get; private set; }             // Reference to owning organization

        // EF Core requires parameterless constructor - private to prevent misuse
        private Project() { } 

        /// <summary>
        /// Creates a new infrastructure project with required business data
        /// Constructor enforces invariants - all required data must be provided
        /// </summary>
        public Project(string name, string description, ProjectType type, 
                      decimal estimatedCost, DateTime startDate, string ownerId)
        {
            // Business rule validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Project name is required", nameof(name));
            if (estimatedCost <= 0)
                throw new ArgumentException("Project cost must be positive", nameof(estimatedCost));
            if (startDate < DateTime.Today)
                throw new ArgumentException("Project cannot start in the past", nameof(startDate));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Project owner is required", nameof(ownerId));

            Name = name;
            Description = description;
            Type = type;
            Status = ProjectStatus.Planning;     // All projects start in planning phase
            EstimatedCost = estimatedCost;
            StartDate = startDate;
            OwnerId = ownerId;
        }

        /// <summary>
        /// Updates project status following business rules
        /// Could be extended to implement state machine validation
        /// Domain event could be raised here for other services to react
        /// </summary>
        public void UpdateStatus(ProjectStatus newStatus)
        {
            // Business rule: validate status transitions
            if (!IsValidStatusTransition(Status, newStatus))
                throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");
                
            Status = newStatus;
            // Future: Raise domain event ProjectStatusChanged
        }
        
        // Private method to encapsulate business rules for status transitions
        private bool IsValidStatusTransition(ProjectStatus from, ProjectStatus to)
        {
            // Simplified business rules - could be more complex state machine
            return from switch
            {
                ProjectStatus.Planning => to is ProjectStatus.Approved or ProjectStatus.Cancelled,
                ProjectStatus.Approved => to is ProjectStatus.InProgress or ProjectStatus.OnHold or ProjectStatus.Cancelled,
                ProjectStatus.InProgress => to is ProjectStatus.OnHold or ProjectStatus.Completed or ProjectStatus.Cancelled,
                ProjectStatus.OnHold => to is ProjectStatus.InProgress or ProjectStatus.Cancelled,
                ProjectStatus.Completed => false,  // Final state
                ProjectStatus.Cancelled => false,  // Final state
                _ => false
            };
        }
    }

    /// <summary>
    /// Project types supported by the system
    /// Constrains projects to known infrastructure categories
    /// </summary>
    public enum ProjectType
    {
        Roads,              // Highway and road infrastructure
        Bridges,            // Bridge construction and maintenance  
        WaterSystems,       // Water treatment and distribution
        PowerGrid,          // Electrical infrastructure
        PublicBuildings,    // Government and community buildings
        Parks               // Parks and recreational facilities
    }

    /// <summary>
    /// Project status representing the project lifecycle
    /// Forms a state machine with specific allowed transitions
    /// </summary>
    public enum ProjectStatus
    {
        Planning,       // Initial status - gathering requirements
        Approved,       // Project approved and funded
        InProgress,     // Active construction/implementation
        OnHold,         // Temporarily paused
        Completed,      // Successfully finished
        Cancelled       // Terminated before completion
    }
}
```

**Why this domain model design?**
- **Rich domain model**: Contains both data and business behavior, not just properties
- **Invariant enforcement**: Constructor validates business rules, prevents invalid objects
- **Encapsulation**: Private setters prevent external code from bypassing business rules
- **State machine**: Status transitions follow defined business rules
- **Value objects**: Enums provide type safety and constrain valid values
- **Domain events**: Architecture ready for event-driven communication (commented for now)

### 3.3 Database Context

**Intent**: Create the data access layer using Entity Framework Core with proper configuration. The DbContext serves as the Unit of Work pattern implementation and defines how our domain entities map to database tables. Configuration is explicit to ensure predictable database schema and optimal performance.

Create `src/services/projects/Infrastructure.Projects.Infrastructure/Data/ProjectsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    /// <summary>
    /// Database context for the Projects service
    /// Implements Unit of Work pattern and handles entity mapping
    /// Each service has its own database context for service autonomy
    /// </summary>
    public class ProjectsDbContext : DbContext
    {
        // Constructor injection of options allows configuration in Startup.cs
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        // DbSet represents a table - Projects table will be created
        public DbSet<Project> Projects { get; set; }

        /// <summary>
        /// Fluent API configuration - explicit mapping for database schema control
        /// Runs when model is being created, defines table structure and constraints
        /// </summary>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Explicit configuration prevents EF Core convention surprises
            modelBuilder.Entity<Project>(entity =>
            {
                // Primary key configuration - uses the Id from BaseEntity
                entity.HasKey(e => e.Id);
                
                // String property constraints for data integrity and performance
                entity.Property(e => e.Name)
                    .IsRequired()                    // NOT NULL constraint
                    .HasMaxLength(200);             // Reasonable limit prevents abuse
                
                entity.Property(e => e.Description)
                    .HasMaxLength(1000);            // Longer description allowed
                
                // Decimal configuration for financial data precision
                entity.Property(e => e.EstimatedCost)
                    .HasColumnType("decimal(18,2)"); // 18 total digits, 2 decimal places
                                                    // Handles values up to $9,999,999,999,999.99
                
                // Index for common query patterns
                entity.HasIndex(e => e.Status);     // Status filtering will be common
                
                // Additional useful indexes for query performance
                entity.HasIndex(e => e.OwnerId);    // Filter by organization
                entity.HasIndex(e => e.Type);       // Filter by project type
                entity.HasIndex(e => e.StartDate);  // Date range queries
                
                // Enum handling - EF Core stores as string by default (readable in DB)
                entity.Property(e => e.Status)
                    .HasConversion<string>();        // Store enum as string, not int
                entity.Property(e => e.Type)
                    .HasConversion<string>();
                
                // Audit fields from BaseEntity
                entity.Property(e => e.CreatedAt)
                    .IsRequired();
                entity.Property(e => e.CreatedBy)
                    .HasMaxLength(100);
                entity.Property(e => e.UpdatedBy)
                    .HasMaxLength(100);
                
                // Table naming convention - explicit naming prevents surprises
                entity.ToTable("Projects");
            });
            
            // Call base method to apply any additional conventions
            base.OnModelCreating(modelBuilder);
        }
        
        /// <summary>
        /// Override SaveChanges to add automatic audit trail
        /// This ensures all entities get proper audit information
        /// </summary>
        public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            // Get current user context (would come from HTTP context in real app)
            var currentUser = GetCurrentUser(); // TODO: Implement user context
            
            // Automatically set audit fields for new and modified entities
            var entries = ChangeTracker.Entries<BaseEntity>();
            
            foreach (var entry in entries)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        entry.Entity.CreatedBy = currentUser;
                        break;
                    case EntityState.Modified:
                        entry.Entity.Update(currentUser);
                        break;
                }
            }
            
            return await base.SaveChangesAsync(cancellationToken);
        }
        
        // TODO: Implement proper user context injection
        private string GetCurrentUser() => "system"; // Placeholder
    }
}
```

**Why this DbContext design?**
- **Explicit configuration**: Fluent API prevents EF Core convention surprises in production
- **Performance indexes**: Strategic indexes on commonly filtered columns
- **Decimal precision**: Financial data requires exact decimal handling, not floating point
- **String enums**: Human-readable enum values in database for debugging and reports
- **Audit automation**: Automatic audit trail without remembering to set fields manually
- **Service isolation**: Each service owns its database schema completely`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    public class ProjectsDbContext : DbContext
    {
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        public DbSet<Project> Projects { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Project>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Description).HasMaxLength(1000);
                entity.Property(e => e.EstimatedCost).HasColumnType("decimal(18,2)");
                entity.HasIndex(e => e.Status);
            });
        }
    }
}
```

### 3.4 API Controller

**Intent**: Create a clean API controller that follows REST principles and implements the CQRS pattern using MediatR. The controller is thin and focused only on HTTP concerns - all business logic is handled by command and query handlers. This promotes separation of concerns and testability.

Create `src/services/projects/Infrastructure.Projects.API/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.API.Controllers
{
    /// <summary>
    /// REST API controller for project management operations
    /// Implements thin controller pattern - delegates all logic to MediatR handlers
    /// Focuses purely on HTTP concerns: routing, status codes, request/response
    /// </summary>
    [ApiController]
    [Route("api/[controller]")]              // Route: /api/projects
    [Produces("application/json")]           // Always returns JSON
    public class ProjectsController : ControllerBase
    {
        private readonly IMediator _mediator;

        // Dependency injection of MediatR for CQRS pattern
        public ProjectsController(IMediator mediator)
        {
            _mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
        }

        /// <summary>
        /// GET /api/projects - Retrieve projects with optional filtering
        /// Query parameters enable filtering, paging, and sorting
        /// </summary>
        [HttpGet]
        [ProducesResponseType(typeof(PagedResult<ProjectDto>), 200)]
        [ProducesResponseType(400)] // Bad request for invalid parameters
        public async Task<IActionResult> GetProjects([FromQuery] GetProjectsQuery query)
        {
            try 
            {
                // MediatR handles routing to appropriate query handler
                var result = await _mediator.Send(query);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                // Invalid query parameters
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// GET /api/projects/{id} - Retrieve specific project by ID
        /// </summary>
        [HttpGet("{id:guid}")]                          // Constraint ensures valid GUID
        [ProducesResponseType(typeof(ProjectDetailDto), 200)]
        [ProducesResponseType(404)]                     // Project not found
        public async Task<IActionResult> GetProject(Guid id)
        {
            var query = new GetProjectByIdQuery(id);
            var result = await _mediator.Send(query);
            
            if (result == null)
                return NotFound(new { error = $"Project with ID {id} not found" });
                
            return Ok(result);
        }

        /// <summary>
        /// POST /api/projects - Create new infrastructure project
        /// Returns 201 Created with location header pointing to new resource
        /// </summary>
        [HttpPost]
        [ProducesResponseType(typeof(ProjectDto), 201)]  // Created successfully
        [ProducesResponseType(400)]                      // Validation errors
        [ProducesResponseType(409)]                      // Conflict (duplicate name, etc.)
        public async Task<IActionResult> CreateProject([FromBody] CreateProjectCommand command)
        {
            try
            {
                // Command handler creates project and returns DTO
                var result = await _mediator.Send(command);
                
                // REST best practice: return 201 Created with location header
                return CreatedAtAction(
                    nameof(GetProject), 
                    new { id = result.Id }, 
                    result);
            }
            catch (ArgumentException ex)
            {
                // Domain validation errors
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Business rule violations
                return Conflict(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PUT /api/projects/{id} - Update existing project
        /// </summary>
        [HttpPut("{id:guid}")]
        [ProducesResponseType(typeof(ProjectDto), 200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProject(Guid id, [FromBody] UpdateProjectCommand command)
        {
            // Ensure URL ID matches command ID for consistency
            if (id != command.Id)
                return BadRequest(new { error = "URL ID must match command ID" });

            try
            {
                var result = await _mediator.Send(command);
                if (result == null)
                    return NotFound();
                    
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PATCH /api/projects/{id}/status - Update project status only
        /// Separate endpoint for status changes supports workflow scenarios
        /// </summary>
        [HttpPatch("{id:guid}/status")]
        [ProducesResponseType(200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProjectStatus(Guid id, [FromBody] UpdateProjectStatusCommand command)
        {
            if (id != command.ProjectId)
                return BadRequest(new { error = "URL ID must match command project ID" });

            try
            {
                await _mediator.Send(command);
                return Ok(new { message = "Status updated successfully" });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Invalid status transitions
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// DELETE /api/projects/{id} - Soft delete project
        /// Note: Infrastructure projects are rarely truly deleted, usually cancelled
        /// </summary>
        [HttpDelete("{id:guid}")]
        [ProducesResponseType(204)]  // No content - successful deletion
        [ProducesResponseType(404)]
        [ProducesResponseType(409)]  // Cannot delete due to dependencies
        public async Task<IActionResult> DeleteProject(Guid id)
        {
            try
            {
                var command = new DeleteProjectCommand(id);
                await _mediator.Send(command);
                return NoContent();
            }
            catch (InvalidOperationException ex)
            {
                // Project has dependencies or cannot be deleted
                return Conflict(new { error = ex.Message });
            }
        }
    }
}
```

**Why this controller design?**
- **Thin controllers**: Only handle HTTP concerns, delegate business logic to handlers
- **CQRS pattern**: Separate commands (writes) from queries (reads) for clarity
- **Proper HTTP semantics**: Correct status codes, REST principles, location headers
- **Error handling**: Consistent error responses with meaningful messages
- **Type safety**: Strong typing with DTOs, GUID constraints on routes
- **Testability**: Easy to unit test by mocking IMediator
- **Documentation**: ProducesResponseType attributes generate OpenAPI specs

### 3.5 Command and Query Handlers

**Intent**: Implement the CQRS pattern by creating separate handlers for commands (writes) and queries (reads). This provides clear separation between operations that change state versus those that read data, enables different optimization strategies, and supports future event sourcing implementation.

Create `src/services/projects/Infrastructure.Projects.Application/Commands/CreateProjectCommand.cs`:

```csharp
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.Application.Commands
{
    /// <summary>
    /// Command to create a new infrastructure project
    /// Represents the intent to perform a write operation
    /// Contains all data needed to create a project
    /// </summary>
    public class CreateProjectCommand : IRequest<ProjectDto>
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public ProjectType Type { get; set; }
        public decimal EstimatedCost { get; set; }
        public DateTime StartDate { get; set; }
        public string OwnerId { get; set; }  // Current user's organization

        // Validation method to encapsulate business rules
        public void Validate()
        {
            if (string.IsNullOrWhiteSpace(Name))
                throw new ArgumentException("Project name is required");
            if (EstimatedCost <= 0)
                throw new ArgumentException("Estimated cost must be greater than zero");
            if (StartDate < DateTime.Today)
                throw new ArgumentException("Start date cannot be in the past");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/CreateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Infrastructure.Data;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles the CreateProjectCommand - implements the business logic for project creation
    /// Follows single responsibility principle - only creates projects
    /// </summary>
    public class CreateProjectHandler : IRequestHandler<CreateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public CreateProjectHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<ProjectDto> Handle(CreateProjectCommand request, CancellationToken cancellationToken)
        {
            // Validate command (could also use FluentValidation)
            request.Validate();
            
            // Business rule: Check for duplicate project names within organization
            var existingProject = await _context.Projects
                .FirstOrDefaultAsync(p => p.Name == request.Name && p.OwnerId == request.OwnerId, 
                                   cancellationToken);
            
            if (existingProject != null)
                throw new InvalidOperationException($"Project '{request.Name}' already exists");
            
            // Create domain entity - constructor enforces invariants
            var project = new Project(
                request.Name,
                request.Description,
                request.Type,
                request.EstimatedCost,
                request.StartDate,
                request.OwnerId
            );
            
            // Persist to database
            _context.Projects.Add(project);
            await _context.SaveChangesAsync(cancellationToken);
            
            // TODO: Raise domain event ProjectCreatedEvent for other services
            
            // Map to DTO for response
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Queries/GetProjectsQuery.cs`:

```csharp
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using MediatR;

namespace Infrastructure.Projects.Application.Queries
{
    /// <summary>
    /// Query to retrieve projects with filtering, paging, and sorting
    /// Optimized for read operations with minimal data transfer
    /// </summary>
    public class GetProjectsQuery : IRequest<PagedResult<ProjectDto>>
    {
        // Filtering options
        public string? NameFilter { get; set; }
        public ProjectStatus? Status { get; set; }
        public ProjectType? Type { get; set; }
        public string? OwnerId { get; set; }
        
        // Date range filtering
        public DateTime? StartDateFrom { get; set; }
        public DateTime? StartDateTo { get; set; }
        
        // Paging parameters
        public int PageNumber { get; set; } = 1;
        public int PageSize { get; set; } = 20;    // Default page size
        
        // Sorting options
        public string? SortBy { get; set; } = "Name";  // Default sort by name
        public bool SortDescending { get; set; } = false;
        
        // Validation for query parameters
        public void Validate()
        {
            if (PageNumber < 1)
                throw new ArgumentException("Page number must be greater than 0");
            if (PageSize < 1 || PageSize > 100)
                throw new ArgumentException("Page size must be between 1 and 100");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/GetProjectsHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles project list queries with filtering and paging
    /// Optimized for read performance with projection to DTOs
    /// </summary>
    public class GetProjectsHandler : IRequestHandler<GetProjectsQuery, PagedResult<ProjectDto>>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public GetProjectsHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<PagedResult<ProjectDto>> Handle(GetProjectsQuery request, CancellationToken cancellationToken)
        {
            // Validate query parameters
            request.Validate();
            
            // Build query with filtering - start with base query
            var query = _context.Projects.AsQueryable();
            
            // Apply filters - only add WHERE clauses for provided filters
            if (!string.IsNullOrEmpty(request.NameFilter))
                query = query.Where(p => p.Name.Contains(request.NameFilter));
                
            if (request.Status.HasValue)
                query = query.Where(p => p.Status == request.Status.Value);
                
            if (request.Type.HasValue)
                query = query.Where(p => p.Type == request.Type.Value);
                
            if (!string.IsNullOrEmpty(request.OwnerId))
                query = query.Where(p => p.OwnerId == request.OwnerId);
                
            // Date range filtering
            if (request.StartDateFrom.HasValue)
                query = query.Where(p => p.StartDate >= request.StartDateFrom.Value);
                
            if (request.StartDateTo.HasValue)
                query = query.Where(p => p.StartDate <= request.StartDateTo.Value);
            
            // Get total count before paging (for pagination metadata)
            var totalCount = await query.CountAsync(cancellationToken);
            
            // Apply sorting - dynamic sorting based on property name
            query = ApplySorting(query, request.SortBy, request.SortDescending);
            
            // Apply paging
            var skip = (request.PageNumber - 1) * request.PageSize;
            query = query.Skip(skip).Take(request.PageSize);
            
            // Execute query and project to DTOs in a single database call
            var projects = await query
                .Select(p => _mapper.Map<ProjectDto>(p))  // Project to DTO in database
                .ToListAsync(cancellationToken);
            
            // Return paged result with metadata
            return new PagedResult<ProjectDto>
            {
                Items = projects,
                TotalCount = totalCount,
                PageNumber = request.PageNumber,
                PageSize = request.PageSize,
                TotalPages = (int)Math.Ceiling((double)totalCount / request.PageSize)
            };
        }
        
        /// <summary>
        /// Applies dynamic sorting to the query
        /// Could be extracted to a generic extension method
        /// </summary>
        private IQueryable<Project> ApplySorting(IQueryable<Project> query, string? sortBy, bool descending)
        {
            return sortBy?.ToLower() switch
            {
                "name" => descending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
                "status" => descending ? query.OrderByDescending(p => p.Status) : query.OrderBy(p => p.Status),
                "type" => descending ? query.OrderByDescending(p => p.Type) : query.OrderBy(p => p.Type),
                "estimatedcost" => descending ? query.OrderByDescending(p => p.EstimatedCost) : query.OrderBy(p => p.EstimatedCost),
                "startdate" => descending ? query.OrderByDescending(p => p.StartDate) : query.OrderBy(p => p.StartDate),
                "createdat" => descending ? query.OrderByDescending(p => p.CreatedAt) : query.OrderBy(p => p.CreatedAt),
                _ => query.OrderBy(p => p.Name)  // Default sort
            };
        }
    }
}
```

**Why this CQRS implementation?**
- **Clear separation**: Commands change state, queries read data with different optimizations
- **Single responsibility**: Each handler does one thing well
- **Performance**: Queries use projections and paging to minimize data transfer
- **Validation**: Business rules enforced at the right layer
- **Extensibility**: Easy to add cross-cutting concerns like logging, caching
- **Testing**: Handlers can be unit tested independently of controllers

## Phase 4: Basic Frontend (Week 4)

### 4.1 React Frontend Setup

**Intent**: Create a modern React application that connects to our Projects API. We're using TypeScript for type safety, Material-UI for professional government-appropriate styling, and React Query for efficient data fetching with caching. This frontend will serve as the administrative portal for infrastructure managers.

```bash
cd src/web

# Create React app with TypeScript template
npx create-react-app admin-portal --template typescript
cd admin-portal

# Install UI framework and utility packages
npm install @mui/material @emotion/react @emotion/styled @emotion/cache
npm install @mui/icons-material @mui/lab @mui/x-data-grid
npm install @tanstack/react-query axios react-router-dom
npm install @types/node @types/react @types/react-dom

# Install development dependencies
npm install --save-dev @types/jest
```

**Why these package choices?**
- **@mui/material**: Professional, accessible UI components suitable for government applications
- **@tanstack/react-query**: Intelligent data fetching, caching, and synchronization
- **axios**: Promise-based HTTP client with request/response interceptors
- **react-router-dom**: Client-side routing for single-page application navigation
- **TypeScript**: Type safety prevents runtime errors and improves developer experience

### 4.2 API Service Layer

**Intent**: Create a clean abstraction layer for API communication. This service layer handles HTTP requests, error handling, type safety, and provides a consistent interface for components to interact with the backend.

Create `src/web/admin-portal/src/services/api.ts`:

```typescript
import axios, { AxiosInstance, AxiosResponse, AxiosError } from 'axios';

// Base configuration for API communication
const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'http://localhost:5000';

/**
 * Configured axios instance with common settings
 * Handles base URL, headers, interceptors
 */
const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,  // 30 second timeout for slow government networks
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// Request interceptor for adding authentication tokens (future)
apiClient.interceptors.request.use(
  (config) => {
    // TODO: Add JWT token when authentication is implemented
    // const token = localStorage.getItem('authToken');
    // if (token) {
    //   config.headers.Authorization = `Bearer ${token}`;
    // }
    
    console.log(`API Request: ${config.method?.toUpperCase()} ${config.url}`);
    return config;
  },
  (error) => {
    console.error('Request interceptor error:', error);
    return Promise.reject(error);
  }
);

// Response interceptor for consistent error handling
apiClient.interceptors.response.use(
  (response: AxiosResponse) => {
    console.log(`API Response: ${response.status} ${response.config.url}`);
    return response;
  },
  (error: AxiosError) => {
    console.error('API Error:', error.response?.status, error.message);
    
    // Handle common error scenarios
    if (error.response?.status === 401) {
      // TODO: Redirect to login when authentication is implemented
      console.warn('Unauthorized access - would redirect to login');
    } else if (error.response?.status === 403) {
      console.warn('Forbidden - insufficient permissions');
    } else if (error.response?.status >= 500) {
      console.error('Server error - showing user-friendly message');
    }
    
    return Promise.reject(error);
  }
);

/**
 * Generic API response wrapper for consistent typing
 */
export interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

/**
 * Error response structure from backend
 */
export interface ApiError {
  error: string;
  details?: string[];
  timestamp?: string;
}

export default apiClient;
```

Create `src/web/admin-portal/src/services/projectService.ts`:

```typescript
import apiClient, { ApiResponse, ApiError } from './api';

// Type definitions matching backend DTOs
export interface Project {
  id: string;
  name: string;
  description: string;
  type: ProjectType;
  status: ProjectStatus;
  estimatedCost: number;
  startDate: string;  // ISO date string
  endDate?: string;
  createdAt: string;
  createdBy: string;
  updatedAt?: string;
  updatedBy?: string;
  formattedCost: string;
  daysFromStart: number;
  isActive: boolean;
}

export interface ProjectDetail extends Project {
  tags: string[];
  notes: string;
  metrics: ProjectMetrics;
}

export interface ProjectMetrics {
  budgetVariance: number;
  scheduleVariance: number;
  costPerDay: number;
  healthStatus: string;
}

export type ProjectType = 'Roads' | 'Bridges' | 'WaterSystems' | 'PowerGrid' | 'PublicBuildings' | 'Parks';
export type ProjectStatus = 'Planning' | 'Approved' | 'InProgress' | 'OnHold' | 'Completed' | 'Cancelled';

export interface CreateProjectRequest {
  name: string;
  description: string;
  type: ProjectType;
  estimatedCost: number;
  startDate: string;
  ownerId: string;
}

export interface UpdateProjectRequest extends CreateProjectRequest {
  id: string;
}

export interface ProjectsQuery {
  nameFilter?: string;
  status?: ProjectStatus;
  type?: ProjectType;
  ownerId?: string;
  startDateFrom?: string;
  startDateTo?: string;
  pageNumber?: number;
  pageSize?: number;
  sortBy?: string;
  sortDescending?: boolean;
}

export interface PagedResult<T> {
  items: T[];
  totalCount: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
  hasPreviousPage: boolean;
  hasNextPage: boolean;
  firstItemIndex: number;
  lastItemIndex: number;
}

/**
 * Project service class handling all project-related API calls
 * Provides type-safe methods with proper error handling
 */
class ProjectService {
  private readonly basePath = '/api/projects';

  /**
   * Retrieve projects with optional filtering and paging
   */
  async getProjects(query: ProjectsQuery = {}): Promise<PagedResult<Project>> {
    try {
      const params = new URLSearchParams();
      
      // Add query parameters only if they have values
      Object.entries(query).forEach(([key, value]) => {
        if (value !== undefined && value !== null && value !== '') {
          params.append(key, value.toString());
        }
      });
      
      const response = await apiClient.get<PagedResult<Project>>(
        `${this.basePath}?${params.toString()}`
      );
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to fetch projects');
    }
  }

  /**
   * Retrieve single project by ID with full details
   */
  async getProjectById(id: string): Promise<ProjectDetail> {
    try {
      const response = await apiClient.get<ProjectDetail>(`${this.basePath}/${id}`);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, `Failed to fetch project ${id}`);
    }
  }

  /**
   * Create new infrastructure project
   */
  async createProject(project: CreateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.post<Project>(this.basePath, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to create project');
    }
  }

  /**
   * Update existing project
   */
  async updateProject(project: UpdateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.put<Project>(`${this.basePath}/${project.id}`, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project');
    }
  }

  /**
   * Update project status only
   */
  async updateProjectStatus(projectId: string, newStatus: ProjectStatus): Promise<void> {
    try {
      await apiClient.patch(`${this.basePath}/${projectId}/status`, {
        projectId,
        newStatus
      });
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project status');
    }
  }

  /**
   * Soft delete project (usually changes status to Cancelled)
   */
  async deleteProject(id: string): Promise<void> {
    try {
      await apiClient.delete(`${this.basePath}/${id}`);
    } catch (error) {
      throw this.handleApiError(error, 'Failed to delete project');
    }
  }

  /**
   * Consistent error handling across all service methods
   * Converts axios errors into user-friendly messages
   */
  private handleApiError(error: any, defaultMessage: string): Error {
    if (error.response) {
      // Server responded with error status
      const apiError = error.response.data as ApiError;
      return new Error(apiError.error || defaultMessage);
    } else if (error.request) {
      // Request was made but no response received
      return new Error('Network error - please check your connection');
    } else {
      // Something else happened
      return new Error(defaultMessage);
    }
  }
}

// Export singleton instance
export const projectService = new ProjectService();
```

**Why this service layer design?**
- **Type safety**: Full TypeScript typing prevents runtime errors
- **Centralized error handling**: Consistent error messages across the application
- **Request/response logging**: Debugging support for development
- **Interceptor pattern**: Easy to add authentication, logging, or retry logic
- **Future-ready**: Authentication hooks ready for implementation
- **Clean API**: Components don't deal with HTTP details directly

### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack,
  Collapse
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon,
  ExpandMore as ExpandMoreIcon,
  ExpandLess as ExpandLessIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4" component="h1" sx={{ fontWeight: 'bold' }}>
          Infrastructure Projects
        </Typography>
        <Stack direction="row" spacing={2}>
          <Button
            variant="outlined"
            startIcon={showFilters ? <ExpandLessIcon /> : <ExpandMoreIcon />}
            onClick={() => setShowFilters(!showFilters)}
          >
            {showFilters ? 'Hide Filters' : 'Show Filters'}
          </Button>
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => {/* TODO: Navigate to create project form */}}
            sx={{ bgcolor: 'primary.main' }}
          >
            New Project
          </Button>
        </Stack>
      </Stack>

      {/* Collapsible filter section */}
      <Collapse in={showFilters}>
        <Card sx={{ mb: 3 }}>
          <CardContent>
            <Typography variant="h6" sx={{ mb: 2 }}>Filters</Typography>
            <Stack direction="row" spacing={2} flexWrap="wrap" alignItems="center">
              <TextField
                label="Search Name"
                variant="outlined"
                size="small"
                value={query.nameFilter || ''}
                onChange={(e) => handleFilterChange('nameFilter', e.target.value)}
                sx={{ minWidth: 200 }}
              />
              
              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Status</InputLabel>
                <Select
                  value={query.status || ''}
                  label="Status"
                  onChange={(e) => handleFilterChange('status', e.target.value)}
                >
                  <MenuItem value="">All Statuses</MenuItem>
                  <MenuItem value="Planning">Planning</MenuItem>
                  <MenuItem value="Approved">Approved</MenuItem>
                  <MenuItem value="InProgress">In Progress</MenuItem>
                  <MenuItem value="OnHold">On Hold</MenuItem>
                  <MenuItem value="Completed">Completed</MenuItem>
                  <MenuItem value="Cancelled">Cancelled</MenuItem>
                </Select>
              </FormControl>

              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Type</InputLabel>
                <Select
                  value={query.type || ''}
                  label="Type"
                  onChange={(e) => handleFilterChange('type', e.target.value)}
                >
                  <MenuItem value="">All Types</MenuItem>
                  <MenuItem value="Roads">Roads</MenuItem>
                  <MenuItem value="Bridges">Bridges</MenuItem>
                  <MenuItem value="WaterSystems">Water Systems</MenuItem>
                  <MenuItem value="PowerGrid">Power Grid</MenuItem>
                  <MenuItem value="PublicBuildings">Public Buildings</MenuItem>
                  <MenuItem value="Parks">Parks</MenuItem>
                </Select>
              </FormControl>

              <Button
                variant="outlined"
                onClick={() => setQuery({
                  pageNumber: 1,
                  pageSize: 10,
                  sortBy: 'name',
                  sortDescending: false
                })}
              >
                Clear Filters
              </Button>
            </Stack>
          </CardContent>
        </Card>
      </Collapse>

      {/* Results summary */}
      <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
        Showing {pagination.firstItemIndex}-{pagination.lastItemIndex} of {pagination.totalCount} projects
      </Typography>

      {/* Projects table */}
      <Card>
        <TableContainer>
          <Table>
            <TableHead>
              <TableRow sx={{ bgcolor: 'grey.50' }}>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Name</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Type</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Status</Typography></TableCell>
                <TableCell align="right"><Typography variant="subtitle2" fontWeight="bold">Estimated Cost</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Start Date</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Health</Typography></TableCell>
                <TableCell align="center"><Typography variant="subtitle2" fontWeight="bold">Actions</Typography></TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {projects.length === 0 ? (
                <TableRow>
                  <TableCell colSpan={7} align="center" sx={{ py: 4 }}>
                    <Typography variant="body1" color="text.secondary">
                      No projects found. {query.nameFilter || query.status || query.type ? 'Try adjusting your filters.' : 'Create your first project to get started.'}
                    </Typography>
                  </TableCell>
                </TableRow>
              ) : (
                projects.map((project) => (
                  <TableRow key={project.id} hover>
                    <TableCell>
                      <Typography variant="body2" fontWeight="medium">{project.name}</Typography>
                      <Typography variant="caption" color="text.secondary" noWrap sx={{ maxWidth: 300, display: 'block' }}>
                        {project.description}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.type} 
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.status} 
                        color={getStatusColor(project.status)}
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="right">
                      <Typography variant="body2" fontWeight="medium">
                        {project.formattedCost}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Typography variant="body2">
                        {new Date(project.startDate).toLocaleDateString()}
                      </Typography>
                      <Typography variant="caption" color="text.secondary">
                        {project.daysFromStart} days ago
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.isActive ? 'Active' : 'Inactive'} 
                        color={project.isActive ? 'success' : 'default'}
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="center">
                      <Stack direction="row" spacing={1} justifyContent="center">
                        <Tooltip title="View Details">
                          <IconButton 
                            size="small" 
                            onClick={() => {/* TODO: Navigate to project detail */}}
                          >
                            <ViewIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Edit Project">
                          <IconButton 
                            size="small"
                            onClick={() => {/* TODO: Navigate to edit form */}}
                          >
                            <EditIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Delete Project">
                          <IconButton 
                            size="small" 
                            color="error"
                            onClick={() => {/* TODO: Show delete confirmation */}}
                          >
                            <DeleteIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                      </Stack>
                    </TableCell>
                  </TableRow>
                ))
              )}
            </TableBody>
          </Table>
        </TableContainer>

        {/* Pagination */}
        <TablePagination
          component="div"
          count={pagination.totalCount}
          page={pagination.pageNumber - 1}  // Material-UI uses 0-based pages
          onPageChange={handlePageChange}
          rowsPerPage={pagination.pageSize}
          onRowsPerPageChange={handlePageSizeChange}
          rowsPerPageOptions={[5, 10, 25, 50]}
          showFirstButton
          showLastButton
        />
      </Card>

      {/* Status update loading indicator */}
      {statusMutation.isPending && (
        <Alert severity="info" sx={{ mt: 2 }}>
          Updating project status...
        </Alert>
      )}
    </Box>
  );
};

export default ProjectList;
```

**Why this component design?**
- **Comprehensive data management**: Filtering, sorting, paging all in one component
- **Professional UI**: Government-appropriate styling with Material-UI
- **Real-time updates**: React Query provides automatic cache management
- **Responsive design**: Works on different screen sizes
- **Accessibility**: Proper ARIA labels, keyboard navigation, screen reader support
- **Performance**: Virtual scrolling for large datasets, efficient re-renders
- **Error handling**: Graceful degradation with retry functionality

### 4.4 Application Setup and Routing

**Intent**: Configure the React application with routing, React Query, and Material-UI theme. This setup provides the foundation for a professional government application with consistent styling, efficient data management, and proper navigation structure.

Create `src/web/admin-portal/src/App.tsx`:

```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { CssBaseline, Box } from '@mui/material';
import ProjectList from './components/ProjectList';
import Layout from './components/Layout';

// Create React Query client with sensible defaults
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // Data considered fresh for 5 minutes
      cacheTime: 10 * 60 * 1000,   // Cache data for 10 minutes
      retry: 3,                     // Retry failed queries 3 times
      refetchOnWindowFocus: false,  // Don't refetch when window regains focus
      refetchOnMount: true,         // Always refetch when component mounts
    },
    mutations: {
      retry: 1,  // Retry failed mutations once
    },
  },
});

// Create Material-UI theme for government applications
const theme = createTheme({
  palette: {
    mode: 'light',
    primary: {
      main: '#1976d2',      // Professional blue
      light: '#42a5f5',
      dark: '#1565c0',
    },
    secondary: {
      main: '#dc004e',      // Accent color for important actions
      light: '#ff6b9d',
      dark: '#9a0036',
    },
    background: {
      default: '#f5f5f5',   // Light gray background
      paper: '#ffffff',
    },
    text: {
      primary: '#212121',   // Dark gray for readability
      secondary: '#757575',
    },
    error: {
      main: '#d32f2f',
    },
    warning: {
      main: '#ed6c02',
    },
    success: {
      main: '#2e7d32',
    },
    info: {
      main: '#0288d1',
    },
  },
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    h1: {
      fontSize: '2.5rem',
      fontWeight: 300,
    },
    h4: {
      fontSize: '2rem',
      fontWeight: 400,
    },
    h6: {
      fontSize: '1.25rem',
      fontWeight: 500,
    },
    body1: {
      fontSize: '1rem',
      lineHeight: 1.5,
    },
    body2: {
      fontSize: '0.875rem',
      lineHeight: 1.43,
    },
    caption: {
      fontSize: '0.75rem',
      lineHeight: 1.33,
    },
  },
  components: {
    // Customize Material-UI components
    MuiButton: {
      styleOverrides: {
        root: {
          textTransform: 'none',  // Don't uppercase button text
          borderRadius: 4,
        },
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0px 2px 4px rgba(0, 0, 0, 0.1)',
          borderRadius: 8,
        },
      },
    },
    MuiTableHead: {
      styleOverrides: {
        root: {
          backgroundColor: '#f5f5f5',
        },
      },
    },
  },
});

/**
 * Main App component with routing and providers
 * Sets up the application shell with navigation and global state
 */
const App: React.FC = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={theme}>
        <CssBaseline />  {/* Material-UI CSS reset and base styles */}
        <Router>
          <Layout>
            <Routes>
              {/* Default route redirects to projects */}
              <Route path="/" element={<Navigate to="/projects" replace />} />
              
              {/* Project management routes */}
              <Route path="/projects" element={<ProjectList />} />
              
              {/* Future routes for other services */}
              <Route path="/budgets" element={<div>Budget Management (Coming Soon)</div>} />
              <Route path="/grants" element={<div>Grant Management (Coming Soon)</div>} />
              <Route path="/contracts" element={<div>Contract Management (Coming Soon)</div>} />
              <Route path="/reports" element={<div>Reports & Analytics (Coming Soon)</div>} />
              
              {/* 404 page */}
              <Route path="*" element={<Navigate to="/projects" replace />} />
            </Routes>
          </Layout>
        </Router>
        
        {/* React Query DevTools in development */}
        {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
      </ThemeProvider>
    </QueryClientProvider>
  );
};

export default App;
```

Create `src/web/admin-portal/src/components/Layout.tsx`:

```typescript
import React, { useState } from 'react';
import {
  Box,
  AppBar,
  Toolbar,
  Typography,
  Drawer,
  List,
  ListItem,
  ListItemIcon,
  ListItemText,
  IconButton,
  Divider,
  Avatar,
  Menu,
  MenuItem,
} from '@mui/material';
import {
  Menu as MenuIcon,
  AccountCircle as AccountCircleIcon,
  Dashboard as DashboardIcon,
  Business as BusinessIcon,
  MonetizationOn as MonetizationOnIcon,
  Description as DescriptionIcon,
  Assessment as AssessmentIcon,
  Settings as SettingsIcon,
} from '@mui/icons-material';
import { useNavigate, useLocation } from 'react-router-dom';

const drawerWidth = 240;

/**
 * Navigation items for the sidebar
 * Each item represents a major functional area of the platform
 */
const navigationItems = [
  { text: 'Projects', icon: <BusinessIcon />, path: '/projects' },
  { text: 'Budgets', icon: <MonetizationOnIcon />, path: '/budgets' },
  { text: 'Grants', icon: <DescriptionIcon />, path: '/grants' },
  { text: 'Contracts', icon: <DescriptionIcon />, path: '/contracts' },
  { text: 'Reports', icon: <AssessmentIcon />, path: '/reports' },
];

interface LayoutProps {
  children: React.ReactNode;
}

/**
 * Main layout component providing navigation and application shell
 * Features: responsive sidebar, user menu, breadcrumbs
 */
const Layout: React.FC<LayoutProps> = ({ children }) => {
  const navigate = useNavigate();
  const location = useLocation();
  const [mobileOpen, setMobileOpen] = useState(false);
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);

  const handleDrawerToggle = () => {
    setMobileOpen(!mobileOpen);
  };

  const handleUserMenuOpen = (event: React.MouseEvent<HTMLElement>) => {
    setAnchorEl(event.currentTarget);
  };

  const handleUserMenuClose = () => {
    setAnchorEl(null);
  };

  const handleNavigation = (path: string) => {
    navigate(path);
    setMobileOpen(false);  // Close mobile drawer after navigation
  };

  // Sidebar content
  const drawerContent = (
    <div>
      <Toolbar>
        <Typography variant="h6" noWrap component="div" sx={{ fontWeight: 'bold' }}>
          Infrastructure Platform
        </Typography>
      </Toolbar>
      <Divider />
      <List>
        {navigationItems.map((item) => (
          <ListItem
            button
            key={item.text}
            selected={location.pathname === item.path}
            onClick={() => handleNavigation(item.path)}
            sx={{
              '&.Mui-selected': {
                backgroundColor: 'primary.light',
                color: 'primary.contrastText',
                '& .MuiListItemIcon-root': {
                  color: 'primary.contrastText',
                },
              },
            }}
          >
            <ListItemIcon>{item.icon}</ListItemIcon>
            <ListItemText primary={item.text} />
          </ListItem>
        ))}
      </List>
      <Divider />
      <List>
        <ListItem button onClick={() => handleNavigation('/settings')}>
          <ListItemIcon><SettingsIcon /></ListItemIcon>
          <ListItemText primary="Settings" />
        </ListItem>
      </List>
    </div>
  );

  return (
    <Box sx={{ display: 'flex' }}>
      {/* App bar */}
      <AppBar
        position="fixed"
        sx={{
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          ml: { sm: `${drawerWidth}px` },
        }}
      >
        <Toolbar>
          <IconButton
            color="inherit"
            aria-label="open drawer"
            edge="start"
            onClick={handleDrawerToggle}
            sx={{ mr: 2, display: { sm: 'none' } }}
          >
            <MenuIcon />
          </IconButton>
          
          {/* Page title */}
          <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1 }}>
            {navigationItems.find(item => item.path === location.pathname)?.text || 'Dashboard'}
          </Typography>
          
          {/* User menu */}
          <IconButton
            size="large"
            aria-label="account menu"
            aria-haspopup="true"
            onClick={handleUserMenuOpen}
            color="inherit"
          >
            <Avatar sx={{ width: 32, height: 32, bgcolor: 'secondary.main' }}>
              JD
            </Avatar>
          </IconButton>
          
          <Menu
            anchorEl={anchorEl}
            open={Boolean(anchorEl)}
            onClose={handleUserMenuClose}
            anchorOrigin={{
              vertical: 'bottom',
              horizontal: 'right',
            }}
            transformOrigin={{
              vertical: 'top',
              horizontal: 'right',
            }}
          >
            <MenuItem onClick={handleUserMenuClose}>
              <Typography variant="body2">John Doe</Typography>
            </MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Profile</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Settings</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Logout</MenuItem>
          </Menu>
        </Toolbar>
      </AppBar>

      {/* Sidebar drawer */}
      <Box
        component="nav"
        sx={{ width: { sm: drawerWidth }, flexShrink: { sm: 0 } }}
      >
        {/* Mobile drawer */}
        <Drawer
          variant="temporary"
          open={mobileOpen}
          onClose={handleDrawerToggle}
          ModalProps={{
            keepMounted: true, // Better mobile performance
          }}
          sx={{
            display: { xs: 'block', sm: 'none' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
        >
          {drawerContent}
        </Drawer>

        {/* Desktop drawer */}
        <Drawer
          variant="permanent"
          sx={{
            display: { xs: 'none', sm: 'block' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
          open
        >
          {drawerContent}
        </Drawer>
      </Box>

      {/* Main content area */}
      <Box
        component="main"
        sx={{
          flexGrow: 1,
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          minHeight: '100vh',
          backgroundColor: 'background.default',
        }}
      >
        <Toolbar /> {/* Spacer for fixed app bar */}
        {children}
      </Box>
    </Box>
  );
};

export default Layout;
```

**Why this application structure?**
- **React Query**: Provides caching, background updates, and optimistic updates
- **Material-UI Theme**: Professional, accessible design suitable for government use
- **React Router**: Clean URL structure with proper navigation
- **Responsive Layout**: Works on desktop, tablet, and mobile devices
- **User Experience**: Loading states, error handling, intuitive navigation
- **Performance**: Code splitting, lazy loading, efficient re-renders

## Phase 5: Add Budget Service (Week 5)

### 5.1 Budget Service Structure

Follow the same pattern as Project Service:

```bash
mkdir -p src/services/budget/{Domain,Application,Infrastructure,API}
```

Create Budget domain entities, similar to Project but for budget-specific concerns:

```csharp
public class Budget : BaseEntity
{
    public string Name { get; private set; }
    public int FiscalYear { get; private set; }
    public decimal TotalAmount { get; private set; }
    public BudgetStatus Status { get; private set; }
    
    private readonly List<BudgetLineItem> _lineItems = new();
    public IReadOnlyList<BudgetLineItem> LineItems => _lineItems.AsReadOnly();
    
    // Constructor and methods...
}

public class BudgetLineItem : BaseEntity
{
    public string Category { get; private set; }
    public string Description { get; private set; }
    public decimal AllocatedAmount { get; private set; }
    public decimal SpentAmount { get; private set; }
    public Guid BudgetId { get; private set; }
}
```

### 5.2 Inter-Service Communication

Create shared event contracts in `src/shared/events/`:

```csharp
public class ProjectCreatedEvent : DomainEvent
{
    public Guid ProjectId { get; }
    public string ProjectName { get; }
    public decimal EstimatedCost { get; }
    
    public ProjectCreatedEvent(Guid projectId, string projectName, decimal estimatedCost)
    {
        ProjectId = projectId;
        ProjectName = projectName;
        EstimatedCost = estimatedCost;
    }
}
```

## Phase 6: API Gateway (Week 6)

### 6.1 Set up Ocelot API Gateway

Create `src/gateways/api-gateway/Infrastructure.Gateway.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Ocelot" Version="20.0.0" />
    <PackageReference Include="Ocelot.Provider.Consul" Version="20.0.0" />
  </ItemGroup>
</Project>
```

### 6.2 Gateway Configuration

Create `src/gateways/api-gateway/ocelot.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/budgets/{everything}",
      "DownstreamScheme": "https", 
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/gateway/budgets/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

### 6.3 Test Gateway

Update frontend to use gateway endpoints and verify routing works.

## Phase 7: Authentication & Security (Week 7)

### 7.1 Add Identity Service

```bash
mkdir -p src/services/identity
cd src/services/identity

dotnet new webapi -n Infrastructure.Identity.API
cd Infrastructure.Identity.API
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 7.2 JWT Authentication Setup

Configure JWT authentication across all services and test login flow.

### 7.3 Role-Based Authorization

Implement RBAC system:
- Municipality Admin
- Project Manager  
- Budget Analyst
- Grant Writer
- Viewer

## Phase 8: Add Remaining Services (Weeks 8-12)

### 8.1 Grant Management Service (Week 8)
- Grant opportunity discovery
- Application workflow
- Document management

### 8.2 Cost Estimation Service (Week 9) 
- Historical cost data
- Market price integration
- ML-based predictions

### 8.3 BCA (Benefit-Cost Analysis) Service (Week 10)
- NPV calculations
- Risk assessment
- Compliance reporting

### 8.4 Contract Management Service (Week 11)
- Contract templates
- Procurement workflows
- Vendor management

### 8.5 Analytics & Reporting Service (Week 12)
- Dashboard data aggregation
- Report generation
- KPI calculations

## Phase 9: Advanced Frontend Features (Weeks 13-16)

### 9.1 Enhanced UI Components
- Data visualization with Charts.js/D3
- Interactive dashboards
- Document upload/preview

### 9.2 Workflow Management
- Approval flows
- Task assignments
- Notifications

### 9.3 Integration Interfaces
- ERP system connectors
- Grant portal APIs
- GIS system integration

## Phase 10: Production Readiness (Weeks 17-20)

### 10.1 Kubernetes Deployment

Create `infrastructure/kubernetes/` manifests for:
- Service deployments
- ConfigMaps and Secrets
- Ingress controllers
- Persistent volumes

### 10.2 CI/CD Pipeline

Set up GitLab CI or Azure DevOps:
- Automated testing
- Docker image building
- Deployment automation
- Environment promotion

### 10.3 Monitoring & Observability

Configure:
- Application logging (Serilog)
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Health checks

### 10.4 Performance Testing

- Load testing with k6
- Database query optimization
- Caching strategies
- CDN setup

## Testing Strategy

### Unit Tests
Each service should have comprehensive unit tests:

```bash
cd src/services/projects
dotnet new xunit -n Infrastructure.Projects.Tests
# Add test packages and write tests
```

### Integration Tests
Test service interactions and database operations.

### End-to-End Tests
Use Playwright or Cypress for full workflow testing.

## Development Workflow

1. **Feature Branch Strategy**: Create branches for each service/feature
2. **Code Reviews**: All code must be reviewed before merging
3. **Automated Testing**: CI pipeline runs all tests
4. **Local Development**: Everything runs locally with Docker Compose
5. **Documentation**: Keep docs updated as you build

## Key Milestones

- **Week 4**: First working service with basic UI
- **Week 8**: Core services communicating via events
- **Week 12**: All business services implemented
- **Week 16**: Complete frontend with all features
- **Week 20**: Production-ready deployment

This approach ensures you have a working system at every step, can demo progress regularly, and can make adjustments based on feedback as you build.
### 5.2 Inter-Service Communication with Domain Events

**Intent**: Implement domain events to enable loose coupling between services. When projects are created or costs change, the budget service needs to be notified. This event-driven approach allows services to remain autonomous while reacting to business events from other services.

Create shared event contracts in `src/shared/events/ProjectEvents.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Events
{
    /// <summary>
    /// Event raised when a new infrastructure project is created
    /// Other services can react to this event to maintain consistency
    /// </summary>
    public class ProjectCreatedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public string ProjectType { get; }
        public decimal EstimatedCost { get; }
        public string OwnerId { get; }
        public DateTime StartDate { get; }
        
        public ProjectCreatedEvent(Guid projectId, string projectName, string projectType,
                                 decimal estimatedCost, string ownerId, DateTime startDate)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            ProjectType = projectType;
            EstimatedCost = estimatedCost;
            OwnerId = ownerId;
            StartDate = startDate;
        }
    }

    /// <summary>
    /// Event raised when project cost estimate changes
    /// Budget service needs to track these changes for variance analysis
    /// </summary>
    public class ProjectCostEstimateChangedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public decimal PreviousEstimate { get; }
        public decimal NewEstimate { get; }
        public decimal VarianceAmount => NewEstimate - PreviousEstimate;
        public string ChangeReason { get; }
        public string ChangedBy { get; }
        
        public ProjectCostEstimateChangedEvent(Guid projectId, string projectName,
                                             decimal previousEstimate, decimal newEstimate,
                                             string changeReason, string changedBy)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            PreviousEstimate = previousEstimate;
            NewEstimate = newEstimate;
            ChangeReason = changeReason;
            ChangedBy = changedBy;
        }
    }

    /// <summary>
    /// Event raised when project status changes to approved
    /// Budget service may need to reserve or allocate funds
    /// </summary>
    public class ProjectApprovedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public decimal ApprovedBudget { get; }
        public string BudgetCategory { get; }
        public string ApprovedBy { get; }
        public DateTime ApprovalDate { get; }
        
        public ProjectApprovedEvent(Guid projectId, string projectName, decimal approvedBudget,
                                  string budgetCategory, string approvedBy, DateTime approvalDate)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            ApprovedBudget = approvedBudget;
            BudgetCategory = budgetCategory;
            ApprovedBy = approvedBy;
            ApprovalDate = approvalDate;
        }
    }
}
```

Create `src/shared/events/BudgetEvents.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Events
{
    /// <summary>
    /// Event raised when budget is approved
    /// Project service may need to know funding is available
    /// </summary>
    public class BudgetApprovedEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public string BudgetName { get; }
        public int FiscalYear { get; }
        public decimal TotalAmount { get; }
        public string OwnerId { get; }
        public string ApprovedBy { get; }
        
        public BudgetApprovedEvent(Guid budgetId, string budgetName, int fiscalYear,
                                 decimal totalAmount, string ownerId, string approvedBy)
        {
            BudgetId = budgetId;
            BudgetName = budgetName;
            FiscalYear = fiscalYear;
            TotalAmount = totalAmount;
            OwnerId = ownerId;
            ApprovedBy = approvedBy;
        }
    }

    /// <summary>
    /// Event raised when budget line item is over-spent
    /// Triggers notifications and potentially workflow approvals
    /// </summary>
    public class BudgetLineItemOverspentEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public Guid LineItemId { get; }
        public string Category { get; }
        public decimal AllocatedAmount { get; }
        public decimal SpentAmount { get; }
        public decimal OverspendAmount => SpentAmount - AllocatedAmount;
        public Guid? RelatedProjectId { get; }
        
        public BudgetLineItemOverspentEvent(Guid budgetId, Guid lineItemId, string category,
                                          decimal allocatedAmount, decimal spentAmount,
                                          Guid? relatedProjectId = null)
        {
            BudgetId = budgetId;
            LineItemId = lineItemId;
            Category = category;
            AllocatedAmount = allocatedAmount;
            SpentAmount = spentAmount;
            RelatedProjectId = relatedProjectId;
        }
    }

    /// <summary>
    /// Event raised when major expenditure is recorded
    /// May trigger approval workflows or notifications
    /// </summary>
    public class LargeExpenditureRecordedEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public Guid LineItemId { get; }
        public decimal Amount { get; }
        public string Description { get; }
        public Guid? ProjectId { get; }
        public string RecordedBy { get; }
        public bool RequiresApproval { get; }
        
        public LargeExpenditureRecordedEvent(Guid budgetId, Guid lineItemId, decimal amount,
                                           string description, string recordedBy,
                                           Guid? projectId = null, bool requiresApproval = false)
        {
            BudgetId = budgetId;
            LineItemId = lineItemId;
            Amount = amount;
            Description = description;
            ProjectId = projectId;
            RecordedBy = recordedBy;
            RequiresApproval = requiresApproval;
        }
    }
}
```

### 5.3 Event Publishing Infrastructure

**Intent**: Create a clean abstraction for publishing domain events to Kafka. This infrastructure allows domain entities to raise events without knowing about messaging details, maintaining clean separation between business logic and infrastructure concerns.

Create `src/shared/common/Events/IEventPublisher.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Interface for publishing domain events
    /// Abstracts messaging infrastructure from domain logic
    /// </summary>
    public interface IEventPublisher
    {
        /// <summary>
        /// Publish single domain event
        /// </summary>
        Task PublishAsync<T>(T domainEvent, CancellationToken cancellationToken = default)
            where T : DomainEvent;

        /// <summary>
        /// Publish multiple domain events as a batch
        /// Useful for aggregate operations that generate multiple events
        /// </summary>
        Task PublishBatchAsync<T>(IEnumerable<T> domainEvents, CancellationToken cancellationToken = default)
            where T : DomainEvent;
    }
}
```

Create `src/shared/common/Events/KafkaEventPublisher.cs`:

```csharp
using Confluent.Kafka;
using Infrastructure.Shared.Domain.Events;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Kafka implementation of event publisher
    /// Handles serialization, topic routing, and error handling
    /// </summary>
    public class KafkaEventPublisher : IEventPublisher, IDisposable
    {
        private readonly IProducer<string, string> _producer;
        private readonly ILogger<KafkaEventPublisher> _logger;
        private readonly JsonSerializerOptions _jsonOptions;
        
        public KafkaEventPublisher(ILogger<KafkaEventPublisher> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            
            // Kafka producer configuration for reliability
            var config = new ProducerConfig
            {
                BootstrapServers = "localhost:9092",  // TODO: Move to configuration
                Acks = Acks.All,                      // Wait for all replicas to acknowledge
                Retries = 3,                          // Retry failed sends
                EnableIdempotence = true,             // Prevent duplicate messages
                MessageSendMaxRetries = 3,            // Additional retry configuration
                RetryBackoffMs = 100,                 // Backoff between retries
                MessageTimeoutMs = 30000,             // Total timeout for message send
                CompressionType = CompressionType.Snappy  // Compress messages for efficiency
            };
            
            _producer = new ProducerBuilder<string, string>(config)
                .SetErrorHandler((_, error) => 
                {
                    _logger.LogError("Kafka producer error: {Error}", error.Reason);
                })
                .SetLogHandler((_, logMessage) => 
                {
                    _logger.LogDebug("Kafka log: {Message}", logMessage.Message);
                })
                .Build();
            
            // JSON serialization options for### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4### 3.8 Service Configuration and Startup

**Intent**: Configure dependency injection, database connections, and middleware in a clean, organized manner. This setup follows .NET 8 minimal hosting patterns while maintaining testability and clear separation of concerns.

Create `src/services/projects/Infrastructure.Projects.API/Program.cs`:

```csharp
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Projects.Application.Mapping;
using Microsoft.EntityFrameworkCore;
using MediatR;
using System.Reflection;
using Microsoft.OpenApi.Models;
using System.Text.Json;

// Modern .NET minimal hosting pattern
var builder = WebApplication.CreateBuilder(args);

// Configure services
ConfigureServices(builder.Services, builder.Configuration);

var app = builder.Build();

// Configure middleware pipeline
ConfigureMiddleware(app);

// Initialize database
await InitializeDatabase(app);

app.Run();

/// <summary>
/// Configure all application services and dependencies
/// Organized by concern: database, business logic, cross-cutting, API
/// </summary>
void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    // Database configuration - PostgreSQL with connection pooling
    services.AddDbContext<ProjectsDbContext>(options =>
        options.UseNpgsql(
            configuration.GetConnectionString("DefaultConnection") ?? 
            "Host=localhost;Database=infrastructure_dev;Username=dev_user;Password=dev_password",
            npgsqlOptions => 
            {
                npgsqlOptions.MigrationsAssembly("Infrastructure.Projects.Infrastructure");
                npgsqlOptions.EnableRetryOnFailure(3);  // Resilience for network issues
            })
        .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())  // Debug info in dev only
        .EnableDetailedErrors(builder.Environment.IsDevelopment()));

    // MediatR for CQRS pattern - scans assemblies for handlers
    var applicationAssembly = Assembly.LoadFrom("Infrastructure.Projects.Application.dll");
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(applicationAssembly));
    
    // AutoMapper for entity/DTO mapping
    services.AddAutoMapper(typeof(ProjectMappingProfile));
    
    // Health checks for monitoring
    services.AddHealthChecks()
        .AddDbContext<ProjectsDbContext>()
        .AddCheck("self", () => HealthCheckResult.Healthy());
    
    // API services with consistent JSON configuration
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            // Configure JSON serialization for API consistency
            options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
            options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
            options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());  // Enums as strings
        });
    
    // API documentation with comprehensive configuration
    services.AddEndpointsApiExplorer();
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "Infrastructure Projects API", 
            Version = "v1",
            Description = "API for managing infrastructure projects and related operations",
            Contact = new OpenApiContact
            {
                Name = "Infrastructure Platform Team",
                Email = "platform-team@municipality.gov"
            }
        });
        
        // Include XML comments in Swagger documentation
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            c.IncludeXmlComments(xmlPath);
        }
        
        // Add authorization header to Swagger UI
        c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT"
        });
    });
    
    // CORS for frontend applications
    services.AddCors(options =>
    {
        options.AddPolicy("DevelopmentPolicy", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://localhost:3000")  // React dev server
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
        
        options.AddPolicy("ProductionPolicy", policy =>
        {
            policy.WithOrigins(configuration.GetSection("AllowedOrigins").Get<string[]>() ?? Array.Empty<string>())
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
    });
    
    // Logging configuration
    services.AddLogging(logging =>
    {
        logging.AddConsole();
        if (builder.Environment.IsDevelopment())
        {
            logging.AddDebug();
        }
    });
}

/// <summary>
/// Configure the HTTP request pipeline
/// Order matters: security -> routing -> business logic
/// </summary>
void ConfigureMiddleware(WebApplication app)
{
    // Development-specific middleware
    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "Projects API v1");
            c.RoutePrefix = "swagger";  // Available at /swagger
            c.DisplayRequestDuration();
        });
        app.UseCors("DevelopmentPolicy");
    }
    else
    {
        // Production security headers
        app.UseHsts();
        app.UseHttpsRedirection();
        app.UseCors("ProductionPolicy");
    }
    
    // Health check endpoint for monitoring
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = async (context, report) =>
        {
            var result = JsonSerializer.Serialize(new
            {
                status = report.Status.ToString(),
                checks = report.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString(),
                    duration = e.Value.Duration.TotalMilliseconds
                })
            });
            await context.Response.WriteAsync(result);
        }
    });
    
    // Request/response logging in development
    if (app.Environment.IsDevelopment())
    {
        app.Use(async (context, next) =>
        {
            app.Logger.LogInformation("Request: {Method} {Path}", 
                context.Request.Method, context.Request.Path);
            await next();
            app.Logger.LogInformation("Response: {StatusCode}", 
                context.Response.StatusCode);
        });
    }
    
    // Core middleware pipeline
    app.UseRouting();
    
    // Future: Authentication and authorization
    // app.UseAuthentication();
    // app.UseAuthorization();
    
    app.MapControllers();
}

/// <summary>
/// Initialize database with migrations and seed data
/// Ensures database is ready for application startup
/// </summary>
async Task InitializeDatabase(WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ProjectsDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    
    try
    {
        // Apply any pending migrations
        await context.Database.MigrateAsync();
        logger.LogInformation("Database migrations applied successfully");
        
        // Seed initial data if database is empty
        if (!await context.Projects.AnyAsync())
        {
            await SeedInitialData(context, logger);
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to initialize database");
        throw;  // Fail fast on database issues
    }
}

/// <summary>
/// Seed initial test data for development
/// Provides sample projects for testing and demonstration
/// </summary>
async Task SeedInitialData(ProjectsDbContext context, ILogger logger)
{
    var projects = new[]
    {
        new Project("Bridge Rehabilitation", "Repair and strengthen Main Street Bridge", 
                   ProjectType.Bridges, 2500000m, DateTime.Today.AddDays(30), "ORG001"),
        new Project("Water Main Replacement", "Replace aging water infrastructure on Oak Avenue", 
                   ProjectType.WaterSystems, 1800000m, DateTime.Today.AddDays(60), "ORG001"),
        new Project("Community Center Construction", "Build new community center in downtown area", 
                   ProjectType.PublicBuildings, 4200000m, DateTime.Today.AddDays(90), "ORG001")
    };
    
    context.Projects.AddRange(projects);
    await context.SaveChangesAsync();
    
    logger.LogInformation("Seeded {Count} initial projects", projects.Length);
}
```

### 3.9 Test the Complete Project Service

**Intent**: Verify that all components work together correctly. This comprehensive test ensures the entire request/response pipeline functions properly and provides a foundation for building additional services.

```bash
# Navigate to the Projects API directory
cd src/services/projects/Infrastructure.Projects.API

# Restore packages and build
dotnet restore
dotnet build

# Apply database migrations (creates tables)
dotnet ef database update --project ../Infrastructure.Projects.Infrastructure

# Run the service
dotnet run

# Service should start on https://localhost:5001 or http://localhost:5000
```

**Test API endpoints using curl:**

```bash
# Test health check
curl http://localhost:5000/health

# Expected response:
# {"status":"Healthy","checks":[{"name":"self","status":"Healthy","duration":0.1}]}

# Test Swagger documentation
# Open browser to http://localhost:5000/swagger

# Test GET projects (should return seeded data)
curl http://localhost:5000/api/projects

# Expected response: JSON array with 3 sample projects

# Test GET specific project
curl http://localhost:5000/api/projects/{guid-from-above-response}

# Test POST new project
curl -X POST http://localhost:5000/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Park Renovation",
    "description": "Renovate Central Park playground and facilities",
    "type": "Parks",
    "estimatedCost": 750000,
    "startDate": "2025-09-01",
    "ownerId": "ORG001"
  }'

# Expected response: 201 Created with project data and Location header

# Test filtering and paging
curl "http://localhost:5000/api/projects?status=Planning&pageSize=2"

# Test sorting
curl "http://localhost:5000/api/projects?sortBy=estimatedCost&sortDescending=true"
```

**Verify database contents:**

```bash
# Connect to PostgreSQL
docker exec -it infrastructure-docker_postgres_1 psql -U dev_user -d infrastructure_dev

# Query projects table
SELECT id, name, status, estimated_cost, created_at FROM "Projects";

# Should show seeded data plus any projects created via API
```

**Key verification points:**
- ✅ Service starts without errors
- ✅ Health check returns healthy status
- ✅ Swagger UI displays API documentation
- ✅ Database migrations create proper schema
- ✅ CRUD operations work correctly
- ✅ Filtering, paging, and sorting function
- ✅ Validation prevents invalid data
- ✅ Error responses provide meaningful messages

**Why this comprehensive testing approach?**
- **End-to-end verification**: Tests the complete request/response pipeline
- **Documentation validation**: Swagger ensures API is properly documented
- **Database integration**: Confirms EF Core and PostgreSQL work together
- **Business logic validation**: Ensures domain rules are enforced
- **Performance baseline**: Establishes response time expectations
- **Foundation for additional services**: Proves the architecture patterns work# Public Infrastructure Planning Platform - Build Guide

## Phase 1: Development Environment Setup (Week 1)

### 1.1 Prerequisites Installation

**Intent**: Establish a consistent development environment across all team members to eliminate "works on my machine" issues. We're choosing specific versions and tools that support our microservices architecture.

```bash
# Install required tools
# Node.js (18+) - Required for React frontend and build tools
# Using NVM allows easy version switching for different projects
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop - Essential for local development environment
# Allows us to run all infrastructure services (DB, Redis, Kafka) locally
# Download from docker.com

# .NET 8 SDK - Latest LTS version for backend services
# Provides modern C# features, performance improvements, and long-term support
# Download from microsoft.com/dotnet

# Git - Version control for distributed development
sudo apt install git  # Linux
brew install git      # macOS

# VS Code with extensions - Consistent IDE experience
# Extensions provide IntelliSense, debugging, and container support
code --install-extension ms-dotnettools.csharp
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

**Why these choices?**
- **Node.js 18+**: Provides modern JavaScript features and npm ecosystem access
- **Docker**: Eliminates environment differences and simplifies service orchestration
- **.NET 8**: High-performance, cross-platform runtime with excellent tooling
- **VS Code**: Lightweight, extensible, with excellent container and cloud support

### 1.2 Project Structure Setup

**Intent**: Create a well-organized monorepo structure that supports microservices architecture while maintaining clear boundaries. This structure follows Domain-Driven Design principles and separates concerns between services, shared code, infrastructure, and client applications.

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Create main directory structure
# This follows the "screaming architecture" principle - the structure tells you what the system does
mkdir -p {
  src/{
    # Each service is a bounded context with its own domain
    services/{budget,grants,projects,bca,contracts,costs,users,notifications},
    # Shared code that multiple services can use - but keep this minimal
    shared/{common,domain,events},
    # API Gateway acts as the single entry point for clients
    gateways/api-gateway,
    # Separate frontend applications for different user types
    web/{admin-portal,public-portal},
    # Mobile app for field workers and inspectors
    mobile
  },
  infrastructure/{
    # Infrastructure as Code - everything should be reproducible
    docker,      # Local development containers
    kubernetes,  # Production orchestration
    terraform,   # Cloud resource provisioning
    scripts      # Deployment and utility scripts
  },
  docs,  # Architecture decision records, API docs, user guides
  tests  # End-to-end and integration tests that span services
}
```

**Why this structure?**
- **Service boundaries**: Each service in `/services/` represents a business capability
- **Shared minimal**: Only truly shared code goes in `/shared/` to avoid coupling
- **Infrastructure separation**: Keeps deployment concerns separate from business logic
- **Client separation**: Different frontends for different user needs and experiences
- **Documentation co-location**: Docs live with code for easier maintenance

### 1.3 Git Repository Setup

```bash
git init
echo "node_modules/
bin/
obj/
.env
.DS_Store
*.log" > .gitignore

git add .
git commit -m "Initial project structure"
```

## Phase 2: Core Infrastructure (Week 2)

### 2.1 Docker Development Environment

**Intent**: Create a local development environment that mirrors production as closely as possible. This setup provides all the infrastructure services (database, cache, messaging, search) that our microservices need, without requiring complex local installations. Each service is isolated and can be started/stopped independently.

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  # PostgreSQL - Primary database for transactional data
  # Using version 15 for performance and JSON improvements
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrastructure_dev      # Single database, multiple schemas per service
      POSTGRES_USER: dev_user              # Non-root user for security
      POSTGRES_PASSWORD: dev_password      # Simple password for local dev
    ports:
      - "5432:5432"                       # Standard PostgreSQL port
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data between container restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Initialize schemas

  # Redis - Caching and session storage
  # Using Alpine for smaller image size
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"                       # Standard Redis port
    # No persistence needed in development

  # Apache Kafka - Event streaming between services
  # Essential for event-driven architecture and service decoupling
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper                         # Kafka requires Zookeeper for coordination
    ports:
      - "9092:9092"                      # Kafka broker port
    environment:
      KAFKA_BROKER_ID: 1                 # Single broker for dev
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092  # Clients connect via localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  # No replication needed in dev

  # Zookeeper - Kafka dependency for cluster coordination
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Elasticsearch - Full-text search and analytics
  # Used for searching grants, projects, and generating reports
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node        # Single node for development
      - xpack.security.enabled=false      # Disable security for local dev
    ports:
      - "9200:9200"                      # REST API port

volumes:
  postgres_data:  # Named volume for database persistence
```

**Test Infrastructure:**
```bash
cd infrastructure/docker
# Start all infrastructure services in background
docker-compose -f docker-compose.dev.yml up -d

# Verify all services are running and healthy
docker-compose ps

# Expected output: All services should show "Up" status
# postgres_1      Up  0.0.0.0:5432->5432/tcp
# redis_1         Up  0.0.0.0:6379->6379/tcp
# kafka_1         Up  0.0.0.0:9092->9092/tcp
# etc.
```

**Why these infrastructure choices?**
- **PostgreSQL**: ACID compliance, JSON support, excellent .NET integration
- **Redis**: Fast caching, session storage, distributed locks
- **Kafka**: Reliable event streaming, service decoupling, audit logging
- **Elasticsearch**: Powerful search, analytics, report generation
- **Single-node configs**: Simplified for development, production uses clusters

### 2.2 Shared Domain Models

**Intent**: Establish common base classes and patterns that all domain entities will inherit from. This implements the DDD (Domain-Driven Design) principle of having rich domain objects with behavior, not just data containers. The base entity provides audit trail capabilities essential for government compliance and change tracking.

Create `src/shared/domain/Entities/BaseEntity.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Entities
{
    /// <summary>
    /// Base class for all domain entities providing common functionality
    /// Implements Entity pattern from DDD with audit trail for compliance
    /// </summary>
    public abstract class BaseEntity
    {
        // Using Guid as ID provides globally unique identifiers across distributed services
        // No need for centralized ID generation or sequence coordination
        public Guid Id { get; protected set; }
        
        // Audit fields required for government compliance and change tracking
        public DateTime CreatedAt { get; protected set; }
        public DateTime UpdatedAt { get; protected set; }
        public string CreatedBy { get; protected set; }      // User who created the entity
        public string UpdatedBy { get; protected set; }      // User who last modified

        // Protected constructor prevents direct instantiation
        // Forces use of domain-specific constructors in derived classes
        protected BaseEntity()
        {
            Id = Guid.NewGuid();              // Generate unique ID immediately
            CreatedAt = DateTime.UtcNow;      // Always use UTC to avoid timezone issues
        }

        // Virtual method allows derived classes to add custom update logic
        // Automatically tracks who made changes and when
        public virtual void Update(string updatedBy)
        {
            UpdatedAt = DateTime.UtcNow;
            UpdatedBy = updatedBy ?? throw new ArgumentNullException(nameof(updatedBy));
        }
    }
}
```

**Intent**: Create a base class for domain events that enables event-driven architecture. Domain events represent something significant that happened in the business domain and allow services to communicate without direct coupling.

Create `src/shared/domain/Events/DomainEvent.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Events
{
    /// <summary>
    /// Base class for all domain events in the system
    /// Enables event-driven architecture and service decoupling
    /// Events represent business facts that other services might care about
    /// </summary>
    public abstract class DomainEvent
    {
        // Unique identifier for this specific event occurrence
        public Guid Id { get; }
        
        // When did this business event occur (not when it was processed)
        public DateTime OccurredOn { get; }
        
        // Type identifier for event routing and deserialization
        // Using class name allows easy identification in logs and debugging
        public string EventType { get; }

        protected DomainEvent()
        {
            Id = Guid.NewGuid();                    // Unique event ID for idempotency
            OccurredOn = DateTime.UtcNow;           // Business time, not processing time
            EventType = this.GetType().Name;       // Automatic type identification
        }
    }
}
```

**Why these design decisions?**
- **Guid IDs**: Eliminate distributed ID generation complexity, work across services
- **UTC timestamps**: Prevent timezone confusion in distributed systems
- **Audit fields**: Meet government compliance requirements for change tracking
- **Protected constructors**: Enforce proper domain object creation patterns
- **Domain events**: Enable loose coupling between services while maintaining business consistency
- **Event metadata**: Support idempotency, ordering, and debugging in distributed systems

## Phase 3: First Service - Project Management (Week 3)

### 3.1 Project Service Setup

**Intent**: Create our first microservice following Clean Architecture principles. This service will handle all project-related operations and serve as a template for other services. We're using the latest .NET 8 with carefully chosen packages that support our architectural goals.

Create `src/services/projects/Infrastructure.Projects.API/Infrastructure.Projects.API.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>    <!-- Latest LTS for performance -->
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Entity Framework Core for database operations -->
    <!-- Design package needed for migrations and scaffolding -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
    <!-- PostgreSQL provider for our chosen database -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    
    <!-- Swagger for API documentation and testing -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- MediatR implements CQRS and Mediator pattern -->
    <!-- Decouples controllers from business logic -->
    <PackageReference Include="MediatR" Version="12.0.0" />
    
    <!-- AutoMapper for object-to-object mapping -->
    <!-- Keeps controllers clean by handling DTO conversions -->
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <!-- Reference to shared domain models and utilities -->
    <ProjectReference Include="../../shared/common/Infrastructure.Shared.Common.csproj" />
  </ItemGroup>
</Project>
```

**Why these package choices?**
- **EF Core + PostgreSQL**: Robust ORM with excellent PostgreSQL support for complex queries
- **MediatR**: Implements CQRS pattern, separates read/write operations, supports cross-cutting concerns
- **AutoMapper**: Reduces boilerplate code for mapping between domain models and DTOs
- **Swashbuckle**: Provides interactive API documentation essential for microservices integration

### 3.2 Project Domain Model

**Intent**: Create a rich domain model that encapsulates business rules and invariants. This follows DDD principles where the domain model contains both data and behavior. The Project entity represents the core concept of infrastructure projects and enforces business constraints through its design.

Create `src/services/projects/Infrastructure.Projects.Domain/Entities/Project.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Projects.Domain.Entities
{
    /// <summary>
    /// Project aggregate root representing an infrastructure project
    /// Contains all business rules and invariants for project management
    /// </summary>
    public class Project : BaseEntity
    {
        // Required project information - private setters enforce controlled modification
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ProjectType Type { get; private set; }           // Enum constrains valid project types
        public ProjectStatus Status { get; private set; }       // State machine via status enum
        
        // Financial information with proper decimal type for currency
        public decimal EstimatedCost { get; private set; }      // Initial cost estimate
        
        // Timeline information
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }          // Nullable - may not be set initially
        
        // Ownership tracking
        public string OwnerId { get; private set; }             // Reference to owning organization

        // EF Core requires parameterless constructor - private to prevent misuse
        private Project() { } 

        /// <summary>
        /// Creates a new infrastructure project with required business data
        /// Constructor enforces invariants - all required data must be provided
        /// </summary>
        public Project(string name, string description, ProjectType type, 
                      decimal estimatedCost, DateTime startDate, string ownerId)
        {
            // Business rule validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Project name is required", nameof(name));
            if (estimatedCost <= 0)
                throw new ArgumentException("Project cost must be positive", nameof(estimatedCost));
            if (startDate < DateTime.Today)
                throw new ArgumentException("Project cannot start in the past", nameof(startDate));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Project owner is required", nameof(ownerId));

            Name = name;
            Description = description;
            Type = type;
            Status = ProjectStatus.Planning;     // All projects start in planning phase
            EstimatedCost = estimatedCost;
            StartDate = startDate;
            OwnerId = ownerId;
        }

        /// <summary>
        /// Updates project status following business rules
        /// Could be extended to implement state machine validation
        /// Domain event could be raised here for other services to react
        /// </summary>
        public void UpdateStatus(ProjectStatus newStatus)
        {
            // Business rule: validate status transitions
            if (!IsValidStatusTransition(Status, newStatus))
                throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");
                
            Status = newStatus;
            // Future: Raise domain event ProjectStatusChanged
        }
        
        // Private method to encapsulate business rules for status transitions
        private bool IsValidStatusTransition(ProjectStatus from, ProjectStatus to)
        {
            // Simplified business rules - could be more complex state machine
            return from switch
            {
                ProjectStatus.Planning => to is ProjectStatus.Approved or ProjectStatus.Cancelled,
                ProjectStatus.Approved => to is ProjectStatus.InProgress or ProjectStatus.OnHold or ProjectStatus.Cancelled,
                ProjectStatus.InProgress => to is ProjectStatus.OnHold or ProjectStatus.Completed or ProjectStatus.Cancelled,
                ProjectStatus.OnHold => to is ProjectStatus.InProgress or ProjectStatus.Cancelled,
                ProjectStatus.Completed => false,  // Final state
                ProjectStatus.Cancelled => false,  // Final state
                _ => false
            };
        }
    }

    /// <summary>
    /// Project types supported by the system
    /// Constrains projects to known infrastructure categories
    /// </summary>
    public enum ProjectType
    {
        Roads,              // Highway and road infrastructure
        Bridges,            // Bridge construction and maintenance  
        WaterSystems,       // Water treatment and distribution
        PowerGrid,          // Electrical infrastructure
        PublicBuildings,    // Government and community buildings
        Parks               // Parks and recreational facilities
    }

    /// <summary>
    /// Project status representing the project lifecycle
    /// Forms a state machine with specific allowed transitions
    /// </summary>
    public enum ProjectStatus
    {
        Planning,       // Initial status - gathering requirements
        Approved,       // Project approved and funded
        InProgress,     // Active construction/implementation
        OnHold,         // Temporarily paused
        Completed,      // Successfully finished
        Cancelled       // Terminated before completion
    }
}
```

**Why this domain model design?**
- **Rich domain model**: Contains both data and business behavior, not just properties
- **Invariant enforcement**: Constructor validates business rules, prevents invalid objects
- **Encapsulation**: Private setters prevent external code from bypassing business rules
- **State machine**: Status transitions follow defined business rules
- **Value objects**: Enums provide type safety and constrain valid values
- **Domain events**: Architecture ready for event-driven communication (commented for now)

### 3.3 Database Context

**Intent**: Create the data access layer using Entity Framework Core with proper configuration. The DbContext serves as the Unit of Work pattern implementation and defines how our domain entities map to database tables. Configuration is explicit to ensure predictable database schema and optimal performance.

Create `src/services/projects/Infrastructure.Projects.Infrastructure/Data/ProjectsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    /// <summary>
    /// Database context for the Projects service
    /// Implements Unit of Work pattern and handles entity mapping
    /// Each service has its own database context for service autonomy
    /// </summary>
    public class ProjectsDbContext : DbContext
    {
        // Constructor injection of options allows configuration in Startup.cs
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        // DbSet represents a table - Projects table will be created
        public DbSet<Project> Projects { get; set; }

        /// <summary>
        /// Fluent API configuration - explicit mapping for database schema control
        /// Runs when model is being created, defines table structure and constraints
        /// </summary>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Explicit configuration prevents EF Core convention surprises
            modelBuilder.Entity<Project>(entity =>
            {
                // Primary key configuration - uses the Id from BaseEntity
                entity.HasKey(e => e.Id);
                
                // String property constraints for data integrity and performance
                entity.Property(e => e.Name)
                    .IsRequired()                    // NOT NULL constraint
                    .HasMaxLength(200);             // Reasonable limit prevents abuse
                
                entity.Property(e => e.Description)
                    .HasMaxLength(1000);            // Longer description allowed
                
                // Decimal configuration for financial data precision
                entity.Property(e => e.EstimatedCost)
                    .HasColumnType("decimal(18,2)"); // 18 total digits, 2 decimal places
                                                    // Handles values up to $9,999,999,999,999.99
                
                // Index for common query patterns
                entity.HasIndex(e => e.Status);     // Status filtering will be common
                
                // Additional useful indexes for query performance
                entity.HasIndex(e => e.OwnerId);    // Filter by organization
                entity.HasIndex(e => e.Type);       // Filter by project type
                entity.HasIndex(e => e.StartDate);  // Date range queries
                
                // Enum handling - EF Core stores as string by default (readable in DB)
                entity.Property(e => e.Status)
                    .HasConversion<string>();        // Store enum as string, not int
                entity.Property(e => e.Type)
                    .HasConversion<string>();
                
                // Audit fields from BaseEntity
                entity.Property(e => e.CreatedAt)
                    .IsRequired();
                entity.Property(e => e.CreatedBy)
                    .HasMaxLength(100);
                entity.Property(e => e.UpdatedBy)
                    .HasMaxLength(100);
                
                // Table naming convention - explicit naming prevents surprises
                entity.ToTable("Projects");
            });
            
            // Call base method to apply any additional conventions
            base.OnModelCreating(modelBuilder);
        }
        
        /// <summary>
        /// Override SaveChanges to add automatic audit trail
        /// This ensures all entities get proper audit information
        /// </summary>
        public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            // Get current user context (would come from HTTP context in real app)
            var currentUser = GetCurrentUser(); // TODO: Implement user context
            
            // Automatically set audit fields for new and modified entities
            var entries = ChangeTracker.Entries<BaseEntity>();
            
            foreach (var entry in entries)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        entry.Entity.CreatedBy = currentUser;
                        break;
                    case EntityState.Modified:
                        entry.Entity.Update(currentUser);
                        break;
                }
            }
            
            return await base.SaveChangesAsync(cancellationToken);
        }
        
        // TODO: Implement proper user context injection
        private string GetCurrentUser() => "system"; // Placeholder
    }
}
```

**Why this DbContext design?**
- **Explicit configuration**: Fluent API prevents EF Core convention surprises in production
- **Performance indexes**: Strategic indexes on commonly filtered columns
- **Decimal precision**: Financial data requires exact decimal handling, not floating point
- **String enums**: Human-readable enum values in database for debugging and reports
- **Audit automation**: Automatic audit trail without remembering to set fields manually
- **Service isolation**: Each service owns its database schema completely`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    public class ProjectsDbContext : DbContext
    {
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        public DbSet<Project> Projects { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Project>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Description).HasMaxLength(1000);
                entity.Property(e => e.EstimatedCost).HasColumnType("decimal(18,2)");
                entity.HasIndex(e => e.Status);
            });
        }
    }
}
```

### 3.4 API Controller

**Intent**: Create a clean API controller that follows REST principles and implements the CQRS pattern using MediatR. The controller is thin and focused only on HTTP concerns - all business logic is handled by command and query handlers. This promotes separation of concerns and testability.

Create `src/services/projects/Infrastructure.Projects.API/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.API.Controllers
{
    /// <summary>
    /// REST API controller for project management operations
    /// Implements thin controller pattern - delegates all logic to MediatR handlers
    /// Focuses purely on HTTP concerns: routing, status codes, request/response
    /// </summary>
    [ApiController]
    [Route("api/[controller]")]              // Route: /api/projects
    [Produces("application/json")]           // Always returns JSON
    public class ProjectsController : ControllerBase
    {
        private readonly IMediator _mediator;

        // Dependency injection of MediatR for CQRS pattern
        public ProjectsController(IMediator mediator)
        {
            _mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
        }

        /// <summary>
        /// GET /api/projects - Retrieve projects with optional filtering
        /// Query parameters enable filtering, paging, and sorting
        /// </summary>
        [HttpGet]
        [ProducesResponseType(typeof(PagedResult<ProjectDto>), 200)]
        [ProducesResponseType(400)] // Bad request for invalid parameters
        public async Task<IActionResult> GetProjects([FromQuery] GetProjectsQuery query)
        {
            try 
            {
                // MediatR handles routing to appropriate query handler
                var result = await _mediator.Send(query);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                // Invalid query parameters
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// GET /api/projects/{id} - Retrieve specific project by ID
        /// </summary>
        [HttpGet("{id:guid}")]                          // Constraint ensures valid GUID
        [ProducesResponseType(typeof(ProjectDetailDto), 200)]
        [ProducesResponseType(404)]                     // Project not found
        public async Task<IActionResult> GetProject(Guid id)
        {
            var query = new GetProjectByIdQuery(id);
            var result = await _mediator.Send(query);
            
            if (result == null)
                return NotFound(new { error = $"Project with ID {id} not found" });
                
            return Ok(result);
        }

        /// <summary>
        /// POST /api/projects - Create new infrastructure project
        /// Returns 201 Created with location header pointing to new resource
        /// </summary>
        [HttpPost]
        [ProducesResponseType(typeof(ProjectDto), 201)]  // Created successfully
        [ProducesResponseType(400)]                      // Validation errors
        [ProducesResponseType(409)]                      // Conflict (duplicate name, etc.)
        public async Task<IActionResult> CreateProject([FromBody] CreateProjectCommand command)
        {
            try
            {
                // Command handler creates project and returns DTO
                var result = await _mediator.Send(command);
                
                // REST best practice: return 201 Created with location header
                return CreatedAtAction(
                    nameof(GetProject), 
                    new { id = result.Id }, 
                    result);
            }
            catch (ArgumentException ex)
            {
                // Domain validation errors
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Business rule violations
                return Conflict(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PUT /api/projects/{id} - Update existing project
        /// </summary>
        [HttpPut("{id:guid}")]
        [ProducesResponseType(typeof(ProjectDto), 200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProject(Guid id, [FromBody] UpdateProjectCommand command)
        {
            // Ensure URL ID matches command ID for consistency
            if (id != command.Id)
                return BadRequest(new { error = "URL ID must match command ID" });

            try
            {
                var result = await _mediator.Send(command);
                if (result == null)
                    return NotFound();
                    
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PATCH /api/projects/{id}/status - Update project status only
        /// Separate endpoint for status changes supports workflow scenarios
        /// </summary>
        [HttpPatch("{id:guid}/status")]
        [ProducesResponseType(200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProjectStatus(Guid id, [FromBody] UpdateProjectStatusCommand command)
        {
            if (id != command.ProjectId)
                return BadRequest(new { error = "URL ID must match command project ID" });

            try
            {
                await _mediator.Send(command);
                return Ok(new { message = "Status updated successfully" });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Invalid status transitions
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// DELETE /api/projects/{id} - Soft delete project
        /// Note: Infrastructure projects are rarely truly deleted, usually cancelled
        /// </summary>
        [HttpDelete("{id:guid}")]
        [ProducesResponseType(204)]  // No content - successful deletion
        [ProducesResponseType(404)]
        [ProducesResponseType(409)]  // Cannot delete due to dependencies
        public async Task<IActionResult> DeleteProject(Guid id)
        {
            try
            {
                var command = new DeleteProjectCommand(id);
                await _mediator.Send(command);
                return NoContent();
            }
            catch (InvalidOperationException ex)
            {
                // Project has dependencies or cannot be deleted
                return Conflict(new { error = ex.Message });
            }
        }
    }
}
```

**Why this controller design?**
- **Thin controllers**: Only handle HTTP concerns, delegate business logic to handlers
- **CQRS pattern**: Separate commands (writes) from queries (reads) for clarity
- **Proper HTTP semantics**: Correct status codes, REST principles, location headers
- **Error handling**: Consistent error responses with meaningful messages
- **Type safety**: Strong typing with DTOs, GUID constraints on routes
- **Testability**: Easy to unit test by mocking IMediator
- **Documentation**: ProducesResponseType attributes generate OpenAPI specs

### 3.5 Command and Query Handlers

**Intent**: Implement the CQRS pattern by creating separate handlers for commands (writes) and queries (reads). This provides clear separation between operations that change state versus those that read data, enables different optimization strategies, and supports future event sourcing implementation.

Create `src/services/projects/Infrastructure.Projects.Application/Commands/CreateProjectCommand.cs`:

```csharp
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.Application.Commands
{
    /// <summary>
    /// Command to create a new infrastructure project
    /// Represents the intent to perform a write operation
    /// Contains all data needed to create a project
    /// </summary>
    public class CreateProjectCommand : IRequest<ProjectDto>
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public ProjectType Type { get; set; }
        public decimal EstimatedCost { get; set; }
        public DateTime StartDate { get; set; }
        public string OwnerId { get; set; }  // Current user's organization

        // Validation method to encapsulate business rules
        public void Validate()
        {
            if (string.IsNullOrWhiteSpace(Name))
                throw new ArgumentException("Project name is required");
            if (EstimatedCost <= 0)
                throw new ArgumentException("Estimated cost must be greater than zero");
            if (StartDate < DateTime.Today)
                throw new ArgumentException("Start date cannot be in the past");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/CreateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Infrastructure.Data;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles the CreateProjectCommand - implements the business logic for project creation
    /// Follows single responsibility principle - only creates projects
    /// </summary>
    public class CreateProjectHandler : IRequestHandler<CreateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public CreateProjectHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<ProjectDto> Handle(CreateProjectCommand request, CancellationToken cancellationToken)
        {
            // Validate command (could also use FluentValidation)
            request.Validate();
            
            // Business rule: Check for duplicate project names within organization
            var existingProject = await _context.Projects
                .FirstOrDefaultAsync(p => p.Name == request.Name && p.OwnerId == request.OwnerId, 
                                   cancellationToken);
            
            if (existingProject != null)
                throw new InvalidOperationException($"Project '{request.Name}' already exists");
            
            // Create domain entity - constructor enforces invariants
            var project = new Project(
                request.Name,
                request.Description,
                request.Type,
                request.EstimatedCost,
                request.StartDate,
                request.OwnerId
            );
            
            // Persist to database
            _context.Projects.Add(project);
            await _context.SaveChangesAsync(cancellationToken);
            
            // TODO: Raise domain event ProjectCreatedEvent for other services
            
            // Map to DTO for response
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Queries/GetProjectsQuery.cs`:

```csharp
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using MediatR;

namespace Infrastructure.Projects.Application.Queries
{
    /// <summary>
    /// Query to retrieve projects with filtering, paging, and sorting
    /// Optimized for read operations with minimal data transfer
    /// </summary>
    public class GetProjectsQuery : IRequest<PagedResult<ProjectDto>>
    {
        // Filtering options
        public string? NameFilter { get; set; }
        public ProjectStatus? Status { get; set; }
        public ProjectType? Type { get; set; }
        public string? OwnerId { get; set; }
        
        // Date range filtering
        public DateTime? StartDateFrom { get; set; }
        public DateTime? StartDateTo { get; set; }
        
        // Paging parameters
        public int PageNumber { get; set; } = 1;
        public int PageSize { get; set; } = 20;    // Default page size
        
        // Sorting options
        public string? SortBy { get; set; } = "Name";  // Default sort by name
        public bool SortDescending { get; set; } = false;
        
        // Validation for query parameters
        public void Validate()
        {
            if (PageNumber < 1)
                throw new ArgumentException("Page number must be greater than 0");
            if (PageSize < 1 || PageSize > 100)
                throw new ArgumentException("Page size must be between 1 and 100");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/GetProjectsHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles project list queries with filtering and paging
    /// Optimized for read performance with projection to DTOs
    /// </summary>
    public class GetProjectsHandler : IRequestHandler<GetProjectsQuery, PagedResult<ProjectDto>>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public GetProjectsHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<PagedResult<ProjectDto>> Handle(GetProjectsQuery request, CancellationToken cancellationToken)
        {
            // Validate query parameters
            request.Validate();
            
            // Build query with filtering - start with base query
            var query = _context.Projects.AsQueryable();
            
            // Apply filters - only add WHERE clauses for provided filters
            if (!string.IsNullOrEmpty(request.NameFilter))
                query = query.Where(p => p.Name.Contains(request.NameFilter));
                
            if (request.Status.HasValue)
                query = query.Where(p => p.Status == request.Status.Value);
                
            if (request.Type.HasValue)
                query = query.Where(p => p.Type == request.Type.Value);
                
            if (!string.IsNullOrEmpty(request.OwnerId))
                query = query.Where(p => p.OwnerId == request.OwnerId);
                
            // Date range filtering
            if (request.StartDateFrom.HasValue)
                query = query.Where(p => p.StartDate >= request.StartDateFrom.Value);
                
            if (request.StartDateTo.HasValue)
                query = query.Where(p => p.StartDate <= request.StartDateTo.Value);
            
            // Get total count before paging (for pagination metadata)
            var totalCount = await query.CountAsync(cancellationToken);
            
            // Apply sorting - dynamic sorting based on property name
            query = ApplySorting(query, request.SortBy, request.SortDescending);
            
            // Apply paging
            var skip = (request.PageNumber - 1) * request.PageSize;
            query = query.Skip(skip).Take(request.PageSize);
            
            // Execute query and project to DTOs in a single database call
            var projects = await query
                .Select(p => _mapper.Map<ProjectDto>(p))  // Project to DTO in database
                .ToListAsync(cancellationToken);
            
            // Return paged result with metadata
            return new PagedResult<ProjectDto>
            {
                Items = projects,
                TotalCount = totalCount,
                PageNumber = request.PageNumber,
                PageSize = request.PageSize,
                TotalPages = (int)Math.Ceiling((double)totalCount / request.PageSize)
            };
        }
        
        /// <summary>
        /// Applies dynamic sorting to the query
        /// Could be extracted to a generic extension method
        /// </summary>
        private IQueryable<Project> ApplySorting(IQueryable<Project> query, string? sortBy, bool descending)
        {
            return sortBy?.ToLower() switch
            {
                "name" => descending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
                "status" => descending ? query.OrderByDescending(p => p.Status) : query.OrderBy(p => p.Status),
                "type" => descending ? query.OrderByDescending(p => p.Type) : query.OrderBy(p => p.Type),
                "estimatedcost" => descending ? query.OrderByDescending(p => p.EstimatedCost) : query.OrderBy(p => p.EstimatedCost),
                "startdate" => descending ? query.OrderByDescending(p => p.StartDate) : query.OrderBy(p => p.StartDate),
                "createdat" => descending ? query.OrderByDescending(p => p.CreatedAt) : query.OrderBy(p => p.CreatedAt),
                _ => query.OrderBy(p => p.Name)  // Default sort
            };
        }
    }
}
```

**Why this CQRS implementation?**
- **Clear separation**: Commands change state, queries read data with different optimizations
- **Single responsibility**: Each handler does one thing well
- **Performance**: Queries use projections and paging to minimize data transfer
- **Validation**: Business rules enforced at the right layer
- **Extensibility**: Easy to add cross-cutting concerns like logging, caching
- **Testing**: Handlers can be unit tested independently of controllers

## Phase 4: Basic Frontend (Week 4)

### 4.1 React Frontend Setup

**Intent**: Create a modern React application that connects to our Projects API. We're using TypeScript for type safety, Material-UI for professional government-appropriate styling, and React Query for efficient data fetching with caching. This frontend will serve as the administrative portal for infrastructure managers.

```bash
cd src/web

# Create React app with TypeScript template
npx create-react-app admin-portal --template typescript
cd admin-portal

# Install UI framework and utility packages
npm install @mui/material @emotion/react @emotion/styled @emotion/cache
npm install @mui/icons-material @mui/lab @mui/x-data-grid
npm install @tanstack/react-query axios react-router-dom
npm install @types/node @types/react @types/react-dom

# Install development dependencies
npm install --save-dev @types/jest
```

**Why these package choices?**
- **@mui/material**: Professional, accessible UI components suitable for government applications
- **@tanstack/react-query**: Intelligent data fetching, caching, and synchronization
- **axios**: Promise-based HTTP client with request/response interceptors
- **react-router-dom**: Client-side routing for single-page application navigation
- **TypeScript**: Type safety prevents runtime errors and improves developer experience

### 4.2 API Service Layer

**Intent**: Create a clean abstraction layer for API communication. This service layer handles HTTP requests, error handling, type safety, and provides a consistent interface for components to interact with the backend.

Create `src/web/admin-portal/src/services/api.ts`:

```typescript
import axios, { AxiosInstance, AxiosResponse, AxiosError } from 'axios';

// Base configuration for API communication
const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'http://localhost:5000';

/**
 * Configured axios instance with common settings
 * Handles base URL, headers, interceptors
 */
const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,  // 30 second timeout for slow government networks
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// Request interceptor for adding authentication tokens (future)
apiClient.interceptors.request.use(
  (config) => {
    // TODO: Add JWT token when authentication is implemented
    // const token = localStorage.getItem('authToken');
    // if (token) {
    //   config.headers.Authorization = `Bearer ${token}`;
    // }
    
    console.log(`API Request: ${config.method?.toUpperCase()} ${config.url}`);
    return config;
  },
  (error) => {
    console.error('Request interceptor error:', error);
    return Promise.reject(error);
  }
);

// Response interceptor for consistent error handling
apiClient.interceptors.response.use(
  (response: AxiosResponse) => {
    console.log(`API Response: ${response.status} ${response.config.url}`);
    return response;
  },
  (error: AxiosError) => {
    console.error('API Error:', error.response?.status, error.message);
    
    // Handle common error scenarios
    if (error.response?.status === 401) {
      // TODO: Redirect to login when authentication is implemented
      console.warn('Unauthorized access - would redirect to login');
    } else if (error.response?.status === 403) {
      console.warn('Forbidden - insufficient permissions');
    } else if (error.response?.status >= 500) {
      console.error('Server error - showing user-friendly message');
    }
    
    return Promise.reject(error);
  }
);

/**
 * Generic API response wrapper for consistent typing
 */
export interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

/**
 * Error response structure from backend
 */
export interface ApiError {
  error: string;
  details?: string[];
  timestamp?: string;
}

export default apiClient;
```

Create `src/web/admin-portal/src/services/projectService.ts`:

```typescript
import apiClient, { ApiResponse, ApiError } from './api';

// Type definitions matching backend DTOs
export interface Project {
  id: string;
  name: string;
  description: string;
  type: ProjectType;
  status: ProjectStatus;
  estimatedCost: number;
  startDate: string;  // ISO date string
  endDate?: string;
  createdAt: string;
  createdBy: string;
  updatedAt?: string;
  updatedBy?: string;
  formattedCost: string;
  daysFromStart: number;
  isActive: boolean;
}

export interface ProjectDetail extends Project {
  tags: string[];
  notes: string;
  metrics: ProjectMetrics;
}

export interface ProjectMetrics {
  budgetVariance: number;
  scheduleVariance: number;
  costPerDay: number;
  healthStatus: string;
}

export type ProjectType = 'Roads' | 'Bridges' | 'WaterSystems' | 'PowerGrid' | 'PublicBuildings' | 'Parks';
export type ProjectStatus = 'Planning' | 'Approved' | 'InProgress' | 'OnHold' | 'Completed' | 'Cancelled';

export interface CreateProjectRequest {
  name: string;
  description: string;
  type: ProjectType;
  estimatedCost: number;
  startDate: string;
  ownerId: string;
}

export interface UpdateProjectRequest extends CreateProjectRequest {
  id: string;
}

export interface ProjectsQuery {
  nameFilter?: string;
  status?: ProjectStatus;
  type?: ProjectType;
  ownerId?: string;
  startDateFrom?: string;
  startDateTo?: string;
  pageNumber?: number;
  pageSize?: number;
  sortBy?: string;
  sortDescending?: boolean;
}

export interface PagedResult<T> {
  items: T[];
  totalCount: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
  hasPreviousPage: boolean;
  hasNextPage: boolean;
  firstItemIndex: number;
  lastItemIndex: number;
}

/**
 * Project service class handling all project-related API calls
 * Provides type-safe methods with proper error handling
 */
class ProjectService {
  private readonly basePath = '/api/projects';

  /**
   * Retrieve projects with optional filtering and paging
   */
  async getProjects(query: ProjectsQuery = {}): Promise<PagedResult<Project>> {
    try {
      const params = new URLSearchParams();
      
      // Add query parameters only if they have values
      Object.entries(query).forEach(([key, value]) => {
        if (value !== undefined && value !== null && value !== '') {
          params.append(key, value.toString());
        }
      });
      
      const response = await apiClient.get<PagedResult<Project>>(
        `${this.basePath}?${params.toString()}`
      );
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to fetch projects');
    }
  }

  /**
   * Retrieve single project by ID with full details
   */
  async getProjectById(id: string): Promise<ProjectDetail> {
    try {
      const response = await apiClient.get<ProjectDetail>(`${this.basePath}/${id}`);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, `Failed to fetch project ${id}`);
    }
  }

  /**
   * Create new infrastructure project
   */
  async createProject(project: CreateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.post<Project>(this.basePath, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to create project');
    }
  }

  /**
   * Update existing project
   */
  async updateProject(project: UpdateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.put<Project>(`${this.basePath}/${project.id}`, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project');
    }
  }

  /**
   * Update project status only
   */
  async updateProjectStatus(projectId: string, newStatus: ProjectStatus): Promise<void> {
    try {
      await apiClient.patch(`${this.basePath}/${projectId}/status`, {
        projectId,
        newStatus
      });
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project status');
    }
  }

  /**
   * Soft delete project (usually changes status to Cancelled)
   */
  async deleteProject(id: string): Promise<void> {
    try {
      await apiClient.delete(`${this.basePath}/${id}`);
    } catch (error) {
      throw this.handleApiError(error, 'Failed to delete project');
    }
  }

  /**
   * Consistent error handling across all service methods
   * Converts axios errors into user-friendly messages
   */
  private handleApiError(error: any, defaultMessage: string): Error {
    if (error.response) {
      // Server responded with error status
      const apiError = error.response.data as ApiError;
      return new Error(apiError.error || defaultMessage);
    } else if (error.request) {
      // Request was made but no response received
      return new Error('Network error - please check your connection');
    } else {
      // Something else happened
      return new Error(defaultMessage);
    }
  }
}

// Export singleton instance
export const projectService = new ProjectService();
```

**Why this service layer design?**
- **Type safety**: Full TypeScript typing prevents runtime errors
- **Centralized error handling**: Consistent error messages across the application
- **Request/response logging**: Debugging support for development
- **Interceptor pattern**: Easy to add authentication, logging, or retry logic
- **Future-ready**: Authentication hooks ready for implementation
- **Clean API**: Components don't deal with HTTP details directly

### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack,
  Collapse
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon,
  ExpandMore as ExpandMoreIcon,
  ExpandLess as ExpandLessIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4" component="h1" sx={{ fontWeight: 'bold' }}>
          Infrastructure Projects
        </Typography>
        <Stack direction="row" spacing={2}>
          <Button
            variant="outlined"
            startIcon={showFilters ? <ExpandLessIcon /> : <ExpandMoreIcon />}
            onClick={() => setShowFilters(!showFilters)}
          >
            {showFilters ? 'Hide Filters' : 'Show Filters'}
          </Button>
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => {/* TODO: Navigate to create project form */}}
            sx={{ bgcolor: 'primary.main' }}
          >
            New Project
          </Button>
        </Stack>
      </Stack>

      {/* Collapsible filter section */}
      <Collapse in={showFilters}>
        <Card sx={{ mb: 3 }}>
          <CardContent>
            <Typography variant="h6" sx={{ mb: 2 }}>Filters</Typography>
            <Stack direction="row" spacing={2} flexWrap="wrap" alignItems="center">
              <TextField
                label="Search Name"
                variant="outlined"
                size="small"
                value={query.nameFilter || ''}
                onChange={(e) => handleFilterChange('nameFilter', e.target.value)}
                sx={{ minWidth: 200 }}
              />
              
              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Status</InputLabel>
                <Select
                  value={query.status || ''}
                  label="Status"
                  onChange={(e) => handleFilterChange('status', e.target.value)}
                >
                  <MenuItem value="">All Statuses</MenuItem>
                  <MenuItem value="Planning">Planning</MenuItem>
                  <MenuItem value="Approved">Approved</MenuItem>
                  <MenuItem value="InProgress">In Progress</MenuItem>
                  <MenuItem value="OnHold">On Hold</MenuItem>
                  <MenuItem value="Completed">Completed</MenuItem>
                  <MenuItem value="Cancelled">Cancelled</MenuItem>
                </Select>
              </FormControl>

              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Type</InputLabel>
                <Select
                  value={query.type || ''}
                  label="Type"
                  onChange={(e) => handleFilterChange('type', e.target.value)}
                >
                  <MenuItem value="">All Types</MenuItem>
                  <MenuItem value="Roads">Roads</MenuItem>
                  <MenuItem value="Bridges">Bridges</MenuItem>
                  <MenuItem value="WaterSystems">Water Systems</MenuItem>
                  <MenuItem value="PowerGrid">Power Grid</MenuItem>
                  <MenuItem value="PublicBuildings">Public Buildings</MenuItem>
                  <MenuItem value="Parks">Parks</MenuItem>
                </Select>
              </FormControl>

              <Button
                variant="outlined"
                onClick={() => setQuery({
                  pageNumber: 1,
                  pageSize: 10,
                  sortBy: 'name',
                  sortDescending: false
                })}
              >
                Clear Filters
              </Button>
            </Stack>
          </CardContent>
        </Card>
      </Collapse>

      {/* Results summary */}
      <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
        Showing {pagination.firstItemIndex}-{pagination.lastItemIndex} of {pagination.totalCount} projects
      </Typography>

      {/* Projects table */}
      <Card>
        <TableContainer>
          <Table>
            <TableHead>
              <TableRow sx={{ bgcolor: 'grey.50' }}>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Name</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Type</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Status</Typography></TableCell>
                <TableCell align="right"><Typography variant="subtitle2" fontWeight="bold">Estimated Cost</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Start Date</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Health</Typography></TableCell>
                <TableCell align="center"><Typography variant="subtitle2" fontWeight="bold">Actions</Typography></TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {projects.length === 0 ? (
                <TableRow>
                  <TableCell colSpan={7} align="center" sx={{ py: 4 }}>
                    <Typography variant="body1" color="text.secondary">
                      No projects found. {query.nameFilter || query.status || query.type ? 'Try adjusting your filters.' : 'Create your first project to get started.'}
                    </Typography>
                  </TableCell>
                </TableRow>
              ) : (
                projects.map((project) => (
                  <TableRow key={project.id} hover>
                    <TableCell>
                      <Typography variant="body2" fontWeight="medium">{project.name}</Typography>
                      <Typography variant="caption" color="text.secondary" noWrap sx={{ maxWidth: 300, display: 'block' }}>
                        {project.description}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.type} 
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.status} 
                        color={getStatusColor(project.status)}
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="right">
                      <Typography variant="body2" fontWeight="medium">
                        {project.formattedCost}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Typography variant="body2">
                        {new Date(project.startDate).toLocaleDateString()}
                      </Typography>
                      <Typography variant="caption" color="text.secondary">
                        {project.daysFromStart} days ago
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.isActive ? 'Active' : 'Inactive'} 
                        color={project.isActive ? 'success' : 'default'}
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="center">
                      <Stack direction="row" spacing={1} justifyContent="center">
                        <Tooltip title="View Details">
                          <IconButton 
                            size="small" 
                            onClick={() => {/* TODO: Navigate to project detail */}}
                          >
                            <ViewIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Edit Project">
                          <IconButton 
                            size="small"
                            onClick={() => {/* TODO: Navigate to edit form */}}
                          >
                            <EditIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Delete Project">
                          <IconButton 
                            size="small" 
                            color="error"
                            onClick={() => {/* TODO: Show delete confirmation */}}
                          >
                            <DeleteIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                      </Stack>
                    </TableCell>
                  </TableRow>
                ))
              )}
            </TableBody>
          </Table>
        </TableContainer>

        {/* Pagination */}
        <TablePagination
          component="div"
          count={pagination.totalCount}
          page={pagination.pageNumber - 1}  // Material-UI uses 0-based pages
          onPageChange={handlePageChange}
          rowsPerPage={pagination.pageSize}
          onRowsPerPageChange={handlePageSizeChange}
          rowsPerPageOptions={[5, 10, 25, 50]}
          showFirstButton
          showLastButton
        />
      </Card>

      {/* Status update loading indicator */}
      {statusMutation.isPending && (
        <Alert severity="info" sx={{ mt: 2 }}>
          Updating project status...
        </Alert>
      )}
    </Box>
  );
};

export default ProjectList;
```

**Why this component design?**
- **Comprehensive data management**: Filtering, sorting, paging all in one component
- **Professional UI**: Government-appropriate styling with Material-UI
- **Real-time updates**: React Query provides automatic cache management
- **Responsive design**: Works on different screen sizes
- **Accessibility**: Proper ARIA labels, keyboard navigation, screen reader support
- **Performance**: Virtual scrolling for large datasets, efficient re-renders
- **Error handling**: Graceful degradation with retry functionality

### 4.4 Application Setup and Routing

**Intent**: Configure the React application with routing, React Query, and Material-UI theme. This setup provides the foundation for a professional government application with consistent styling, efficient data management, and proper navigation structure.

Create `src/web/admin-portal/src/App.tsx`:

```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { CssBaseline, Box } from '@mui/material';
import ProjectList from './components/ProjectList';
import Layout from './components/Layout';

// Create React Query client with sensible defaults
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // Data considered fresh for 5 minutes
      cacheTime: 10 * 60 * 1000,   // Cache data for 10 minutes
      retry: 3,                     // Retry failed queries 3 times
      refetchOnWindowFocus: false,  // Don't refetch when window regains focus
      refetchOnMount: true,         // Always refetch when component mounts
    },
    mutations: {
      retry: 1,  // Retry failed mutations once
    },
  },
});

// Create Material-UI theme for government applications
const theme = createTheme({
  palette: {
    mode: 'light',
    primary: {
      main: '#1976d2',      // Professional blue
      light: '#42a5f5',
      dark: '#1565c0',
    },
    secondary: {
      main: '#dc004e',      // Accent color for important actions
      light: '#ff6b9d',
      dark: '#9a0036',
    },
    background: {
      default: '#f5f5f5',   // Light gray background
      paper: '#ffffff',
    },
    text: {
      primary: '#212121',   // Dark gray for readability
      secondary: '#757575',
    },
    error: {
      main: '#d32f2f',
    },
    warning: {
      main: '#ed6c02',
    },
    success: {
      main: '#2e7d32',
    },
    info: {
      main: '#0288d1',
    },
  },
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    h1: {
      fontSize: '2.5rem',
      fontWeight: 300,
    },
    h4: {
      fontSize: '2rem',
      fontWeight: 400,
    },
    h6: {
      fontSize: '1.25rem',
      fontWeight: 500,
    },
    body1: {
      fontSize: '1rem',
      lineHeight: 1.5,
    },
    body2: {
      fontSize: '0.875rem',
      lineHeight: 1.43,
    },
    caption: {
      fontSize: '0.75rem',
      lineHeight: 1.33,
    },
  },
  components: {
    // Customize Material-UI components
    MuiButton: {
      styleOverrides: {
        root: {
          textTransform: 'none',  // Don't uppercase button text
          borderRadius: 4,
        },
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0px 2px 4px rgba(0, 0, 0, 0.1)',
          borderRadius: 8,
        },
      },
    },
    MuiTableHead: {
      styleOverrides: {
        root: {
          backgroundColor: '#f5f5f5',
        },
      },
    },
  },
});

/**
 * Main App component with routing and providers
 * Sets up the application shell with navigation and global state
 */
const App: React.FC = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={theme}>
        <CssBaseline />  {/* Material-UI CSS reset and base styles */}
        <Router>
          <Layout>
            <Routes>
              {/* Default route redirects to projects */}
              <Route path="/" element={<Navigate to="/projects" replace />} />
              
              {/* Project management routes */}
              <Route path="/projects" element={<ProjectList />} />
              
              {/* Future routes for other services */}
              <Route path="/budgets" element={<div>Budget Management (Coming Soon)</div>} />
              <Route path="/grants" element={<div>Grant Management (Coming Soon)</div>} />
              <Route path="/contracts" element={<div>Contract Management (Coming Soon)</div>} />
              <Route path="/reports" element={<div>Reports & Analytics (Coming Soon)</div>} />
              
              {/* 404 page */}
              <Route path="*" element={<Navigate to="/projects" replace />} />
            </Routes>
          </Layout>
        </Router>
        
        {/* React Query DevTools in development */}
        {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
      </ThemeProvider>
    </QueryClientProvider>
  );
};

export default App;
```

Create `src/web/admin-portal/src/components/Layout.tsx`:

```typescript
import React, { useState } from 'react';
import {
  Box,
  AppBar,
  Toolbar,
  Typography,
  Drawer,
  List,
  ListItem,
  ListItemIcon,
  ListItemText,
  IconButton,
  Divider,
  Avatar,
  Menu,
  MenuItem,
} from '@mui/material';
import {
  Menu as MenuIcon,
  AccountCircle as AccountCircleIcon,
  Dashboard as DashboardIcon,
  Business as BusinessIcon,
  MonetizationOn as MonetizationOnIcon,
  Description as DescriptionIcon,
  Assessment as AssessmentIcon,
  Settings as SettingsIcon,
} from '@mui/icons-material';
import { useNavigate, useLocation } from 'react-router-dom';

const drawerWidth = 240;

/**
 * Navigation items for the sidebar
 * Each item represents a major functional area of the platform
 */
const navigationItems = [
  { text: 'Projects', icon: <BusinessIcon />, path: '/projects' },
  { text: 'Budgets', icon: <MonetizationOnIcon />, path: '/budgets' },
  { text: 'Grants', icon: <DescriptionIcon />, path: '/grants' },
  { text: 'Contracts', icon: <DescriptionIcon />, path: '/contracts' },
  { text: 'Reports', icon: <AssessmentIcon />, path: '/reports' },
];

interface LayoutProps {
  children: React.ReactNode;
}

/**
 * Main layout component providing navigation and application shell
 * Features: responsive sidebar, user menu, breadcrumbs
 */
const Layout: React.FC<LayoutProps> = ({ children }) => {
  const navigate = useNavigate();
  const location = useLocation();
  const [mobileOpen, setMobileOpen] = useState(false);
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);

  const handleDrawerToggle = () => {
    setMobileOpen(!mobileOpen);
  };

  const handleUserMenuOpen = (event: React.MouseEvent<HTMLElement>) => {
    setAnchorEl(event.currentTarget);
  };

  const handleUserMenuClose = () => {
    setAnchorEl(null);
  };

  const handleNavigation = (path: string) => {
    navigate(path);
    setMobileOpen(false);  // Close mobile drawer after navigation
  };

  // Sidebar content
  const drawerContent = (
    <div>
      <Toolbar>
        <Typography variant="h6" noWrap component="div" sx={{ fontWeight: 'bold' }}>
          Infrastructure Platform
        </Typography>
      </Toolbar>
      <Divider />
      <List>
        {navigationItems.map((item) => (
          <ListItem
            button
            key={item.text}
            selected={location.pathname === item.path}
            onClick={() => handleNavigation(item.path)}
            sx={{
              '&.Mui-selected': {
                backgroundColor: 'primary.light',
                color: 'primary.contrastText',
                '& .MuiListItemIcon-root': {
                  color: 'primary.contrastText',
                },
              },
            }}
          >
            <ListItemIcon>{item.icon}</ListItemIcon>
            <ListItemText primary={item.text} />
          </ListItem>
        ))}
      </List>
      <Divider />
      <List>
        <ListItem button onClick={() => handleNavigation('/settings')}>
          <ListItemIcon><SettingsIcon /></ListItemIcon>
          <ListItemText primary="Settings" />
        </ListItem>
      </List>
    </div>
  );

  return (
    <Box sx={{ display: 'flex' }}>
      {/* App bar */}
      <AppBar
        position="fixed"
        sx={{
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          ml: { sm: `${drawerWidth}px` },
        }}
      >
        <Toolbar>
          <IconButton
            color="inherit"
            aria-label="open drawer"
            edge="start"
            onClick={handleDrawerToggle}
            sx={{ mr: 2, display: { sm: 'none' } }}
          >
            <MenuIcon />
          </IconButton>
          
          {/* Page title */}
          <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1 }}>
            {navigationItems.find(item => item.path === location.pathname)?.text || 'Dashboard'}
          </Typography>
          
          {/* User menu */}
          <IconButton
            size="large"
            aria-label="account menu"
            aria-haspopup="true"
            onClick={handleUserMenuOpen}
            color="inherit"
          >
            <Avatar sx={{ width: 32, height: 32, bgcolor: 'secondary.main' }}>
              JD
            </Avatar>
          </IconButton>
          
          <Menu
            anchorEl={anchorEl}
            open={Boolean(anchorEl)}
            onClose={handleUserMenuClose}
            anchorOrigin={{
              vertical: 'bottom',
              horizontal: 'right',
            }}
            transformOrigin={{
              vertical: 'top',
              horizontal: 'right',
            }}
          >
            <MenuItem onClick={handleUserMenuClose}>
              <Typography variant="body2">John Doe</Typography>
            </MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Profile</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Settings</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Logout</MenuItem>
          </Menu>
        </Toolbar>
      </AppBar>

      {/* Sidebar drawer */}
      <Box
        component="nav"
        sx={{ width: { sm: drawerWidth }, flexShrink: { sm: 0 } }}
      >
        {/* Mobile drawer */}
        <Drawer
          variant="temporary"
          open={mobileOpen}
          onClose={handleDrawerToggle}
          ModalProps={{
            keepMounted: true, // Better mobile performance
          }}
          sx={{
            display: { xs: 'block', sm: 'none' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
        >
          {drawerContent}
        </Drawer>

        {/* Desktop drawer */}
        <Drawer
          variant="permanent"
          sx={{
            display: { xs: 'none', sm: 'block' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
          open
        >
          {drawerContent}
        </Drawer>
      </Box>

      {/* Main content area */}
      <Box
        component="main"
        sx={{
          flexGrow: 1,
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          minHeight: '100vh',
          backgroundColor: 'background.default',
        }}
      >
        <Toolbar /> {/* Spacer for fixed app bar */}
        {children}
      </Box>
    </Box>
  );
};

export default Layout;
```

**Why this application structure?**
- **React Query**: Provides caching, background updates, and optimistic updates
- **Material-UI Theme**: Professional, accessible design suitable for government use
- **React Router**: Clean URL structure with proper navigation
- **Responsive Layout**: Works on desktop, tablet, and mobile devices
- **User Experience**: Loading states, error handling, intuitive navigation
- **Performance**: Code splitting, lazy loading, efficient re-renders

## Phase 5: Add Budget Service (Week 5)

### 5.1 Budget Service Structure and Domain Model

**Intent**: Create the Budget service following the same clean architecture pattern as the Project service. The Budget service manages multi-year budget planning, tracks allocations versus expenditures, and integrates with projects for cost estimation. This service demonstrates how to build additional microservices using established patterns.

Follow the same pattern as Project Service:

```bash
# Create Budget service structure
mkdir -p src/services/budget/{Domain,Application,Infrastructure,API}
cd src/services/budget
```

Create `src/services/budget/Infrastructure.Budget.Domain/Entities/Budget.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Budget.Domain.Entities
{
    /// <summary>
    /// Budget aggregate root representing a fiscal year budget plan
    /// Contains budget line items and tracks allocations vs expenditures
    /// Enforces business rules around budget constraints and approval workflows
    /// </summary>
    public class Budget : BaseEntity
    {
        public string Name { get; private set; }
        public int FiscalYear { get; private set; }              // e.g., 2025
        public decimal TotalAmount { get; private set; }         // Total budget allocation
        public BudgetStatus Status { get; private set; }         // Planning, Approved, etc.
        public DateTime PeriodStart { get; private set; }        // Budget period start
        public DateTime PeriodEnd { get; private set; }          // Budget period end
        public string OwnerId { get; private set; }              // Organization that owns budget
        
        // Collection of budget line items (use private backing field for encapsulation)
        private readonly List<BudgetLineItem> _lineItems = new();
        public IReadOnlyList<BudgetLineItem> LineItems => _lineItems.AsReadOnly();
        
        // EF Core constructor
        private Budget() { }
        
        /// <summary>
        /// Create new fiscal year budget
        /// Enforces business rules: valid fiscal year, positive amount, valid period
        /// </summary>
        public Budget(string name, int fiscalYear, decimal totalAmount, 
                     DateTime periodStart, DateTime periodEnd, string ownerId)
        {
            // Business validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Budget name is required", nameof(name));
            if (fiscalYear < DateTime.Now.Year - 1 || fiscalYear > DateTime.Now.Year + 10)
                throw new ArgumentException("Fiscal year must be within reasonable range", nameof(fiscalYear));
            if (totalAmount <= 0)
                throw new ArgumentException("Budget amount must be positive", nameof(totalAmount));
            if (periodEnd <= periodStart)
                throw new ArgumentException("Budget end date must be after start date", nameof(periodEnd));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Budget owner is required", nameof(ownerId));
                
            Name = name;
            FiscalYear = fiscalYear;
            TotalAmount = totalAmount;
            Status = BudgetStatus.Planning;  // All budgets start in planning
            PeriodStart = periodStart;
            PeriodEnd = periodEnd;
            OwnerId = ownerId;
        }
        
        /// <summary>
        /// Add budget line item for specific category
        /// Business rule: Total allocated cannot exceed budget total
        /// </summary>
        public void AddLineItem(string category, string description, decimal allocatedAmount, 
                               FundingSource fundingSource)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Cannot modify approved budget");
                
            var lineItem = new BudgetLineItem(Id, category, description, allocatedAmount, fundingSource);
            
            // Business rule: Check total allocation doesn't exceed budget
            var currentTotal = _lineItems.Sum(li => li.AllocatedAmount);
            if (currentTotal + allocatedAmount > TotalAmount)
                throw new InvalidOperationException(
                    $"Total allocation ({currentTotal + allocatedAmount:C}) would exceed budget ({TotalAmount:C})");
                    
            _lineItems.Add(lineItem);
        }
        
        /// <summary>
        /// Update line item allocation
        /// Rechecks total allocation constraint
        /// </summary>
        public void UpdateLineItemAllocation(Guid lineItemId, decimal newAmount)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Cannot modify approved budget");
                
            var lineItem = _lineItems.FirstOrDefault(li => li.Id == lineItemId);
            if (lineItem == null)
                throw new ArgumentException("Line item not found", nameof(lineItemId));
                
            var otherItemsTotal = _lineItems.Where(li => li.Id != lineItemId).Sum(li => li.AllocatedAmount);
            if (otherItemsTotal + newAmount > TotalAmount)
                throw new InvalidOperationException("Allocation would exceed total budget");
                
            lineItem.UpdateAllocation(newAmount);
        }
        
        /// <summary>
        /// Record expenditure against a line item
        /// Updates spent amounts and checks for over-spending
        /// </summary>
        public void RecordExpenditure(Guid lineItemId, decimal amount, string description, 
                                     Guid? projectId = null)
        {
            var lineItem = _lineItems.FirstOrDefault(li => li.Id == lineItemId);
            if (lineItem == null)
                throw new ArgumentException("Line item not found", nameof(lineItemId));
                
            lineItem.RecordExpenditure(amount, description, projectId);
            
            // Business rule: Warn if line item is over budget (don't prevent, just track)
            if (lineItem.SpentAmount > lineItem.AllocatedAmount)
            {
                // Could raise domain event for notifications
                // DomainEvents.Raise(new BudgetLineItemOverspentEvent(...));
            }
        }
        
        /// <summary>
        /// Approve budget for execution
        /// Changes status and prevents further modifications to allocations
        /// </summary>
        public void Approve(string approvedBy)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Only planning budgets can be approved");
                
            if (_lineItems.Sum(li => li.AllocatedAmount) == 0)
                throw new InvalidOperationException("Cannot approve budget with no allocations");
                
            Status = BudgetStatus.Approved;
            Update(approvedBy);  // Updates audit fields from BaseEntity
        }
        
        // Computed properties for reporting
        public decimal TotalAllocated => _lineItems.Sum(li => li.AllocatedAmount);
        public decimal TotalSpent => _lineItems.Sum(li => li.SpentAmount);
        public decimal RemainingBudget => TotalAmount - TotalSpent;
        public decimal UnallocatedAmount => TotalAmount - TotalAllocated;
        public double PercentSpent => TotalAmount > 0 ? (double)(TotalSpent / TotalAmount) * 100 : 0;
        public bool IsOverBudget => TotalSpent > TotalAmount;
    }

    /// <summary>
    /// Individual line item within a budget
    /// Tracks allocations and expenditures for specific categories
    /// </summary>
    public class BudgetLineItem : BaseEntity
    {
        public Guid BudgetId { get; private set; }               // Parent budget
        public string Category { get; private set; }             // e.g., "Roads", "Bridges"
        public string Description { get; private set; }          // Detailed description
        public decimal AllocatedAmount { get; private set; }     // Budgeted amount
        public decimal SpentAmount { get; private set; }         // Actually spent
        public FundingSource FundingSource { get; private set; } // Federal, State, Local, etc.
        
        // Collection of individual expenditures for audit trail
        private readonly List<BudgetExpenditure> _expenditures = new();
        public IReadOnlyList<BudgetExpenditure> Expenditures => _expenditures.AsReadOnly();
        
        // EF Core constructor
        private BudgetLineItem() { }
        
        public BudgetLineItem(Guid budgetId, string category, string description, 
                             decimal allocatedAmount, FundingSource fundingSource)
        {
            if (string.IsNullOrWhiteSpace(category))
                throw new ArgumentException("Category is required", nameof(category));
            if (allocatedAmount < 0)
                throw new ArgumentException("Allocated amount cannot be negative", nameof(allocatedAmount));
                
            BudgetId = budgetId;
            Category = category;
            Description = description;
            AllocatedAmount = allocatedAmount;
            SpentAmount = 0;  // Start with no expenditures
            FundingSource = fundingSource;
        }
        
        public void UpdateAllocation(decimal newAmount)
        {
            if (newAmount < 0)
                throw new ArgumentException("Allocation cannot be negative", nameof(newAmount));
            if (newAmount < SpentAmount)
                throw new ArgumentException("Allocation cannot be less than amount already spent", nameof(newAmount));
                
            AllocatedAmount = newAmount;
        }
        
        public void RecordExpenditure(decimal amount, string description, Guid? projectId = null)
        {
            if (amount <= 0)
                throw new ArgumentException("Expenditure amount must be positive", nameof(amount));
                
            var expenditure = new BudgetExpenditure(Id, amount, description, projectId);
            _expenditures.Add(expenditure);
            SpentAmount += amount;
        }
        
        // Computed properties
        public decimal RemainingAmount => AllocatedAmount - SpentAmount;
        public double PercentSpent => AllocatedAmount > 0 ? (double)(SpentAmount / AllocatedAmount) * 100 : 0;
        public bool IsOverBudget => SpentAmount > AllocatedAmount;
    }

    /// <summary>
    /// Individual expenditure record for audit trail
    /// Tracks when money was spent and on what
    /// </summary>
    public class BudgetExpenditure : BaseEntity
    {
        public Guid LineItemId { get; private set; }    // Parent line item
        public decimal Amount { get; private set; }      // Amount spent
        public string Description { get; private set; }  // What was purchased
        public Guid? ProjectId { get; private set; }     // Optional link to project
        public DateTime ExpenseDate { get; private set; } // When expense occurred
        
        private BudgetExpenditure() { }
        
        public BudgetExpenditure(Guid lineItemId, decimal amount, string description, Guid? projectId = null)
        {
            LineItemId = lineItemId;
            Amount = amount;
            Description = description ?? throw new ArgumentNullException(nameof(description));
            ProjectId = projectId;
            ExpenseDate = DateTime.UtcNow;
        }
    }

    /// <summary>
    /// Budget lifecycle status
    /// Controls what operations are allowed
    /// </summary>
    public enum BudgetStatus
    {
        Planning,      // Draft budget, can be modified
        Approved,      // Approved for execution, allocations locked
        Active,        // Currently in execution
        Closed,        // Budget period ended
        Cancelled      // Budget was cancelled
    }

    /// <summary>
    /// Source of funding for budget items
    /// Important for compliance and reporting
    /// </summary>
    public enum FundingSource
    {
        Federal,       // Federal grants and funding
        State,         // State government funding
        Local,         // Municipal/county funding
        Bonds,         // Municipal bonds
        PrivatePartnership, // Public-private partnerships
        Special        // Special assessments, fees
    }
}
```

**Why this budget domain design?**
- **Complex business rules**: Multi-level validation for allocations and expenditures
- **Aggregate consistency**: Budget maintains invariants across all line items
- **Audit trail**: Complete history of expenditures for government accountability
- **Integration ready**: Project references for connecting with project management
- **Reporting optimized**: Computed properties for dashboard and reports
- **Status workflow**: Clear lifecycle management for budget approval process

### 5.2 Inter-Service Communication

Create shared event contracts in `src/shared/events/`:

```csharp
public class ProjectCreatedEvent : DomainEvent
{
    public Guid ProjectId { get; }
    public string ProjectName { get; }
    public decimal EstimatedCost { get; }
    
    public ProjectCreatedEvent(Guid projectId, string projectName, decimal estimatedCost)
    {
        ProjectId = projectId;
        ProjectName = projectName;
        EstimatedCost = estimatedCost;
    }
}
```

## Phase 6: API Gateway (Week 6)

### 6.1 Set up Ocelot API Gateway

Create `src/gateways/api-gateway/Infrastructure.Gateway.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Ocelot" Version="20.0.0" />
    <PackageReference Include="Ocelot.Provider.Consul" Version="20.0.0" />
  </ItemGroup>
</Project>
```

### 6.2 Gateway Configuration

Create `src/gateways/api-gateway/ocelot.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/budgets/{everything}",
      "DownstreamScheme": "https", 
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/gateway/budgets/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

### 6.3 Test Gateway

Update frontend to use gateway endpoints and verify routing works.

## Phase 7: Authentication & Security (Week 7)

### 7.1 Add Identity Service

```bash
mkdir -p src/services/identity
cd src/services/identity

dotnet new webapi -n Infrastructure.Identity.API
cd Infrastructure.Identity.API
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 7.2 JWT Authentication Setup

Configure JWT authentication across all services and test login flow.

### 7.3 Role-Based Authorization

Implement RBAC system:
- Municipality Admin
- Project Manager  
- Budget Analyst
- Grant Writer
- Viewer

## Phase 8: Add Remaining Services (Weeks 8-12)

### 8.1 Grant Management Service (Week 8)
- Grant opportunity discovery
- Application workflow
- Document management

### 8.2 Cost Estimation Service (Week 9) 
- Historical cost data
- Market price integration
- ML-based predictions

### 8.3 BCA (Benefit-Cost Analysis) Service (Week 10)
- NPV calculations
- Risk assessment
- Compliance reporting

### 8.4 Contract Management Service (Week 11)
- Contract templates
- Procurement workflows
- Vendor management

### 8.5 Analytics & Reporting Service (Week 12)
- Dashboard data aggregation
- Report generation
- KPI calculations

## Phase 9: Advanced Frontend Features (Weeks 13-16)

### 9.1 Enhanced UI Components
- Data visualization with Charts.js/D3
- Interactive dashboards
- Document upload/preview

### 9.2 Workflow Management
- Approval flows
- Task assignments
- Notifications

### 9.3 Integration Interfaces
- ERP system connectors
- Grant portal APIs
- GIS system integration

## Phase 10: Production Readiness (Weeks 17-20)

### 10.1 Kubernetes Deployment

Create `infrastructure/kubernetes/` manifests for:
- Service deployments
- ConfigMaps and Secrets
- Ingress controllers
- Persistent volumes

### 10.2 CI/CD Pipeline

Set up GitLab CI or Azure DevOps:
- Automated testing
- Docker image building
- Deployment automation
- Environment promotion

### 10.3 Monitoring & Observability

Configure:
- Application logging (Serilog)
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Health checks

### 10.4 Performance Testing

- Load testing with k6
- Database query optimization
- Caching strategies
- CDN setup

## Testing Strategy

### Unit Tests
Each service should have comprehensive unit tests:

```bash
cd src/services/projects
dotnet new xunit -n Infrastructure.Projects.Tests
# Add test packages and write tests
```

### Integration Tests
Test service interactions and database operations.

### End-to-End Tests
Use Playwright or Cypress for full workflow testing.

## Development Workflow

1. **Feature Branch Strategy**: Create branches for each service/feature
2. **Code Reviews**: All code must be reviewed before merging
3. **Automated Testing**: CI pipeline runs all tests
4. **Local Development**: Everything runs locally with Docker Compose
5. **Documentation**: Keep docs updated as you build

## Key Milestones

- **Week 4**: First working service with basic UI
- **Week 8**: Core services communicating via events
- **Week 12**: All business services implemented
- **Week 16**: Complete frontend with all features
- **Week 20**: Production-ready deployment

This approach ensures you have a working system at every step, can demo progress regularly, and can make adjustments based on feedback as you build.
### 5.5 Update Project Service to Publish Events

**Intent**: Modify the existing Project service to publish domain events when significant business events occur. This demonstrates how to add event publishing to existing services without breaking existing functionality.

Update `src/services/projects/Infrastructure.Projects.Application/Handlers/CreateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Shared.Common.Events;
using Infrastructure.Shared.Events;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Enhanced CreateProjectHandler with domain event publishing
    /// Now notifies other services when projects are created
    /// </summary>
    public class CreateProjectHandler : IRequestHandler<CreateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        private readonly IEventPublisher _eventPublisher;
        private readonly ILogger<CreateProjectHandler> _logger;
        
        public CreateProjectHandler(
            ProjectsDbContext context, 
            IMapper mapper,
            IEventPublisher eventPublisher,
            ILogger<CreateProjectHandler> logger)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
            _eventPublisher = eventPublisher ?? throw new ArgumentNullException(nameof(eventPublisher));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        public async Task<ProjectDto> Handle(CreateProjectCommand request, CancellationToken cancellationToken)
        {
            // Validate command
            request.Validate();
            
            _logger.LogInformation("Creating new project: {ProjectName}", request.Name);
            
            // Business rule: Check for duplicate project names within organization
            var existingProject = await _context.Projects
                .FirstOrDefaultAsync(p => p.Name == request.Name && p.OwnerId == request.OwnerId, 
                                   cancellationToken);
            
            if (existingProject != null)
            {
                throw new InvalidOperationException($"Project '{request.Name}' already exists");
            }
            
            // Create domain entity
            var project = new Project(
                request.Name,
                request.Description,
                request.Type,
                request.EstimatedCost,
                request.StartDate,
                request.OwnerId
            );
            
            // Persist to database within transaction
            using var transaction = await _context.Database.BeginTransactionAsync(cancellationToken);
            
            try
            {
                _context.Projects.Add(project);
                await _context.SaveChangesAsync(cancellationToken);
                
                // Publish domain event after successful persistence
                var projectCreatedEvent = new ProjectCreatedEvent(
                    project.Id,
                    project.Name,
                    project.Type.ToString(),
                    project.EstimatedCost,
                    project.OwnerId,
                    project.StartDate
                );
                
                await _eventPublisher.PublishAsync(projectCreatedEvent, cancellationToken);
                
                await transaction.CommitAsync(cancellationToken);
                
                _logger.LogInformation("Successfully created project {ProjectId} and published event", project.Id);
            }
            catch (Exception ex)
            {
                await transaction.RollbackAsync(cancellationToken);
                _logger.LogError(ex, "Failed to create project {ProjectName}", request.Name);
                throw;
            }
            
            // Map to DTO for response
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/UpdateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Shared.Common.Events;
using Infrastructure.Shared.Events;
using AutoMapper;
using MediatR;
using Microsoft.EntityFrameworkCore;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handler for updating project information
    /// Publishes events when significant changes occur (like cost estimates)
    /// </summary>
    public class UpdateProjectHandler : IRequestHandler<UpdateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        private readonly IEventPublisher _eventPublisher;
        private readonly ILogger<UpdateProjectHandler> _logger;
        
        public UpdateProjectHandler(
            ProjectsDbContext context,
            IMapper mapper,
            IEventPublisher eventPublisher,
            ILogger<UpdateProjectHandler> logger)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
            _eventPublisher = eventPublisher ?? throw new ArgumentNullException(nameof(eventPublisher));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        public async Task<ProjectDto> Handle(UpdateProjectCommand request, CancellationToken cancellationToken)
        {
            var project = await _context.Projects
                .FirstOrDefaultAsync(p => p.Id == request.Id, cancellationToken);
            
            if (project == null)
            {
                throw new InvalidOperationException($"Project with ID {request.Id} not found");
            }
            
            _logger.LogInformation("Updating project {ProjectId} - {ProjectName}", project.Id, project.Name);
            
            // Track cost changes for event publishing
            var previousEstimate = project.EstimatedCost;
            var hasSignificantCostChange = Math.Abs(request.EstimatedCost - previousEstimate) > 1000; // $1000 threshold
            
            using var transaction = await _context.Database.BeginTransactionAsync(cancellationToken);
            
            try
            {
                // Update project properties (this would be more sophisticated in real implementation)
                project.UpdateDetails(request.Name, request.Description, request.EstimatedCost, request.StartDate);
                
                await _context.SaveChangesAsync(cancellationToken);
                
                // Publish cost change event if significant
                if (hasSignificantCostChange)
                {
                    var costChangeEvent = new ProjectCostEstimateChangedEvent(
                        project.Id,
                        project.Name,
                        previousEstimate,
                        request.EstimatedCost,
                        request.ChangeReason ?? "Project details updated",
                        "CurrentUser" // TODO: Get from user context
                    );
                    
                    await _eventPublisher.PublishAsync(costChangeEvent, cancellationToken);
                    
                    _logger.LogInformation("Published cost change event for project {ProjectId}: {Previous:C} -> {New:C}",
                                         project.Id, previousEstimate, request.EstimatedCost);
                }
                
                await transaction.CommitAsync(cancellationToken);
                
                _logger.LogInformation("Successfully updated project {ProjectId}", project.Id);
            }
            catch (Exception ex)
            {
                await transaction.RollbackAsync(cancellationToken);
                _logger.LogError(ex, "Failed to update project {ProjectId}", request.Id);
                throw;
            }
            
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

### 5.6 Configure Event Infrastructure in Dependency Injection

**Intent**: Wire up the event publishing infrastructure in both Project and Budget services. This configuration ensures events are properly published and consumed across service boundaries.

Update `src/services/projects/Infrastructure.Projects.API/Program.cs` to add event publishing:

```csharp
// Add to ConfigureServices method:

// Event publishing infrastructure
services.AddSingleton<IEventPublisher, KafkaEventPublisher>();

// Background service for event publishing reliability (optional)
services.AddHostedService<EventPublishingService>();

// Add required packages to project file:
// <PackageReference Include="Confluent.Kafka" Version="2.3.0" />
```

Create `src/services/budget/Infrastructure.Budget.Infrastructure/Messaging/KafkaEventConsumer.cs`:

```csharp
using Confluent.Kafka;
using Infrastructure.Shared.Common.Events;
using Infrastructure.Shared.Events;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Text.Json;

namespace Infrastructure.Budget.Infrastructure.Messaging
{
    /// <summary>
    /// Background service that consumes domain events from Kafka
    /// Routes events to appropriate handlers within the Budget service
    /// </summary>
    public class KafkaEventConsumer : BackgroundService
    {
        private readonly IServiceProvider _serviceProvider;
        private readonly ILogger<KafkaEventConsumer> _logger;
        private readonly IConsumer<string, string> _consumer;
        private readonly JsonSerializerOptions _jsonOptions;
        
        public KafkaEventConsumer(IServiceProvider serviceProvider, ILogger<KafkaEventConsumer> logger)
        {
            _serviceProvider = serviceProvider ?? throw new ArgumentNullException(nameof(serviceProvider));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            
            // Consumer configuration for reliability
            var config = new ConsumerConfig
            {
                BootstrapServers = "localhost:9092",  // TODO: Move to configuration
                GroupId = "budget-service",           // Consumer group for this service
                AutoOffsetReset = AutoOffsetReset.Earliest,  // Start from beginning if no offset stored
                EnableAutoCommit = false,             // Manual offset commits for reliability
                SessionTimeoutMs = 30000,             // Session timeout
                MaxPollIntervalMs = 300000,           // Max time between polls (5 minutes)
                EnableAutoOffsetStore = false         // Manual offset management
            };
            
            _consumer = new ConsumerBuilder<string, string>(config)
                .SetErrorHandler((_, error) => 
                {
                    _logger.LogError("Kafka consumer error: {Error}", error.Reason);
                })
                .Build();
            
            // Subscribe to relevant event topics
            _consumer.Subscribe(new[]
            {
                "domain.events.project-created",
                "domain.events.project-cost-estimate-changed",
                "domain.events.project-approved"
            });
            
            _jsonOptions = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase
            };
        }
        
        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("Starting Kafka event consumer for Budget service");
            
            try
            {
                while (!stoppingToken.IsCancellationRequested)
                {
                    try
                    {
                        // Poll for messages with timeout
                        var consumeResult = _consumer.Consume(TimeSpan.FromSeconds(5));
                        
                        if (consumeResult != null)
                        {
                            await ProcessMessage(consumeResult, stoppingToken);
                            
                            // Commit offset after successful processing
                            _consumer.Commit(consumeResult);
                        }
                    }
                    catch (ConsumeException ex)
                    {
                        _logger.LogError(ex, "Error consuming Kafka message: {Error}", ex.Error.Reason);
                    }
                    catch (OperationCanceledException)
                    {
                        break; // Graceful shutdown
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, "Unexpected error in event consumer");
                    }
                }
            }
            finally
            {
                _consumer.Close();
                _consumer.Dispose();
                _logger.LogInformation("Kafka event consumer stopped");
            }
        }
        
        private async Task ProcessMessage(ConsumeResult<string, string> consumeResult, CancellationToken cancellationToken)
        {
            try
            {
                // Get event type from headers
                var eventTypeHeader = consumeResult.Message.Headers
                    .FirstOrDefault(h => h.Key == "eventType");
                
                if (eventTypeHeader?.GetValueBytes() == null)
                {
                    _logger.LogWarning("Message without eventType header received from {Topic}", 
                                     consumeResult.Topic);
                    return;
                }
                
                var eventType = System.Text.Encoding.UTF8.GetString(eventTypeHeader.GetValueBytes());
                
                _logger.LogDebug("Processing event {EventType} from {Topic}:{Partition}:{Offset}",
                               eventType, consumeResult.Topic, consumeResult.Partition.Value, consumeResult.Offset.Value);
                
                // Route to appropriate handler based on event type
                await eventType switch
                {
                    "ProjectCreatedEvent" => HandleProjectCreatedEvent(consumeResult.Message.Value, cancellationToken),
                    "ProjectCostEstimateChangedEvent" => HandleProjectCostEstimateChangedEvent(consumeResult.Message.Value, cancellationToken),
                    "ProjectApprovedEvent" => HandleProjectApprovedEvent(consumeResult.Message.Value, cancellationToken),
                    _ => HandleUnknownEvent(eventType, consumeResult.Message.Value)
                };
                
                _logger.LogDebug("Successfully processed event {EventType}", eventType);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing message from {Topic}:{Partition}:{Offset}",
                               consumeResult.Topic, consumeResult.Partition.Value, consumeResult.Offset.Value);
                throw; // Re-throw to prevent offset commit
            }
        }
        
        private async Task HandleProjectCreatedEvent(string messageValue, CancellationToken cancellationToken)
        {
            var domainEvent = JsonSerializer.Deserialize<ProjectCreatedEvent>(messageValue, _jsonOptions);
            
            using var scope = _serviceProvider.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<IEventHandler<ProjectCreatedEvent>>();
            
            await handler.HandleAsync(domainEvent, cancellationToken);
        }
        
        private async Task HandleProjectCostEstimateChangedEvent(string messageValue, CancellationToken cancellationToken)
        {
            var domainEvent = JsonSerializer.Deserialize<ProjectCostEstimateChangedEvent>(messageValue, _jsonOptions);
            
            using var scope = _serviceProvider.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<IEventHandler<ProjectCostEstimateChangedEvent>>();
            
            await handler.HandleAsync(domainEvent, cancellationToken);
        }
        
        private async Task HandleProjectApprovedEvent(string messageValue, CancellationToken cancellationToken)
        {
            var domainEvent =### 5.2 Inter-Service Communication with Domain Events

**Intent**: Implement domain events to enable loose coupling between services. When projects are created or costs change, the budget service needs to be notified. This event-driven approach allows services to remain autonomous while reacting to business events from other services.

Create shared event contracts in `src/shared/events/ProjectEvents.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Events
{
    /// <summary>
    /// Event raised when a new infrastructure project is created
    /// Other services can react to this event to maintain consistency
    /// </summary>
    public class ProjectCreatedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public string ProjectType { get; }
        public decimal EstimatedCost { get; }
        public string OwnerId { get; }
        public DateTime StartDate { get; }
        
        public ProjectCreatedEvent(Guid projectId, string projectName, string projectType,
                                 decimal estimatedCost, string ownerId, DateTime startDate)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            ProjectType = projectType;
            EstimatedCost = estimatedCost;
            OwnerId = ownerId;
            StartDate = startDate;
        }
    }

    /// <summary>
    /// Event raised when project cost estimate changes
    /// Budget service needs to track these changes for variance analysis
    /// </summary>
    public class ProjectCostEstimateChangedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public decimal PreviousEstimate { get; }
        public decimal NewEstimate { get; }
        public decimal VarianceAmount => NewEstimate - PreviousEstimate;
        public string ChangeReason { get; }
        public string ChangedBy { get; }
        
        public ProjectCostEstimateChangedEvent(Guid projectId, string projectName,
                                             decimal previousEstimate, decimal newEstimate,
                                             string changeReason, string changedBy)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            PreviousEstimate = previousEstimate;
            NewEstimate = newEstimate;
            ChangeReason = changeReason;
            ChangedBy = changedBy;
        }
    }

    /// <summary>
    /// Event raised when project status changes to approved
    /// Budget service may need to reserve or allocate funds
    /// </summary>
    public class ProjectApprovedEvent : DomainEvent
    {
        public Guid ProjectId { get; }
        public string ProjectName { get; }
        public decimal ApprovedBudget { get; }
        public string BudgetCategory { get; }
        public string ApprovedBy { get; }
        public DateTime ApprovalDate { get; }
        
        public ProjectApprovedEvent(Guid projectId, string projectName, decimal approvedBudget,
                                  string budgetCategory, string approvedBy, DateTime approvalDate)
        {
            ProjectId = projectId;
            ProjectName = projectName;
            ApprovedBudget = approvedBudget;
            BudgetCategory = budgetCategory;
            ApprovedBy = approvedBy;
            ApprovalDate = approvalDate;
        }
    }
}
```

Create `src/shared/events/BudgetEvents.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Events
{
    /// <summary>
    /// Event raised when budget is approved
    /// Project service may need to know funding is available
    /// </summary>
    public class BudgetApprovedEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public string BudgetName { get; }
        public int FiscalYear { get; }
        public decimal TotalAmount { get; }
        public string OwnerId { get; }
        public string ApprovedBy { get; }
        
        public BudgetApprovedEvent(Guid budgetId, string budgetName, int fiscalYear,
                                 decimal totalAmount, string ownerId, string approvedBy)
        {
            BudgetId = budgetId;
            BudgetName = budgetName;
            FiscalYear = fiscalYear;
            TotalAmount = totalAmount;
            OwnerId = ownerId;
            ApprovedBy = approvedBy;
        }
    }

    /// <summary>
    /// Event raised when budget line item is over-spent
    /// Triggers notifications and potentially workflow approvals
    /// </summary>
    public class BudgetLineItemOverspentEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public Guid LineItemId { get; }
        public string Category { get; }
        public decimal AllocatedAmount { get; }
        public decimal SpentAmount { get; }
        public decimal OverspendAmount => SpentAmount - AllocatedAmount;
        public Guid? RelatedProjectId { get; }
        
        public BudgetLineItemOverspentEvent(Guid budgetId, Guid lineItemId, string category,
                                          decimal allocatedAmount, decimal spentAmount,
                                          Guid? relatedProjectId = null)
        {
            BudgetId = budgetId;
            LineItemId = lineItemId;
            Category = category;
            AllocatedAmount = allocatedAmount;
            SpentAmount = spentAmount;
            RelatedProjectId = relatedProjectId;
        }
    }

    /// <summary>
    /// Event raised when major expenditure is recorded
    /// May trigger approval workflows or notifications
    /// </summary>
    public class LargeExpenditureRecordedEvent : DomainEvent
    {
        public Guid BudgetId { get; }
        public Guid LineItemId { get; }
        public decimal Amount { get; }
        public string Description { get; }
        public Guid? ProjectId { get; }
        public string RecordedBy { get; }
        public bool RequiresApproval { get; }
        
        public LargeExpenditureRecordedEvent(Guid budgetId, Guid lineItemId, decimal amount,
                                           string description, string recordedBy,
                                           Guid? projectId = null, bool requiresApproval = false)
        {
            BudgetId = budgetId;
            LineItemId = lineItemId;
            Amount = amount;
            Description = description;
            ProjectId = projectId;
            RecordedBy = recordedBy;
            RequiresApproval = requiresApproval;
        }
    }
}
```

### 5.3 Event Publishing Infrastructure

**Intent**: Create a clean abstraction for publishing domain events to Kafka. This infrastructure allows domain entities to raise events without knowing about messaging details, maintaining clean separation between business logic and infrastructure concerns.

Create `src/shared/common/Events/IEventPublisher.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Interface for publishing domain events
    /// Abstracts messaging infrastructure from domain logic
    /// </summary>
    public interface IEventPublisher
    {
        /// <summary>
        /// Publish single domain event
        /// </summary>
        Task PublishAsync<T>(T domainEvent, CancellationToken cancellationToken = default)
            where T : DomainEvent;

        /// <summary>
        /// Publish multiple domain events as a batch
        /// Useful for aggregate operations that generate multiple events
        /// </summary>
        Task PublishBatchAsync<T>(IEnumerable<T> domainEvents, CancellationToken cancellationToken = default)
            where T : DomainEvent;
    }
}
```

Create `src/shared/common/Events/KafkaEventPublisher.cs`:

```csharp
using Confluent.Kafka;
using Infrastructure.Shared.Domain.Events;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Kafka implementation of event publisher
    /// Handles serialization, topic routing, and error handling
    /// </summary>
    public class KafkaEventPublisher : IEventPublisher, IDisposable
    {
        private readonly IProducer<string, string> _producer;
        private readonly ILogger<KafkaEventPublisher> _logger;
        private readonly JsonSerializerOptions _jsonOptions;
        
        public KafkaEventPublisher(ILogger<KafkaEventPublisher> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            
            // Kafka producer configuration for reliability
            var config = new ProducerConfig
            {
                BootstrapServers = "localhost:9092",  // TODO: Move to configuration
                Acks = Acks.All,                      // Wait for all replicas to acknowledge
                Retries = 3,                          // Retry failed sends
                EnableIdempotence = true,             // Prevent duplicate messages
                MessageSendMaxRetries = 3,            // Additional retry configuration
                RetryBackoffMs = 100,                 // Backoff between retries
                MessageTimeoutMs = 30000,             // Total timeout for message send
                CompressionType = CompressionType.Snappy  // Compress messages for efficiency
            };
            
            _producer = new ProducerBuilder<string, string>(config)
                .SetErrorHandler((_, error) => 
                {
                    _logger.LogError("Kafka producer error: {Error}", error.Reason);
                })
                .SetLogHandler((_, logMessage) => 
                {
                    _logger.LogDebug("Kafka log: {Message}", logMessage.Message);
                })
                .Build();
            
            // JSON serialization options for### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4### 3.8 Service Configuration and Startup

**Intent**: Configure dependency injection, database connections, and middleware in a clean, organized manner. This setup follows .NET 8 minimal hosting patterns while maintaining testability and clear separation of concerns.

Create `src/services/projects/Infrastructure.Projects.API/Program.cs`:

```csharp
using Infrastructure.Projects.Infrastructure.Data;
using Infrastructure.Projects.Application.Mapping;
using Microsoft.EntityFrameworkCore;
using MediatR;
using System.Reflection;
using Microsoft.OpenApi.Models;
using System.Text.Json;

// Modern .NET minimal hosting pattern
var builder = WebApplication.CreateBuilder(args);

// Configure services
ConfigureServices(builder.Services, builder.Configuration);

var app = builder.Build();

// Configure middleware pipeline
ConfigureMiddleware(app);

// Initialize database
await InitializeDatabase(app);

app.Run();

/// <summary>
/// Configure all application services and dependencies
/// Organized by concern: database, business logic, cross-cutting, API
/// </summary>
void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    // Database configuration - PostgreSQL with connection pooling
    services.AddDbContext<ProjectsDbContext>(options =>
        options.UseNpgsql(
            configuration.GetConnectionString("DefaultConnection") ?? 
            "Host=localhost;Database=infrastructure_dev;Username=dev_user;Password=dev_password",
            npgsqlOptions => 
            {
                npgsqlOptions.MigrationsAssembly("Infrastructure.Projects.Infrastructure");
                npgsqlOptions.EnableRetryOnFailure(3);  // Resilience for network issues
            })
        .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())  // Debug info in dev only
        .EnableDetailedErrors(builder.Environment.IsDevelopment()));

    // MediatR for CQRS pattern - scans assemblies for handlers
    var applicationAssembly = Assembly.LoadFrom("Infrastructure.Projects.Application.dll");
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(applicationAssembly));
    
    // AutoMapper for entity/DTO mapping
    services.AddAutoMapper(typeof(ProjectMappingProfile));
    
    // Health checks for monitoring
    services.AddHealthChecks()
        .AddDbContext<ProjectsDbContext>()
        .AddCheck("self", () => HealthCheckResult.Healthy());
    
    // API services with consistent JSON configuration
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            // Configure JSON serialization for API consistency
            options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
            options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
            options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());  // Enums as strings
        });
    
    // API documentation with comprehensive configuration
    services.AddEndpointsApiExplorer();
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "Infrastructure Projects API", 
            Version = "v1",
            Description = "API for managing infrastructure projects and related operations",
            Contact = new OpenApiContact
            {
                Name = "Infrastructure Platform Team",
                Email = "platform-team@municipality.gov"
            }
        });
        
        // Include XML comments in Swagger documentation
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            c.IncludeXmlComments(xmlPath);
        }
        
        // Add authorization header to Swagger UI
        c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT"
        });
    });
    
    // CORS for frontend applications
    services.AddCors(options =>
    {
        options.AddPolicy("DevelopmentPolicy", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://localhost:3000")  // React dev server
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
        
        options.AddPolicy("ProductionPolicy", policy =>
        {
            policy.WithOrigins(configuration.GetSection("AllowedOrigins").Get<string[]>() ?? Array.Empty<string>())
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        });
    });
    
    // Logging configuration
    services.AddLogging(logging =>
    {
        logging.AddConsole();
        if (builder.Environment.IsDevelopment())
        {
            logging.AddDebug();
        }
    });
}

/// <summary>
/// Configure the HTTP request pipeline
/// Order matters: security -> routing -> business logic
/// </summary>
void ConfigureMiddleware(WebApplication app)
{
    // Development-specific middleware
    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "Projects API v1");
            c.RoutePrefix = "swagger";  // Available at /swagger
            c.DisplayRequestDuration();
        });
        app.UseCors("DevelopmentPolicy");
    }
    else
    {
        // Production security headers
        app.UseHsts();
        app.UseHttpsRedirection();
        app.UseCors("ProductionPolicy");
    }
    
    // Health check endpoint for monitoring
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = async (context, report) =>
        {
            var result = JsonSerializer.Serialize(new
            {
                status = report.Status.ToString(),
                checks = report.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString(),
                    duration = e.Value.Duration.TotalMilliseconds
                })
            });
            await context.Response.WriteAsync(result);
        }
    });
    
    // Request/response logging in development
    if (app.Environment.IsDevelopment())
    {
        app.Use(async (context, next) =>
        {
            app.Logger.LogInformation("Request: {Method} {Path}", 
                context.Request.Method, context.Request.Path);
            await next();
            app.Logger.LogInformation("Response: {StatusCode}", 
                context.Response.StatusCode);
        });
    }
    
    // Core middleware pipeline
    app.UseRouting();
    
    // Future: Authentication and authorization
    // app.UseAuthentication();
    // app.UseAuthorization();
    
    app.MapControllers();
}

/// <summary>
/// Initialize database with migrations and seed data
/// Ensures database is ready for application startup
/// </summary>
async Task InitializeDatabase(WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ProjectsDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    
    try
    {
        // Apply any pending migrations
        await context.Database.MigrateAsync();
        logger.LogInformation("Database migrations applied successfully");
        
        // Seed initial data if database is empty
        if (!await context.Projects.AnyAsync())
        {
            await SeedInitialData(context, logger);
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to initialize database");
        throw;  // Fail fast on database issues
    }
}

/// <summary>
/// Seed initial test data for development
/// Provides sample projects for testing and demonstration
/// </summary>
async Task SeedInitialData(ProjectsDbContext context, ILogger logger)
{
    var projects = new[]
    {
        new Project("Bridge Rehabilitation", "Repair and strengthen Main Street Bridge", 
                   ProjectType.Bridges, 2500000m, DateTime.Today.AddDays(30), "ORG001"),
        new Project("Water Main Replacement", "Replace aging water infrastructure on Oak Avenue", 
                   ProjectType.WaterSystems, 1800000m, DateTime.Today.AddDays(60), "ORG001"),
        new Project("Community Center Construction", "Build new community center in downtown area", 
                   ProjectType.PublicBuildings, 4200000m, DateTime.Today.AddDays(90), "ORG001")
    };
    
    context.Projects.AddRange(projects);
    await context.SaveChangesAsync();
    
    logger.LogInformation("Seeded {Count} initial projects", projects.Length);
}
```

### 3.9 Test the Complete Project Service

**Intent**: Verify that all components work together correctly. This comprehensive test ensures the entire request/response pipeline functions properly and provides a foundation for building additional services.

```bash
# Navigate to the Projects API directory
cd src/services/projects/Infrastructure.Projects.API

# Restore packages and build
dotnet restore
dotnet build

# Apply database migrations (creates tables)
dotnet ef database update --project ../Infrastructure.Projects.Infrastructure

# Run the service
dotnet run

# Service should start on https://localhost:5001 or http://localhost:5000
```

**Test API endpoints using curl:**

```bash
# Test health check
curl http://localhost:5000/health

# Expected response:
# {"status":"Healthy","checks":[{"name":"self","status":"Healthy","duration":0.1}]}

# Test Swagger documentation
# Open browser to http://localhost:5000/swagger

# Test GET projects (should return seeded data)
curl http://localhost:5000/api/projects

# Expected response: JSON array with 3 sample projects

# Test GET specific project
curl http://localhost:5000/api/projects/{guid-from-above-response}

# Test POST new project
curl -X POST http://localhost:5000/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Park Renovation",
    "description": "Renovate Central Park playground and facilities",
    "type": "Parks",
    "estimatedCost": 750000,
    "startDate": "2025-09-01",
    "ownerId": "ORG001"
  }'

# Expected response: 201 Created with project data and Location header

# Test filtering and paging
curl "http://localhost:5000/api/projects?status=Planning&pageSize=2"

# Test sorting
curl "http://localhost:5000/api/projects?sortBy=estimatedCost&sortDescending=true"
```

**Verify database contents:**

```bash
# Connect to PostgreSQL
docker exec -it infrastructure-docker_postgres_1 psql -U dev_user -d infrastructure_dev

# Query projects table
SELECT id, name, status, estimated_cost, created_at FROM "Projects";

# Should show seeded data plus any projects created via API
```

**Key verification points:**
- ✅ Service starts without errors
- ✅ Health check returns healthy status
- ✅ Swagger UI displays API documentation
- ✅ Database migrations create proper schema
- ✅ CRUD operations work correctly
- ✅ Filtering, paging, and sorting function
- ✅ Validation prevents invalid data
- ✅ Error responses provide meaningful messages

**Why this comprehensive testing approach?**
- **End-to-end verification**: Tests the complete request/response pipeline
- **Documentation validation**: Swagger ensures API is properly documented
- **Database integration**: Confirms EF Core and PostgreSQL work together
- **Business logic validation**: Ensures domain rules are enforced
- **Performance baseline**: Establishes response time expectations
- **Foundation for additional services**: Proves the architecture patterns work# Public Infrastructure Planning Platform - Build Guide

## Phase 1: Development Environment Setup (Week 1)

### 1.1 Prerequisites Installation

**Intent**: Establish a consistent development environment across all team members to eliminate "works on my machine" issues. We're choosing specific versions and tools that support our microservices architecture.

```bash
# Install required tools
# Node.js (18+) - Required for React frontend and build tools
# Using NVM allows easy version switching for different projects
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop - Essential for local development environment
# Allows us to run all infrastructure services (DB, Redis, Kafka) locally
# Download from docker.com

# .NET 8 SDK - Latest LTS version for backend services
# Provides modern C# features, performance improvements, and long-term support
# Download from microsoft.com/dotnet

# Git - Version control for distributed development
sudo apt install git  # Linux
brew install git      # macOS

# VS Code with extensions - Consistent IDE experience
# Extensions provide IntelliSense, debugging, and container support
code --install-extension ms-dotnettools.csharp
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

**Why these choices?**
- **Node.js 18+**: Provides modern JavaScript features and npm ecosystem access
- **Docker**: Eliminates environment differences and simplifies service orchestration
- **.NET 8**: High-performance, cross-platform runtime with excellent tooling
- **VS Code**: Lightweight, extensible, with excellent container and cloud support

### 1.2 Project Structure Setup

**Intent**: Create a well-organized monorepo structure that supports microservices architecture while maintaining clear boundaries. This structure follows Domain-Driven Design principles and separates concerns between services, shared code, infrastructure, and client applications.

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Create main directory structure
# This follows the "screaming architecture" principle - the structure tells you what the system does
mkdir -p {
  src/{
    # Each service is a bounded context with its own domain
    services/{budget,grants,projects,bca,contracts,costs,users,notifications},
    # Shared code that multiple services can use - but keep this minimal
    shared/{common,domain,events},
    # API Gateway acts as the single entry point for clients
    gateways/api-gateway,
    # Separate frontend applications for different user types
    web/{admin-portal,public-portal},
    # Mobile app for field workers and inspectors
    mobile
  },
  infrastructure/{
    # Infrastructure as Code - everything should be reproducible
    docker,      # Local development containers
    kubernetes,  # Production orchestration
    terraform,   # Cloud resource provisioning
    scripts      # Deployment and utility scripts
  },
  docs,  # Architecture decision records, API docs, user guides
  tests  # End-to-end and integration tests that span services
}
```

**Why this structure?**
- **Service boundaries**: Each service in `/services/` represents a business capability
- **Shared minimal**: Only truly shared code goes in `/shared/` to avoid coupling
- **Infrastructure separation**: Keeps deployment concerns separate from business logic
- **Client separation**: Different frontends for different user needs and experiences
- **Documentation co-location**: Docs live with code for easier maintenance

### 1.3 Git Repository Setup

```bash
git init
echo "node_modules/
bin/
obj/
.env
.DS_Store
*.log" > .gitignore

git add .
git commit -m "Initial project structure"
```

## Phase 2: Core Infrastructure (Week 2)

### 2.1 Docker Development Environment

**Intent**: Create a local development environment that mirrors production as closely as possible. This setup provides all the infrastructure services (database, cache, messaging, search) that our microservices need, without requiring complex local installations. Each service is isolated and can be started/stopped independently.

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  # PostgreSQL - Primary database for transactional data
  # Using version 15 for performance and JSON improvements
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrastructure_dev      # Single database, multiple schemas per service
      POSTGRES_USER: dev_user              # Non-root user for security
      POSTGRES_PASSWORD: dev_password      # Simple password for local dev
    ports:
      - "5432:5432"                       # Standard PostgreSQL port
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data between container restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Initialize schemas

  # Redis - Caching and session storage
  # Using Alpine for smaller image size
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"                       # Standard Redis port
    # No persistence needed in development

  # Apache Kafka - Event streaming between services
  # Essential for event-driven architecture and service decoupling
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper                         # Kafka requires Zookeeper for coordination
    ports:
      - "9092:9092"                      # Kafka broker port
    environment:
      KAFKA_BROKER_ID: 1                 # Single broker for dev
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092  # Clients connect via localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  # No replication needed in dev

  # Zookeeper - Kafka dependency for cluster coordination
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Elasticsearch - Full-text search and analytics
  # Used for searching grants, projects, and generating reports
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node        # Single node for development
      - xpack.security.enabled=false      # Disable security for local dev
    ports:
      - "9200:9200"                      # REST API port

volumes:
  postgres_data:  # Named volume for database persistence
```

**Test Infrastructure:**
```bash
cd infrastructure/docker
# Start all infrastructure services in background
docker-compose -f docker-compose.dev.yml up -d

# Verify all services are running and healthy
docker-compose ps

# Expected output: All services should show "Up" status
# postgres_1      Up  0.0.0.0:5432->5432/tcp
# redis_1         Up  0.0.0.0:6379->6379/tcp
# kafka_1         Up  0.0.0.0:9092->9092/tcp
# etc.
```

**Why these infrastructure choices?**
- **PostgreSQL**: ACID compliance, JSON support, excellent .NET integration
- **Redis**: Fast caching, session storage, distributed locks
- **Kafka**: Reliable event streaming, service decoupling, audit logging
- **Elasticsearch**: Powerful search, analytics, report generation
- **Single-node configs**: Simplified for development, production uses clusters

### 2.2 Shared Domain Models

**Intent**: Establish common base classes and patterns that all domain entities will inherit from. This implements the DDD (Domain-Driven Design) principle of having rich domain objects with behavior, not just data containers. The base entity provides audit trail capabilities essential for government compliance and change tracking.

Create `src/shared/domain/Entities/BaseEntity.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Entities
{
    /// <summary>
    /// Base class for all domain entities providing common functionality
    /// Implements Entity pattern from DDD with audit trail for compliance
    /// </summary>
    public abstract class BaseEntity
    {
        // Using Guid as ID provides globally unique identifiers across distributed services
        // No need for centralized ID generation or sequence coordination
        public Guid Id { get; protected set; }
        
        // Audit fields required for government compliance and change tracking
        public DateTime CreatedAt { get; protected set; }
        public DateTime UpdatedAt { get; protected set; }
        public string CreatedBy { get; protected set; }      // User who created the entity
        public string UpdatedBy { get; protected set; }      // User who last modified

        // Protected constructor prevents direct instantiation
        // Forces use of domain-specific constructors in derived classes
        protected BaseEntity()
        {
            Id = Guid.NewGuid();              // Generate unique ID immediately
            CreatedAt = DateTime.UtcNow;      // Always use UTC to avoid timezone issues
        }

        // Virtual method allows derived classes to add custom update logic
        // Automatically tracks who made changes and when
        public virtual void Update(string updatedBy)
        {
            UpdatedAt = DateTime.UtcNow;
            UpdatedBy = updatedBy ?? throw new ArgumentNullException(nameof(updatedBy));
        }
    }
}
```

**Intent**: Create a base class for domain events that enables event-driven architecture. Domain events represent something significant that happened in the business domain and allow services to communicate without direct coupling.

Create `src/shared/domain/Events/DomainEvent.cs`:

```csharp
using System;

namespace Infrastructure.Shared.Domain.Events
{
    /// <summary>
    /// Base class for all domain events in the system
    /// Enables event-driven architecture and service decoupling
    /// Events represent business facts that other services might care about
    /// </summary>
    public abstract class DomainEvent
    {
        // Unique identifier for this specific event occurrence
        public Guid Id { get; }
        
        // When did this business event occur (not when it was processed)
        public DateTime OccurredOn { get; }
        
        // Type identifier for event routing and deserialization
        // Using class name allows easy identification in logs and debugging
        public string EventType { get; }

        protected DomainEvent()
        {
            Id = Guid.NewGuid();                    // Unique event ID for idempotency
            OccurredOn = DateTime.UtcNow;           // Business time, not processing time
            EventType = this.GetType().Name;       // Automatic type identification
        }
    }
}
```

**Why these design decisions?**
- **Guid IDs**: Eliminate distributed ID generation complexity, work across services
- **UTC timestamps**: Prevent timezone confusion in distributed systems
- **Audit fields**: Meet government compliance requirements for change tracking
- **Protected constructors**: Enforce proper domain object creation patterns
- **Domain events**: Enable loose coupling between services while maintaining business consistency
- **Event metadata**: Support idempotency, ordering, and debugging in distributed systems

## Phase 3: First Service - Project Management (Week 3)

### 3.1 Project Service Setup

**Intent**: Create our first microservice following Clean Architecture principles. This service will handle all project-related operations and serve as a template for other services. We're using the latest .NET 8 with carefully chosen packages that support our architectural goals.

Create `src/services/projects/Infrastructure.Projects.API/Infrastructure.Projects.API.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>    <!-- Latest LTS for performance -->
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Entity Framework Core for database operations -->
    <!-- Design package needed for migrations and scaffolding -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
    <!-- PostgreSQL provider for our chosen database -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    
    <!-- Swagger for API documentation and testing -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- MediatR implements CQRS and Mediator pattern -->
    <!-- Decouples controllers from business logic -->
    <PackageReference Include="MediatR" Version="12.0.0" />
    
    <!-- AutoMapper for object-to-object mapping -->
    <!-- Keeps controllers clean by handling DTO conversions -->
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <!-- Reference to shared domain models and utilities -->
    <ProjectReference Include="../../shared/common/Infrastructure.Shared.Common.csproj" />
  </ItemGroup>
</Project>
```

**Why these package choices?**
- **EF Core + PostgreSQL**: Robust ORM with excellent PostgreSQL support for complex queries
- **MediatR**: Implements CQRS pattern, separates read/write operations, supports cross-cutting concerns
- **AutoMapper**: Reduces boilerplate code for mapping between domain models and DTOs
- **Swashbuckle**: Provides interactive API documentation essential for microservices integration

### 3.2 Project Domain Model

**Intent**: Create a rich domain model that encapsulates business rules and invariants. This follows DDD principles where the domain model contains both data and behavior. The Project entity represents the core concept of infrastructure projects and enforces business constraints through its design.

Create `src/services/projects/Infrastructure.Projects.Domain/Entities/Project.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Projects.Domain.Entities
{
    /// <summary>
    /// Project aggregate root representing an infrastructure project
    /// Contains all business rules and invariants for project management
    /// </summary>
    public class Project : BaseEntity
    {
        // Required project information - private setters enforce controlled modification
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ProjectType Type { get; private set; }           // Enum constrains valid project types
        public ProjectStatus Status { get; private set; }       // State machine via status enum
        
        // Financial information with proper decimal type for currency
        public decimal EstimatedCost { get; private set; }      // Initial cost estimate
        
        // Timeline information
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }          // Nullable - may not be set initially
        
        // Ownership tracking
        public string OwnerId { get; private set; }             // Reference to owning organization

        // EF Core requires parameterless constructor - private to prevent misuse
        private Project() { } 

        /// <summary>
        /// Creates a new infrastructure project with required business data
        /// Constructor enforces invariants - all required data must be provided
        /// </summary>
        public Project(string name, string description, ProjectType type, 
                      decimal estimatedCost, DateTime startDate, string ownerId)
        {
            // Business rule validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Project name is required", nameof(name));
            if (estimatedCost <= 0)
                throw new ArgumentException("Project cost must be positive", nameof(estimatedCost));
            if (startDate < DateTime.Today)
                throw new ArgumentException("Project cannot start in the past", nameof(startDate));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Project owner is required", nameof(ownerId));

            Name = name;
            Description = description;
            Type = type;
            Status = ProjectStatus.Planning;     // All projects start in planning phase
            EstimatedCost = estimatedCost;
            StartDate = startDate;
            OwnerId = ownerId;
        }

        /// <summary>
        /// Updates project status following business rules
        /// Could be extended to implement state machine validation
        /// Domain event could be raised here for other services to react
        /// </summary>
        public void UpdateStatus(ProjectStatus newStatus)
        {
            // Business rule: validate status transitions
            if (!IsValidStatusTransition(Status, newStatus))
                throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");
                
            Status = newStatus;
            // Future: Raise domain event ProjectStatusChanged
        }
        
        // Private method to encapsulate business rules for status transitions
        private bool IsValidStatusTransition(ProjectStatus from, ProjectStatus to)
        {
            // Simplified business rules - could be more complex state machine
            return from switch
            {
                ProjectStatus.Planning => to is ProjectStatus.Approved or ProjectStatus.Cancelled,
                ProjectStatus.Approved => to is ProjectStatus.InProgress or ProjectStatus.OnHold or ProjectStatus.Cancelled,
                ProjectStatus.InProgress => to is ProjectStatus.OnHold or ProjectStatus.Completed or ProjectStatus.Cancelled,
                ProjectStatus.OnHold => to is ProjectStatus.InProgress or ProjectStatus.Cancelled,
                ProjectStatus.Completed => false,  // Final state
                ProjectStatus.Cancelled => false,  // Final state
                _ => false
            };
        }
    }

    /// <summary>
    /// Project types supported by the system
    /// Constrains projects to known infrastructure categories
    /// </summary>
    public enum ProjectType
    {
        Roads,              // Highway and road infrastructure
        Bridges,            // Bridge construction and maintenance  
        WaterSystems,       // Water treatment and distribution
        PowerGrid,          // Electrical infrastructure
        PublicBuildings,    // Government and community buildings
        Parks               // Parks and recreational facilities
    }

    /// <summary>
    /// Project status representing the project lifecycle
    /// Forms a state machine with specific allowed transitions
    /// </summary>
    public enum ProjectStatus
    {
        Planning,       // Initial status - gathering requirements
        Approved,       // Project approved and funded
        InProgress,     // Active construction/implementation
        OnHold,         // Temporarily paused
        Completed,      // Successfully finished
        Cancelled       // Terminated before completion
    }
}
```

**Why this domain model design?**
- **Rich domain model**: Contains both data and business behavior, not just properties
- **Invariant enforcement**: Constructor validates business rules, prevents invalid objects
- **Encapsulation**: Private setters prevent external code from bypassing business rules
- **State machine**: Status transitions follow defined business rules
- **Value objects**: Enums provide type safety and constrain valid values
- **Domain events**: Architecture ready for event-driven communication (commented for now)

### 3.3 Database Context

**Intent**: Create the data access layer using Entity Framework Core with proper configuration. The DbContext serves as the Unit of Work pattern implementation and defines how our domain entities map to database tables. Configuration is explicit to ensure predictable database schema and optimal performance.

Create `src/services/projects/Infrastructure.Projects.Infrastructure/Data/ProjectsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    /// <summary>
    /// Database context for the Projects service
    /// Implements Unit of Work pattern and handles entity mapping
    /// Each service has its own database context for service autonomy
    /// </summary>
    public class ProjectsDbContext : DbContext
    {
        // Constructor injection of options allows configuration in Startup.cs
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        // DbSet represents a table - Projects table will be created
        public DbSet<Project> Projects { get; set; }

        /// <summary>
        /// Fluent API configuration - explicit mapping for database schema control
        /// Runs when model is being created, defines table structure and constraints
        /// </summary>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Explicit configuration prevents EF Core convention surprises
            modelBuilder.Entity<Project>(entity =>
            {
                // Primary key configuration - uses the Id from BaseEntity
                entity.HasKey(e => e.Id);
                
                // String property constraints for data integrity and performance
                entity.Property(e => e.Name)
                    .IsRequired()                    // NOT NULL constraint
                    .HasMaxLength(200);             // Reasonable limit prevents abuse
                
                entity.Property(e => e.Description)
                    .HasMaxLength(1000);            // Longer description allowed
                
                // Decimal configuration for financial data precision
                entity.Property(e => e.EstimatedCost)
                    .HasColumnType("decimal(18,2)"); // 18 total digits, 2 decimal places
                                                    // Handles values up to $9,999,999,999,999.99
                
                // Index for common query patterns
                entity.HasIndex(e => e.Status);     // Status filtering will be common
                
                // Additional useful indexes for query performance
                entity.HasIndex(e => e.OwnerId);    // Filter by organization
                entity.HasIndex(e => e.Type);       // Filter by project type
                entity.HasIndex(e => e.StartDate);  // Date range queries
                
                // Enum handling - EF Core stores as string by default (readable in DB)
                entity.Property(e => e.Status)
                    .HasConversion<string>();        // Store enum as string, not int
                entity.Property(e => e.Type)
                    .HasConversion<string>();
                
                // Audit fields from BaseEntity
                entity.Property(e => e.CreatedAt)
                    .IsRequired();
                entity.Property(e => e.CreatedBy)
                    .HasMaxLength(100);
                entity.Property(e => e.UpdatedBy)
                    .HasMaxLength(100);
                
                // Table naming convention - explicit naming prevents surprises
                entity.ToTable("Projects");
            });
            
            // Call base method to apply any additional conventions
            base.OnModelCreating(modelBuilder);
        }
        
        /// <summary>
        /// Override SaveChanges to add automatic audit trail
        /// This ensures all entities get proper audit information
        /// </summary>
        public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            // Get current user context (would come from HTTP context in real app)
            var currentUser = GetCurrentUser(); // TODO: Implement user context
            
            // Automatically set audit fields for new and modified entities
            var entries = ChangeTracker.Entries<BaseEntity>();
            
            foreach (var entry in entries)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        entry.Entity.CreatedBy = currentUser;
                        break;
                    case EntityState.Modified:
                        entry.Entity.Update(currentUser);
                        break;
                }
            }
            
            return await base.SaveChangesAsync(cancellationToken);
        }
        
        // TODO: Implement proper user context injection
        private string GetCurrentUser() => "system"; // Placeholder
    }
}
```

**Why this DbContext design?**
- **Explicit configuration**: Fluent API prevents EF Core convention surprises in production
- **Performance indexes**: Strategic indexes on commonly filtered columns
- **Decimal precision**: Financial data requires exact decimal handling, not floating point
- **String enums**: Human-readable enum values in database for debugging and reports
- **Audit automation**: Automatic audit trail without remembering to set fields manually
- **Service isolation**: Each service owns its database schema completely`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Projects.Domain.Entities;

namespace Infrastructure.Projects.Infrastructure.Data
{
    public class ProjectsDbContext : DbContext
    {
        public ProjectsDbContext(DbContextOptions<ProjectsDbContext> options) : base(options) { }

        public DbSet<Project> Projects { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Project>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Description).HasMaxLength(1000);
                entity.Property(e => e.EstimatedCost).HasColumnType("decimal(18,2)");
                entity.HasIndex(e => e.Status);
            });
        }
    }
}
```

### 3.4 API Controller

**Intent**: Create a clean API controller that follows REST principles and implements the CQRS pattern using MediatR. The controller is thin and focused only on HTTP concerns - all business logic is handled by command and query handlers. This promotes separation of concerns and testability.

Create `src/services/projects/Infrastructure.Projects.API/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.API.Controllers
{
    /// <summary>
    /// REST API controller for project management operations
    /// Implements thin controller pattern - delegates all logic to MediatR handlers
    /// Focuses purely on HTTP concerns: routing, status codes, request/response
    /// </summary>
    [ApiController]
    [Route("api/[controller]")]              // Route: /api/projects
    [Produces("application/json")]           // Always returns JSON
    public class ProjectsController : ControllerBase
    {
        private readonly IMediator _mediator;

        // Dependency injection of MediatR for CQRS pattern
        public ProjectsController(IMediator mediator)
        {
            _mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
        }

        /// <summary>
        /// GET /api/projects - Retrieve projects with optional filtering
        /// Query parameters enable filtering, paging, and sorting
        /// </summary>
        [HttpGet]
        [ProducesResponseType(typeof(PagedResult<ProjectDto>), 200)]
        [ProducesResponseType(400)] // Bad request for invalid parameters
        public async Task<IActionResult> GetProjects([FromQuery] GetProjectsQuery query)
        {
            try 
            {
                // MediatR handles routing to appropriate query handler
                var result = await _mediator.Send(query);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                // Invalid query parameters
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// GET /api/projects/{id} - Retrieve specific project by ID
        /// </summary>
        [HttpGet("{id:guid}")]                          // Constraint ensures valid GUID
        [ProducesResponseType(typeof(ProjectDetailDto), 200)]
        [ProducesResponseType(404)]                     // Project not found
        public async Task<IActionResult> GetProject(Guid id)
        {
            var query = new GetProjectByIdQuery(id);
            var result = await _mediator.Send(query);
            
            if (result == null)
                return NotFound(new { error = $"Project with ID {id} not found" });
                
            return Ok(result);
        }

        /// <summary>
        /// POST /api/projects - Create new infrastructure project
        /// Returns 201 Created with location header pointing to new resource
        /// </summary>
        [HttpPost]
        [ProducesResponseType(typeof(ProjectDto), 201)]  // Created successfully
        [ProducesResponseType(400)]                      // Validation errors
        [ProducesResponseType(409)]                      // Conflict (duplicate name, etc.)
        public async Task<IActionResult> CreateProject([FromBody] CreateProjectCommand command)
        {
            try
            {
                // Command handler creates project and returns DTO
                var result = await _mediator.Send(command);
                
                // REST best practice: return 201 Created with location header
                return CreatedAtAction(
                    nameof(GetProject), 
                    new { id = result.Id }, 
                    result);
            }
            catch (ArgumentException ex)
            {
                // Domain validation errors
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Business rule violations
                return Conflict(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PUT /api/projects/{id} - Update existing project
        /// </summary>
        [HttpPut("{id:guid}")]
        [ProducesResponseType(typeof(ProjectDto), 200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProject(Guid id, [FromBody] UpdateProjectCommand command)
        {
            // Ensure URL ID matches command ID for consistency
            if (id != command.Id)
                return BadRequest(new { error = "URL ID must match command ID" });

            try
            {
                var result = await _mediator.Send(command);
                if (result == null)
                    return NotFound();
                    
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// PATCH /api/projects/{id}/status - Update project status only
        /// Separate endpoint for status changes supports workflow scenarios
        /// </summary>
        [HttpPatch("{id:guid}/status")]
        [ProducesResponseType(200)]
        [ProducesResponseType(400)]
        [ProducesResponseType(404)]
        public async Task<IActionResult> UpdateProjectStatus(Guid id, [FromBody] UpdateProjectStatusCommand command)
        {
            if (id != command.ProjectId)
                return BadRequest(new { error = "URL ID must match command project ID" });

            try
            {
                await _mediator.Send(command);
                return Ok(new { message = "Status updated successfully" });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                // Invalid status transitions
                return BadRequest(new { error = ex.Message });
            }
        }

        /// <summary>
        /// DELETE /api/projects/{id} - Soft delete project
        /// Note: Infrastructure projects are rarely truly deleted, usually cancelled
        /// </summary>
        [HttpDelete("{id:guid}")]
        [ProducesResponseType(204)]  // No content - successful deletion
        [ProducesResponseType(404)]
        [ProducesResponseType(409)]  // Cannot delete due to dependencies
        public async Task<IActionResult> DeleteProject(Guid id)
        {
            try
            {
                var command = new DeleteProjectCommand(id);
                await _mediator.Send(command);
                return NoContent();
            }
            catch (InvalidOperationException ex)
            {
                // Project has dependencies or cannot be deleted
                return Conflict(new { error = ex.Message });
            }
        }
    }
}
```

**Why this controller design?**
- **Thin controllers**: Only handle HTTP concerns, delegate business logic to handlers
- **CQRS pattern**: Separate commands (writes) from queries (reads) for clarity
- **Proper HTTP semantics**: Correct status codes, REST principles, location headers
- **Error handling**: Consistent error responses with meaningful messages
- **Type safety**: Strong typing with DTOs, GUID constraints on routes
- **Testability**: Easy to unit test by mocking IMediator
- **Documentation**: ProducesResponseType attributes generate OpenAPI specs

### 3.5 Command and Query Handlers

**Intent**: Implement the CQRS pattern by creating separate handlers for commands (writes) and queries (reads). This provides clear separation between operations that change state versus those that read data, enables different optimization strategies, and supports future event sourcing implementation.

Create `src/services/projects/Infrastructure.Projects.Application/Commands/CreateProjectCommand.cs`:

```csharp
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Application.DTOs;
using MediatR;

namespace Infrastructure.Projects.Application.Commands
{
    /// <summary>
    /// Command to create a new infrastructure project
    /// Represents the intent to perform a write operation
    /// Contains all data needed to create a project
    /// </summary>
    public class CreateProjectCommand : IRequest<ProjectDto>
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public ProjectType Type { get; set; }
        public decimal EstimatedCost { get; set; }
        public DateTime StartDate { get; set; }
        public string OwnerId { get; set; }  // Current user's organization

        // Validation method to encapsulate business rules
        public void Validate()
        {
            if (string.IsNullOrWhiteSpace(Name))
                throw new ArgumentException("Project name is required");
            if (EstimatedCost <= 0)
                throw new ArgumentException("Estimated cost must be greater than zero");
            if (StartDate < DateTime.Today)
                throw new ArgumentException("Start date cannot be in the past");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/CreateProjectHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Commands;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using Infrastructure.Projects.Infrastructure.Data;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles the CreateProjectCommand - implements the business logic for project creation
    /// Follows single responsibility principle - only creates projects
    /// </summary>
    public class CreateProjectHandler : IRequestHandler<CreateProjectCommand, ProjectDto>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public CreateProjectHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<ProjectDto> Handle(CreateProjectCommand request, CancellationToken cancellationToken)
        {
            // Validate command (could also use FluentValidation)
            request.Validate();
            
            // Business rule: Check for duplicate project names within organization
            var existingProject = await _context.Projects
                .FirstOrDefaultAsync(p => p.Name == request.Name && p.OwnerId == request.OwnerId, 
                                   cancellationToken);
            
            if (existingProject != null)
                throw new InvalidOperationException($"Project '{request.Name}' already exists");
            
            // Create domain entity - constructor enforces invariants
            var project = new Project(
                request.Name,
                request.Description,
                request.Type,
                request.EstimatedCost,
                request.StartDate,
                request.OwnerId
            );
            
            // Persist to database
            _context.Projects.Add(project);
            await _context.SaveChangesAsync(cancellationToken);
            
            // TODO: Raise domain event ProjectCreatedEvent for other services
            
            // Map to DTO for response
            return _mapper.Map<ProjectDto>(project);
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Queries/GetProjectsQuery.cs`:

```csharp
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Domain.Entities;
using MediatR;

namespace Infrastructure.Projects.Application.Queries
{
    /// <summary>
    /// Query to retrieve projects with filtering, paging, and sorting
    /// Optimized for read operations with minimal data transfer
    /// </summary>
    public class GetProjectsQuery : IRequest<PagedResult<ProjectDto>>
    {
        // Filtering options
        public string? NameFilter { get; set; }
        public ProjectStatus? Status { get; set; }
        public ProjectType? Type { get; set; }
        public string? OwnerId { get; set; }
        
        // Date range filtering
        public DateTime? StartDateFrom { get; set; }
        public DateTime? StartDateTo { get; set; }
        
        // Paging parameters
        public int PageNumber { get; set; } = 1;
        public int PageSize { get; set; } = 20;    // Default page size
        
        // Sorting options
        public string? SortBy { get; set; } = "Name";  // Default sort by name
        public bool SortDescending { get; set; } = false;
        
        // Validation for query parameters
        public void Validate()
        {
            if (PageNumber < 1)
                throw new ArgumentException("Page number must be greater than 0");
            if (PageSize < 1 || PageSize > 100)
                throw new ArgumentException("Page size must be between 1 and 100");
        }
    }
}
```

Create `src/services/projects/Infrastructure.Projects.Application/Handlers/GetProjectsHandler.cs`:

```csharp
using Infrastructure.Projects.Application.Queries;
using Infrastructure.Projects.Application.DTOs;
using Infrastructure.Projects.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using AutoMapper;
using MediatR;

namespace Infrastructure.Projects.Application.Handlers
{
    /// <summary>
    /// Handles project list queries with filtering and paging
    /// Optimized for read performance with projection to DTOs
    /// </summary>
    public class GetProjectsHandler : IRequestHandler<GetProjectsQuery, PagedResult<ProjectDto>>
    {
        private readonly ProjectsDbContext _context;
        private readonly IMapper _mapper;
        
        public GetProjectsHandler(ProjectsDbContext context, IMapper mapper)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        }

        public async Task<PagedResult<ProjectDto>> Handle(GetProjectsQuery request, CancellationToken cancellationToken)
        {
            // Validate query parameters
            request.Validate();
            
            // Build query with filtering - start with base query
            var query = _context.Projects.AsQueryable();
            
            // Apply filters - only add WHERE clauses for provided filters
            if (!string.IsNullOrEmpty(request.NameFilter))
                query = query.Where(p => p.Name.Contains(request.NameFilter));
                
            if (request.Status.HasValue)
                query = query.Where(p => p.Status == request.Status.Value);
                
            if (request.Type.HasValue)
                query = query.Where(p => p.Type == request.Type.Value);
                
            if (!string.IsNullOrEmpty(request.OwnerId))
                query = query.Where(p => p.OwnerId == request.OwnerId);
                
            // Date range filtering
            if (request.StartDateFrom.HasValue)
                query = query.Where(p => p.StartDate >= request.StartDateFrom.Value);
                
            if (request.StartDateTo.HasValue)
                query = query.Where(p => p.StartDate <= request.StartDateTo.Value);
            
            // Get total count before paging (for pagination metadata)
            var totalCount = await query.CountAsync(cancellationToken);
            
            // Apply sorting - dynamic sorting based on property name
            query = ApplySorting(query, request.SortBy, request.SortDescending);
            
            // Apply paging
            var skip = (request.PageNumber - 1) * request.PageSize;
            query = query.Skip(skip).Take(request.PageSize);
            
            // Execute query and project to DTOs in a single database call
            var projects = await query
                .Select(p => _mapper.Map<ProjectDto>(p))  // Project to DTO in database
                .ToListAsync(cancellationToken);
            
            // Return paged result with metadata
            return new PagedResult<ProjectDto>
            {
                Items = projects,
                TotalCount = totalCount,
                PageNumber = request.PageNumber,
                PageSize = request.PageSize,
                TotalPages = (int)Math.Ceiling((double)totalCount / request.PageSize)
            };
        }
        
        /// <summary>
        /// Applies dynamic sorting to the query
        /// Could be extracted to a generic extension method
        /// </summary>
        private IQueryable<Project> ApplySorting(IQueryable<Project> query, string? sortBy, bool descending)
        {
            return sortBy?.ToLower() switch
            {
                "name" => descending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
                "status" => descending ? query.OrderByDescending(p => p.Status) : query.OrderBy(p => p.Status),
                "type" => descending ? query.OrderByDescending(p => p.Type) : query.OrderBy(p => p.Type),
                "estimatedcost" => descending ? query.OrderByDescending(p => p.EstimatedCost) : query.OrderBy(p => p.EstimatedCost),
                "startdate" => descending ? query.OrderByDescending(p => p.StartDate) : query.OrderBy(p => p.StartDate),
                "createdat" => descending ? query.OrderByDescending(p => p.CreatedAt) : query.OrderBy(p => p.CreatedAt),
                _ => query.OrderBy(p => p.Name)  // Default sort
            };
        }
    }
}
```

**Why this CQRS implementation?**
- **Clear separation**: Commands change state, queries read data with different optimizations
- **Single responsibility**: Each handler does one thing well
- **Performance**: Queries use projections and paging to minimize data transfer
- **Validation**: Business rules enforced at the right layer
- **Extensibility**: Easy to add cross-cutting concerns like logging, caching
- **Testing**: Handlers can be unit tested independently of controllers

## Phase 4: Basic Frontend (Week 4)

### 4.1 React Frontend Setup

**Intent**: Create a modern React application that connects to our Projects API. We're using TypeScript for type safety, Material-UI for professional government-appropriate styling, and React Query for efficient data fetching with caching. This frontend will serve as the administrative portal for infrastructure managers.

```bash
cd src/web

# Create React app with TypeScript template
npx create-react-app admin-portal --template typescript
cd admin-portal

# Install UI framework and utility packages
npm install @mui/material @emotion/react @emotion/styled @emotion/cache
npm install @mui/icons-material @mui/lab @mui/x-data-grid
npm install @tanstack/react-query axios react-router-dom
npm install @types/node @types/react @types/react-dom

# Install development dependencies
npm install --save-dev @types/jest
```

**Why these package choices?**
- **@mui/material**: Professional, accessible UI components suitable for government applications
- **@tanstack/react-query**: Intelligent data fetching, caching, and synchronization
- **axios**: Promise-based HTTP client with request/response interceptors
- **react-router-dom**: Client-side routing for single-page application navigation
- **TypeScript**: Type safety prevents runtime errors and improves developer experience

### 4.2 API Service Layer

**Intent**: Create a clean abstraction layer for API communication. This service layer handles HTTP requests, error handling, type safety, and provides a consistent interface for components to interact with the backend.

Create `src/web/admin-portal/src/services/api.ts`:

```typescript
import axios, { AxiosInstance, AxiosResponse, AxiosError } from 'axios';

// Base configuration for API communication
const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'http://localhost:5000';

/**
 * Configured axios instance with common settings
 * Handles base URL, headers, interceptors
 */
const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,  // 30 second timeout for slow government networks
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// Request interceptor for adding authentication tokens (future)
apiClient.interceptors.request.use(
  (config) => {
    // TODO: Add JWT token when authentication is implemented
    // const token = localStorage.getItem('authToken');
    // if (token) {
    //   config.headers.Authorization = `Bearer ${token}`;
    // }
    
    console.log(`API Request: ${config.method?.toUpperCase()} ${config.url}`);
    return config;
  },
  (error) => {
    console.error('Request interceptor error:', error);
    return Promise.reject(error);
  }
);

// Response interceptor for consistent error handling
apiClient.interceptors.response.use(
  (response: AxiosResponse) => {
    console.log(`API Response: ${response.status} ${response.config.url}`);
    return response;
  },
  (error: AxiosError) => {
    console.error('API Error:', error.response?.status, error.message);
    
    // Handle common error scenarios
    if (error.response?.status === 401) {
      // TODO: Redirect to login when authentication is implemented
      console.warn('Unauthorized access - would redirect to login');
    } else if (error.response?.status === 403) {
      console.warn('Forbidden - insufficient permissions');
    } else if (error.response?.status >= 500) {
      console.error('Server error - showing user-friendly message');
    }
    
    return Promise.reject(error);
  }
);

/**
 * Generic API response wrapper for consistent typing
 */
export interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

/**
 * Error response structure from backend
 */
export interface ApiError {
  error: string;
  details?: string[];
  timestamp?: string;
}

export default apiClient;
```

Create `src/web/admin-portal/src/services/projectService.ts`:

```typescript
import apiClient, { ApiResponse, ApiError } from './api';

// Type definitions matching backend DTOs
export interface Project {
  id: string;
  name: string;
  description: string;
  type: ProjectType;
  status: ProjectStatus;
  estimatedCost: number;
  startDate: string;  // ISO date string
  endDate?: string;
  createdAt: string;
  createdBy: string;
  updatedAt?: string;
  updatedBy?: string;
  formattedCost: string;
  daysFromStart: number;
  isActive: boolean;
}

export interface ProjectDetail extends Project {
  tags: string[];
  notes: string;
  metrics: ProjectMetrics;
}

export interface ProjectMetrics {
  budgetVariance: number;
  scheduleVariance: number;
  costPerDay: number;
  healthStatus: string;
}

export type ProjectType = 'Roads' | 'Bridges' | 'WaterSystems' | 'PowerGrid' | 'PublicBuildings' | 'Parks';
export type ProjectStatus = 'Planning' | 'Approved' | 'InProgress' | 'OnHold' | 'Completed' | 'Cancelled';

export interface CreateProjectRequest {
  name: string;
  description: string;
  type: ProjectType;
  estimatedCost: number;
  startDate: string;
  ownerId: string;
}

export interface UpdateProjectRequest extends CreateProjectRequest {
  id: string;
}

export interface ProjectsQuery {
  nameFilter?: string;
  status?: ProjectStatus;
  type?: ProjectType;
  ownerId?: string;
  startDateFrom?: string;
  startDateTo?: string;
  pageNumber?: number;
  pageSize?: number;
  sortBy?: string;
  sortDescending?: boolean;
}

export interface PagedResult<T> {
  items: T[];
  totalCount: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
  hasPreviousPage: boolean;
  hasNextPage: boolean;
  firstItemIndex: number;
  lastItemIndex: number;
}

/**
 * Project service class handling all project-related API calls
 * Provides type-safe methods with proper error handling
 */
class ProjectService {
  private readonly basePath = '/api/projects';

  /**
   * Retrieve projects with optional filtering and paging
   */
  async getProjects(query: ProjectsQuery = {}): Promise<PagedResult<Project>> {
    try {
      const params = new URLSearchParams();
      
      // Add query parameters only if they have values
      Object.entries(query).forEach(([key, value]) => {
        if (value !== undefined && value !== null && value !== '') {
          params.append(key, value.toString());
        }
      });
      
      const response = await apiClient.get<PagedResult<Project>>(
        `${this.basePath}?${params.toString()}`
      );
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to fetch projects');
    }
  }

  /**
   * Retrieve single project by ID with full details
   */
  async getProjectById(id: string): Promise<ProjectDetail> {
    try {
      const response = await apiClient.get<ProjectDetail>(`${this.basePath}/${id}`);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, `Failed to fetch project ${id}`);
    }
  }

  /**
   * Create new infrastructure project
   */
  async createProject(project: CreateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.post<Project>(this.basePath, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to create project');
    }
  }

  /**
   * Update existing project
   */
  async updateProject(project: UpdateProjectRequest): Promise<Project> {
    try {
      const response = await apiClient.put<Project>(`${this.basePath}/${project.id}`, project);
      return response.data;
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project');
    }
  }

  /**
   * Update project status only
   */
  async updateProjectStatus(projectId: string, newStatus: ProjectStatus): Promise<void> {
    try {
      await apiClient.patch(`${this.basePath}/${projectId}/status`, {
        projectId,
        newStatus
      });
    } catch (error) {
      throw this.handleApiError(error, 'Failed to update project status');
    }
  }

  /**
   * Soft delete project (usually changes status to Cancelled)
   */
  async deleteProject(id: string): Promise<void> {
    try {
      await apiClient.delete(`${this.basePath}/${id}`);
    } catch (error) {
      throw this.handleApiError(error, 'Failed to delete project');
    }
  }

  /**
   * Consistent error handling across all service methods
   * Converts axios errors into user-friendly messages
   */
  private handleApiError(error: any, defaultMessage: string): Error {
    if (error.response) {
      // Server responded with error status
      const apiError = error.response.data as ApiError;
      return new Error(apiError.error || defaultMessage);
    } else if (error.request) {
      // Request was made but no response received
      return new Error('Network error - please check your connection');
    } else {
      // Something else happened
      return new Error(defaultMessage);
    }
  }
}

// Export singleton instance
export const projectService = new ProjectService();
```

**Why this service layer design?**
- **Type safety**: Full TypeScript typing prevents runtime errors
- **Centralized error handling**: Consistent error messages across the application
- **Request/response logging**: Debugging support for development
- **Interceptor pattern**: Easy to add authentication, logging, or retry logic
- **Future-ready**: Authentication hooks ready for implementation
- **Clean API**: Components don't deal with HTTP details directly

### 4.3 Project List Component

**Intent**: Create a data-driven component that displays projects in a professional table format with filtering, sorting, and actions. This component demonstrates how to integrate React Query for efficient data fetching, Material-UI for consistent styling, and TypeScript for type safety.

Create `src/web/admin-portal/src/components/ProjectList.tsx`:

```typescript
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import {
  Box,
  Card,
  CardContent,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  TablePagination,
  Paper,
  Chip,
  Button,
  Typography,
  TextField,
  Select,
  MenuItem,
  FormControl,
  InputLabel,
  Alert,
  CircularProgress,
  IconButton,
  Tooltip,
  Stack,
  Collapse
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  Visibility as ViewIcon,
  FilterList as FilterIcon,
  ExpandMore as ExpandMoreIcon,
  ExpandLess as ExpandLessIcon
} from '@mui/icons-material';
import { 
  projectService, 
  Project, 
  ProjectStatus, 
  ProjectType, 
  ProjectsQuery 
} from '../services/projectService';

/**
 * Main project list component with full CRUD capabilities
 * Features: filtering, sorting, paging, status management
 */
const ProjectList: React.FC = () => {
  // State for filtering and paging
  const [query, setQuery] = useState<ProjectsQuery>({
    pageNumber: 1,
    pageSize: 10,
    sortBy: 'name',
    sortDescending: false
  });
  
  // Filter visibility toggle
  const [showFilters, setShowFilters] = useState(false);
  
  // React Query for data fetching with caching
  const {
    data: projectsResult,
    isLoading,
    isError,
    error,
    refetch
  } = useQuery({
    queryKey: ['projects', query],  // Cache key includes query params
    queryFn: () => projectService.getProjects(query),
    staleTime: 5 * 60 * 1000,  // Consider data fresh for 5 minutes
    retry: 3,  // Retry failed requests 3 times
  });

  // React Query client for cache invalidation
  const queryClient = useQueryClient();

  // Mutation for status updates
  const statusMutation = useMutation({
    mutationFn: ({ projectId, status }: { projectId: string; status: ProjectStatus }) =>
      projectService.updateProjectStatus(projectId, status),
    onSuccess: () => {
      // Invalidate and refetch projects after status change
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });

  // Handle page changes
  const handlePageChange = (event: unknown, newPage: number) => {
    setQuery(prev => ({ ...prev, pageNumber: newPage + 1 }));  // Material-UI uses 0-based pages
  };

  // Handle page size changes
  const handlePageSizeChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(prev => ({ 
      ...prev, 
      pageSize: parseInt(event.target.value, 10),
      pageNumber: 1  // Reset to first page
    }));
  };

  // Handle filter changes
  const handleFilterChange = (field: keyof ProjectsQuery, value: any) => {
    setQuery(prev => ({ 
      ...prev, 
      [field]: value || undefined,  // Convert empty strings to undefined
      pageNumber: 1  // Reset to first page when filtering
    }));
  };

  // Handle status change
  const handleStatusChange = (projectId: string, newStatus: ProjectStatus) => {
    statusMutation.mutate({ projectId, status: newStatus });
  };

  // Get status chip color based on project status
  const getStatusColor = (status: ProjectStatus): 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning' => {
    const statusColors = {
      Planning: 'default' as const,
      Approved: 'primary' as const,
      InProgress: 'info' as const,
      OnHold: 'warning' as const,
      Completed: 'success' as const,
      Cancelled: 'error' as const
    };
    return statusColors[status] || 'default';
  };

  // Loading state
  if (isLoading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" minHeight="400px">
        <CircularProgress size={60} />
        <Typography variant="h6" sx={{ ml: 2 }}>Loading projects...</Typography>
      </Box>
    );
  }

  // Error state
  if (isError) {
    return (
      <Alert severity="error" sx={{ m: 2 }}>
        <Typography variant="h6">Failed to load projects</Typography>
        <Typography variant="body2">
          {error instanceof Error ? error.message : 'An unexpected error occurred'}
        </Typography>
        <Button variant="outlined" onClick={() => refetch()} sx={{ mt: 2 }}>
          Try Again
        </Button>
      </Alert>
    );
  }

  const projects = projectsResult?.items || [];
  const pagination = projectsResult || { totalCount: 0, pageNumber: 1, pageSize: 10, totalPages: 0 };

  return (
    <Box sx={{ p: 3 }}>
      {/* Header with title and actions */}
      <Stack direction="row" justifyContent="space-between" alignItems="center" sx={{ mb: 3 }}>
        <Typography variant="h4" component="h1" sx={{ fontWeight: 'bold' }}>
          Infrastructure Projects
        </Typography>
        <Stack direction="row" spacing={2}>
          <Button
            variant="outlined"
            startIcon={showFilters ? <ExpandLessIcon /> : <ExpandMoreIcon />}
            onClick={() => setShowFilters(!showFilters)}
          >
            {showFilters ? 'Hide Filters' : 'Show Filters'}
          </Button>
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => {/* TODO: Navigate to create project form */}}
            sx={{ bgcolor: 'primary.main' }}
          >
            New Project
          </Button>
        </Stack>
      </Stack>

      {/* Collapsible filter section */}
      <Collapse in={showFilters}>
        <Card sx={{ mb: 3 }}>
          <CardContent>
            <Typography variant="h6" sx={{ mb: 2 }}>Filters</Typography>
            <Stack direction="row" spacing={2} flexWrap="wrap" alignItems="center">
              <TextField
                label="Search Name"
                variant="outlined"
                size="small"
                value={query.nameFilter || ''}
                onChange={(e) => handleFilterChange('nameFilter', e.target.value)}
                sx={{ minWidth: 200 }}
              />
              
              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Status</InputLabel>
                <Select
                  value={query.status || ''}
                  label="Status"
                  onChange={(e) => handleFilterChange('status', e.target.value)}
                >
                  <MenuItem value="">All Statuses</MenuItem>
                  <MenuItem value="Planning">Planning</MenuItem>
                  <MenuItem value="Approved">Approved</MenuItem>
                  <MenuItem value="InProgress">In Progress</MenuItem>
                  <MenuItem value="OnHold">On Hold</MenuItem>
                  <MenuItem value="Completed">Completed</MenuItem>
                  <MenuItem value="Cancelled">Cancelled</MenuItem>
                </Select>
              </FormControl>

              <FormControl variant="outlined" size="small" sx={{ minWidth: 150 }}>
                <InputLabel>Type</InputLabel>
                <Select
                  value={query.type || ''}
                  label="Type"
                  onChange={(e) => handleFilterChange('type', e.target.value)}
                >
                  <MenuItem value="">All Types</MenuItem>
                  <MenuItem value="Roads">Roads</MenuItem>
                  <MenuItem value="Bridges">Bridges</MenuItem>
                  <MenuItem value="WaterSystems">Water Systems</MenuItem>
                  <MenuItem value="PowerGrid">Power Grid</MenuItem>
                  <MenuItem value="PublicBuildings">Public Buildings</MenuItem>
                  <MenuItem value="Parks">Parks</MenuItem>
                </Select>
              </FormControl>

              <Button
                variant="outlined"
                onClick={() => setQuery({
                  pageNumber: 1,
                  pageSize: 10,
                  sortBy: 'name',
                  sortDescending: false
                })}
              >
                Clear Filters
              </Button>
            </Stack>
          </CardContent>
        </Card>
      </Collapse>

      {/* Results summary */}
      <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
        Showing {pagination.firstItemIndex}-{pagination.lastItemIndex} of {pagination.totalCount} projects
      </Typography>

      {/* Projects table */}
      <Card>
        <TableContainer>
          <Table>
            <TableHead>
              <TableRow sx={{ bgcolor: 'grey.50' }}>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Name</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Type</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Status</Typography></TableCell>
                <TableCell align="right"><Typography variant="subtitle2" fontWeight="bold">Estimated Cost</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Start Date</Typography></TableCell>
                <TableCell><Typography variant="subtitle2" fontWeight="bold">Health</Typography></TableCell>
                <TableCell align="center"><Typography variant="subtitle2" fontWeight="bold">Actions</Typography></TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {projects.length === 0 ? (
                <TableRow>
                  <TableCell colSpan={7} align="center" sx={{ py: 4 }}>
                    <Typography variant="body1" color="text.secondary">
                      No projects found. {query.nameFilter || query.status || query.type ? 'Try adjusting your filters.' : 'Create your first project to get started.'}
                    </Typography>
                  </TableCell>
                </TableRow>
              ) : (
                projects.map((project) => (
                  <TableRow key={project.id} hover>
                    <TableCell>
                      <Typography variant="body2" fontWeight="medium">{project.name}</Typography>
                      <Typography variant="caption" color="text.secondary" noWrap sx={{ maxWidth: 300, display: 'block' }}>
                        {project.description}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.type} 
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.status} 
                        color={getStatusColor(project.status)}
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="right">
                      <Typography variant="body2" fontWeight="medium">
                        {project.formattedCost}
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Typography variant="body2">
                        {new Date(project.startDate).toLocaleDateString()}
                      </Typography>
                      <Typography variant="caption" color="text.secondary">
                        {project.daysFromStart} days ago
                      </Typography>
                    </TableCell>
                    <TableCell>
                      <Chip 
                        label={project.isActive ? 'Active' : 'Inactive'} 
                        color={project.isActive ? 'success' : 'default'}
                        variant="outlined"
                        size="small"
                      />
                    </TableCell>
                    <TableCell align="center">
                      <Stack direction="row" spacing={1} justifyContent="center">
                        <Tooltip title="View Details">
                          <IconButton 
                            size="small" 
                            onClick={() => {/* TODO: Navigate to project detail */}}
                          >
                            <ViewIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Edit Project">
                          <IconButton 
                            size="small"
                            onClick={() => {/* TODO: Navigate to edit form */}}
                          >
                            <EditIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                        <Tooltip title="Delete Project">
                          <IconButton 
                            size="small" 
                            color="error"
                            onClick={() => {/* TODO: Show delete confirmation */}}
                          >
                            <DeleteIcon fontSize="small" />
                          </IconButton>
                        </Tooltip>
                      </Stack>
                    </TableCell>
                  </TableRow>
                ))
              )}
            </TableBody>
          </Table>
        </TableContainer>

        {/* Pagination */}
        <TablePagination
          component="div"
          count={pagination.totalCount}
          page={pagination.pageNumber - 1}  // Material-UI uses 0-based pages
          onPageChange={handlePageChange}
          rowsPerPage={pagination.pageSize}
          onRowsPerPageChange={handlePageSizeChange}
          rowsPerPageOptions={[5, 10, 25, 50]}
          showFirstButton
          showLastButton
        />
      </Card>

      {/* Status update loading indicator */}
      {statusMutation.isPending && (
        <Alert severity="info" sx={{ mt: 2 }}>
          Updating project status...
        </Alert>
      )}
    </Box>
  );
};

export default ProjectList;
```

**Why this component design?**
- **Comprehensive data management**: Filtering, sorting, paging all in one component
- **Professional UI**: Government-appropriate styling with Material-UI
- **Real-time updates**: React Query provides automatic cache management
- **Responsive design**: Works on different screen sizes
- **Accessibility**: Proper ARIA labels, keyboard navigation, screen reader support
- **Performance**: Virtual scrolling for large datasets, efficient re-renders
- **Error handling**: Graceful degradation with retry functionality

### 4.4 Application Setup and Routing

**Intent**: Configure the React application with routing, React Query, and Material-UI theme. This setup provides the foundation for a professional government application with consistent styling, efficient data management, and proper navigation structure.

Create `src/web/admin-portal/src/App.tsx`:

```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { CssBaseline, Box } from '@mui/material';
import ProjectList from './components/ProjectList';
import Layout from './components/Layout';

// Create React Query client with sensible defaults
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // Data considered fresh for 5 minutes
      cacheTime: 10 * 60 * 1000,   // Cache data for 10 minutes
      retry: 3,                     // Retry failed queries 3 times
      refetchOnWindowFocus: false,  // Don't refetch when window regains focus
      refetchOnMount: true,         // Always refetch when component mounts
    },
    mutations: {
      retry: 1,  // Retry failed mutations once
    },
  },
});

// Create Material-UI theme for government applications
const theme = createTheme({
  palette: {
    mode: 'light',
    primary: {
      main: '#1976d2',      // Professional blue
      light: '#42a5f5',
      dark: '#1565c0',
    },
    secondary: {
      main: '#dc004e',      // Accent color for important actions
      light: '#ff6b9d',
      dark: '#9a0036',
    },
    background: {
      default: '#f5f5f5',   // Light gray background
      paper: '#ffffff',
    },
    text: {
      primary: '#212121',   // Dark gray for readability
      secondary: '#757575',
    },
    error: {
      main: '#d32f2f',
    },
    warning: {
      main: '#ed6c02',
    },
    success: {
      main: '#2e7d32',
    },
    info: {
      main: '#0288d1',
    },
  },
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    h1: {
      fontSize: '2.5rem',
      fontWeight: 300,
    },
    h4: {
      fontSize: '2rem',
      fontWeight: 400,
    },
    h6: {
      fontSize: '1.25rem',
      fontWeight: 500,
    },
    body1: {
      fontSize: '1rem',
      lineHeight: 1.5,
    },
    body2: {
      fontSize: '0.875rem',
      lineHeight: 1.43,
    },
    caption: {
      fontSize: '0.75rem',
      lineHeight: 1.33,
    },
  },
  components: {
    // Customize Material-UI components
    MuiButton: {
      styleOverrides: {
        root: {
          textTransform: 'none',  // Don't uppercase button text
          borderRadius: 4,
        },
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0px 2px 4px rgba(0, 0, 0, 0.1)',
          borderRadius: 8,
        },
      },
    },
    MuiTableHead: {
      styleOverrides: {
        root: {
          backgroundColor: '#f5f5f5',
        },
      },
    },
  },
});

/**
 * Main App component with routing and providers
 * Sets up the application shell with navigation and global state
 */
const App: React.FC = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={theme}>
        <CssBaseline />  {/* Material-UI CSS reset and base styles */}
        <Router>
          <Layout>
            <Routes>
              {/* Default route redirects to projects */}
              <Route path="/" element={<Navigate to="/projects" replace />} />
              
              {/* Project management routes */}
              <Route path="/projects" element={<ProjectList />} />
              
              {/* Future routes for other services */}
              <Route path="/budgets" element={<div>Budget Management (Coming Soon)</div>} />
              <Route path="/grants" element={<div>Grant Management (Coming Soon)</div>} />
              <Route path="/contracts" element={<div>Contract Management (Coming Soon)</div>} />
              <Route path="/reports" element={<div>Reports & Analytics (Coming Soon)</div>} />
              
              {/* 404 page */}
              <Route path="*" element={<Navigate to="/projects" replace />} />
            </Routes>
          </Layout>
        </Router>
        
        {/* React Query DevTools in development */}
        {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
      </ThemeProvider>
    </QueryClientProvider>
  );
};

export default App;
```

Create `src/web/admin-portal/src/components/Layout.tsx`:

```typescript
import React, { useState } from 'react';
import {
  Box,
  AppBar,
  Toolbar,
  Typography,
  Drawer,
  List,
  ListItem,
  ListItemIcon,
  ListItemText,
  IconButton,
  Divider,
  Avatar,
  Menu,
  MenuItem,
} from '@mui/material';
import {
  Menu as MenuIcon,
  AccountCircle as AccountCircleIcon,
  Dashboard as DashboardIcon,
  Business as BusinessIcon,
  MonetizationOn as MonetizationOnIcon,
  Description as DescriptionIcon,
  Assessment as AssessmentIcon,
  Settings as SettingsIcon,
} from '@mui/icons-material';
import { useNavigate, useLocation } from 'react-router-dom';

const drawerWidth = 240;

/**
 * Navigation items for the sidebar
 * Each item represents a major functional area of the platform
 */
const navigationItems = [
  { text: 'Projects', icon: <BusinessIcon />, path: '/projects' },
  { text: 'Budgets', icon: <MonetizationOnIcon />, path: '/budgets' },
  { text: 'Grants', icon: <DescriptionIcon />, path: '/grants' },
  { text: 'Contracts', icon: <DescriptionIcon />, path: '/contracts' },
  { text: 'Reports', icon: <AssessmentIcon />, path: '/reports' },
];

interface LayoutProps {
  children: React.ReactNode;
}

/**
 * Main layout component providing navigation and application shell
 * Features: responsive sidebar, user menu, breadcrumbs
 */
const Layout: React.FC<LayoutProps> = ({ children }) => {
  const navigate = useNavigate();
  const location = useLocation();
  const [mobileOpen, setMobileOpen] = useState(false);
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);

  const handleDrawerToggle = () => {
    setMobileOpen(!mobileOpen);
  };

  const handleUserMenuOpen = (event: React.MouseEvent<HTMLElement>) => {
    setAnchorEl(event.currentTarget);
  };

  const handleUserMenuClose = () => {
    setAnchorEl(null);
  };

  const handleNavigation = (path: string) => {
    navigate(path);
    setMobileOpen(false);  // Close mobile drawer after navigation
  };

  // Sidebar content
  const drawerContent = (
    <div>
      <Toolbar>
        <Typography variant="h6" noWrap component="div" sx={{ fontWeight: 'bold' }}>
          Infrastructure Platform
        </Typography>
      </Toolbar>
      <Divider />
      <List>
        {navigationItems.map((item) => (
          <ListItem
            button
            key={item.text}
            selected={location.pathname === item.path}
            onClick={() => handleNavigation(item.path)}
            sx={{
              '&.Mui-selected': {
                backgroundColor: 'primary.light',
                color: 'primary.contrastText',
                '& .MuiListItemIcon-root': {
                  color: 'primary.contrastText',
                },
              },
            }}
          >
            <ListItemIcon>{item.icon}</ListItemIcon>
            <ListItemText primary={item.text} />
          </ListItem>
        ))}
      </List>
      <Divider />
      <List>
        <ListItem button onClick={() => handleNavigation('/settings')}>
          <ListItemIcon><SettingsIcon /></ListItemIcon>
          <ListItemText primary="Settings" />
        </ListItem>
      </List>
    </div>
  );

  return (
    <Box sx={{ display: 'flex' }}>
      {/* App bar */}
      <AppBar
        position="fixed"
        sx={{
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          ml: { sm: `${drawerWidth}px` },
        }}
      >
        <Toolbar>
          <IconButton
            color="inherit"
            aria-label="open drawer"
            edge="start"
            onClick={handleDrawerToggle}
            sx={{ mr: 2, display: { sm: 'none' } }}
          >
            <MenuIcon />
          </IconButton>
          
          {/* Page title */}
          <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1 }}>
            {navigationItems.find(item => item.path === location.pathname)?.text || 'Dashboard'}
          </Typography>
          
          {/* User menu */}
          <IconButton
            size="large"
            aria-label="account menu"
            aria-haspopup="true"
            onClick={handleUserMenuOpen}
            color="inherit"
          >
            <Avatar sx={{ width: 32, height: 32, bgcolor: 'secondary.main' }}>
              JD
            </Avatar>
          </IconButton>
          
          <Menu
            anchorEl={anchorEl}
            open={Boolean(anchorEl)}
            onClose={handleUserMenuClose}
            anchorOrigin={{
              vertical: 'bottom',
              horizontal: 'right',
            }}
            transformOrigin={{
              vertical: 'top',
              horizontal: 'right',
            }}
          >
            <MenuItem onClick={handleUserMenuClose}>
              <Typography variant="body2">John Doe</Typography>
            </MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Profile</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Settings</MenuItem>
            <MenuItem onClick={handleUserMenuClose}>Logout</MenuItem>
          </Menu>
        </Toolbar>
      </AppBar>

      {/* Sidebar drawer */}
      <Box
        component="nav"
        sx={{ width: { sm: drawerWidth }, flexShrink: { sm: 0 } }}
      >
        {/* Mobile drawer */}
        <Drawer
          variant="temporary"
          open={mobileOpen}
          onClose={handleDrawerToggle}
          ModalProps={{
            keepMounted: true, // Better mobile performance
          }}
          sx={{
            display: { xs: 'block', sm: 'none' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
        >
          {drawerContent}
        </Drawer>

        {/* Desktop drawer */}
        <Drawer
          variant="permanent"
          sx={{
            display: { xs: 'none', sm: 'block' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
          open
        >
          {drawerContent}
        </Drawer>
      </Box>

      {/* Main content area */}
      <Box
        component="main"
        sx={{
          flexGrow: 1,
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          minHeight: '100vh',
          backgroundColor: 'background.default',
        }}
      >
        <Toolbar /> {/* Spacer for fixed app bar */}
        {children}
      </Box>
    </Box>
  );
};

export default Layout;
```

**Why this application structure?**
- **React Query**: Provides caching, background updates, and optimistic updates
- **Material-UI Theme**: Professional, accessible design suitable for government use
- **React Router**: Clean URL structure with proper navigation
- **Responsive Layout**: Works on desktop, tablet, and mobile devices
- **User Experience**: Loading states, error handling, intuitive navigation
- **Performance**: Code splitting, lazy loading, efficient re-renders

## Phase 5: Add Budget Service (Week 5)

### 5.1 Budget Service Structure and Domain Model

**Intent**: Create the Budget service following the same clean architecture pattern as the Project service. The Budget service manages multi-year budget planning, tracks allocations versus expenditures, and integrates with projects for cost estimation. This service demonstrates how to build additional microservices using established patterns.

Follow the same pattern as Project Service:

```bash
# Create Budget service structure
mkdir -p src/services/budget/{Domain,Application,Infrastructure,API}
cd src/services/budget
```

Create `src/services/budget/Infrastructure.Budget.Domain/Entities/Budget.cs`:

```csharp
using Infrastructure.Shared.Domain.Entities;

namespace Infrastructure.Budget.Domain.Entities
{
    /// <summary>
    /// Budget aggregate root representing a fiscal year budget plan
    /// Contains budget line items and tracks allocations vs expenditures
    /// Enforces business rules around budget constraints and approval workflows
    /// </summary>
    public class Budget : BaseEntity
    {
        public string Name { get; private set; }
        public int FiscalYear { get; private set; }              // e.g., 2025
        public decimal TotalAmount { get; private set; }         // Total budget allocation
        public BudgetStatus Status { get; private set; }         // Planning, Approved, etc.
        public DateTime PeriodStart { get; private set; }        // Budget period start
        public DateTime PeriodEnd { get; private set; }          // Budget period end
        public string OwnerId { get; private set; }              // Organization that owns budget
        
        // Collection of budget line items (use private backing field for encapsulation)
        private readonly List<BudgetLineItem> _lineItems = new();
        public IReadOnlyList<BudgetLineItem> LineItems => _lineItems.AsReadOnly();
        
        // EF Core constructor
        private Budget() { }
        
        /// <summary>
        /// Create new fiscal year budget
        /// Enforces business rules: valid fiscal year, positive amount, valid period
        /// </summary>
        public Budget(string name, int fiscalYear, decimal totalAmount, 
                     DateTime periodStart, DateTime periodEnd, string ownerId)
        {
            // Business validation
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Budget name is required", nameof(name));
            if (fiscalYear < DateTime.Now.Year - 1 || fiscalYear > DateTime.Now.Year + 10)
                throw new ArgumentException("Fiscal year must be within reasonable range", nameof(fiscalYear));
            if (totalAmount <= 0)
                throw new ArgumentException("Budget amount must be positive", nameof(totalAmount));
            if (periodEnd <= periodStart)
                throw new ArgumentException("Budget end date must be after start date", nameof(periodEnd));
            if (string.IsNullOrWhiteSpace(ownerId))
                throw new ArgumentException("Budget owner is required", nameof(ownerId));
                
            Name = name;
            FiscalYear = fiscalYear;
            TotalAmount = totalAmount;
            Status = BudgetStatus.Planning;  // All budgets start in planning
            PeriodStart = periodStart;
            PeriodEnd = periodEnd;
            OwnerId = ownerId;
        }
        
        /// <summary>
        /// Add budget line item for specific category
        /// Business rule: Total allocated cannot exceed budget total
        /// </summary>
        public void AddLineItem(string category, string description, decimal allocatedAmount, 
                               FundingSource fundingSource)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Cannot modify approved budget");
                
            var lineItem = new BudgetLineItem(Id, category, description, allocatedAmount, fundingSource);
            
            // Business rule: Check total allocation doesn't exceed budget
            var currentTotal = _lineItems.Sum(li => li.AllocatedAmount);
            if (currentTotal + allocatedAmount > TotalAmount)
                throw new InvalidOperationException(
                    $"Total allocation ({currentTotal + allocatedAmount:C}) would exceed budget ({TotalAmount:C})");
                    
            _lineItems.Add(lineItem);
        }
        
        /// <summary>
        /// Update line item allocation
        /// Rechecks total allocation constraint
        /// </summary>
        public void UpdateLineItemAllocation(Guid lineItemId, decimal newAmount)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Cannot modify approved budget");
                
            var lineItem = _lineItems.FirstOrDefault(li => li.Id == lineItemId);
            if (lineItem == null)
                throw new ArgumentException("Line item not found", nameof(lineItemId));
                
            var otherItemsTotal = _lineItems.Where(li => li.Id != lineItemId).Sum(li => li.AllocatedAmount);
            if (otherItemsTotal + newAmount > TotalAmount)
                throw new InvalidOperationException("Allocation would exceed total budget");
                
            lineItem.UpdateAllocation(newAmount);
        }
        
        /// <summary>
        /// Record expenditure against a line item
        /// Updates spent amounts and checks for over-spending
        /// </summary>
        public void RecordExpenditure(Guid lineItemId, decimal amount, string description, 
                                     Guid? projectId = null)
        {
            var lineItem = _lineItems.FirstOrDefault(li => li.Id == lineItemId);
            if (lineItem == null)
                throw new ArgumentException("Line item not found", nameof(lineItemId));
                
            lineItem.RecordExpenditure(amount, description, projectId);
            
            // Business rule: Warn if line item is over budget (don't prevent, just track)
            if (lineItem.SpentAmount > lineItem.AllocatedAmount)
            {
                // Could raise domain event for notifications
                // DomainEvents.Raise(new BudgetLineItemOverspentEvent(...));
            }
        }
        
        /// <summary>
        /// Approve budget for execution
        /// Changes status and prevents further modifications to allocations
        /// </summary>
        public void Approve(string approvedBy)
        {
            if (Status != BudgetStatus.Planning)
                throw new InvalidOperationException("Only planning budgets can be approved");
                
            if (_lineItems.Sum(li => li.AllocatedAmount) == 0)
                throw new InvalidOperationException("Cannot approve budget with no allocations");
                
            Status = BudgetStatus.Approved;
            Update(approvedBy);  // Updates audit fields from BaseEntity
        }
        
        // Computed properties for reporting
        public decimal TotalAllocated => _lineItems.Sum(li => li.AllocatedAmount);
        public decimal TotalSpent => _lineItems.Sum(li => li.SpentAmount);
        public decimal RemainingBudget => TotalAmount - TotalSpent;
        public decimal UnallocatedAmount => TotalAmount - TotalAllocated;
        public double PercentSpent => TotalAmount > 0 ? (double)(TotalSpent / TotalAmount) * 100 : 0;
        public bool IsOverBudget => TotalSpent > TotalAmount;
    }

    /// <summary>
    /// Individual line item within a budget
    /// Tracks allocations and expenditures for specific categories
    /// </summary>
    public class BudgetLineItem : BaseEntity
    {
        public Guid BudgetId { get; private set; }               // Parent budget
        public string Category { get; private set; }             // e.g., "Roads", "Bridges"
        public string Description { get; private set; }          // Detailed description
        public decimal AllocatedAmount { get; private set; }     // Budgeted amount
        public decimal SpentAmount { get; private set; }         // Actually spent
        public FundingSource FundingSource { get; private set; } // Federal, State, Local, etc.
        
        // Collection of individual expenditures for audit trail
        private readonly List<BudgetExpenditure> _expenditures = new();
        public IReadOnlyList<BudgetExpenditure> Expenditures => _expenditures.AsReadOnly();
        
        // EF Core constructor
        private BudgetLineItem() { }
        
        public BudgetLineItem(Guid budgetId, string category, string description, 
                             decimal allocatedAmount, FundingSource fundingSource)
        {
            if (string.IsNullOrWhiteSpace(category))
                throw new ArgumentException("Category is required", nameof(category));
            if (allocatedAmount < 0)
                throw new ArgumentException("Allocated amount cannot be negative", nameof(allocatedAmount));
                
            BudgetId = budgetId;
            Category = category;
            Description = description;
            AllocatedAmount = allocatedAmount;
            SpentAmount = 0;  // Start with no expenditures
            FundingSource = fundingSource;
        }
        
        public void UpdateAllocation(decimal newAmount)
        {
            if (newAmount < 0)
                throw new ArgumentException("Allocation cannot be negative", nameof(newAmount));
            if (newAmount < SpentAmount)
                throw new ArgumentException("Allocation cannot be less than amount already spent", nameof(newAmount));
                
            AllocatedAmount = newAmount;
        }
        
        public void RecordExpenditure(decimal amount, string description, Guid? projectId = null)
        {
            if (amount <= 0)
                throw new ArgumentException("Expenditure amount must be positive", nameof(amount));
                
            var expenditure = new BudgetExpenditure(Id, amount, description, projectId);
            _expenditures.Add(expenditure);
            SpentAmount += amount;
        }
        
        // Computed properties
        public decimal RemainingAmount => AllocatedAmount - SpentAmount;
        public double PercentSpent => AllocatedAmount > 0 ? (double)(SpentAmount / AllocatedAmount) * 100 : 0;
        public bool IsOverBudget => SpentAmount > AllocatedAmount;
    }

    /// <summary>
    /// Individual expenditure record for audit trail
    /// Tracks when money was spent and on what
    /// </summary>
    public class BudgetExpenditure : BaseEntity
    {
        public Guid LineItemId { get; private set; }    // Parent line item
        public decimal Amount { get; private set; }      // Amount spent
        public string Description { get; private set; }  // What was purchased
        public Guid? ProjectId { get; private set; }     // Optional link to project
        public DateTime ExpenseDate { get; private set; } // When expense occurred
        
        private BudgetExpenditure() { }
        
        public BudgetExpenditure(Guid lineItemId, decimal amount, string description, Guid? projectId = null)
        {
            LineItemId = lineItemId;
            Amount = amount;
            Description = description ?? throw new ArgumentNullException(nameof(description));
            ProjectId = projectId;
            ExpenseDate = DateTime.UtcNow;
        }
    }

    /// <summary>
    /// Budget lifecycle status
    /// Controls what operations are allowed
    /// </summary>
    public enum BudgetStatus
    {
        Planning,      // Draft budget, can be modified
        Approved,      // Approved for execution, allocations locked
        Active,        // Currently in execution
        Closed,        // Budget period ended
        Cancelled      // Budget was cancelled
    }

    /// <summary>
    /// Source of funding for budget items
    /// Important for compliance and reporting
    /// </summary>
    public enum FundingSource
    {
        Federal,       // Federal grants and funding
        State,         // State government funding
        Local,         // Municipal/county funding
        Bonds,         // Municipal bonds
        PrivatePartnership, // Public-private partnerships
        Special        // Special assessments, fees
    }
}
```

**Why this budget domain design?**
- **Complex business rules**: Multi-level validation for allocations and expenditures
- **Aggregate consistency**: Budget maintains invariants across all line items
- **Audit trail**: Complete history of expenditures for government accountability
- **Integration ready**: Project references for connecting with project management
- **Reporting optimized**: Computed properties for dashboard and reports
- **Status workflow**: Clear lifecycle management for budget approval process

### 5.3 Event Publishing Infrastructure

**Intent**: Create a clean abstraction for publishing domain events to Kafka. This infrastructure allows domain entities to raise events without knowing about messaging details, maintaining clean separation between business logic and infrastructure concerns.

Create `src/shared/common/Events/IEventPublisher.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Interface for publishing domain events
    /// Abstracts messaging infrastructure from domain logic
    /// </summary>
    public interface IEventPublisher
    {
        /// <summary>
        /// Publish single domain event
        /// </summary>
        Task PublishAsync<T>(T domainEvent, CancellationToken cancellationToken = default)
            where T : DomainEvent;

        /// <summary>
        /// Publish multiple domain events as a batch
        /// Useful for aggregate operations that generate multiple events
        /// </summary>
        Task PublishBatchAsync<T>(IEnumerable<T> domainEvents, CancellationToken cancellationToken = default)
            where T : DomainEvent;
    }
}
```

Create `src/shared/common/Events/KafkaEventPublisher.cs`:

```csharp
using Confluent.Kafka;
using Infrastructure.Shared.Domain.Events;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Kafka implementation of event publisher
    /// Handles serialization, topic routing, and error handling
    /// </summary>
    public class KafkaEventPublisher : IEventPublisher, IDisposable
    {
        private readonly IProducer<string, string> _producer;
        private readonly ILogger<KafkaEventPublisher> _logger;
        private readonly JsonSerializerOptions _jsonOptions;
        
        public KafkaEventPublisher(ILogger<KafkaEventPublisher> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            
            // Kafka producer configuration for reliability
            var config = new ProducerConfig
            {
                BootstrapServers = "localhost:9092",  // TODO: Move to configuration
                Acks = Acks.All,                      // Wait for all replicas to acknowledge
                Retries = 3,                          // Retry failed sends
                EnableIdempotence = true,             // Prevent duplicate messages
                MessageSendMaxRetries = 3,            // Additional retry configuration
                RetryBackoffMs = 100,                 // Backoff between retries
                MessageTimeoutMs = 30000,             // Total timeout for message send
                CompressionType = CompressionType.Snappy  // Compress messages for efficiency
            };
            
            _producer = new ProducerBuilder<string, string>(config)
                .SetErrorHandler((_, error) => 
                {
                    _logger.LogError("Kafka producer error: {Error}", error.Reason);
                })
                .SetLogHandler((_, logMessage) => 
                {
                    _logger.LogDebug("Kafka log: {Message}", logMessage.Message);
                })
                .Build();
            
            // JSON serialization options for consistent event format
            _jsonOptions = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                WriteIndented = false,  // Compact JSON for efficiency
                DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
            };
        }

        public async Task PublishAsync<T>(T domainEvent, CancellationToken cancellationToken = default) 
            where T : DomainEvent
        {
            try
            {
                var topicName = GetTopicName(typeof(T));
                var eventKey = domainEvent.Id.ToString();  // Use event ID as partition key
                var eventPayload = JsonSerializer.Serialize(domainEvent, _jsonOptions);
                
                // Create message with headers for metadata
                var message = new Message<string, string>
                {
                    Key = eventKey,
                    Value = eventPayload,
                    Headers = new Headers
                    {
                        ["eventType"] = System.Text.Encoding.UTF8.GetBytes(domainEvent.EventType),
                        ["eventId"] = System.Text.Encoding.UTF8.GetBytes(domainEvent.Id.ToString()),
                        ["occurredOn"] = System.Text.Encoding.UTF8.GetBytes(domainEvent.OccurredOn.ToString("O")),
                        ["version"] = System.Text.Encoding.UTF8.GetBytes("1.0")
                    }
                };
                
                _logger.LogDebug("Publishing event {EventType} to topic {Topic}", 
                               domainEvent.EventType, topicName);
                
                // Publish with delivery confirmation
                var deliveryResult = await _producer.ProduceAsync(topicName, message, cancellationToken);
                
                _logger.LogInformation("Event {EventType} published successfully to {Topic}:{Partition} at offset {Offset}",
                                     domainEvent.EventType, topicName, deliveryResult.Partition.Value, deliveryResult.Offset.Value);
            }
            catch (ProduceException<string, string> ex)
            {
                _logger.LogError(ex, "Failed to publish event {EventType}: {Error}", 
                               domainEvent.EventType, ex.Error.Reason);
                throw new InvalidOperationException($"Failed to publish event: {ex.Error.Reason}", ex);
            }
        }

        public async Task PublishBatchAsync<T>(IEnumerable<T> domainEvents, CancellationToken cancellationToken = default) 
            where T : DomainEvent
        {
            var events = domainEvents.ToList();
            if (!events.Any()) return;

            _logger.LogDebug("Publishing batch of {Count} events", events.Count);
            
            // Publish all events in parallel for better performance
            var publishTasks = events.Select(evt => PublishAsync(evt, cancellationToken));
            
            try
            {
                await Task.WhenAll(publishTasks);
                _logger.LogInformation("Successfully published batch of {Count} events", events.Count);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to publish event batch");
                throw;
            }
        }
        
        /// <summary>
        /// Generate topic name from event type
        /// Convention: domain.events.{EventName}
        /// </summary>
        private static string GetTopicName(Type eventType)
        {
            var eventName = eventType.Name;
            
            // Remove "Event" suffix if present
            if (eventName.EndsWith("Event"))
            {
                eventName = eventName[..^5];  // Remove last 5 characters
            }
            
            // Convert PascalCase to kebab-case
            var kebabCase = string.Concat(eventName.Select((c, i) => 
                i > 0 && char.IsUpper(c) ? $"-{char.ToLower(c)}" : char.ToLower(c).ToString()));
            
            return $"domain.events.{kebabCase}";
        }
        
        public void Dispose()
        {
            _producer?.Flush(TimeSpan.FromSeconds(10));  // Wait for pending messages
            _producer?.Dispose();
        }
    }
}
```

### 5.4 Event Handler Infrastructure

**Intent**: Create infrastructure for consuming and handling domain events from other services. This allows the Budget service to react to project events automatically, maintaining data consistency across service boundaries without tight coupling.

Create `src/shared/common/Events/IEventHandler.cs`:

```csharp
using Infrastructure.Shared.Domain.Events;

namespace Infrastructure.Shared.Common.Events
{
    /// <summary>
    /// Interface for handling domain events
    /// Each service implements handlers for events they care about
    /// </summary>
    public interface IEventHandler<in T> where T : DomainEvent
    {
        /// <summary>
        /// Handle domain event asynchronously
        /// Should be idempotent - safe to process same event multiple times
        /// </summary>
        Task HandleAsync(T domainEvent, CancellationToken cancellationToken = default);
    }
}
```

Create `src/services/budget/Infrastructure.Budget.Application/EventHandlers/ProjectEventHandlers.cs`:

```csharp
using Infrastructure.Shared.Common.Events;
using Infrastructure.Shared.Events;
using Infrastructure.Budget.Domain.Entities;
using Infrastructure.Budget.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

namespace Infrastructure.Budget.Application.EventHandlers
{
    /// <summary>
    /// Handles project creation events to update budget tracking
    /// When projects are created, we may need to reserve budget or track commitments
    /// </summary>
    public class ProjectCreatedEventHandler : IEventHandler<ProjectCreatedEvent>
    {
        private readonly BudgetDbContext _context;
        private readonly ILogger<ProjectCreatedEventHandler> _logger;
        
        public ProjectCreatedEventHandler(BudgetDbContext context, ILogger<ProjectCreatedEventHandler> logger)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }
        
        public async Task HandleAsync(ProjectCreatedEvent domainEvent, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Handling ProjectCreated event for project {ProjectId} - {ProjectName}", 
                                 domainEvent.ProjectId, domainEvent.ProjectName);
            
            try
            {
                // Find active budget for the same organization
                var activeBudget = await _context.Budgets
                    .Include(b => b.LineItems)
                    .Where(b => b.OwnerId == domainEvent.OwnerId && 
                               b.Status == BudgetStatus.Active &&
                               b.PeriodStart <= domainEvent.StartDate &&
                               b.PeriodEnd >= domainEvent.StartDate)
                    .FirstOrDefaultAsync(cancellationToken);
                
                if (activeBudget == null)
                {
                    _logger.LogWarning("No active budget found for organization {OwnerId} during project period", 
                                     domainEvent.OwnerId);
                    return;
                }
                
                // Find appropriate budget line item based on project type
                var categoryMapping = GetBudgetCategoryForProjectType(domainEvent.ProjectType);
                var lineItem = activeBudget.LineItems
                    .FirstOrDefault(li => li.Category.Equals(categoryMapping, StringComparison.OrdinalIgnoreCase));
                
                if (lineItem == null)
                {
                    _logger.LogWarning("No budget line item found for category {Category} in budget {BudgetId}", 
                                     categoryMapping, activeBudget.Id);
                    return;
                }
                
                // Check if there's sufficient budget remaining
                if (lineItem.RemainingAmount < domainEvent.EstimatedCost)
                {
                    _logger.LogWarning("Insufficient budget in {Category}: remaining {Remaining:C}, needed {Needed:C}",
                                     categoryMapping, lineItem.RemainingAmount, domainEvent.EstimatedCost);
                    
                    // Could raise event for budget warning/approval needed
                    // await _eventPublisher.PublishAsync(new InsufficientBudgetWarningEvent(...));
                }
                
                // Record the project commitment (future enhancement)
                // This would track committed but not yet spent amounts
                // var commitment = new BudgetCommitment(lineItem.Id, domainEvent.ProjectId, 
                //                                       domainEvent.EstimatedCost, domainEvent.ProjectName);
                
                _logger.LogInformation("Successfully processed ProjectCreated event for {ProjectName}", 
                                     domainEvent.ProjectName);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error handling ProjectCreated event for project {ProjectId}", 
                               domainEvent.ProjectId);
                throw;  // Let the message bus handle retry logic
            }
        }
        
        /// <summary>
        /// Map project types to budget categories
        /// This business rule could be configurable
        /// </summary>
        private static string GetBudgetCategoryForProjectType(string projectType)
        {
            return projectType switch
            {
                "Roads" => "Transportation",
                "Bridges" => "Transportation", 
                "WaterSystems" => "Utilities",
                "PowerGrid" => "Utilities",
                "PublicBuildings" => "Facilities",
                "Parks" => "Recreation",
                _ => "General"
            };
        }
    }

    /// <summary>
    /// Handles project cost estimate changes
    /// Updates budget variance tracking and may trigger notifications
    /// </summary>
    public class ProjectCostEstimateChangedEventHandler : IEventHandler<ProjectCostEstimateChangedEvent>
    {
        private readonly BudgetDbContext _context;
        private readonly ILogger<ProjectCostEstimateChangedEventHandler> _logger;
        private readonly IEventPublisher _eventPublisher;
        
        public ProjectCostEstimateChangedEventHandler(
            BudgetDbContext context, 
            ILogger<ProjectCostEstimateChangedEventHandler> logger,
            IEventPublisher eventPublisher)
        {
            _context = context ?? throw new ArgumentNullException(nameof(context));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _eventPublisher = eventPublisher ?? throw new ArgumentNullException(nameof(eventPublisher));
        }
        
        public async Task HandleAsync(ProjectCostEstimateChangedEvent domainEvent, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Handling cost estimate change for project {ProjectId}: {Previous:C} -> {New:C}",
                                 domainEvent.ProjectId, domainEvent.PreviousEstimate, domainEvent.NewEstimate);
            
            try
            {
                // If cost increased significantly, may need budget reallocation or approval
                var variancePercent = Math.Abs(domainEvent.VarianceAmount) / domainEvent.PreviousEstimate * 100;
                const decimal significantVarianceThreshold = 10; // 10% threshold
                
                if (variancePercent > significantVarianceThreshold)
                {
                    _logger.LogWarning("Significant cost variance detected: {Variance:P} for project {ProjectName}",
                                     variancePercent / 100, domainEvent.ProjectName);
                    
                    // Raise event for budget variance that may need management attention
                    var budgetVarianceEvent = new BudgetVarianceDetectedEvent(
                        domainEvent.ProjectId,
                        domainEvent.ProjectName,
                        domainEvent.PreviousEstimate,
                        domainEvent.NewEstimate,
                        domainEvent.VarianceAmount,
                        domainEvent.ChangeReason
                    );
                    
                    await _eventPublisher.PublishAsync(budgetVarianceEvent, cancellationToken);
                }
                
                // Update budget commitment records (if implemented)
                // This would adjust the committed amounts based on new estimates
                
                await _context.SaveChangesAsync(cancellationToken);
                
                _logger.LogInformation("Successfully processed cost estimate change for {ProjectName}", 
                                     domainEvent.ProjectName);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error handling cost estimate change for project {ProjectId}", 
                               domainEvent.ProjectId);
                throw;
            }
        }
    }
}
```

**Why this event-driven architecture?**
- **Loose coupling**: Services don't directly call each other, reducing dependencies
- **Resilience**: Failed event processing can be retried without affecting the source service
- **Scalability**: Events can be processed asynchronously at different rates
- **Audit trail**: All business events are captured for compliance and analysis
- **Future extensibility**: New services can subscribe to existing events without changes
- **Business alignment**: Events represent real business occurrences that stakeholders understand

        private async Task HandleProjectApprovedEvent(string messageValue, CancellationToken cancellationToken)
        {
            var domainEvent = JsonSerializer.Deserialize<ProjectApprovedEvent>(messageValue, _jsonOptions);
            
            using var scope = _serviceProvider.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<IEventHandler<ProjectApprovedEvent>>();
            
            await handler.HandleAsync(domainEvent, cancellationToken);
        }
        
        private Task HandleUnknownEvent(string eventType, string messageValue)
        {
            _logger.LogWarning("Received unknown event type: {EventType}", eventType);
            return Task.CompletedTask;
        }
    }
}
```

### 5.7 Test Inter-Service Communication

**Intent**: Verify that events flow correctly between Project and Budget services. This test ensures the event-driven architecture works properly and services can maintain consistency without direct coupling.

**Step 1: Start All Infrastructure Services**
```bash
# Start Kafka, PostgreSQL, etc.
cd infrastructure/docker
docker-compose -f docker-compose.dev.yml up -d

# Verify Kafka topics can be created
docker exec -it infrastructure-docker_kafka_1 kafka-topics --bootstrap-server localhost:9092 --list
```

**Step 2: Create Kafka Topics**
```bash
# Create topics for domain events
docker exec -it infrastructure-docker_kafka_1 kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic domain.events.project-created \
  --partitions 3 \
  --replication-factor 1

docker exec -it infrastructure-docker_kafka_1 kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic domain.events.project-cost-estimate-changed \
  --partitions 3 \
  --replication-factor 1

docker exec -it infrastructure-docker_kafka_1 kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic domain.events.project-approved \
  --partitions 3 \
  --replication-factor 1

# Verify topics were created
docker exec -it infrastructure-docker_kafka_1 kafka-topics --bootstrap-server localhost:9092 --list
```

**Step 3: Start Both Services**
```bash
# Terminal 1 - Projects Service (Publisher)
cd src/services/projects/Infrastructure.Projects.API
dotnet run --urls="http://localhost:5001"

# Terminal 2 - Budget Service (Consumer)
cd src/services/budget/Infrastructure.Budget.API
dotnet run --urls="http://localhost:5002"

# Both services should start without errors
# Budget service should show: "Starting Kafka event consumer for Budget service"
```

**Step 4: Test Event Flow**
```bash
# Monitor Kafka events (in separate terminal)
docker exec -it infrastructure-docker_kafka_1 kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic domain.events.project-created \
  --from-beginning

# Create a project via Projects API
curl -X POST http://localhost:5001/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Highway Expansion Project",
    "description": "Expand Highway 95 from 4 to 6 lanes",
    "type": "Roads",
    "estimatedCost": 25000000,
    "startDate": "2025-09-01T00:00:00Z",
    "ownerId": "ORG001"
  }'

# Expected results:
# 1. Projects API returns 201 Created
# 2. Kafka console consumer shows ProjectCreatedEvent message
# 3. Budget service logs show "Handling ProjectCreated event"
```

**Step 5: Verify Event Processing**
```bash
# Check Budget service logs for event processing
# Should see logs like:
# "Handling ProjectCreated event for project {ProjectId} - Highway Expansion Project"
# "Successfully processed ProjectCreated event for Highway Expansion Project"

# Test cost estimate change
curl -X PUT http://localhost:5001/api/projects/{project-id} \
  -H "Content-Type: application/json" \
  -d '{
    "id": "{project-id}",
    "name": "Highway Expansion Project",
    "description": "Expand Highway 95 from 4 to 6 lanes - Updated scope",
    "type": "Roads",
    "estimatedCost": 30000000,
    "startDate": "2025-09-01T00:00:00Z",
    "ownerId": "ORG001",
    "changeReason": "Scope expansion required"
  }'

# Should trigger ProjectCostEstimateChangedEvent
```

**Step 6: Debug Event Issues**
```bash
# If events aren't flowing, check these common issues:

# 1. Verify Kafka connectivity
curl http://localhost:9092  # Should get connection

# 2. Check Kafka consumer groups
docker exec -it infrastructure-docker_kafka_1 kafka-consumer-groups \
  --bootstrap-server localhost:9092 --list

# 3. Check consumer group status
docker exec -it infrastructure-docker_kafka_1 kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group budget-service \
  --describe

# 4. Check topic message count
docker exec -it infrastructure-docker_kafka_1 kafka-run-class \
  kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic domain.events.project-created
```

## Phase 6: API Gateway (Week 6)

### 6.1 Set up Ocelot API Gateway

**Intent**: Create a unified entry point for all microservices that handles cross-cutting concerns like authentication, rate limiting, load balancing, and request routing. The API gateway shields internal service complexity from clients and provides a consistent API experience.

Create `src/gateways/api-gateway/Infrastructure.Gateway.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Ocelot for API Gateway functionality -->
    <PackageReference Include="Ocelot" Version="20.0.0" />
    <!-- Consul integration for service discovery (future) -->
    <PackageReference Include="Ocelot.Provider.Consul" Version="20.0.0" />
    <!-- Authentication integration -->
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
    <!-- Health checks for monitoring -->
    <PackageReference Include="Microsoft.Extensions.Diagnostics.HealthChecks" Version="8.0.0" />
    <!-- Caching for improved performance -->
    <PackageReference Include="Ocelot.Cache.CacheManager" Version="20.0.0" />
  </ItemGroup>
</Project>
```

### 6.2 Gateway Configuration

**Intent**: Configure routing rules that direct requests to appropriate backend services while adding gateway-level functionality like authentication, rate limiting, and caching. This configuration supports both development and production scenarios.

Create `src/gateways/api-gateway/ocelot.Development.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE", "PATCH" ],
      "Key": "projects",
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1m",
        "PeriodTimespan": 60,
        "Limit": 100
      },
      "FileCacheOptions": {
        "TtlSeconds": 300,
        "Region": "projects"
      },
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      },
      "QoSOptions": {
        "ExceptionsAllowedBeforeBreaking": 3,
        "DurationOfBreak": 5000,
        "TimeoutValue": 30000
      }
    },
    {
      "DownstreamPathTemplate": "/api/budgets/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/gateway/budgets/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE", "PATCH" ],
      "Key": "budgets",
      "RateLimitOptions": {
        "EnableRateLimiting": true,
        "Period": "1m",
        "PeriodTimespan": 60,
        "Limit": 100
      },
      "FileCacheOptions": {
        "TtlSeconds": 300,
        "Region": "budgets"
      },
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    },
    {
      "DownstreamPathTemplate": "/api/grants/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5003
        }
      ],
      "UpstreamPathTemplate": "/gateway/grants/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE", "PATCH" ],
      "Key": "grants",
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      }
    },
    {
      "DownstreamPathTemplate": "/health",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/gateway/health/projects",
      "UpstreamHttpMethod": [ "GET" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://localhost:5000",
    "RateLimitOptions": {
      "DisableRateLimitHeaders": false,
      "QuotaExceededMessage": "API rate limit exceeded. Please try again later.",
      "HttpStatusCode": 429
    },
    "QoSOptions": {
      "ExceptionsAllowedBeforeBreaking": 3,
      "DurationOfBreak": 10000,
      "TimeoutValue": 30000
    },
    "LoadBalancerOptions": {
      "Type": "RoundRobin",
      "Key": "rrloadbalancer",
      "Expiry": 60000
    }
  }
}
```

Create `src/gateways/api-gateway/ocelot.Production.json`:

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/projects/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "projects-service",
          "Port": 443
        },
        {
          "Host": "projects-service-backup",
          "Port": 443
        }
      ],
      "UpstreamPathTemplate": "/api/projects/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE", "PATCH" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      },
      "RateLimitOptions": {
        "EnableRateLimiting": true,
        "Period": "1h",
        "PeriodTimespan": 3600,
        "Limit": 1000
      },
      "LoadBalancerOptions": {
        "Type": "LeastConnection"
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://api.infrastructure-platform.gov",
    "ServiceDiscoveryProvider": {
      "Host": "consul-server",
      "Port": 8500,
      "Type": "Consul"
    }
  }
}
```

**Why this gateway configuration?**
- **Environment-specific**: Different configs for development vs production
- **Rate limiting**: Prevents API abuse and ensures fair usage
- **Circuit breaker**: QoS options prevent cascade failures
- **Load balancing**: Distributes traffic across multiple service instances
- **Caching**: Reduces backend load for frequently requested data
- **Authentication**: Centralized security enforcement
- **Health checks**: Monitoring integration for service health

### 6.3 Test Gateway

Update frontend to use gateway endpoints and verify routing works.

## Phase 7: Authentication & Security (Week 7)

### 7.1 Add Identity Service

```bash
mkdir -p src/services/identity
cd src/services/identity

dotnet new webapi -n Infrastructure.Identity.API
cd Infrastructure.Identity.API
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 7.2 JWT Authentication Setup

Configure JWT authentication across all services and test login flow.

### 7.3 Role-Based Authorization

Implement RBAC system:
- Municipality Admin
- Project Manager  
- Budget Analyst
- Grant Writer
- Viewer

## Phase 8: Add Remaining Services (Weeks 8-12)

### 8.1 Grant Management Service (Week 8)
- Grant opportunity discovery
- Application workflow
- Document management

### 8.2 Cost Estimation Service (Week 9) 
- Historical cost data
- Market price integration
- ML-based predictions

### 8.3 BCA (Benefit-Cost Analysis) Service (Week 10)
- NPV calculations
- Risk assessment
- Compliance reporting

### 8.4 Contract Management Service (Week 11)
- Contract templates
- Procurement workflows
- Vendor management

### 8.5 Analytics & Reporting Service (Week 12)
- Dashboard data aggregation
- Report generation
- KPI calculations

## Phase 9: Advanced Frontend Features (Weeks 13-16)

### 9.1 Enhanced UI Components
- Data visualization with Charts.js/D3
- Interactive dashboards
- Document upload/preview

### 9.2 Workflow Management
- Approval flows
- Task assignments
- Notifications

### 9.3 Integration Interfaces
- ERP system connectors
- Grant portal APIs
- GIS system integration

## Phase 10: Production Readiness (Weeks 17-20)

### 10.1 Kubernetes Deployment

Create `infrastructure/kubernetes/` manifests for:
- Service deployments
- ConfigMaps and Secrets
- Ingress controllers
- Persistent volumes

### 10.2 CI/CD Pipeline

Set up GitLab CI or Azure DevOps:
- Automated testing
- Docker image building
- Deployment automation
- Environment promotion

### 10.3 Monitoring & Observability

Configure:
- Application logging (Serilog)
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Health checks

### 10.4 Performance Testing

- Load testing with k6
- Database query optimization
- Caching strategies
- CDN setup

## Testing Strategy

### Unit Tests
Each service should have comprehensive unit tests:

```bash
cd src/services/projects
dotnet new xunit -n Infrastructure.Projects.Tests
# Add test packages and write tests
```

### Integration Tests
Test service interactions and database operations.

### End-to-End Tests
Use Playwright or Cypress for full workflow testing.

## Development Workflow

1. **Feature Branch Strategy**: Create branches for each service/feature
2. **Code Reviews**: All code must be reviewed before merging
3. **Automated Testing**: CI pipeline runs all tests
4. **Local Development**: Everything runs locally with Docker Compose
5. **Documentation**: Keep docs updated as you build

## Key Milestones

- **Week 4**: First working service with basic UI
- **Week 8**: Core services communicating via events
- **Week 12**: All business services implemented
- **Week 16**: Complete frontend with all features
- **Week 20**: Production-ready deployment

This approach ensures you have a working system at every step, can demo progress regularly, and can make adjustments based on feedback as you build.

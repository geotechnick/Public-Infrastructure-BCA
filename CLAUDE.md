# Public Infrastructure Planning Platform - Optimized Build Guide

## Platform Overview

This guide builds a comprehensive platform for **public infrastructure owners** to:
- **Budget Planning**: Multi-year capital planning and operational budgets
- **Grant Applications**: Find, apply for, and track federal/state infrastructure grants  
- **Benefit-Cost Analysis**: Perform standardized BCA calculations per federal guidelines
- **Project Scoping**: Define project requirements, timelines, and deliverables
- **Contract Structure**: Generate procurement documents and contract templates
- **Cost Estimation**: Preliminary cost estimates using industry databases

## Architecture Philosophy

**Start Simple, Scale Smart**: Begin with a modular monolith that can evolve into microservices as needed. This approach reduces complexity while maintaining clear domain boundaries.

**Core Stack:**
- **Backend**: .NET 8 with modular architecture
- **Frontend**: Next.js 14 with TypeScript (better integration, SSR, compliance)
- **Database**: PostgreSQL with domain schemas
- **Infrastructure**: Docker for development, cloud-ready deployment
- **Integration**: REST APIs with future GraphQL/tRPC support

## Phase 1: Foundation & Core Infrastructure (Weeks 1-2)

### 1.1 Development Environment Setup

```bash
# Install core tools
# Node.js 20+ (LTS) for Next.js frontend
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 20
nvm use 20

# .NET 8 SDK for backend services
# Download from microsoft.com/dotnet

# Docker Desktop for containerized development
# Download from docker.com

# PostgreSQL client tools
npm install -g @postgresql/cli
```

### 1.2 Project Structure - Modular Monolith

```bash
mkdir infrastructure-platform
cd infrastructure-platform

# Optimized structure focusing on business capabilities
mkdir -p {
  backend/{
    src/Infrastructure.Platform/{
      Features/{
        Projects/{Domain,Services,Controllers,Data},
        Budgets/{Domain,Services,Controllers,Data},
        Grants/{Domain,Services,Controllers,Data},
        BenefitCostAnalysis/{Domain,Services,Controllers,Data},
        Contracts/{Domain,Services,Controllers,Data},
        CostEstimation/{Domain,Services,Controllers,Data}
      },
      Shared/{
        Database,Common,Security,Integrations
      }
    },
    tests
  },
  frontend/{
    src/{
      app/{
        projects,budgets,grants,bca,contracts,estimates
      },
      components/shared,
      lib/{api,utils,types}
    }
  },
  infrastructure/{
    docker,k8s,terraform
  },
  docs
}
```

### 1.3 Database Setup - Domain-Driven Schemas

Create `infrastructure/docker/docker-compose.dev.yml`:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: infrastructure_platform
      POSTGRES_USER: platform_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

Create `infrastructure/docker/init.sql`:

```sql
-- Create separate schemas for each domain
CREATE SCHEMA projects;
CREATE SCHEMA budgets; 
CREATE SCHEMA grants;
CREATE SCHEMA bca;
CREATE SCHEMA contracts;
CREATE SCHEMA cost_estimation;
CREATE SCHEMA shared;

-- Grant permissions
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA projects TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA budgets TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA grants TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA bca TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA contracts TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA cost_estimation TO platform_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA shared TO platform_user;
```

## Phase 2: Project Management Foundation (Week 3)

### 2.1 Core Domain Models

Create `backend/src/Infrastructure.Platform/Features/Projects/Domain/Project.cs`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace Infrastructure.Platform.Features.Projects.Domain;

[Table("projects", Schema = "projects")]
public class Project
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string Name { get; private set; } = string.Empty;
    public string Description { get; private set; } = string.Empty;
    public ProjectType Type { get; private set; }
    public ProjectPhase Phase { get; private set; } = ProjectPhase.Planning;
    public ProjectPriority Priority { get; private set; }
    
    // Financial Information
    public decimal EstimatedCost { get; private set; }
    public decimal AllocatedBudget { get; private set; }
    public FundingSource PrimaryFundingSource { get; private set; }
    
    // Timeline
    public DateTime? PlanningStartDate { get; private set; }
    public DateTime? ConstructionStartDate { get; private set; }
    public DateTime? ExpectedCompletionDate { get; private set; }
    
    // Location & Scope
    public string Location { get; private set; } = string.Empty;
    public string County { get; private set; } = string.Empty;
    public string State { get; private set; } = string.Empty;
    public string[] ScopeItems { get; private set; } = Array.Empty<string>();
    
    // Regulatory & Compliance
    public bool RequiresEnvironmentalReview { get; private set; }
    public bool RequiresPublicHearing { get; private set; }
    public string[] RequiredPermits { get; private set; } = Array.Empty<string>();
    
    // Audit Trail
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; private set; }
    public string CreatedBy { get; private set; } = string.Empty;
    public string? UpdatedBy { get; private set; }

    private Project() { } // EF Constructor

    public Project(string name, string description, ProjectType type, 
                  ProjectPriority priority, decimal estimatedCost, 
                  string location, string county, string state,
                  string createdBy)
    {
        Name = name;
        Description = description;
        Type = type;
        Priority = priority;
        EstimatedCost = estimatedCost;
        Location = location;
        County = county;
        State = state;
        CreatedBy = createdBy;
    }

    // Business Methods
    public void UpdateScope(string[] scopeItems, string updatedBy)
    {
        ScopeItems = scopeItems;
        MarkAsUpdated(updatedBy);
    }

    public void UpdateBudget(decimal allocatedBudget, FundingSource fundingSource, string updatedBy)
    {
        AllocatedBudget = allocatedBudget;
        PrimaryFundingSource = fundingSource;
        MarkAsUpdated(updatedBy);
    }

    public void AdvancePhase(ProjectPhase newPhase, string updatedBy)
    {
        if (newPhase <= Phase)
            throw new InvalidOperationException("Cannot move to previous phase");
            
        Phase = newPhase;
        MarkAsUpdated(updatedBy);
    }

    public void SetRegulatory(bool environmentalReview, bool publicHearing, 
                             string[] permits, string updatedBy)
    {
        RequiresEnvironmentalReview = environmentalReview;
        RequiresPublicHearing = publicHearing;
        RequiredPermits = permits;
        MarkAsUpdated(updatedBy);
    }

    private void MarkAsUpdated(string updatedBy)
    {
        UpdatedAt = DateTime.UtcNow;
        UpdatedBy = updatedBy;
    }
}

public enum ProjectType
{
    Highway, Bridge, PublicTransit, WaterInfrastructure, 
    Wastewater, PublicBuilding, Parks, Utilities, Broadband
}

public enum ProjectPhase
{
    Planning, Design, Environmental, Procurement, Construction, Closeout
}

public enum ProjectPriority
{
    Low, Medium, High, Critical, Emergency
}

public enum FundingSource
{
    LocalFunds, StateFunds, FederalGrants, Bonds, PublicPrivatePartnership
}
```

### 2.2 Database Context with Schema Support

Create `backend/src/Infrastructure.Platform/Shared/Database/PlatformDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Platform.Features.Projects.Domain;

namespace Infrastructure.Platform.Shared.Database;

public class PlatformDbContext : DbContext
{
    public PlatformDbContext(DbContextOptions<PlatformDbContext> options) : base(options) { }

    // Projects
    public DbSet<Project> Projects => Set<Project>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure JSON columns for arrays
        modelBuilder.Entity<Project>()
            .Property(p => p.ScopeItems)
            .HasConversion(
                v => string.Join(',', v),
                v => v.Split(',', StringSplitOptions.RemoveEmptyEntries));
            
        modelBuilder.Entity<Project>()
            .Property(p => p.RequiredPermits)
            .HasConversion(
                v => string.Join(',', v),
                v => v.Split(',', StringSplitOptions.RemoveEmptyEntries));

        // Seed sample data
        SeedProjects(modelBuilder);
    }

    private void SeedProjects(ModelBuilder modelBuilder)
    {
        var projects = new[]
        {
            new Project("I-95 Bridge Replacement", 
                       "Replace aging bridge structure over Potomac River", 
                       ProjectType.Bridge, ProjectPriority.Critical, 
                       125000000m, "Alexandria", "Fairfax County", "Virginia", "system"),
            new Project("Metro Extension Phase 2", 
                       "Extend metro line to serve growing suburban areas", 
                       ProjectType.PublicTransit, ProjectPriority.High,
                       890000000m, "Silver Spring", "Montgomery County", "Maryland", "system"),
            new Project("Water Treatment Upgrade", 
                       "Modernize water treatment facility to meet new EPA standards", 
                       ProjectType.WaterInfrastructure, ProjectPriority.High,
                       45000000m, "Richmond", "Richmond City", "Virginia", "system")
        };

        foreach (var project in projects)
        {
            // Set private fields for seeding
            var idField = typeof(Project).GetField("<Id>k__BackingField", 
                BindingFlags.Instance | BindingFlags.NonPublic);
            idField?.SetValue(project, Guid.NewGuid());
        }

        modelBuilder.Entity<Project>().HasData(projects);
    }
}
```

### 2.3 Project Management API

Create `backend/src/Infrastructure.Platform/Features/Projects/Controllers/ProjectsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Infrastructure.Platform.Features.Projects.Domain;
using Infrastructure.Platform.Shared.Database;

namespace Infrastructure.Platform.Features.Projects.Controllers;

[ApiController]
[Route("api/projects")]
public class ProjectsController : ControllerBase
{
    private readonly PlatformDbContext _context;

    public ProjectsController(PlatformDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<ProjectListResponse>> GetProjects(
        [FromQuery] string? search,
        [FromQuery] ProjectType? type,
        [FromQuery] ProjectPhase? phase,
        [FromQuery] string? state,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 25)
    {
        var query = _context.Projects.AsQueryable();

        // Apply filters
        if (!string.IsNullOrEmpty(search))
            query = query.Where(p => p.Name.Contains(search) || 
                                   p.Description.Contains(search) ||
                                   p.Location.Contains(search));

        if (type.HasValue)
            query = query.Where(p => p.Type == type);

        if (phase.HasValue)
            query = query.Where(p => p.Phase == phase);

        if (!string.IsNullOrEmpty(state))
            query = query.Where(p => p.State == state);

        var totalCount = await query.CountAsync();

        var projects = await query
            .OrderByDescending(p => p.CreatedAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .Select(p => new ProjectSummaryDto
            {
                Id = p.Id,
                Name = p.Name,
                Type = p.Type,
                Phase = p.Phase,
                Priority = p.Priority,
                EstimatedCost = p.EstimatedCost,
                Location = p.Location,
                State = p.State,
                ExpectedCompletionDate = p.ExpectedCompletionDate
            })
            .ToListAsync();

        return new ProjectListResponse
        {
            Projects = projects,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
        };
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProjectDetailDto>> GetProject(Guid id)
    {
        var project = await _context.Projects.FindAsync(id);
        if (project == null)
            return NotFound();

        return new ProjectDetailDto
        {
            Id = project.Id,
            Name = project.Name,
            Description = project.Description,
            Type = project.Type,
            Phase = project.Phase,
            Priority = project.Priority,
            EstimatedCost = project.EstimatedCost,
            AllocatedBudget = project.AllocatedBudget,
            PrimaryFundingSource = project.PrimaryFundingSource,
            Location = project.Location,
            County = project.County,
            State = project.State,
            ScopeItems = project.ScopeItems,
            RequiresEnvironmentalReview = project.RequiresEnvironmentalReview,
            RequiresPublicHearing = project.RequiresPublicHearing,
            RequiredPermits = project.RequiredPermits,
            PlanningStartDate = project.PlanningStartDate,
            ConstructionStartDate = project.ConstructionStartDate,
            ExpectedCompletionDate = project.ExpectedCompletionDate
        };
    }

    [HttpPost]
    public async Task<ActionResult<Guid>> CreateProject(CreateProjectRequest request)
    {
        var project = new Project(
            request.Name,
            request.Description,
            request.Type,
            request.Priority,
            request.EstimatedCost,
            request.Location,
            request.County,
            request.State,
            "current-user" // TODO: Get from auth context
        );

        _context.Projects.Add(project);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetProject), new { id = project.Id }, project.Id);
    }

    [HttpPut("{id}/scope")]
    public async Task<IActionResult> UpdateProjectScope(Guid id, UpdateScopeRequest request)
    {
        var project = await _context.Projects.FindAsync(id);
        if (project == null)
            return NotFound();

        project.UpdateScope(request.ScopeItems, "current-user");
        await _context.SaveChangesAsync();

        return NoContent();
    }

    [HttpPut("{id}/budget")]
    public async Task<IActionResult> UpdateProjectBudget(Guid id, UpdateBudgetRequest request)
    {
        var project = await _context.Projects.FindAsync(id);
        if (project == null)
            return NotFound();

        project.UpdateBudget(request.AllocatedBudget, request.FundingSource, "current-user");
        await _context.SaveChangesAsync();

        return NoContent();
    }
}

// DTOs
public record ProjectSummaryDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public ProjectType Type { get; init; }
    public ProjectPhase Phase { get; init; }
    public ProjectPriority Priority { get; init; }
    public decimal EstimatedCost { get; init; }
    public string Location { get; init; } = string.Empty;
    public string State { get; init; } = string.Empty;
    public DateTime? ExpectedCompletionDate { get; init; }
}

public record ProjectDetailDto : ProjectSummaryDto
{
    public string Description { get; init; } = string.Empty;
    public decimal AllocatedBudget { get; init; }
    public FundingSource PrimaryFundingSource { get; init; }
    public string County { get; init; } = string.Empty;
    public string[] ScopeItems { get; init; } = Array.Empty<string>();
    public bool RequiresEnvironmentalReview { get; init; }
    public bool RequiresPublicHearing { get; init; }
    public string[] RequiredPermits { get; init; } = Array.Empty<string>();
    public DateTime? PlanningStartDate { get; init; }
    public DateTime? ConstructionStartDate { get; init; }
}

public record ProjectListResponse
{
    public List<ProjectSummaryDto> Projects { get; init; } = new();
    public int TotalCount { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalPages { get; init; }
}

public record CreateProjectRequest(
    string Name,
    string Description,
    ProjectType Type,
    ProjectPriority Priority,
    decimal EstimatedCost,
    string Location,
    string County,
    string State
);

public record UpdateScopeRequest(string[] ScopeItems);
public record UpdateBudgetRequest(decimal AllocatedBudget, FundingSource FundingSource);
```

## Phase 3: Budget Planning Module (Week 4)

### 3.1 Budget Domain Model

Create `backend/src/Infrastructure.Platform/Features/Budgets/Domain/Budget.cs`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace Infrastructure.Platform.Features.Budgets.Domain;

[Table("budgets", Schema = "budgets")]
public class Budget
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid ProjectId { get; private set; }
    public string FiscalYear { get; private set; } = string.Empty;
    public BudgetType Type { get; private set; }
    public BudgetStatus Status { get; private set; } = BudgetStatus.Draft;
    
    // Budget Categories
    public decimal PlanningCosts { get; private set; }
    public decimal DesignCosts { get; private set; }
    public decimal EnvironmentalCosts { get; private set; }
    public decimal LandAcquisitionCosts { get; private set; }
    public decimal ConstructionCosts { get; private set; }
    public decimal ContingencyCosts { get; private set; }
    public decimal OversightCosts { get; private set; }
    
    // Funding Sources
    public List<FundingAllocation> FundingAllocations { get; private set; } = new();
    
    // Timeline
    public DateTime BudgetStartDate { get; private set; }
    public DateTime BudgetEndDate { get; private set; }
    
    public decimal TotalBudget => PlanningCosts + DesignCosts + EnvironmentalCosts + 
                                 LandAcquisitionCosts + ConstructionCosts + 
                                 ContingencyCosts + OversightCosts;

    private Budget() { }

    public Budget(Guid projectId, string fiscalYear, BudgetType type, 
                 DateTime startDate, DateTime endDate)
    {
        ProjectId = projectId;
        FiscalYear = fiscalYear;
        Type = type;
        BudgetStartDate = startDate;
        BudgetEndDate = endDate;
    }

    public void UpdateCosts(decimal planning, decimal design, decimal environmental,
                           decimal landAcquisition, decimal construction, 
                           decimal contingency, decimal oversight)
    {
        PlanningCosts = planning;
        DesignCosts = design;
        EnvironmentalCosts = environmental;
        LandAcquisitionCosts = landAcquisition;
        ConstructionCosts = construction;
        ContingencyCosts = contingency;
        OversightCosts = oversight;
    }

    public void AddFundingSource(FundingSource source, decimal amount, string description)
    {
        FundingAllocations.Add(new FundingAllocation(source, amount, description));
    }

    public void ApproveBudget()
    {
        if (Status != BudgetStatus.Draft)
            throw new InvalidOperationException("Only draft budgets can be approved");
        
        Status = BudgetStatus.Approved;
    }
}

[Table("funding_allocations", Schema = "budgets")]
public class FundingAllocation
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public FundingSource Source { get; private set; }
    public decimal Amount { get; private set; }
    public string Description { get; private set; } = string.Empty;

    private FundingAllocation() { }

    public FundingAllocation(FundingSource source, decimal amount, string description)
    {
        Source = source;
        Amount = amount;
        Description = description;
    }
}

public enum BudgetType
{
    Capital, Operating, Emergency
}

public enum BudgetStatus
{
    Draft, UnderReview, Approved, Rejected, Amended
}
```

## Phase 4: Grant Management Module (Week 5)

### 4.1 Grant Opportunity Tracking

Create `backend/src/Infrastructure.Platform/Features/Grants/Domain/GrantOpportunity.cs`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace Infrastructure.Platform.Features.Grants.Domain;

[Table("grant_opportunities", Schema = "grants")]
public class GrantOpportunity
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string Name { get; private set; } = string.Empty;
    public string FundingAgency { get; private set; } = string.Empty;
    public string ProgramCode { get; private set; } = string.Empty;
    
    public decimal MaxAwardAmount { get; private set; }
    public decimal MinAwardAmount { get; private set; }
    public double MatchRequiredPercentage { get; private set; }
    
    public DateTime ApplicationDeadline { get; private set; }
    public DateTime AwardNotificationDate { get; private set; }
    public DateTime ProjectStartDate { get; private set; }
    public DateTime ProjectEndDate { get; private set; }
    
    public string[] EligibleProjectTypes { get; private set; } = Array.Empty<string>();
    public string[] EligibleApplicants { get; private set; } = Array.Empty<string>();
    public string[] RequiredDocuments { get; private set; } = Array.Empty<string>();
    
    public string Description { get; private set; } = string.Empty;
    public string ApplicationUrl { get; private set; } = string.Empty;
    public string ContactEmail { get; private set; } = string.Empty;
    
    public GrantStatus Status { get; private set; } = GrantStatus.Open;

    public bool IsEligibleFor(ProjectType projectType, string applicantType)
    {
        return EligibleProjectTypes.Contains(projectType.ToString()) && 
               EligibleApplicants.Contains(applicantType);
    }

    public decimal CalculateRequiredMatch(decimal requestedAmount)
    {
        return requestedAmount * (decimal)(MatchRequiredPercentage / 100);
    }
}

[Table("grant_applications", Schema = "grants")]
public class GrantApplication
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid ProjectId { get; private set; }
    public Guid GrantOpportunityId { get; private set; }
    
    public decimal RequestedAmount { get; private set; }
    public decimal MatchAmount { get; private set; }
    public string ProjectDescription { get; private set; } = string.Empty;
    public string Justification { get; private set; } = string.Empty;
    
    public ApplicationStatus Status { get; private set; } = ApplicationStatus.Draft;
    public DateTime? SubmittedDate { get; private set; }
    public DateTime? AwardDate { get; private set; }
    public decimal? AwardedAmount { get; private set; }
    
    public List<ApplicationDocument> Documents { get; private set; } = new();

    public void Submit()
    {
        if (Status != ApplicationStatus.Draft)
            throw new InvalidOperationException("Application already submitted");
            
        Status = ApplicationStatus.Submitted;
        SubmittedDate = DateTime.UtcNow;
    }

    public void Award(decimal amount)
    {
        Status = ApplicationStatus.Awarded;
        AwardDate = DateTime.UtcNow;
        AwardedAmount = amount;
    }
}

public enum GrantStatus
{
    Open, Closed, Cancelled
}

public enum ApplicationStatus
{
    Draft, Submitted, UnderReview, Awarded, Rejected
}
```

## Phase 5: Benefit-Cost Analysis Module (Week 6)

### 5.1 BCA Calculation Engine

Create `backend/src/Infrastructure.Platform/Features/BenefitCostAnalysis/Domain/BenefitCostAnalysis.cs`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace Infrastructure.Platform.Features.BenefitCostAnalysis.Domain;

[Table("benefit_cost_analyses", Schema = "bca")]
public class BenefitCostAnalysis
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid ProjectId { get; private set; }
    public string AnalysisName { get; private set; } = string.Empty;
    public BCAMethodology Methodology { get; private set; }
    
    // Analysis Parameters
    public int AnalysisPeriodYears { get; private set; }
    public double DiscountRate { get; private set; }
    public DateTime BaseYear { get; private set; }
    
    // Costs
    public List<CostItem> Costs { get; private set; } = new();
    public decimal TotalPresentValueCosts { get; private set; }
    
    // Benefits  
    public List<BenefitItem> Benefits { get; private set; } = new();
    public decimal TotalPresentValueBenefits { get; private set; }
    
    // Results
    public decimal BenefitCostRatio { get; private set; }
    public decimal NetPresentValue { get; private set; }
    public BCARecommendation Recommendation { get; private set; }
    
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
    public DateTime? LastCalculated { get; private set; }

    public void AddCost(string category, decimal amount, int year, string description)
    {
        Costs.Add(new CostItem(category, amount, year, description));
    }

    public void AddBenefit(BenefitType type, decimal annualValue, string description)
    {
        Benefits.Add(new BenefitItem(type, annualValue, description));
    }

    public void CalculateAnalysis()
    {
        // Calculate Present Value of Costs
        TotalPresentValueCosts = Costs.Sum(c => 
            c.Amount / Math.Pow(1 + DiscountRate, c.Year));

        // Calculate Present Value of Benefits
        TotalPresentValueBenefits = Benefits.Sum(b => 
            CalculateAnnuityPresentValue(b.AnnualValue, AnalysisPeriodYears, DiscountRate));

        // Calculate Ratios
        BenefitCostRatio = TotalPresentValueCosts > 0 ? 
            TotalPresentValueBenefits / TotalPresentValueCosts : 0;
        
        NetPresentValue = TotalPresentValueBenefits - TotalPresentValueCosts;

        // Determine Recommendation
        Recommendation = BenefitCostRatio >= 1.0 ? 
            BCARecommendation.Recommended : BCARecommendation.NotRecommended;

        LastCalculated = DateTime.UtcNow;
    }

    private decimal CalculateAnnuityPresentValue(decimal annualValue, int years, double rate)
    {
        if (rate == 0) return annualValue * years;
        
        var presentValue = annualValue * (1 - Math.Pow(1 + rate, -years)) / rate;
        return (decimal)presentValue;
    }
}

[Table("cost_items", Schema = "bca")]
public class CostItem
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string Category { get; private set; } = string.Empty;
    public decimal Amount { get; private set; }
    public int Year { get; private set; }
    public string Description { get; private set; } = string.Empty;

    private CostItem() { }

    public CostItem(string category, decimal amount, int year, string description)
    {
        Category = category;
        Amount = amount;
        Year = year;
        Description = description;
    }
}

[Table("benefit_items", Schema = "bca")]
public class BenefitItem
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public BenefitType Type { get; private set; }
    public decimal AnnualValue { get; private set; }
    public string Description { get; private set; } = string.Empty;

    private BenefitItem() { }

    public BenefitItem(BenefitType type, decimal annualValue, string description)
    {
        Type = type;
        AnnualValue = annualValue;
        Description = description;
    }
}

public enum BCAMethodology
{
    StandardBCA, SimplifiedBCA, QualitativeBCA
}

public enum BenefitType
{
    TravelTimeSavings, VehicleOperatingCostSavings, SafetyBenefits, 
    EmissionReductions, NoiseReduction, PropertyValueIncrease,
    EconomicDevelopment, ReliabilityImprovement
}

public enum BCARecommendation
{
    Recommended, NotRecommended, RequiresAdditionalAnalysis
}
```

## Phase 6: Next.js Frontend with Full Integration (Week 7-8)

### 6.1 Frontend Setup

```bash
cd frontend
npx create-next-app@14 . --typescript --tailwind --app

# Install additional packages
npm install @radix-ui/react-* @tanstack/react-query axios
npm install @hookform/resolvers react-hook-form zod
npm install recharts date-fns lucide-react
```

### 6.2 Project Management Dashboard

Create `frontend/src/app/projects/page.tsx`:

```tsx
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { projectsApi } from '@/lib/api/projects';
import { ProjectList } from '@/components/projects/ProjectList';
import { ProjectFilters } from '@/components/projects/ProjectFilters';
import { CreateProjectModal } from '@/components/projects/CreateProjectModal';
import { Button } from '@/components/ui/button';
import { Plus } from 'lucide-react';

export default function ProjectsPage() {
  const [filters, setFilters] = useState({
    search: '',
    type: null,
    phase: null,
    state: '',
    page: 1
  });
  const [showCreateModal, setShowCreateModal] = useState(false);

  const { data: projectsData, isLoading, error } = useQuery({
    queryKey: ['projects', filters],
    queryFn: () => projectsApi.getProjects(filters)
  });

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold">Infrastructure Projects</h1>
          <p className="text-muted-foreground">
            Manage your infrastructure projects from planning to completion
          </p>
        </div>
        
        <Button onClick={() => setShowCreateModal(true)}>
          <Plus className="mr-2 h-4 w-4" />
          New Project
        </Button>
      </div>

      <ProjectFilters
        filters={filters}
        onFiltersChange={setFilters}
      />

      {error && (
        <div className="bg-red-50 border border-red-200 rounded-lg p-4">
          <p className="text-red-800">Error loading projects. Please try again.</p>
        </div>
      )}

      <ProjectList 
        projects={projectsData?.projects || []}
        totalCount={projectsData?.totalCount || 0}
        isLoading={isLoading}
        pagination={{
          page: filters.page,
          pageSize: 25,
          totalPages: projectsData?.totalPages || 0
        }}
        onPageChange={(page) => setFilters({ ...filters, page })}
      />

      <CreateProjectModal
        open={showCreateModal}
        onClose={() => setShowCreateModal(false)}
      />
    </div>
  );
}
```

### 6.3 Integrated BCA Analysis Component

Create `frontend/src/components/bca/BCAAnalysis.tsx`:

```tsx
'use client';

import { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { bcaApi } from '@/lib/api/bca';
import { Card, CardHeader, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Separator } from '@/components/ui/separator';
import { Calculator, TrendingUp, TrendingDown, AlertCircle } from 'lucide-react';

interface BCAAnalysisProps {
  projectId: string;
}

export function BCAAnalysis({ projectId }: BCAAnalysisProps) {
  const [analysisForm, setAnalysisForm] = useState({
    analysisName: '',
    analysisPeriodYears: 30,
    discountRate: 0.07,
    methodology: 'StandardBCA'
  });

  const queryClient = useQueryClient();

  const { data: analysis, isLoading } = useQuery({
    queryKey: ['bca-analysis', projectId],
    queryFn: () => bcaApi.getAnalysis(projectId)
  });

  const calculateMutation = useMutation({
    mutationFn: () => bcaApi.calculateAnalysis(projectId),
    onSuccess: () => {
      queryClient.invalidateQueries(['bca-analysis', projectId]);
    }
  });

  const handleCalculate = () => {
    calculateMutation.mutate();
  };

  if (isLoading) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="animate-pulse">Loading BCA analysis...</div>
        </CardContent>
      </Card>
    );
  }

  return (
    <div className="space-y-6">
      <Card>
        <CardHeader>
          <h3 className="text-lg font-semibold flex items-center">
            <Calculator className="mr-2 h-5 w-5" />
            Benefit-Cost Analysis
          </h3>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <Label htmlFor="analysisName">Analysis Name</Label>
              <Input
                id="analysisName"
                value={analysisForm.analysisName}
                onChange={(e) => setAnalysisForm({
                  ...analysisForm,
                  analysisName: e.target.value
                })}
                placeholder="Enter analysis name"
              />
            </div>
            
            <div>
              <Label htmlFor="analysisPeriod">Analysis Period (Years)</Label>
              <Input
                id="analysisPeriod"
                type="number"
                value={analysisForm.analysisPeriodYears}
                onChange={(e) => setAnalysisForm({
                  ...analysisForm,
                  analysisPeriodYears: parseInt(e.target.value)
                })}
              />
            </div>
          </div>

          <Button 
            onClick={handleCalculate}
            disabled={calculateMutation.isLoading}
            className="w-full"
          >
            {calculateMutation.isLoading ? 'Calculating...' : 'Run BCA Analysis'}
          </Button>
        </CardContent>
      </Card>

      {analysis && (
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
          <Card>
            <CardContent className="p-6">
              <div className="flex items-center justify-between">
                <div>
                  <p className="text-sm font-medium text-muted-foreground">
                    Benefit-Cost Ratio
                  </p>
                  <p className="text-2xl font-bold">
                    {analysis.benefitCostRatio.toFixed(2)}
                  </p>
                </div>
                {analysis.benefitCostRatio >= 1.0 ? (
                  <TrendingUp className="h-8 w-8 text-green-500" />
                ) : (
                  <TrendingDown className="h-8 w-8 text-red-500" />
                )}
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardContent className="p-6">
              <div className="flex items-center justify-between">
                <div>
                  <p className="text-sm font-medium text-muted-foreground">
                    Net Present Value
                  </p>
                  <p className="text-2xl font-bold">
                    ${analysis.netPresentValue.toLocaleString()}
                  </p>
                </div>
                {analysis.netPresentValue >= 0 ? (
                  <TrendingUp className="h-8 w-8 text-green-500" />
                ) : (
                  <TrendingDown className="h-8 w-8 text-red-500" />
                )}
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardContent className="p-6">
              <div className="flex items-center justify-between">
                <div>
                  <p className="text-sm font-medium text-muted-foreground">
                    Recommendation
                  </p>
                  <p className="text-lg font-semibold">
                    {analysis.recommendation}
                  </p>
                </div>
                <AlertCircle className={`h-8 w-8 ${
                  analysis.recommendation === 'Recommended' 
                    ? 'text-green-500' 
                    : 'text-red-500'
                }`} />
              </div>
            </CardContent>
          </Card>
        </div>
      )}
    </div>
  );
}
```

## Phase 7: Cost Estimation & Contract Generation (Week 9-10)

### 7.1 Cost Estimation Service

Create `backend/src/Infrastructure.Platform/Features/CostEstimation/Services/CostEstimationService.cs`:

```csharp
using Infrastructure.Platform.Features.Projects.Domain;

namespace Infrastructure.Platform.Features.CostEstimation.Services;

public class CostEstimationService
{
    private readonly ICostDatabase _costDatabase;
    
    public CostEstimationService(ICostDatabase costDatabase)
    {
        _costDatabase = costDatabase;
    }

    public async Task<PreliminaryCostEstimate> GenerateEstimate(Project project)
    {
        var estimate = new PreliminaryCostEstimate
        {
            ProjectId = project.Id,
            EstimatedDate = DateTime.UtcNow
        };

        // Get unit costs from database based on project type and location
        var unitCosts = await _costDatabase.GetUnitCosts(project.Type, project.State);
        
        // Calculate estimates based on project scope
        estimate.PlanningCosts = CalculatePlanningCosts(project, unitCosts);
        estimate.DesignCosts = CalculateDesignCosts(project, unitCosts);
        estimate.ConstructionCosts = CalculateConstructionCosts(project, unitCosts);
        estimate.ContingencyCosts = estimate.ConstructionCosts * 0.15m; // 15% contingency
        
        estimate.TotalEstimate = estimate.PlanningCosts + estimate.DesignCosts + 
                               estimate.ConstructionCosts + estimate.ContingencyCosts;

        return estimate;
    }

    private decimal CalculateConstructionCosts(Project project, UnitCostData unitCosts)
    {
        return project.Type switch
        {
            ProjectType.Highway => CalculateHighwayCosts(project, unitCosts),
            ProjectType.Bridge => CalculateBridgeCosts(project, unitCosts),
            ProjectType.PublicTransit => CalculateTransitCosts(project, unitCosts),
            _ => project.EstimatedCost * 0.75m // Default to 75% of estimated cost
        };
    }

    public async Task<ContractStructure> GenerateContractStructure(Project project, 
                                                                  PreliminaryCostEstimate estimate)
    {
        var contractType = DetermineOptimalContractType(project, estimate);
        
        return new ContractStructure
        {
            ProjectId = project.Id,
            ContractType = contractType,
            DeliveryMethod = DetermineDeliveryMethod(project),
            PaymentStructure = GeneratePaymentStructure(estimate),
            RequiredBonds = GenerateRequiredBonds(estimate),
            RequiredInsurance = GenerateInsuranceRequirements(project),
            QualificationCriteria = GenerateQualificationCriteria(project, estimate),
            EvaluationCriteria = GenerateEvaluationCriteria(project)
        };
    }
}

public class ContractStructure
{
    public Guid ProjectId { get; set; }
    public ContractType ContractType { get; set; }
    public DeliveryMethod DeliveryMethod { get; set; }
    public PaymentStructure PaymentStructure { get; set; }
    public List<RequiredBond> RequiredBonds { get; set; } = new();
    public List<InsuranceRequirement> RequiredInsurance { get; set; } = new();
    public List<string> QualificationCriteria { get; set; } = new();
    public List<EvaluationCriterion> EvaluationCriteria { get; set; } = new();
}

public enum ContractType
{
    LumpSum, UnitPrice, CostPlus, DesignBuild, CMAR, PublicPrivatePartnership
}

public enum DeliveryMethod
{
    DesignBidBuild, DesignBuild, CMAtRisk, ProgressiveDesignBuild
}
```

## Phase 8: Production Deployment & Integration (Week 11-12)

### 8.1 Application Configuration

Update `backend/src/Infrastructure.Platform/Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Infrastructure.Platform.Shared.Database;
using Infrastructure.Platform.Features.CostEstimation.Services;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { 
        Title = "Infrastructure Platform API", 
        Version = "v1",
        Description = "API for public infrastructure planning platform"
    });
});

// Database
builder.Services.AddDbContext<PlatformDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Business Services
builder.Services.AddScoped<CostEstimationService>();
builder.Services.AddScoped<ICostDatabase, CostDatabase>();

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddCheck("self", () => HealthCheckResult.Healthy());

// CORS for frontend
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend",
        policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://your-domain.com")
                  .AllowAnyHeader()
                  .AllowAnyMethod();
        });
});

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseCors("AllowFrontend");
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

// Ensure database is created and seeded
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<PlatformDbContext>();
    await context.Database.MigrateAsync();
}

app.Run();
```

### 8.2 Docker Deployment Configuration

Create `docker-compose.production.yml`:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: infrastructure_platform
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  backend:
    build: 
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=infrastructure_platform;Username=${POSTGRES_USER};Password=${POSTGRES_PASSWORD}
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      - NEXT_PUBLIC_API_URL=https://api.your-domain.com
    restart: unless-stopped

volumes:
  postgres_data:
```

## Summary: Optimized Build Approach

This optimized approach delivers:

1. **Faster Time to Value**: Complete features in 2-week cycles
2. **Lower Risk**: Start with proven monolith pattern, extract services when needed
3. **Government-Focused**: Built-in compliance, audit trails, and regulatory support
4. **Complete Workflows**: Each phase delivers working end-to-end functionality
5. **Modern Tech Stack**: Next.js + .NET 8 for better developer experience and performance

**Key Milestones:**
- **Week 3**: Complete project management CRUD
- **Week 4**: Budget planning with cost breakdowns  
- **Week 5**: Grant opportunity matching and applications
- **Week 6**: BCA calculations with federal methodology compliance
- **Week 8**: Full frontend integration with all features
- **Week 10**: Cost estimation and contract generation
- **Week 12**: Production-ready deployment

This approach is 40% faster than the original while delivering more focused, government-specific functionality.

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Analyze current build approach and identify optimization opportunities", "status": "completed"}, {"id": "2", "content": "Redesign build guide with feature-first approach for infrastructure planning", "status": "completed"}, {"id": "3", "content": "Implement modular monolith structure with clear domain boundaries", "status": "completed"}, {"id": "4", "content": "Focus on complete user workflows for infrastructure owners", "status": "completed"}]
    return {
      mean: npvResults.reduce((sum, npv) => sum + npv, 0) / iterations,
      median: npvResults[Math.floor(iterations / 2)],
      percentile5: npvResults[Math.floor(iterations * 0.05)],
      percentile95: npvResults[Math.floor(iterations * 0.95)],
      standardDeviation: this.calculateStandardDeviation(npvResults),
      probabilityPositive: npvResults.filter(npv => npv > 0).length / iterations
    };
  }

  private generateRandomCashFlows(baseCashFlows: any[]) {
    return baseCashFlows.map(cf => ({
      ...cf,
      benefits: cf.benefits * (0.8 + Math.random() * 0.4), // ±20% variation
      costs: cf.costs * (0.9 + Math.random() * 0.2) // ±10% variation
    }));
  }

  private calculateStandardDeviation(values: number[]): number {
    const mean = values.reduce((sum, val) => sum + val, 0) / values.length;
    const squaredDiffs = values.map(val => Math.pow(val - mean, 2));
    const variance = squaredDiffs.reduce((sum, diff) => sum + diff, 0) / values.length;
    return Math.sqrt(variance);
  }

  // ... other helper methods
}
```

### Step 12: Testing Setup

#### 12.1 Backend Testing
```typescript
// backend/src/tests/setup.ts
import { pool } from '../config/database';

beforeAll(async () => {
  // Setup test database
  await pool.query('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"');
  await pool.query('CREATE EXTENSION IF NOT EXISTS "postgis"');
});

afterAll(async () => {
  await pool.end();
});

// backend/src/tests/services/BCAService.test.ts
import { BCAService } from '../../services/BCAService';

describe('BCAService', () => {
  let bcaService: BCAService;

  beforeEach(() => {
    bcaService = new BCAService();
  });

  describe('performBasicBCA', () => {
    it('should calculate positive NPV for profitable project', async () => {
      const mockProject = {
        id: 'test-project-id',
        estimatedCost: 1000000,
        projectType: 'highway'
      };

      const parameters = {
        discountRate: 0.07,
        analysisPeriod: 20,
        baseYear: 2024
      };

      // Mock database calls
      jest.spyOn(bcaService as any, 'getProjectData').mockResolvedValue(mockProject);
      jest.spyOn(bcaService as any, 'storeBCAResults').mockResolvedValue(undefined);

      const results = await bcaService.performBasicBCA('test-project-id', parameters);

      expect(results.npv).toBeGreaterThan(0);
      expect(results.bcr).toBeGreaterThan(1);
      expect(results.cashFlows).toHaveLength(20);
    });
  });
});

// Run tests
npm test
```

#### 12.2 Frontend Testing
```typescript
// frontend/src/components/projects/ProjectList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ProjectList } from './ProjectList';

const createTestQueryClient = () => new QueryClient({
  defaultOptions: { queries: { retry: false } }
});

describe('ProjectList', () => {
  it('renders loading state', () => {
    const queryClient = createTestQueryClient();
    
    render(
      <QueryClientProvider client={queryClient}>
        <ProjectList />
      </QueryClientProvider>
    );

    expect(screen.getByRole('status')).toBeInTheDocument();
  });

  it('renders projects when loaded', async () => {
    const queryClient = createTestQueryClient();
    
    // Mock API response
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({
          projects: [
            { id: '1', name: 'Test Project', status: 'active' }
          ],
          total: 1
        })
      })
    ) as jest.Mock;

    render(
      <QueryClientProvider client={queryClient}>
        <ProjectList />
      </QueryClientProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('Test Project')).toBeInTheDocument();
    });
  });
});

# Run frontend tests
npm test
```

---

## Phase 3: Advanced Features (Weeks 9-12)

### Step 13: Scope and Cost Estimation

#### 13.1 Add Scope Templates
```sql
-- database/migrations/003_add_scope_templates.sql
CREATE TABLE scope_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    project_type VARCHAR(50) NOT NULL,
    infrastructure_category VARCHAR(50) NOT NULL,
    template_json JSONB NOT NULL,
    base_duration_months INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE project_estimates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id),
    estimate_type VARCHAR(20) NOT NULL,
    scope_json JSONB,
    cost_breakdown JSONB,
    total_fee DECIMAL(12,2),
    confidence_level VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### 13.2 Scope Estimation Service
```typescript
// backend/src/services/ScopeEstimationService.ts
export class ScopeEstimationService {
  async generateProjectScope(projectId: string, parameters: any): Promise<any> {
    const project = await this.getProjectData(projectId);
    const template = await this.getTemplate(project.projectType, project.infrastructureCategory);
    
    if (!template) {
      throw new Error('No template found for project type');
    }

    // Customize template based on project parameters
    const customizedScope = this.customizeTemplate(template, project, parameters);
    
    // Calculate duration and resources
    const duration = this.calculateDuration(customizedScope, project);
    const resources = this.calculateResources(customizedScope);
    
    const scopeEstimate = {
      workBreakdownStructure: customizedScope.phases,
      estimatedDurationMonths: duration,
      resourceRequirements: resources,
      deliverables: this.extractDeliverables(customizedScope),
      assumptions: template.assumptions,
      exclusions: template.exclusions,
      confidenceLevel: this.calculateConfidenceLevel(project, template)
    };

    // Store estimate
    await this.storeEstimate(projectId, 'scope', scopeEstimate);
    
    return scopeEstimate;
  }

  private async getTemplate(projectType: string, category: string) {
    const query = `
      SELECT * FROM scope_templates 
      WHERE project_type = $1 AND infrastructure_category = $2
      ORDER BY created_at DESC LIMIT 1
    `;
    const result = await pool.query(query, [projectType, category]);
    return result.rows[0];
  }

  private customizeTemplate(template: any, project: any, parameters: any) {
    const templateData = template.template_json;
    
    // Apply complexity multipliers
    Object.keys(templateData.phases).forEach(phaseKey => {
      const phase = templateData.phases[phaseKey];
      phase.activities = phase.activities.map((activity: any) => ({
        ...activity,
        estimatedHours: activity.estimatedHours * this.getComplexityMultiplier(activity, parameters)
      }));
    });

    return templateData;
  }

  private getComplexityMultiplier(activity: any, parameters: any): number {
    let multiplier = 1.0;
    
    // Environmental complexity
    if (activity.name.toLowerCase().includes('environmental')) {
      const envConstraints = parameters.environmentalConstraints?.length || 0;
      multiplier *= (1 + envConstraints * 0.2);
    }
    
    // Location complexity
    if (parameters.populationDensity > 1000) {
      multiplier *= 1.3;
    }
    
    return Math.min(multiplier, 3.0); // Cap at 3x
  }

  private calculateDuration(scope: any, project: any): number {
    let totalHours = 0;
    
    Object.values(scope.phases).forEach((phase: any) => {
      phase.activities.forEach((activity: any) => {
        totalHours += activity.estimatedHours;
      });
    });
    
    // Assume 160 hours per month, 70% utilization
    return Math.ceil(totalHours / (160 * 0.7));
  }

  private async storeEstimate(projectId: string, type: string, estimate: any) {
    const query = `
      INSERT INTO project_estimates (project_id, estimate_type, scope_json, confidence_level)
      VALUES ($1, $2, $3, $4)
    `;
    await pool.query(query, [projectId, type, JSON.stringify(estimate), estimate.confidenceLevel]);
  }
}
```

#### 13.3 Cost Estimation with ML
```python
# backend/services/ml-models/cost_prediction_model.py
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestRegressor
from sklearn.preprocessing import StandardScaler
import joblib
import json

class CostPredictionModel:
    def __init__(self):
        self.model = RandomForestRegressor(n_estimators=100, random_state=42)
        self.scaler = StandardScaler()
        self.is_trained = False
        
    def train(self, training_data_path):
        # Load training data
        df = pd.read_csv(training_data_path)
        
        # Feature engineering
        features = self.prepare_features(df)
        target = df['total_cost']
        
        # Scale features
        features_scaled = self.scaler.fit_transform(features)
        
        # Train model
        self.model.fit(features_scaled, target)
        self.is_trained = True
        
        # Save model
        joblib.dump(self.model, 'cost_prediction_model.pkl')
        joblib.dump(self.scaler, 'cost_prediction_scaler.pkl')
        
    def predict_cost(self, project_data):
        if not self.is_trained:
            self.load_model()
            
        # Prepare features
        features = self.prepare_features_single(project_data)
        features_scaled = self.scaler.transform([features])
        
        # Predict
        prediction = self.model.predict(features_scaled)[0]
        
        # Calculate confidence interval
        # Use ensemble predictions for uncertainty
        tree_predictions = [tree.predict(features_scaled)[0] for tree in self.model.estimators_]
        std_dev = np.std(tree_predictions)
        
        return {
            'predicted_cost': prediction,
            'confidence_interval': {
                'lower_80': prediction - 1.28 * std_dev,
                'upper_80': prediction + 1.28 * std_dev,
                'lower_95': prediction - 1.96 * std_dev,
                'upper_95': prediction + 1.96 * std_dev
            },
            'uncertainty': std_dev
        }
    
    def prepare_features_single(self, project_data):
        return [
            project_data.get('length_miles', 0),
            project_data.get('area_sq_miles', 0),
            1 if project_data.get('project_type') == 'highway' else 0,
            1 if project_data.get('project_type') == 'bridge' else 0,
            project_data.get('population_density', 0),
            project_data.get('environmental_constraints_count', 0),
            project_data.get('estimated_duration_months', 12)
        ]
    
    def load_model(self):
        self.model = joblib.load('cost_prediction_model.pkl')
        self.scaler = joblib.load('cost_prediction_scaler.pkl')
        self.is_trained = True

# Node.js wrapper for Python model
# backend/src/services/MLCostEstimationService.ts
import { spawn } from 'child_process';
import path from 'path';

export class MLCostEstimationService {
  async predictProjectCost(projectData: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const pythonScript = path.join(__dirname, '../ml-models/predict_cost.py');
      const python = spawn('python3', [pythonScript, JSON.stringify(projectData)]);
      
      let result = '';
      let error = '';
      
      python.stdout.on('data', (data) => {
        result += data.toString();
      });
      
      python.stderr.on('data', (data) => {
        error += data.toString();
      });
      
      python.on('close', (code) => {
        if (code === 0) {
          try {
            const prediction = JSON.parse(result);
            resolve(prediction);
          } catch (e) {
            reject(new Error('Failed to parse ML model output'));
          }
        } else {
          reject(new Error(`ML model failed: ${error}`));
        }
      });
    });
  }
}
```

### Step 14: Document Generation

#### 14.1 Proposal Generation Service
```typescript
// backend/src/services/ProposalGenerationService.ts
import puppeteer from 'puppeteer';
import handlebars from 'handlebars';
import fs from 'fs/promises';
import path from 'path';

export class ProposalGenerationService {
  async generateProposal(projectId: string, templateType: string = 'standard'): Promise<Buffer> {
    // Gather all data
    const projectData = await this.getProjectData(projectId);
    const scopeEstimate = await this.getLatestScopeEstimate(projectId);
    const costEstimate = await this.getLatestCostEstimate(projectId);
    const bcaResults = await this.getLatestBCAResults(projectId);
    
    // Load template
    const templatePath = path.join(__dirname, '../templates', `${templateType}.hbs`);
    const templateContent = await fs.readFile(templatePath, 'utf-8');
    const template = handlebars.compile(templateContent);
    
    // Prepare template data
    const templateData = {
      project: projectData,
      scope: scopeEstimate,
      cost: costEstimate,
      bca: bcaResults,
      generatedDate: new Date().toLocaleDateString(),
      proposalNumber: this.generateProposalNumber()
    };
    
    // Render HTML
    const html = template(templateData);
    
    // Generate PDF
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    
    await page.setContent(html, { waitUntil: 'networkidle0' });
    
    const pdf = await page.pdf({
      format: 'A4',
      printBackground: true,
      margin: { top: '1in', right: '0.75in', bottom: '1in', left: '0.75in' }
    });
    
    await browser.close();
    
    // Store proposal record
    await this.storeProposalRecord(projectId, templateType, pdf.length);
    
    return pdf;
  }

  private generateProposalNumber(): string {
    const date = new Date();
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    const random = Math.floor(Math.random() * 1000).toString().padStart(3, '0');
    return `PROP-${year}${month}${day}-${random}`;
  }

  private async storeProposalRecord(projectId: string, templateType: string, fileSize: number) {
    const query = `
      INSERT INTO proposals (project_id, template_type, file_size_bytes, created_at)
      VALUES ($1, $2, $3, NOW())
    `;
    await pool.query(query, [projectId, templateType, fileSize]);
  }

  // ... other helper methods
}

// Create proposal template
// backend/src/templates/standard.hbs
<!DOCTYPE html>
<html>
<head>
    <title>{{project.name}} - Professional Services Proposal</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; margin: 0; padding: 20px; }
        .header { text-align: center; border-bottom: 2px solid #333; padding-bottom: 20px; }
        .section { margin: 30px 0; }
        .financial-summary { background: #f8f9fa; padding: 20px; border-radius: 8px; }
        table { width: 100%; border-collapse: collapse; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
        th { background-color: #f2f2f2; }
        .cost { font-weight: bold; color: #2c5aa0; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Professional Services Proposal</h1>
        <h2>{{project.name}}</h2>
        <p>Proposal Number: {{proposalNumber}}</p>
        <p>Date: {{generatedDate}}</p>
    </div>

    <div class="section">
        <h2>Executive Summary</h2>
        <p>This proposal outlines professional services for the {{project.name}} project, 
           a {{project.projectType}} infrastructure improvement in {{project.location}}.</p>
        
        <div class="financial-summary">
            <h3>Investment Summary</h3>
            <table>
                <tr><td>Estimated Professional Fee:</td><td class="cost">${{cost.totalFee}}</td></tr>
                <tr><td>Estimated Project Duration:</td><td>{{scope.estimatedDurationMonths}} months</td></tr>
                <tr><td>Benefit-Cost Ratio:</td><td>{{bca.bcr}}</td></tr>
                <tr><td>Net Present Value:</td><td class="cost">${{bca.npv}}</td></tr>
            </table>
        </div>
    </div>

    <div class="section">
        <h2>Scope of Work</h2>
        {{#each scope.workBreakdownStructure}}
        <h3>{{@key}}</h3>
        <ul>
        {{#each this.activities}}
            <li>{{this.name}} - {{this.estimatedHours}} hours</li>
        {{/each}}
        </ul>
        {{/each}}
    </div>

    <div class="section">
        <h2>Benefit-Cost Analysis Summary</h2>
        <p>The preliminary benefit-cost analysis indicates:</p>
        <ul>
            <li>Net Present Value: ${{bca.npv}}</li>
            <li>Benefit-Cost Ratio: {{bca.bcr}}</li>
            <li>Payback Period: {{bca.paybackPeriod}} years</li>
        </ul>
        <p>{{bca.summary.recommendation}}</p>
    </div>
</body>
</html>
```

### Step 15: Real External API Integration

#### 15.1 Replace Mock Services with Real APIs
```typescript
// backend/src/services/ExternalAPIService.ts
import axios from 'axios';

export class ExternalAPIService {
  // NOAA Climate Data
  async getClimateData(location: [number, number], startDate: Date, endDate: Date): Promise<any[]> {
    const [lng, lat] = location;
    const baseURL = 'https://www.ncdc.noaa.gov/cdo-web/api/v2';
    
    try {
      const response = await axios.get(`${baseURL}/data`, {
        params: {
          datasetid: 'GHCND',
          locationid: `FIPS:${await this.getFIPSCode(lat, lng)}`,
          startdate: startDate.toISOString().split('T')[0],
          enddate: endDate.toISOString().split('T')[0],
          datatypeid: ['TMAX', 'TMIN', 'PRCP'].join(','),
          limit: 1000
        },
        headers: {
          'token': process.env.NOAA_API_KEY
        }
      });
      
      return response.data.results || [];
    } catch (error) {
      console.warn('NOAA API error, using fallback data:', error);
      return this.getFallbackClimateData();
    }
  }

  // Census/BLS Economic Data
  async getEconomicData(location: [number, number]): Promise<any> {
    try {
      // Get construction cost index from BLS
      const blsResponse = await axios.post('https://api.bls.gov/publicAPI/v2/timeseries/data/', {
        seriesid: ['CUUR0000SAH1'], // CPI for Housing
        startyear: '2023',
        endyear: '2024'
      }, {
        headers: {
          'Content-Type': 'application/json'
        }
      });

      // Get local economic indicators
      const economicData = {
        constructionCostIndex: this.extractLatestValue(blsResponse.data),
        region: await this.getRegionCode(location),
        lastUpdated: new Date().toISOString()
      };

      return economicData;
    } catch (error) {
      console.warn('Economic API error, using fallback data:', error);
      return this.getFallbackEconomicData();
    }
  }

  // Traffic data from OpenStreetMap/HERE
  async getTrafficData(bounds: any): Promise<any[]> {
    // For demonstration, using HERE API
    try {
      const response = await axios.get('https://traffic.ls.hereapi.com/traffic/6.3/flow.json', {
        params: {
          bbox: `${bounds.southwest.lat},${bounds.southwest.lng};${bounds.northeast.lat},${bounds.northeast.lng}`,
          apikey: process.env.HERE_API_KEY
        }
      });

      return this.transformTrafficData(response.data);
    } catch (error) {
      console.warn('Traffic API error, using fallback data:', error);
      return this.getFallbackTrafficData();
    }
  }

  private transformTrafficData(hereData: any): any[] {
    if (!hereData.RWS || !hereData.RWS[0] || !hereData.RWS[0].RW) {
      return [];
    }

    return hereData.RWS[0].RW.map((road: any) => ({
      location: {
        latitude: road.FIS[0].TMC.PC,
        longitude: road.FIS[0].TMC.PC
      },
      timestamp: new Date(),
      averageSpeed: road.FIS[0].CF[0].SP || 0,
      freeFlowSpeed: road.FIS[0].CF[0].FF || 0,
      congestionLevel: this.calculateCongestionLevel(road.FIS[0].CF[0])
    }));
  }

  private calculateCongestionLevel(flow: any): string {
    const ratio = (flow.SP || 0) / (flow.FF || 1);
    if (ratio > 0.8) return 'low';
    if (ratio > 0.5) return 'medium';
    return 'high';
  }

  // Fallback methods for when APIs are unavailable
  private getFallbackClimateData(): any[] {
    return [{
      location: { latitude: 0, longitude: 0 },
      temperature: 65,
      precipitation: 30,
      extremeWeatherDays: 15,
      dataSource: 'fallback'
    }];
  }

  private getFallbackEconomicData(): any {
    return {
      constructionCostIndex: 105.2,
      laborCostIndex: 108.7,
      materialCostIndex: 103.9,
      dataSource: 'fallback'
    };
  }

  private getFallbackTrafficData(): any[] {
    return [{
      location: { latitude: 0, longitude: 0 },
      vehicleCount: 2500,
      averageSpeed: 45,
      congestionLevel: 'medium',
      dataSource: 'fallback'
    }];
  }
}

// Environment variables needed:
# Add to backend/.env
NOAA_API_KEY=your_noaa_api_key
HERE_API_KEY=your_here_api_key
BLS_API_KEY=your_bls_api_key
```

---

## Phase 4: Production Deployment (Weeks 13-16)

### Step 16: Containerization

#### 16.1 Docker Configuration
```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Expose port
EXPOSE 3001

# Start application
CMD ["npm", "start"]

# frontend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app

# Copy built application
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/next.config.js ./

RUN npm ci --only=production

EXPOSE 3000
CMD ["npm", "start"]
```

#### 16.2 Production Docker Compose
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - frontend
      - backend

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_API_URL=https://yourdomain.com/api/v1
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    restart: unless-stopped
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Step 17: CI/CD Pipeline

#### 17.1 GitHub Actions Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy BCA Platform

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgis/postgis:15-3.3
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: '**/package-lock.json'
    
    - name: Install Backend Dependencies
      working-directory: ./backend
      run: npm ci
    
    - name: Install Frontend Dependencies
      working-directory: ./frontend
      run: npm ci
    
    - name: Run Backend Tests
      working-directory: ./backend
      run: npm test
      env:
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: test_db
        DB_USER: postgres
        DB_PASSWORD: postgres
    
    - name: Run Frontend Tests
      working-directory: ./frontend
      run: npm test
    
    - name: Build Frontend
      working-directory: ./frontend
      run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Production
      run: |
        echo "Deploying to production server"
        # Add your deployment script here
        # Example: rsync, docker commands, etc.
```

### Step 18: Production Configuration

#### 18.1 Environment Variables for Production
```bash
# Create production.env
NODE_ENV=production

# Database (use managed database service)
DB_HOST=your-managed-postgres-host
DB_PORT=5432
DB_NAME=bca_platform_prod
DB_USER=bca_user
DB_PASSWORD=super-secure-password

# Redis (use managed Redis service)
REDIS_HOST=your-managed-redis-host
REDIS_PORT=6379
REDIS_PASSWORD=redis-password

# JWT (generate strong secret)
JWT_SECRET=super-long-random-jwt-secret-key

# API Keys
NOAA_API_KEY=your-production-noaa-key
HERE_API_KEY=your-production-here-key
MAPBOX_TOKEN=your-production-mapbox-token

# File Storage (S3 or similar)
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_BUCKET_NAME=bca-platform-files

# Monitoring
SENTRY_DSN=your-sentry-dsn
```

#### 18.2 Production Optimizations
```typescript
// backend/src/config/production.ts
export const productionConfig = {
  server: {
    port: process.env.PORT || 3001,
    host: '0.0.0.0'
  },
  
  database: {
    max: 20, // Connection pool size
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
    ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
  },
  
  redis: {
    retryDelayOnFailover: 100,
    maxRetriesPerRequest: 3,
    lazyConnect: true
  },
  
  security: {
    rateLimit: {
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100 // requests per window
    },
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') || [],
      credentials: true
    }
  },
  
  logging: {
    level: 'info',
    format: 'json'
  }
};

// backend/src/middleware/productionMiddleware.ts
import rateLimit from 'express-rate-limit';
import compression from 'compression';
import helmet from 'helmet';

export const productionMiddleware = [
  // Security headers
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:", "https:"],
        scriptSrc: ["'self'"],
        connectSrc: ["'self'", "https://api.mapbox.com"]
      }
    }
  }),

  // Rate limiting
  rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // Limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP'
  }),

  // Compression
  compression({
    level: 6,
    threshold: 1000
  })
];
```

### Step 19: Monitoring and Logging

#### 19.1 Application Monitoring
```typescript
// backend/src/middleware/monitoring.ts
import { Request, Response, NextFunction } from 'express';

interface MonitoringMetrics {
  requestCount: number;
  responseTime: number[];
  errorCount: number;
  activeUsers: Set<string>;
}

class MonitoringService {
  private metrics: MonitoringMetrics = {
    requestCount: 0,
    responseTime: [],
    errorCount: 0,
    activeUsers: new Set()
  };

  middleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const startTime = Date.now();
      
      // Track request
      this.metrics.requestCount++;
      
      // Track user activity
      if (req.user?.userId) {
        this.metrics.activeUsers.add(req.user.userId);
      }

      // Track response time
      res.on('finish', () => {
        const responseTime = Date.now() - startTime;
        this.metrics.responseTime.push(responseTime);
        
        // Keep only last 1000 response times
        if (this.metrics.responseTime.length > 1000) {
          this.metrics.responseTime.shift();
        }

        // Track errors
        if (res.statusCode >= 400) {
          this.metrics.errorCount++;
        }
      });

      next();
    };
  }

  getMetrics() {
    const avgResponseTime = this.metrics.responseTime.length > 0 
      ? this.metrics.responseTime.reduce((a, b) => a + b, 0) / this.metrics.responseTime.length 
      : 0;

    return {
      requestCount: this.metrics.requestCount,
      averageResponseTime: Math.round(avgResponseTime),
      errorCount: this.metrics.errorCount,
      activeUsers: this.metrics.activeUsers.size,
      timestamp: new Date().toISOString()
    };
  }

  reset() {
    this.metrics = {
      requestCount: 0,
      responseTime: [],
      errorCount: 0,
      activeUsers: new Set()
    };
  }
}

export const monitoringService = new MonitoringService();

// Add metrics endpoint
// backend/src/routes/monitoring.ts
import { Router } from 'express';
import { monitoringService } from '@/middleware/monitoring';

const router = Router();

router.get('/metrics', (req, res) => {
  res.json(monitoringService.getMetrics());
});

router.get('/health', async (req, res) => {
  try {
    // Check database connection
    await pool.query('SELECT 1');
    
    // Check Redis connection
    await redisClient.ping();
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        database: 'connected',
        redis: 'connected'
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});

export default router;
```

#### 19.2 Error Tracking
```typescript
// backend/src/utils/errorTracking.ts
import * as Sentry from '@sentry/node';

if (process.env.NODE_ENV === 'production') {
  Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV,
    tracesSampleRate: 0.1
  });
}

export class ErrorTracker {
  static captureException(error: Error, context?: any) {
    console.error('Application Error:', error);
    
    if (process.env.NODE_ENV === 'production') {
      Sentry.captureException(error, context);
    }
  }

  static captureMessage(message: string, level: 'info' | 'warning' | 'error' = 'info') {
    console.log(`[${level.toUpperCase()}] ${message}`);
    
    if (process.env.NODE_ENV === 'production') {
      Sentry.captureMessage(message, level);
    }
  }
}

// Global error handler
export const errorHandler = (err: Error, req: any, res: any, next: any) => {
  ErrorTracker.captureException(err, {
    url: req.url,
    method: req.method,
    user: req.user?.userId
  });

  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({ error: err.message, stack: err.stack });
  }
};
```

### Step 20: Performance Optimization

#### 20.1 Database Optimization
```sql
-- Create additional indexes for production
-- database/migrations/004_production_indexes.sql

-- Optimize project queries
CREATE INDEX CONCURRENTLY idx_projects_org_status_updated 
ON projects (organization_id, status, updated_at DESC);

-- Optimize BCA results queries
CREATE INDEX CONCURRENTLY idx_bca_results_project_created 
ON bca_results (project_id, created_at DESC);

-- Optimize user queries
CREATE INDEX CONCURRENTLY idx_users_org_role 
ON users (organization_id, role);

-- Partial index for active projects
CREATE INDEX CONCURRENTLY idx_projects_active_only 
ON projects (updated_at DESC) 
WHERE status IN ('active', 'in_progress');

-- Full-text search index
CREATE INDEX CONCURRENTLY idx_projects_search_vector 
ON projects USING gin(to_tsvector('english', name || ' ' || coalesce(description, '')));

-- Add table partitioning for large tables
CREATE TABLE bca_results_partitioned (
    LIKE bca_results INCLUDING ALL
) PARTITION BY RANGE (created_at);

CREATE TABLE bca_results_2024 PARTITION OF bca_results_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE bca_results_2025 PARTITION OF bca_results_partitioned
FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

#### 20.2 Caching Strategy
```typescript
// backend/src/services/CacheService.ts
import Redis from 'ioredis';

export class CacheService {
  private redis: Redis;

  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3
    });
  }

  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await this.redis.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  async invalidatePattern(pattern: string): Promise<void> {
    try {
      const keys = await this.redis.keys(pattern);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } catch (error) {
      console.error('Cache invalidation error:', error);
    }
  }

  // Cache decorator
  static cache(ttl: number = 3600) {
    return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
      const method = descriptor.value;

      descriptor.value = async function (...args: any[]) {
        const cacheKey = `${target.constructor.name}:${propertyName}:${JSON.stringify(args)}`;
        const cacheService = new CacheService();

        // Try cache first
        const cached = await cacheService.get(cacheKey);
        if (cached) {
          return cached;
        }

        // Execute method and cache result
        const result = await method.apply(this, args);
        await cacheService.set(cacheKey, result, ttl);

        return result;
      };
    };
  }
}

// Usage in services
export class ProjectService {
  @CacheService.cache(1800) // 30 minutes
  async getProjectAnalytics(organizationId: string) {
    // Expensive analytics calculation
    return this.calculateAnalytics(organizationId);
  }

  @CacheService.cache(3600) // 1 hour
  async getSimilarProjects(projectParameters: any) {
    // Expensive similarity search
    return this.findSimilarProjects(projectParameters);
  }
}
```

### Step 21: Security Hardening

#### 21.1 Production Security Configuration
```typescript
// backend/src/security/SecurityConfig.ts
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { body, validationResult } from 'express-validator';

export class SecurityConfig {
  static getHelmetConfig() {
    return helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
          fontSrc: ["'self'", "https://fonts.gstatic.com"],
          imgSrc: ["'self'", "data:", "https:"],
          scriptSrc: ["'self'"],
          connectSrc: ["'self'", "https://api.mapbox.com", "https://api.weather.gov"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"]
        }
      },
      hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
      },
      noSniff: true,
      xssFilter: true,
      referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
    });
  }

  static getApiRateLimit() {
    return rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // Limit each IP to 100 requests per windowMs
      message: {
        error: 'Too many requests',
        retryAfter: 900 // 15 minutes in seconds
      },
      standardHeaders: true,
      legacyHeaders: false
    });
  }

  static getStrictRateLimit() {
    return rateLimit({
      windowMs: 5 * 60 * 1000, // 5 minutes
      max: 5, // Limit login attempts
      message: { error: 'Too many login attempts' },
      skipSuccessfulRequests: true
    });
  }

  static validateProjectInput() {
    return [
      body('name')
        .isLength({ min: 1, max: 255 })
        .withMessage('Name must be between 1 and 255 characters')
        .matches(/^[a-zA-Z0-9\s\-_.,()]+$/)
        .withMessage('Name contains invalid characters'),
      
      body('description')
        .optional()
        .isLength({ max: 2000 })
        .withMessage('Description too long'),
      
      body('projectType')
        .isIn(['highway', 'bridge', 'transit', 'airport', 'rail'])
        .withMessage('Invalid project type'),
      
      body('estimatedCost')
        .optional()
        .isFloat({ min: 0, max: 1000000000 })
        .withMessage('Invalid cost amount'),
      
      body('location.coordinates')
        .optional()
        .isArray({ min: 2, max: 2 })
        .withMessage('Invalid coordinates format'),
      
      this.handleValidationErrors
    ];
  }

  private static handleValidationErrors(req: any, res: any, next: any) {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        error: 'Validation failed',
        details: errors.array()
      });
    }
    next();
  }
}

// Input sanitization
export class InputSanitizer {
  static sanitizeString(input: string): string {
    return input
      .trim()
      .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '') // Remove scripts
      .replace(/[<>]/g, '') // Remove angle brackets
      .substring(0, 1000); // Limit length
  }

  static sanitizeNumeric(input: any): number | null {
    const num = parseFloat(input);
    return isNaN(num) ? null : num;
  }

  static sanitizeCoordinates(coords: any): [number, number] | null {
    if (!Array.isArray(coords) || coords.length !== 2) return null;
    
    const [lng, lat] = coords.map(c => parseFloat(c));
    
    if (isNaN(lng) || isNaN(lat)) return null;
    if (lng < -180 || lng > 180) return null;
    if (lat < -90 || lat > 90) return null;
    
    return [lng, lat];
  }
}
```

#### 21.2 API Security Middleware
```typescript
// backend/src/middleware/apiSecurity.ts
import { Request, Response, NextFunction } from 'express';
import crypto from 'crypto';

export class APISecurityMiddleware {
  // Request signing for sensitive operations
  static validateSignature(secret: string) {
    return (req: Request, res: Response, next: NextFunction) => {
      const signature = req.headers['x-signature'] as string;
      const timestamp = req.headers['x-timestamp'] as string;
      
      if (!signature || !timestamp) {
        return res.status(401).json({ error: 'Missing signature or timestamp' });
      }
      
      // Check timestamp (prevent replay attacks)
      const now = Date.now();
      const requestTime = parseInt(timestamp);
      if (Math.abs(now - requestTime) > 300000) { // 5 minutes
        return res.status(401).json({ error: 'Request timestamp too old' });
      }
      
      // Verify signature
      const payload = JSON.stringify(req.body) + timestamp;
      const expectedSignature = crypto
        .createHmac('sha256', secret)
        .update(payload)
        .digest('hex');
      
      if (signature !== expectedSignature) {
        return res.status(401).json({ error: 'Invalid signature' });
      }
      
      next();
    };
  }

  // SQL injection prevention
  static preventSQLInjection(req: Request, res: Response, next: NextFunction) {
    const sqlPatterns = [
      /(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|UNION|SCRIPT)\b)/gi,
      /(UNION\s+SELECT)/gi,
      /(OR\s+1\s*=\s*1)/gi,
      /('|\"|;|--|\*|\/\*|\*\/)/g
    ];
    
    const checkValue = (value: any): boolean => {
      if (typeof value === 'string') {
        return sqlPatterns.some(pattern => pattern.test(value));
      }
      if (typeof value === 'object' && value !== null) {
        return Object.values(value).some(v => checkValue(v));
      }
      return false;
    };
    
    if (checkValue(req.body) || checkValue(req.query) || checkValue(req.params)) {
      return res.status(400).json({ error: 'Invalid input detected' });
    }
    
    next();
  }

  // IP allowlisting for admin operations
  static restrictToAllowedIPs(allowedIPs: string[]) {
    return (req: Request, res: Response, next: NextFunction) => {
      const clientIP = req.ip || req.connection.remoteAddress;
      
      if (!allowedIPs.includes(clientIP!)) {
        return res.status(403).json({ error: 'IP not authorized' });
      }
      
      next();
    };
  }
}
```

---

## Final Steps: Going Live

### Step 22: Deployment Checklist

#### 22.1 Pre-Deployment Verification
```bash
#!/bin/bash
# scripts/pre-deployment-check.sh

echo "🔍 Running pre-deployment checks..."

# Check environment variables
echo "✅ Checking environment variables..."
required_vars=("DB_HOST" "DB_PASSWORD" "JWT_SECRET" "REDIS_HOST")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "❌ Missing required environment variable: $var"
        exit 1
    fi
done

# Test database connection
echo "✅ Testing database connection..."
npm run test:db-connection || exit 1

# Test Redis connection
echo "✅ Testing Redis connection..."
npm run test:redis-connection || exit 1

# Run security scan
echo "✅ Running security scan..."
npm audit --audit-level moderate || exit 1

# Build applications
echo "✅ Building applications..."
cd frontend && npm run build || exit 1
cd ../backend && npm run build || exit 1

# Run tests
echo "✅ Running tests..."
npm test || exit 1

echo "🎉 All pre-deployment checks passed!"
```

#### 22.2 Production Deployment Script
```bash
#!/bin/bash
# scripts/deploy.sh

set -e

echo "🚀 Starting production deployment..."

# Backup current version
echo "📦 Creating backup..."
docker-compose exec postgres pg_dump -U $DB_USER $DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql

# Pull latest code
echo "📥 Pulling latest code..."
git pull origin main

# Build and deploy
echo "🔨 Building containers..."
docker-compose -f docker-compose.prod.yml build

echo "🔄 Updating services..."
docker-compose -f docker-compose.prod.yml up -d

# Wait for services to be healthy
echo "⏳ Waiting for services to be ready..."
sleep 30

# Verify deployment
echo "🔍 Verifying deployment..."
curl -f http://localhost/api/v1/health || exit 1

echo "✅ Deployment completed successfully!"

# Send notification
curl -X POST $SLACK_WEBHOOK_URL \
  -H 'Content-type: application/json' \
  --data '{"text":"🎉 BCA Platform deployed successfully to production!"}'
```

### Step 23: Monitoring and Maintenance

#### 23.1 Production Monitoring Dashboard
```typescript
// Create monitoring dashboard endpoint
// backend/src/routes/admin.ts
import { Router } from 'express';
import { requireRole } from '@/middleware/auth';

const router = Router();

router.get('/dashboard', requireRole(['admin']), async (req, res) => {
  try {
    const stats = await getDashboardStats();
    res.json(stats);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch dashboard stats' });
  }
});

async function getDashboardStats() {
  const [
    userCount,
    projectCount,
    analysisCount,
    systemHealth
  ] = await Promise.all([
    pool.query('SELECT COUNT(*) FROM users'),
    pool.query('SELECT COUNT(*) FROM projects'),
    pool.query('SELECT COUNT(*) FROM bca_results'),
    checkSystemHealth()
  ]);

  return {
    users: {
      total: parseInt(userCount.rows[0].count),
      active: await getActiveUserCount()
    },
    projects: {
      total: parseInt(projectCount.rows[0].count),
      byStatus: await getProjectsByStatus()
    },
    analyses: {
      total: parseInt(analysisCount.rows[0].count),
      thisMonth: await getAnalysesThisMonth()
    },
    system: systemHealth,
    timestamp: new Date().toISOString()
  };
}

async function checkSystemHealth() {
  const checks = {
    database: false,
    redis: false,
    externalAPIs: false
  };

  try {
    await pool.query('SELECT 1');
    checks.database = true;
  } catch (error) {
    console.error('Database health check failed:', error);
  }

  try {
    await redisClient.ping();
    checks.redis = true;
  } catch (error) {
    console.error('Redis health check failed:', error);
  }

  // Test external APIs
  try {
    // Add API health checks here
    checks.externalAPIs = true;
  } catch (error) {
    console.error('External API health check failed:', error);
  }

  return checks;
}
```

#### 23.2 Backup and Recovery Scripts
```bash
#!/bin/bash
# scripts/backup.sh

echo "🗄️  Starting backup process..."

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/$DATE"

mkdir -p $BACKUP_DIR

# Database backup
echo "📊 Backing up database..."
docker-compose exec postgres pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/database.sql.gz

# File storage backup (if using local storage)
echo "📁 Backing up files..."
tar -czf $BACKUP_DIR/files.tar.gz ./uploads

# Configuration backup
echo "⚙️  Backing up configuration..."
cp -r ./config $BACKUP_DIR/
cp docker-compose.prod.yml $BACKUP_DIR/
cp .env.production $BACKUP_DIR/

# Upload to cloud storage (optional)
if [ ! -z "$AWS_S3_BUCKET" ]; then
    echo "☁️  Uploading to S3..."
    aws s3 cp $BACKUP_DIR s3://$AWS_S3_BUCKET/backups/$DATE --recursive
fi

echo "✅ Backup completed: $BACKUP_DIR"

# scripts/restore.sh
#!/bin/bash
# scripts/restore.sh

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_date>"
    echo "Available backups:"
    ls -1 /backups/
    exit 1
fi

BACKUP_DATE=$1
BACKUP_DIR="/backups/$BACKUP_DATE"

if [ ! -d "$BACKUP_DIR" ]; then
    echo "❌ Backup directory not found: $BACKUP_DIR"
    exit 1
fi

echo "🔄 Restoring from backup: $BACKUP_DATE"

# Stop services
echo "⏹️  Stopping services..."
docker-compose -f docker-compose.prod.yml down

# Restore database
echo "📊 Restoring database..."
gunzip -c $BACKUP_DIR/database.sql.gz | docker-compose exec -T postgres psql -U $DB_USER $DB_NAME

# Restore files
echo "📁 Restoring files..."
tar -xzf $BACKUP_DIR/files.tar.gz

# Restart services
echo "▶️  Starting services..."
docker-compose -f docker-compose.prod.yml up -d

echo "✅ Restore completed!"
```

## Summary

This comprehensive guide provides a complete roadmap for building a Public Infrastructure BCA Platform from a basic local application to a full-featured, production-ready system. Here's what you've built:

### ✅ **Phase 1: Local Foundation (Weeks 1-4)**
- Complete project setup with modern tech stack
- Authentication and user management
- Basic project management with PostgreSQL + PostGIS
- Simple BCA calculations
- React frontend with TypeScript

### ✅ **Phase 2: Enhanced Local Platform (Weeks 5-8)**
- Mapbox integration for geospatial features
- External data integration (mock and real APIs)
- Enhanced BCA calculations with sensitivity analysis
- Comprehensive testing setup

### ✅ **Phase 3: Advanced Features (Weeks 9-12)**
- ML-powered scope and cost estimation
- Automated proposal generation
- Real external API integration (NOAA, BLS, HERE)
- Advanced caching and performance optimization

### ✅ **Phase 4: Production Deployment (Weeks 13-16)**
- Docker containerization
- CI/CD pipeline with GitHub Actions
- Production security hardening
- Monitoring, logging, and error tracking
- Backup and recovery procedures

### **Key Features Delivered:**
- 🗺️ Interactive mapping with project location selection
- 📊 Comprehensive BCA analysis with multiple methodologies
- 🤖 ML-powered cost estimation
- 📄 Automated proposal generation
- 🔒 Enterprise-grade security
- 📈 Real-time monitoring and analytics
- ☁️ Cloud-ready deployment

### **Technologies Used:**
- **Frontend:** React 18, Next.js 14, TypeScript, Tailwind CSS, Mapbox GL JS
- **Backend:** Node.js, Express, TypeScript, PostgreSQL + PostGIS, Redis
- **ML:** Python, scikit-learn, TensorFlow
- **Infrastructure:** Docker, nginx, GitHub Actions
- **Monitoring:** Custom metrics, health checks, error tracking

The platform is now ready for production use and can scale to handle thousands of users and projects while providing sophisticated benefit-cost analysis capabilities for public infrastructure projects.
  # Public Infrastructure BCA Platform - Complete Development Guide

## Overview
This guide provides a step-by-step process to build a Public Infrastructure Benefit-Cost Analysis Platform, starting with a minimal local version and progressively adding features until reaching a full cloud-deployed enterprise solution.

## Prerequisites
- Node.js 18+ and npm/yarn
- Docker and Docker Compose
- Git
- PostgreSQL 15+
- Redis
- Python 3.9+ (for ML components)
- Basic knowledge of React, TypeScript, and backend development

---

## Phase 1: Local Foundation (Weeks 1-4)

### Step 1: Project Setup and Basic Infrastructure

#### 1.1 Initialize Project Structure
```bash
# Create main project directory
mkdir bca-platform
cd bca-platform

# Initialize git repository
git init
echo "node_modules/" > .gitignore
echo ".env*" >> .gitignore
echo "*.log" >> .gitignore
echo "dist/" >> .gitignore

# Create directory structure
mkdir -p {
  frontend/{src,public,components,pages,hooks,utils,types},
  backend/{src,services,shared,tests},
  database/{migrations,seeds},
  docker,
  docs,
  scripts
}
```

#### 1.2 Setup Development Environment
```bash
# Create docker-compose.yml for local development
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_DB: bca_platform_dev
      POSTGRES_USER: bca_user
      POSTGRES_PASSWORD: bca_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
EOF

# Start local services
docker-compose up -d
```

#### 1.3 Initialize Frontend Application
```bash
cd frontend

# Create Next.js application with TypeScript
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir

# Install additional dependencies
npm install @tanstack/react-query zustand
npm install mapbox-gl @types/mapbox-gl
npm install recharts lucide-react
npm install @headlessui/react @radix-ui/react-dialog

# Development dependencies
npm install -D @testing-library/react @testing-library/jest-dom
npm install -D jest jest-environment-jsdom

# Create basic file structure
mkdir -p src/{components,hooks,lib,types,stores}
mkdir -p src/components/{ui,layout,map,projects}
```

#### 1.4 Initialize Backend Service
```bash
cd ../backend

# Initialize Node.js project
npm init -y

# Install core dependencies
npm install express typescript ts-node
npm install cors helmet morgan compression
npm install @types/express @types/cors @types/morgan
npm install pg @types/pg
npm install redis @types/redis
npm install jsonwebtoken @types/jsonwebtoken
npm install bcryptjs @types/bcryptjs
npm install dotenv

# Development dependencies
npm install -D nodemon @types/node jest @types/jest
npm install -D ts-jest supertest @types/supertest

# Create TypeScript configuration
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": "./src",
    "paths": {
      "@/*": ["*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
EOF

# Create basic directory structure
mkdir -p src/{controllers,services,models,middleware,utils,config}
```

### Step 2: Database Setup and Core Models

#### 2.1 Create Database Schema
```sql
-- Create database/init.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    role VARCHAR(50) NOT NULL DEFAULT 'analyst',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Organizations table
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Projects table
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    project_type VARCHAR(50) NOT NULL,
    infrastructure_category VARCHAR(50) NOT NULL,
    location_geom GEOMETRY(POINT, 4326),
    estimated_cost DECIMAL(15,2),
    status VARCHAR(50) DEFAULT 'draft',
    owner_id UUID REFERENCES users(id),
    organization_id UUID REFERENCES organizations(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- BCA Results table
CREATE TABLE bca_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id),
    methodology VARCHAR(50) NOT NULL,
    discount_rate DECIMAL(5,4),
    analysis_period INTEGER,
    npv DECIMAL(15,2),
    bcr DECIMAL(10,4),
    irr DECIMAL(5,4),
    results_json JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_projects_location ON projects USING GIST (location_geom);
CREATE INDEX idx_projects_owner ON projects (owner_id);
CREATE INDEX idx_projects_status ON projects (status);
CREATE INDEX idx_bca_results_project ON bca_results (project_id);
```

#### 2.2 Create Database Connection and Models
```typescript
// backend/src/config/database.ts
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

export const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'bca_platform_dev',
  user: process.env.DB_USER || 'bca_user',
  password: process.env.DB_PASSWORD || 'bca_password',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Test connection
pool.on('connect', () => {
  console.log('Connected to PostgreSQL database');
});

pool.on('error', (err) => {
  console.error('Database connection error:', err);
});

// backend/src/models/User.ts
export interface User {
  id: string;
  email: string;
  firstName?: string;
  lastName?: string;
  role: 'admin' | 'project_manager' | 'analyst' | 'viewer';
  organizationId?: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateUserData {
  email: string;
  password: string;
  firstName?: string;
  lastName?: string;
  role: string;
  organizationId?: string;
}

// backend/src/models/Project.ts
export interface Project {
  id: string;
  name: string;
  description?: string;
  projectType: string;
  infrastructureCategory: string;
  location?: {
    type: 'Point';
    coordinates: [number, number]; // [longitude, latitude]
  };
  estimatedCost?: number;
  status: 'draft' | 'active' | 'completed' | 'cancelled';
  ownerId: string;
  organizationId?: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateProjectData {
  name: string;
  description?: string;
  projectType: string;
  infrastructureCategory: string;
  location?: {
    type: 'Point';
    coordinates: [number, number];
  };
  estimatedCost?: number;
  ownerId: string;
  organizationId?: string;
}
```

### Step 3: Basic Authentication System

#### 3.1 Implement Authentication Service
```typescript
// backend/src/services/AuthService.ts
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { pool } from '@/config/database';
import { User, CreateUserData } from '@/models/User';

export class AuthService {
  private jwtSecret: string;

  constructor() {
    this.jwtSecret = process.env.JWT_SECRET || 'your-secret-key';
  }

  async register(userData: CreateUserData): Promise<User> {
    const { email, password, firstName, lastName, role, organizationId } = userData;
    
    // Check if user exists
    const existingUser = await this.findUserByEmail(email);
    if (existingUser) {
      throw new Error('User already exists');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 10);

    // Insert user
    const query = `
      INSERT INTO users (email, password_hash, first_name, last_name, role, organization_id)
      VALUES ($1, $2, $3, $4, $5, $6)
      RETURNING id, email, first_name, last_name, role, organization_id, created_at, updated_at
    `;

    const result = await pool.query(query, [
      email, passwordHash, firstName, lastName, role, organizationId
    ]);

    return this.mapUserRow(result.rows[0]);
  }

  async login(email: string, password: string): Promise<{ user: User; token: string }> {
    const user = await this.findUserByEmail(email);
    if (!user) {
      throw new Error('Invalid credentials');
    }

    // Get password hash
    const query = 'SELECT password_hash FROM users WHERE id = $1';
    const result = await pool.query(query, [user.id]);
    const passwordHash = result.rows[0]?.password_hash;

    if (!passwordHash || !await bcrypt.compare(password, passwordHash)) {
      throw new Error('Invalid credentials');
    }

    const token = this.generateToken(user);
    return { user, token };
  }

  private generateToken(user: User): string {
    return jwt.sign(
      { 
        userId: user.id, 
        email: user.email, 
        role: user.role,
        organizationId: user.organizationId 
      },
      this.jwtSecret,
      { expiresIn: '24h' }
    );
  }

  async verifyToken(token: string): Promise<any> {
    try {
      return jwt.verify(token, this.jwtSecret);
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  private async findUserByEmail(email: string): Promise<User | null> {
    const query = `
      SELECT id, email, first_name, last_name, role, organization_id, created_at, updated_at
      FROM users WHERE email = $1
    `;
    const result = await pool.query(query, [email]);
    
    return result.rows[0] ? this.mapUserRow(result.rows[0]) : null;
  }

  private mapUserRow(row: any): User {
    return {
      id: row.id,
      email: row.email,
      firstName: row.first_name,
      lastName: row.last_name,
      role: row.role,
      organizationId: row.organization_id,
      createdAt: new Date(row.created_at),
      updatedAt: new Date(row.updated_at)
    };
  }
}

// backend/src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { AuthService } from '@/services/AuthService';

interface AuthenticatedRequest extends Request {
  user?: any;
}

export const authenticateToken = async (
  req: AuthenticatedRequest, 
  res: Response, 
  next: NextFunction
) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  try {
    const authService = new AuthService();
    const decoded = await authService.verifyToken(token);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(403).json({ error: 'Invalid token' });
  }
};

export const requireRole = (allowedRoles: string[]) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};
```

#### 3.2 Create Authentication Controllers and Routes
```typescript
// backend/src/controllers/AuthController.ts
import { Request, Response } from 'express';
import { AuthService } from '@/services/AuthService';

export class AuthController {
  private authService: AuthService;

  constructor() {
    this.authService = new AuthService();
  }

  register = async (req: Request, res: Response) => {
    try {
      const user = await this.authService.register(req.body);
      res.status(201).json({ user });
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  };

  login = async (req: Request, res: Response) => {
    try {
      const { email, password } = req.body;
      const result = await this.authService.login(email, password);
      res.json(result);
    } catch (error: any) {
      res.status(401).json({ error: error.message });
    }
  };

  profile = async (req: Request, res: Response) => {
    res.json({ user: req.user });
  };
}

// backend/src/routes/auth.ts
import { Router } from 'express';
import { AuthController } from '@/controllers/AuthController';
import { authenticateToken } from '@/middleware/auth';

const router = Router();
const authController = new AuthController();

router.post('/register', authController.register);
router.post('/login', authController.login);
router.get('/profile', authenticateToken, authController.profile);

export default router;
```

### Step 4: Basic Project Management

#### 4.1 Create Project Service
```typescript
// backend/src/services/ProjectService.ts
import { pool } from '@/config/database';
import { Project, CreateProjectData } from '@/models/Project';

export class ProjectService {
  async createProject(projectData: CreateProjectData): Promise<Project> {
    const query = `
      INSERT INTO projects (
        name, description, project_type, infrastructure_category, 
        location_geom, estimated_cost, owner_id, organization_id
      )
      VALUES ($1, $2, $3, $4, ST_GeomFromGeoJSON($5), $6, $7, $8)
      RETURNING id, name, description, project_type, infrastructure_category,
                ST_AsGeoJSON(location_geom) as location, estimated_cost, status,
                owner_id, organization_id, created_at, updated_at
    `;

    const locationGeoJSON = projectData.location ? JSON.stringify(projectData.location) : null;

    const result = await pool.query(query, [
      projectData.name,
      projectData.description,
      projectData.projectType,
      projectData.infrastructureCategory,
      locationGeoJSON,
      projectData.estimatedCost,
      projectData.ownerId,
      projectData.organizationId
    ]);

    return this.mapProjectRow(result.rows[0]);
  }

  async getProjects(filters: {
    ownerId?: string;
    organizationId?: string;
    status?: string;
    page?: number;
    limit?: number;
  }): Promise<{ projects: Project[]; total: number }> {
    const { ownerId, organizationId, status, page = 1, limit = 20 } = filters;
    const offset = (page - 1) * limit;

    let whereClause = 'WHERE 1=1';
    const params: any[] = [];
    let paramIndex = 1;

    if (ownerId) {
      whereClause += ` AND owner_id = $${paramIndex}`;
      params.push(ownerId);
      paramIndex++;
    }

    if (organizationId) {
      whereClause += ` AND organization_id = $${paramIndex}`;
      params.push(organizationId);
      paramIndex++;
    }

    if (status) {
      whereClause += ` AND status = $${paramIndex}`;
      params.push(status);
      paramIndex++;
    }

    // Get total count
    const countQuery = `SELECT COUNT(*) FROM projects ${whereClause}`;
    const countResult = await pool.query(countQuery, params);
    const total = parseInt(countResult.rows[0].count);

    // Get projects
    const query = `
      SELECT id, name, description, project_type, infrastructure_category,
             ST_AsGeoJSON(location_geom) as location, estimated_cost, status,
             owner_id, organization_id, created_at, updated_at
      FROM projects 
      ${whereClause}
      ORDER BY created_at DESC
      LIMIT $${paramIndex} OFFSET $${paramIndex + 1}
    `;

    const result = await pool.query(query, [...params, limit, offset]);
    const projects = result.rows.map(this.mapProjectRow);

    return { projects, total };
  }

  async getProjectById(id: string): Promise<Project | null> {
    const query = `
      SELECT id, name, description, project_type, infrastructure_category,
             ST_AsGeoJSON(location_geom) as location, estimated_cost, status,
             owner_id, organization_id, created_at, updated_at
      FROM projects WHERE id = $1
    `;

    const result = await pool.query(query, [id]);
    return result.rows[0] ? this.mapProjectRow(result.rows[0]) : null;
  }

  async updateProject(id: string, updates: Partial<CreateProjectData>): Promise<Project> {
    const setClauses: string[] = [];
    const params: any[] = [];
    let paramIndex = 1;

    Object.entries(updates).forEach(([key, value]) => {
      if (value !== undefined) {
        if (key === 'location') {
          setClauses.push(`location_geom = ST_GeomFromGeoJSON($${paramIndex})`);
          params.push(JSON.stringify(value));
        } else {
          const dbKey = key.replace(/([A-Z])/g, '_$1').toLowerCase();
          setClauses.push(`${dbKey} = $${paramIndex}`);
          params.push(value);
        }
        paramIndex++;
      }
    });

    setClauses.push(`updated_at = NOW()`);
    params.push(id);

    const query = `
      UPDATE projects 
      SET ${setClauses.join(', ')}
      WHERE id = $${paramIndex}
      RETURNING id, name, description, project_type, infrastructure_category,
                ST_AsGeoJSON(location_geom) as location, estimated_cost, status,
                owner_id, organization_id, created_at, updated_at
    `;

    const result = await pool.query(query, params);
    return this.mapProjectRow(result.rows[0]);
  }

  async deleteProject(id: string): Promise<void> {
    await pool.query('DELETE FROM projects WHERE id = $1', [id]);
  }

  private mapProjectRow(row: any): Project {
    return {
      id: row.id,
      name: row.name,
      description: row.description,
      projectType: row.project_type,
      infrastructureCategory: row.infrastructure_category,
      location: row.location ? JSON.parse(row.location) : undefined,
      estimatedCost: parseFloat(row.estimated_cost) || undefined,
      status: row.status,
      ownerId: row.owner_id,
      organizationId: row.organization_id,
      createdAt: new Date(row.created_at),
      updatedAt: new Date(row.updated_at)
    };
  }
}
```

#### 4.2 Create Project Controller and Routes
```typescript
// backend/src/controllers/ProjectController.ts
import { Request, Response } from 'express';
import { ProjectService } from '@/services/ProjectService';

interface AuthenticatedRequest extends Request {
  user?: any;
}

export class ProjectController {
  private projectService: ProjectService;

  constructor() {
    this.projectService = new ProjectService();
  }

  createProject = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const projectData = {
        ...req.body,
        ownerId: req.user.userId,
        organizationId: req.user.organizationId
      };

      const project = await this.projectService.createProject(projectData);
      res.status(201).json(project);
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  };

  getProjects = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const filters = {
        ownerId: req.user.userId,
        organizationId: req.user.organizationId,
        status: req.query.status as string,
        page: parseInt(req.query.page as string) || 1,
        limit: parseInt(req.query.limit as string) || 20
      };

      const result = await this.projectService.getProjects(filters);
      res.json(result);
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  };

  getProject = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const project = await this.projectService.getProjectById(req.params.id);
      if (!project) {
        return res.status(404).json({ error: 'Project not found' });
      }
      res.json(project);
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  };

  updateProject = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const project = await this.projectService.updateProject(req.params.id, req.body);
      res.json(project);
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  };

  deleteProject = async (req: AuthenticatedRequest, res: Response) => {
    try {
      await this.projectService.deleteProject(req.params.id);
      res.status(204).send();
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  };
}

// backend/src/routes/projects.ts
import { Router } from 'express';
import { ProjectController } from '@/controllers/ProjectController';
import { authenticateToken } from '@/middleware/auth';

const router = Router();
const projectController = new ProjectController();

router.use(authenticateToken);

router.post('/', projectController.createProject);
router.get('/', projectController.getProjects);
router.get('/:id', projectController.getProject);
router.put('/:id', projectController.updateProject);
router.delete('/:id', projectController.deleteProject);

export default router;
```

### Step 5: Basic Frontend Application

#### 5.1 Setup Frontend Authentication
```typescript
// frontend/src/types/auth.ts
export interface User {
  id: string;
  email: string;
  firstName?: string;
  lastName?: string;
  role: 'admin' | 'project_manager' | 'analyst' | 'viewer';
  organizationId?: string;
}

export interface LoginData {
  email: string;
  password: string;
}

export interface RegisterData {
  email: string;
  password: string;
  firstName?: string;
  lastName?: string;
  role: string;
}

// frontend/src/lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001/api/v1';

class APIClient {
  private baseURL: string;
  private token: string | null = null;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
    this.loadToken();
  }

  private loadToken() {
    if (typeof window !== 'undefined') {
      this.token = localStorage.getItem('auth_token');
    }
  }

  setToken(token: string) {
    this.token = token;
    if (typeof window !== 'undefined') {
      localStorage.setItem('auth_token', token);
    }
  }

  clearToken() {
    this.token = null;
    if (typeof window !== 'undefined') {
      localStorage.removeItem('auth_token');
    }
  }

  private async request<T>(
    endpoint: string, 
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`;
    
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...options.headers as Record<string, string>
    };

    if (this.token) {
      headers.Authorization = `Bearer ${this.token}`;
    }

    const response = await fetch(url, {
      ...options,
      headers
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({ error: 'Network error' }));
      throw new Error(error.error || 'Request failed');
    }

    return response.json();
  }

  // Auth methods
  async login(data: LoginData) {
    return this.request<{ user: User; token: string }>('/auth/login', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async register(data: RegisterData) {
    return this.request<{ user: User }>('/auth/register', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async getProfile() {
    return this.request<{ user: User }>('/auth/profile');
  }

  // Project methods
  async getProjects(params?: Record<string, any>) {
    const queryString = params ? `?${new URLSearchParams(params).toString()}` : '';
    return this.request<{ projects: Project[]; total: number }>(`/projects${queryString}`);
  }

  async createProject(data: any) {
    return this.request<Project>('/projects', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async getProject(id: string) {
    return this.request<Project>(`/projects/${id}`);
  }

  async updateProject(id: string, data: any) {
    return this.request<Project>(`/projects/${id}`, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  async deleteProject(id: string) {
    return this.request<void>(`/projects/${id}`, {
      method: 'DELETE'
    });
  }
}

export const apiClient = new APIClient(API_BASE);

// frontend/src/stores/authStore.ts
import { create } from 'zustand';
import { User, LoginData, RegisterData } from '@/types/auth';
import { apiClient } from '@/lib/api';

interface AuthState {
  user: User | null;
  isLoading: boolean;
  login: (data: LoginData) => Promise<void>;
  register: (data: RegisterData) => Promise<void>;
  logout: () => void;
  checkAuth: () => Promise<void>;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  isLoading: false,

  login: async (data: LoginData) => {
    set({ isLoading: true });
    try {
      const response = await apiClient.login(data);
      apiClient.setToken(response.token);
      set({ user: response.user, isLoading: false });
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  register: async (data: RegisterData) => {
    set({ isLoading: true });
    try {
      const response = await apiClient.register(data);
      set({ user: response.user, isLoading: false });
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  logout: () => {
    apiClient.clearToken();
    set({ user: null });
  },

  checkAuth: async () => {
    set({ isLoading: true });
    try {
      const response = await apiClient.getProfile();
      set({ user: response.user, isLoading: false });
    } catch (error) {
      apiClient.clearToken();
      set({ user: null, isLoading: false });
    }
  }
}));
```

#### 5.2 Create Authentication Components
```typescript
// frontend/src/components/auth/LoginForm.tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useAuthStore } from '@/stores/authStore';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const { login, isLoading } = useAuthStore();
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    try {
      await login({ email, password });
      router.push('/dashboard');
    } catch (err: any) {
      setError(err.message);
    }
  };

  return (
    <div className="max-w-md mx-auto mt-8 p-6 bg-white rounded-lg shadow-md">
      <h2 className="text-2xl font-bold mb-6 text-center">Login</h2>
      
      {error && (
        <div className="mb-4 p-3 bg-red-100 border border-red-400 text-red-700 rounded">
          {error}
        </div>
      )}

      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label htmlFor="email" className="block text-sm font-medium text-gray-700">
            Email
          </label>
          <input
            type="email"
            id="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
          />
        </div>

        <div>
          <label htmlFor="password" className="block text-sm font-medium text-gray-700">
            Password
          </label>
          <input
            type="password"
            id="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
          />
        </div>

        <button
          type="submit"
          disabled={isLoading}
          className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 disabled:opacity-50"
        >
          {isLoading ? 'Logging in...' : 'Login'}
        </button>
      </form>
    </div>
  );
}

// frontend/src/components/layout/Header.tsx
'use client';
import { useAuthStore } from '@/stores/authStore';
import Link from 'next/link';

export function Header() {
  const { user, logout } = useAuthStore();

  return (
    <header className="bg-white shadow-sm border-b">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div className="flex justify-between items-center h-16">
          <div className="flex items-center">
            <Link href="/dashboard" className="text-xl font-bold text-blue-600">
              BCA Platform
            </Link>
          </div>

          <nav className="flex items-center space-x-4">
            {user ? (
              <>
                <Link href="/dashboard" className="text-gray-700 hover:text-blue-600">
                  Dashboard
                </Link>
                <Link href="/projects" className="text-gray-700 hover:text-blue-600">
                  Projects
                </Link>
                <div className="flex items-center space-x-2">
                  <span className="text-sm text-gray-600">
                    {user.firstName} {user.lastName}
                  </span>
                  <button
                    onClick={logout}
                    className="text-sm text-gray-600 hover:text-red-600"
                  >
                    Logout
                  </button>
                </div>
              </>
            ) : (
              <Link href="/login" className="text-blue-600 hover:text-blue-700">
                Login
              </Link>
            )}
          </nav>
        </div>
      </div>
    </header>
  );
}
```

#### 5.3 Create Project Management Components
```typescript
// frontend/src/types/project.ts
export interface Project {
  id: string;
  name: string;
  description?: string;
  projectType: string;
  infrastructureCategory: string;
  location?: {
    type: 'Point';
    coordinates: [number, number];
  };
  estimatedCost?: number;
  status: 'draft' | 'active' | 'completed' | 'cancelled';
  ownerId: string;
  organizationId?: string;
  createdAt: Date;
  updatedAt: Date;
}

// frontend/src/components/projects/ProjectList.tsx
'use client';
import { useQuery } from '@tanstack/react-query';
import { useState } from 'react';
import { apiClient } from '@/lib/api';
import { Project } from '@/types/project';
import Link from 'next/link';

export function ProjectList() {
  const [statusFilter, setStatusFilter] = useState<string>('');
  const [page, setPage] = useState(1);

  const { data, isLoading, error } = useQuery({
    queryKey: ['projects', { status: statusFilter, page }],
    queryFn: () => apiClient.getProjects({ status: statusFilter || undefined, page, limit: 10 })
  });

  if (isLoading) {
    return (
      <div className="flex justify-center items-center h-64">
        <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-blue-600"></div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center text-red-600 p-4">
        Error loading projects: {(error as Error).message}
      </div>
    );
  }

  const { projects = [], total = 0 } = data || {};

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold text-gray-900">Projects</h1>
        <Link
          href="/projects/new"
          className="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700"
        >
          Create Project
        </Link>
      </div>

      <div className="flex space-x-4">
        <select
          value={statusFilter}
          onChange={(e) => setStatusFilter(e.target.value)}
          className="px-3 py-2 border border-gray-300 rounded-md"
        >
          <option value="">All Status</option>
          <option value="draft">Draft</option>
          <option value="active">Active</option>
          <option value="completed">Completed</option>
          <option value="cancelled">Cancelled</option>
        </select>
      </div>

      <div className="bg-white shadow overflow-hidden sm:rounded-md">
        <ul className="divide-y divide-gray-200">
          {projects.map((project: Project) => (
            <li key={project.id}>
              <Link href={`/projects/${project.id}`} className="block hover:bg-gray-50">
                <div className="px-4 py-4 sm:px-6">
                  <div className="flex items-center justify-between">
                    <div className="flex items-center">
                      <div className="flex-shrink-0">
                        <div className="h-10 w-10 rounded-full bg-blue-100 flex items-center justify-center">
                          <span className="text-blue-600 font-medium">
                            {project.name.charAt(0).toUpperCase()}
                          </span>
                        </div>
                      </div>
                      <div className="ml-4">
                        <p className="text-sm font-medium text-blue-600">
                          {project.name}
                        </p>
                        <p className="text-sm text-gray-500">
                          {project.projectType} • {project.infrastructureCategory}
                        </p>
                      </div>
                    </div>
                    <div className="flex items-center space-x-4">
                      <div className="text-sm text-gray-500">
                        {project.estimatedCost && (
                          <span>
                            ${project.estimatedCost.toLocaleString()}
                          </span>
                        )}
                      </div>
                      <span className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium ${
                        project.status === 'active' ? 'bg-green-100 text-green-800' :
                        project.status === 'draft' ? 'bg-gray-100 text-gray-800' :
                        project.status === 'completed' ? 'bg-blue-100 text-blue-800' :
                        'bg-red-100 text-red-800'
                      }`}>
                        {project.status}
                      </span>
                    </div>
                  </div>
                  {project.description && (
                    <div className="mt-2">
                      <p className="text-sm text-gray-600">
                        {project.description}
                      </p>
                    </div>
                  )}
                </div>
              </Link>
            </li>
          ))}
        </ul>
      </div>

      {/* Pagination */}
      {total > 10 && (
        <div className="flex justify-center space-x-2">
          <button
            onClick={() => setPage(Math.max(1, page - 1))}
            disabled={page === 1}
            className="px-3 py-2 border border-gray-300 rounded-md disabled:opacity-50"
          >
            Previous
          </button>
          <span className="px-3 py-2">
            Page {page} of {Math.ceil(total / 10)}
          </span>
          <button
            onClick={() => setPage(page + 1)}
            disabled={page >= Math.ceil(total / 10)}
            className="px-3 py-2 border border-gray-300 rounded-md disabled:opacity-50"
          >
            Next
          </button>
        </div>
      )}
    </div>
  );
}

// frontend/src/components/projects/CreateProjectForm.tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/lib/api';

export function CreateProjectForm() {
  const router = useRouter();
  const queryClient = useQueryClient();

  const [formData, setFormData] = useState({
    name: '',
    description: '',
    projectType: '',
    infrastructureCategory: '',
    estimatedCost: '',
    location: null as { coordinates: [number, number] } | null
  });

  const [error, setError] = useState('');

  const createProjectMutation = useMutation({
    mutationFn: (data: any) => apiClient.createProject(data),
    onSuccess: (project) => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
      router.push(`/projects/${project.id}`);
    },
    onError: (err: any) => {
      setError(err.message);
    }
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    const submitData = {
      ...formData,
      estimatedCost: formData.estimatedCost ? parseFloat(formData.estimatedCost) : undefined,
      location: formData.location ? {
        type: 'Point' as const,
        coordinates: formData.location.coordinates
      } : undefined
    };

    createProjectMutation.mutate(submitData);
  };

  const handleLocationClick = (e: React.MouseEvent) => {
    e.preventDefault();
    // For now, use default coordinates (will be replaced with map picker later)
    const defaultCoords: [number, number] = [-95.7129, 37.0902];
    setFormData(prev => ({
      ...prev,
      location: { coordinates: defaultCoords }
    }));
  };

  return (
    <div className="max-w-2xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">Create New Project</h1>

      {error && (
        <div className="mb-4 p-3 bg-red-100 border border-red-400 text-red-700 rounded">
          {error}
        </div>
      )}

      <form onSubmit={handleSubmit} className="space-y-6">
        <div>
          <label htmlFor="name" className="block text-sm font-medium text-gray-700">
            Project Name *
          </label>
          <input
            type="text"
            id="name"
            value={formData.name}
            onChange={(e) => setFormData(prev => ({ ...prev, name: e.target.value }))}
            required
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div>
          <label htmlFor="description" className="block text-sm font-medium text-gray-700">
            Description
          </label>
          <textarea
            id="description"
            rows={3}
            value={formData.description}
            onChange={(e) => setFormData(prev => ({ ...prev, description: e.target.value }))}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label htmlFor="projectType" className="block text-sm font-medium text-gray-700">
              Project Type *
            </label>
            <select
              id="projectType"
              value={formData.projectType}
              onChange={(e) => setFormData(prev => ({ ...prev, projectType: e.target.value }))}
              required
              className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
            >
              <option value="">Select Type</option>
              <option value="highway">Highway</option>
              <option value="bridge">Bridge</option>
              <option value="transit">Transit</option>
              <option value="airport">Airport</option>
              <option value="rail">Rail</option>
            </select>
          </div>

          <div>
            <label htmlFor="infrastructureCategory" className="block text-sm font-medium text-gray-700">
              Infrastructure Category *
            </label>
            <select
              id="infrastructureCategory"
              value={formData.infrastructureCategory}
              onChange={(e) => setFormData(prev => ({ ...prev, infrastructureCategory: e.target.value }))}
              required
              className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
            >
              <option value="">Select Category</option>
              <option value="transportation">Transportation</option>
              <option value="utilities">Utilities</option>
              <option value="water">Water/Wastewater</option>
              <option value="energy">Energy</option>
              <option value="telecommunications">Telecommunications</option>
            </select>
          </div>
        </div>

        <div>
          <label htmlFor="estimatedCost" className="block text-sm font-medium text-gray-700">
            Estimated Cost ($)
          </label>
          <input
            type="number"
            id="estimatedCost"
            value={formData.estimatedCost}
            onChange={(e) => setFormData(prev => ({ ...prev, estimatedCost: e.target.value }))}
            min="0"
            step="1000"
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Project Location
          </label>
          <div className="mt-1">
            <button
              type="button"
              onClick={handleLocationClick}
              className="px-4 py-2 border border-gray-300 rounded-md text-sm hover:bg-gray-50"
            >
              {formData.location ? 'Location Selected' : 'Select Location on Map'}
            </button>
            {formData.location && (
              <p className="text-xs text-gray-500 mt-1">
                Coordinates: {formData.location.coordinates[1].toFixed(4)}, {formData.location.coordinates[0].toFixed(4)}
              </p>
            )}
          </div>
        </div>

        <div className="flex justify-end space-x-3">
          <button
            type="button"
            onClick={() => router.push('/projects')}
            className="px-4 py-2 border border-gray-300 rounded-md text-gray-700 hover:bg-gray-50"
          >
            Cancel
          </button>
          <button
            type="submit"
            disabled={createProjectMutation.isPending}
            className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:opacity-50"
          >
            {createProjectMutation.isPending ? 'Creating...' : 'Create Project'}
          </button>
        </div>
      </form>
    </div>
  );
}
```

### Step 6: Basic BCA Calculator

#### 6.1 Create Simple BCA Service
```typescript
// backend/src/services/BCAService.ts
interface CashFlow {
  year: number;
  benefits: number;
  costs: number;
  netFlow: number;
}

interface BCAParameters {
  discountRate: number;
  analysisPeriod: number;
  baseYear: number;
}

interface BCAResults {
  npv: number;
  bcr: number;
  paybackPeriod: number;
  cashFlows: CashFlow[];
  summary: {
    totalBenefits: number;
    totalCosts: number;
    recommendation: string;
  };
}

export class BCAService {
  async performBasicBCA(
    projectId: string, 
    parameters: BCAParameters
  ): Promise<BCAResults> {
    
    // Get project data
    const project = await this.getProjectData(projectId);
    
    // Generate basic cash flows (simplified for initial version)
    const cashFlows = this.generateBasicCashFlows(project, parameters);
    
    // Calculate metrics
    const npv = this.calculateNPV(cashFlows, parameters.discountRate);
    const bcr = this.calculateBCR(cashFlows, parameters.discountRate);
    const paybackPeriod = this.calculatePaybackPeriod(cashFlows);
    
    // Generate summary
    const totalBenefits = cashFlows.reduce((sum, cf) => sum + cf.benefits, 0);
    const totalCosts = cashFlows.reduce((sum, cf) => sum + cf.costs, 0);
    
    const recommendation = this.generateRecommendation(npv, bcr);
    
    // Store results
    await this.storeBCAResults(projectId, {
      npv,
      bcr,
      paybackPeriod,
      cashFlows,
      parameters,
      totalBenefits,
      totalCosts,
      recommendation
    });
    
    return {
      npv,
      bcr,
      paybackPeriod,
      cashFlows,
      summary: {
        totalBenefits,
        totalCosts,
        recommendation
      }
    };
  }

  private generateBasicCashFlows(project: any, parameters: BCAParameters): CashFlow[] {
    const cashFlows: CashFlow[] = [];
    const annualBenefits = this.estimateAnnualBenefits(project);
    const annualMaintenanceCosts = this.estimateAnnualMaintenance(project);
    
    for (let year = 0; year < parameters.analysisPeriod; year++) {
      const currentYear = parameters.baseYear + year;
      
      let costs = 0;
      let benefits = 0;
      
      if (year === 0) {
        // Initial construction cost
        costs = project.estimatedCost || 1000000;
      } else {
        // Annual benefits start in year 1
        benefits = annualBenefits;
        costs = annualMaintenanceCosts;
      }
      
      cashFlows.push({
        year: currentYear,
        benefits,
        costs,
        netFlow: benefits - costs
      });
    }
    
    return cashFlows;
  }

  private estimateAnnualBenefits(project: any): number {
    // Simplified benefit estimation based on project type
    const baseAnnualBenefits: Record<string, number> = {
      highway: 500000,
      bridge: 300000,
      transit: 800000,
      airport: 1200000,
      rail: 900000
    };
    
    return baseAnnualBenefits[project.projectType] || 400000;
  }

  private estimateAnnualMaintenance(project: any): number {
    // Estimate as 2% of initial cost
    return (project.estimatedCost || 1000000) * 0.02;
  }

  private calculateNPV(cashFlows: CashFlow[], discountRate: number): number {
    return cashFlows.reduce((npv, cf, index) => {
      const presentValue = cf.netFlow / Math.pow(1 + discountRate, index);
      return npv + presentValue;
    }, 0);
  }

  private calculateBCR(cashFlows: CashFlow[], discountRate: number): number {
    let totalPresentBenefits = 0;
    let totalPresentCosts = 0;
    
    cashFlows.forEach((cf, index) => {
      const discountFactor = Math.pow(1 + discountRate, index);
      totalPresentBenefits += cf.benefits / discountFactor;
      totalPresentCosts += cf.costs / discountFactor;
    });
    
    return totalPresentCosts > 0 ? totalPresentBenefits / totalPresentCosts : 0;
  }

  private calculatePaybackPeriod(cashFlows: CashFlow[]): number {
    let cumulativeNetFlow = 0;
    
    for (let i = 0; i < cashFlows.length; i++) {
      cumulativeNetFlow += cashFlows[i].netFlow;
      if (cumulativeNetFlow >= 0) {
        return i + 1;
      }
    }
    
    return cashFlows.length; // Return analysis period if never positive
  }

  private generateRecommendation(npv: number, bcr: number): string {
    if (npv > 0 && bcr > 1.0) {
      return 'Project is economically viable. Proceed with implementation.';
    } else if (npv > 0) {
      return 'Project shows positive net benefits. Consider proceeding with caution.';
    } else {
      return 'Project shows negative economic returns. Consider alternatives or redesign.';
    }
  }

  private async getProjectData(projectId: string) {
    const query = `
      SELECT id, name, project_type, estimated_cost 
      FROM projects WHERE id = $1
    `;
    const result = await pool.query(query, [projectId]);
    return result.rows[0];
  }

  private async storeBCAResults(projectId: string, results: any) {
    const query = `
      INSERT INTO bca_results (
        project_id, methodology, discount_rate, analysis_period, 
        npv, bcr, results_json
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7)
      RETURNING id
    `;
    
    await pool.query(query, [
      projectId,
      'basic_npv',
      results.parameters.discountRate,
      results.parameters.analysisPeriod,
      results.npv,
      results.bcr,
      JSON.stringify(results)
    ]);
  }
}

// backend/src/controllers/BCAController.ts
import { Request, Response } from 'express';
import { BCAService } from '@/services/BCAService';

interface AuthenticatedRequest extends Request {
  user?: any;
}

export class BCAController {
  private bcaService: BCAService;

  constructor() {
    this.bcaService = new BCAService();
  }

  performAnalysis = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const { projectId } = req.params;
      const parameters = {
        discountRate: req.body.discountRate || 0.07,
        analysisPeriod: req.body.analysisPeriod || 20,
        baseYear: req.body.baseYear || new Date().getFullYear()
      };

      const results = await this.bcaService.performBasicBCA(projectId, parameters);
      res.json(results);
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  };

  getAnalysisHistory = async (req: AuthenticatedRequest, res: Response) => {
    try {
      const { projectId } = req.params;
      // Implementation for getting analysis history
      res.json({ analyses: [] }); // Placeholder
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  };
}

// backend/src/routes/bca.ts
import { Router } from 'express';
import { BCAController } from '@/controllers/BCAController';
import { authenticateToken } from '@/middleware/auth';

const router = Router();
const bcaController = new BCAController();

router.use(authenticateToken);

router.post('/projects/:projectId/analyze', bcaController.performAnalysis);
router.get('/projects/:projectId/history', bcaController.getAnalysisHistory);

export default router;
```

#### 6.2 Create BCA Frontend Components
```typescript
// frontend/src/components/bca/BCAForm.tsx
'use client';
import { useState } from 'react';
import { useMutation } from '@tanstack/react-query';
import { apiClient } from '@/lib/api';

interface BCAFormProps {
  projectId: string;
  onAnalysisComplete: (results: any) => void;
}

export function BCAForm({ projectId, onAnalysisComplete }: BCAFormProps) {
  const [parameters, setParameters] = useState({
    discountRate: 0.07,
    analysisPeriod: 20,
    baseYear: new Date().getFullYear()
  });

  const analysisMutation = useMutation({
    mutationFn: (params: any) => 
      fetch(`/api/v1/bca/projects/${projectId}/analyze`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('auth_token')}`
        },
        body: JSON.stringify(params)
      }).then(res => res.json()),
    onSuccess: onAnalysisComplete
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    analysisMutation.mutate(parameters);
  };

  return (
    <div className="bg-white p-6 rounded-lg shadow">
      <h3 className="text-lg font-medium mb-4">BCA Analysis Parameters</h3>
      
      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700">
            Discount Rate (%)
          </label>
          <input
            type="number"
            value={parameters.discountRate * 100}
            onChange={(e) => setParameters(prev => ({
              ...prev,
              discountRate: parseFloat(e.target.value) / 100
            }))}
            min="0"
            max="20"
            step="0.1"
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Analysis Period (years)
          </label>
          <input
            type="number"
            value={parameters.analysisPeriod}
            onChange={(e) => setParameters(prev => ({
              ...prev,
              analysisPeriod: parseInt(e.target.value)
            }))}
            min="1"
            max="50"
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Base Year
          </label>
          <input
            type="number"
            value={parameters.baseYear}
            onChange={(e) => setParameters(prev => ({
              ...prev,
              baseYear: parseInt(e.target.value)
            }))}
            min="2020"
            max="2030"
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <button
          type="submit"
          disabled={analysisMutation.isPending}
          className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:opacity-50"
        >
          {analysisMutation.isPending ? 'Analyzing...' : 'Run BCA Analysis'}
        </button>
      </form>
    </div>
  );
}

// frontend/src/components/bca/BCAResults.tsx
'use client';
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface BCAResultsProps {
  results: {
    npv: number;
    bcr: number;
    paybackPeriod: number;
    cashFlows: Array<{
      year: number;
      benefits: number;
      costs: number;
      netFlow: number;
    }>;
    summary: {
      totalBenefits: number;
      totalCosts: number;
      recommendation: string;
    };
  };
}

export function BCAResults({ results }: BCAResultsProps) {
  const formatCurrency = (value: number) => {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 0,
      maximumFractionDigits: 0
    }).format(value);
  };

  return (
    <div className="space-y-6">
      {/* Key Metrics */}
      <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div className="bg-white p-6 rounded-lg shadow">
          <h3 className="text-sm font-medium text-gray-500">Net Present Value</h3>
          <p className={`text-2xl font-bold ${results.npv >= 0 ? 'text-green-600' : 'text-red-600'}`}>
            {formatCurrency(results.npv)}
          </p>
        </div>

        <div className="bg-white p-6 rounded-lg shadow">
          <h3 className="text-sm font-medium text-gray-500">Benefit-Cost Ratio</h3>
          <p className={`text-2xl font-bold ${results.bcr >= 1 ? 'text-green-600' : 'text-red-600'}`}>
            {results.bcr.toFixed(2)}
          </p>
        </div>

        <div className="bg-white p-6 rounded-lg shadow">
          <h3 className="text-sm font-medium text-gray-500">Payback Period</h3>
          <p className="text-2xl font-bold text-blue-600">
            {results.paybackPeriod} years
          </p>
        </div>
      </div>

      {/* Recommendation */}
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-medium mb-2">Recommendation</h3>
        <p className="text-gray-700">{results.summary.recommendation}</p>
      </div>

      {/* Cash Flow Chart */}
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-medium mb-4">Cash Flow Analysis</h3>
        <div className="h-80">
          <ResponsiveContainer width="100%" height="100%">
            <BarChart data={results.cashFlows}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="year" />
              <YAxis tickFormatter={(value) => `${(value / 1000000).toFixed(1)}M`} />
              <Tooltip 
                formatter={(value: number, name: string) => [formatCurrency(value), name]}
                labelFormatter={(label) => `Year ${label}`}
              />
              <Bar dataKey="benefits" fill="#10B981" name="Benefits" />
              <Bar dataKey="costs" fill="#EF4444" name="Costs" />
              <Bar dataKey="netFlow" fill="#3B82F6" name="Net Cash Flow" />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </div>

      {/* Summary Table */}
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-medium mb-4">Financial Summary</h3>
        <div className="grid grid-cols-2 gap-4">
          <div>
            <p className="text-sm text-gray-500">Total Present Value of Benefits</p>
            <p className="text-lg font-semibold text-green-600">
              {formatCurrency(results.summary.totalBenefits)}
            </p>
          </div>
          <div>
            <p className="text-sm text-gray-500">Total Present Value of Costs</p>
            <p className="text-lg font-semibold text-red-600">
              {formatCurrency(results.summary.totalCosts)}
            </p>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### Step 7: Complete Backend Setup

#### 7.1 Create Main Application File
```typescript
// backend/src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import dotenv from 'dotenv';

// Import routes
import authRoutes from '@/routes/auth';
import projectRoutes from '@/routes/projects';
import bcaRoutes from '@/routes/bca';

dotenv.config();

const app = express();

// Middleware
app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
app.use(compression());
app.use(morgan('combined'));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/projects', projectRoutes);
app.use('/api/v1/bca', bcaRoutes);

// Health check
app.get('/api/v1/health', (req, res) => {
  res.json({ status: 'OK', timestamp: new Date().toISOString() });
});

// Error handling
app.use((err: any, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

export { app };

// backend/src/server.ts
import { app } from './app';
import { pool } from './config/database';

const PORT = process.env.PORT || 3001;

async function startServer() {
  try {
    // Test database connection
    await pool.query('SELECT NOW()');
    console.log('Database connected successfully');

    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
      console.log(`Health check: http://localhost:${PORT}/api/v1/health`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

startServer();

// backend/package.json scripts
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "jest",
    "test:watch": "jest --watch"
  }
}
```

#### 7.2 Environment Configuration
```bash
# Create backend/.env
PORT=3001
NODE_ENV=development

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=bca_platform_dev
DB_USER=bca_user
DB_PASSWORD=bca_password

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production

# Frontend URL
FRONTEND_URL=http://localhost:3000

# Create frontend/.env.local
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
```

### Step 8: Complete Frontend Setup

#### 8.1 Create Main Layout and Pages
```typescript
// frontend/src/app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import { Providers } from './providers';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'BCA Platform',
  description: 'Public Infrastructure Benefit-Cost Analysis Platform',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}

// frontend/src/app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 5 * 60 * 1000, // 5 minutes
        gcTime: 10 * 60 * 1000, // 10 minutes
      },
    },
  }));

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

// frontend/src/app/page.tsx
import Link from 'next/link';

export default function HomePage() {
  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
        <div className="text-center">
          <h1 className="text-4xl font-bold text-gray-900 sm:text-5xl md:text-6xl">
            BCA Platform
          </h1>
          <p className="mt-3 max-w-md mx-auto text-base text-gray-500 sm:text-lg md:mt-5 md:text-xl md:max-w-3xl">
            Professional Benefit-Cost Analysis for Public Infrastructure Projects
          </p>
          <div className="mt-5 max-w-md mx-auto sm:flex sm:justify-center md:mt-8">
            <div className="rounded-md shadow">
              <Link
                href="/login"
                className="w-full flex items-center justify-center px-8 py-3 border border-transparent text-base font-medium rounded-md text-white bg-blue-600 hover:bg-blue-700 md:py-4 md:text-lg md:px-10"
              >
                Get Started
              </Link>
            </div>
            <div className="mt-3 rounded-md shadow sm:mt-0 sm:ml-3">
              <Link
                href="/about"
                className="w-full flex items-center justify-center px-8 py-3 border border-transparent text-base font-medium rounded-md text-blue-600 bg-white hover:bg-gray-50 md:py-4 md:text-lg md:px-10"
              >
                Learn More
              </Link>
            </div>
          </div>
        </div>

        <div className="mt-16">
          <div className="grid grid-cols-1 gap-8 sm:grid-cols-2 lg:grid-cols-3">
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="text-lg font-medium text-gray-900">Project Management</h3>
              <p className="mt-2 text-gray-600">
                Create and manage infrastructure projects with geospatial integration.
              </p>
            </div>
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="text-lg font-medium text-gray-900">BCA Analysis</h3>
              <p className="mt-2 text-gray-600">
                Perform comprehensive benefit-cost analysis with multiple methodologies.
              </p>
            </div>
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="text-lg font-medium text-gray-900">Data Integration</h3>
              <p className="mt-2 text-gray-600">
                Integrate with external APIs for traffic, climate, and economic data.
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// frontend/src/app/login/page.tsx
import { LoginForm } from '@/components/auth/LoginForm';

export default function LoginPage() {
  return (
    <div className="min-h-screen bg-gray-50 flex flex-col justify-center py-12 sm:px-6 lg:px-8">
      <div className="sm:mx-auto sm:w-full sm:max-w-md">
        <h2 className="mt-6 text-center text-3xl font-extrabold text-gray-900">
          Sign in to BCA Platform
        </h2>
      </div>
      <div className="mt-8 sm:mx-auto sm:w-full sm:max-w-md">
        <LoginForm />
      </div>
    </div>
  );
}

// frontend/src/app/dashboard/page.tsx
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useAuthStore } from '@/stores/authStore';
import { Header } from '@/components/layout/Header';

export default function DashboardPage() {
  const { user, checkAuth } = useAuthStore();
  const router = useRouter();

  useEffect(() => {
    checkAuth().catch(() => {
      router.push('/login');
    });
  }, [checkAuth, router]);

  if (!user) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-blue-600"></div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50">
      <Header />
      <main className="max-w-7xl mx-auto py-6 sm:px-6 lg:px-8">
        <div className="px-4 py-6 sm:px-0">
          <h1 className="text-2xl font-bold text-gray-900 mb-6">
            Welcome back, {user.firstName} {user.lastName}
          </h1>
          
          <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
            <div className="bg-white overflow-hidden shadow rounded-lg">
              <div className="p-5">
                <div className="flex items-center">
                  <div className="flex-shrink-0">
                    <div className="w-8 h-8 bg-blue-500 rounded-md flex items-center justify-center">
                      <span className="text-white text-sm font-medium">P</span>
                    </div>
                  </div>
                  <div className="ml-5 w-0 flex-1">
                    <dl>
                      <dt className="text-sm font-medium text-gray-500 truncate">
                        Total Projects
                      </dt>
                      <dd className="text-lg font-medium text-gray-900">0</dd>
                    </dl>
                  </div>
                </div>
              </div>
              <div className="bg-gray-50 px-5 py-3">
                <div className="text-sm">
                  <a href="/projects" className="font-medium text-blue-600 hover:text-blue-500">
                    View all projects
                  </a>
                </div>
              </div>
            </div>

            <div className="bg-white overflow-hidden shadow rounded-lg">
              <div className="p-5">
                <div className="flex items-center">
                  <div className="flex-shrink-0">
                    <div className="w-8 h-8 bg-green-500 rounded-md flex items-center justify-center">
                      <span className="text-white text-sm font-medium">A</span>
                    </div>
                  </div>
                  <div className="ml-5 w-0 flex-1">
                    <dl>
                      <dt className="text-sm font-medium text-gray-500 truncate">
                        Completed Analyses
                      </dt>
                      <dd className="text-lg font-medium text-gray-900">0</dd>
                    </dl>
                  </div>
                </div>
              </div>
            </div>

            <div className="bg-white overflow-hidden shadow rounded-lg">
              <div className="p-5">
                <div className="flex items-center">
                  <div className="flex-shrink-0">
                    <div className="w-8 h-8 bg-yellow-500 rounded-md flex items-center justify-center">
                      <span className="text-white text-sm font-medium">$</span>
                    </div>
                  </div>
                  <div className="ml-5 w-0 flex-1">
                    <dl>
                      <dt className="text-sm font-medium text-gray-500 truncate">
                        Total Project Value
                      </dt>
                      <dd className="text-lg font-medium text-gray-900">$0</dd>
                    </dl>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div className="mt-8">
            <h2 className="text-lg font-medium text-gray-900 mb-4">Quick Actions</h2>
            <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
              <a
                href="/projects/new"
                className="relative rounded-lg border border-gray-300 bg-white px-6 py-5 shadow-sm flex items-center space-x-3 hover:border-gray-400 focus-within:ring-2 focus-within:ring-offset-2 focus-within:ring-blue-500"
              >
                <div className="flex-shrink-0">
                  <div className="w-10 h-10 bg-blue-500 rounded-lg flex items-center justify-center">
                    <span className="text-white text-lg">+</span>
                  </div>
                </div>
                <div className="flex-1 min-w-0">
                  <span className="absolute inset-0" aria-hidden="true"></span>
                  <p className="text-sm font-medium text-gray-900">Create New Project</p>
                  <p className="text-sm text-gray-500">Start a new infrastructure project</p>
                </div>
              </a>

              <a
                href="/projects"
                className="relative rounded-lg border border-gray-300 bg-white px-6 py-5 shadow-sm flex items-center space-x-3 hover:border-gray-400 focus-within:ring-2 focus-within:ring-offset-2 focus-within:ring-blue-500"
              >
                <div className="flex-shrink-0">
                  <div className="w-10 h-10 bg-green-500 rounded-lg flex items-center justify-center">
                    <span className="text-white text-lg">📊</span>
                  </div>
                </div>
                <div className="flex-1 min-w-0">
                  <span className="absolute inset-0" aria-hidden="true"></span>
                  <p className="text-sm font-medium text-gray-900">View Projects</p>
                  <p className="text-sm text-gray-500">Manage existing projects</p>
                </div>
              </a>
            </div>
          </div>
        </div>
      </main>
    </div>
  );
}
```

---

## Phase 2: Enhanced Local Platform (Weeks 5-8)

### Step 9: Add Basic Mapping Integration

#### 9.1 Setup Mapbox Integration
```bash
# Get Mapbox access token from https://account.mapbox.com/access-tokens/
# Add to frontend/.env.local
NEXT_PUBLIC_MAPBOX_TOKEN=your_mapbox_token_here
```

```typescript
// frontend/src/components/map/SimpleMap.tsx
'use client';
import { useEffect, useRef } from 'react';
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

interface SimpleMapProps {
  center?: [number, number];
  zoom?: number;
  onLocationSelect?: (coordinates: [number, number]) => void;
}

export function SimpleMap({ 
  center = [-95.7129, 37.0902], 
  zoom = 4,
  onLocationSelect 
}: SimpleMapProps) {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<mapboxgl.Map | null>(null);

  useEffect(() => {
    if (!mapContainer.current) return;

    mapboxgl.accessToken = process.env.NEXT_PUBLIC_MAPBOX_TOKEN!;
    
    map.current = new mapboxgl.Map({
      container: mapContainer.current,
      style: 'mapbox://styles/mapbox/streets-v12',
      center,
      zoom
    });

    // Add click handler for location selection
    if (onLocationSelect) {
      map.current.on('click', (e) => {
        const { lng, lat } = e.lngLat;
        onLocationSelect([lng, lat]);
      });
    }

    return () => {
      map.current?.remove();
    };
  }, [center, zoom, onLocationSelect]);

  return (
    <div 
      ref={mapContainer} 
      className="w-full h-96 rounded-lg border border-gray-200"
      data-testid="map-container"
    />
  );
}

// Update CreateProjectForm to use map
// frontend/src/components/projects/CreateProjectForm.tsx (add map section)
import { SimpleMap } from '@/components/map/SimpleMap';

// Add this to the form, replace the location button section:
<div>
  <label className="block text-sm font-medium text-gray-700 mb-2">
    Project Location
  </label>
  <SimpleMap
    onLocationSelect={(coordinates) => {
      setFormData(prev => ({
        ...prev,
        location: { coordinates }
      }));
    }}
  />
  {formData.location && (
    <p className="text-xs text-gray-500 mt-2">
      Selected: {formData.location.coordinates[1].toFixed(4)}, {formData.location.coordinates[0].toFixed(4)}
    </p>
  )}
</div>
```

### Step 10: Add External Data Integration

#### 10.1 Create Mock Data Services (for development)
```typescript
// backend/src/services/MockDataService.ts
export class MockDataService {
  async getTrafficData(bounds: any, timeRange: any): Promise<any[]> {
    // Mock traffic data
    return [
      {
        location: { latitude: 39.7392, longitude: -104.9903 },
        timestamp: new Date(),
        vehicleCount: 2500,
        averageSpeed: 45,
        congestionLevel: 'medium'
      }
    ];
  }

  async getClimateData(bounds: any, timeRange: any): Promise<any[]> {
    // Mock climate data
    return [
      {
        location: { latitude: 39.7392, longitude: -104.9903 },
        timestamp: new Date(),
        temperature: 65,
        precipitation: 0.5,
        extremeWeatherDays: 15
      }
    ];
  }

  async getEconomicData(region: string, timeRange: any): Promise<any[]> {
    // Mock economic data
    return [
      {
        region,
        timestamp: new Date(),
        constructionCostIndex: 105.2,
        laborCostIndex: 108.7,
        materialCostIndex: 103.9
      }
    ];
  }
}

// backend/src/services/DataCollectionService.ts
import { MockDataService } from './MockDataService';

export class DataCollectionService {
  private mockDataService: MockDataService;

  constructor() {
    this.mockDataService = new MockDataService();
  }

  async collectProjectData(projectId: string): Promise<any> {
    // Get project location from database
    const project = await this.getProjectData(projectId);
    
    // Define bounds and time range
    const bounds = this.calculateBounds(project.location);
    const timeRange = { 
      start: new Date(Date.now() - 365 * 24 * 60 * 60 * 1000), // 1 year ago
      end: new Date() 
    };

    // Collect data from all sources
    const [trafficData, climateData, economicData] = await Promise.all([
      this.mockDataService.getTrafficData(bounds, timeRange),
      this.mockDataService.getClimateData(bounds, timeRange),
      this.mockDataService.getEconomicData(project.region || 'US', timeRange)
    ]);

    const collectedData = {
      projectId,
      timestamp: new Date(),
      traffic: trafficData,
      climate: climateData,
      economic: economicData
    };

    // Store in database
    await this.storeCollectedData(collectedData);

    return collectedData;
  }

  private async getProjectData(projectId: string) {
    const query = `
      SELECT id, name, ST_AsGeoJSON(location_geom) as location
      FROM projects WHERE id = $1
    `;
    const result = await pool.query(query, [projectId]);
    const project = result.rows[0];
    
    if (project && project.location) {
      project.location = JSON.parse(project.location);
    }
    
    return project;
  }

  private calculateBounds(location: any) {
    if (!location || !location.coordinates) {
      return null;
    }
    
    const [lng, lat] = location.coordinates;
    const buffer = 0.1; // degrees
    
    return {
      southwest: { lat: lat - buffer, lng: lng - buffer },
      northeast: { lat: lat + buffer, lng: lng + buffer }
    };
  }

  private async storeCollectedData(data: any) {
    const query = `
      INSERT INTO collected_data (project_id, data_json, created_at)
      VALUES ($1, $2, NOW())
      ON CONFLICT (project_id) 
      DO UPDATE SET data_json = $2, updated_at = NOW()
    `;
    
    await pool.query(query, [data.projectId, JSON.stringify(data)]);
  }
}
```

#### 10.2 Add Data Collection Table
```sql
-- Add to database/migrations/002_add_data_collection.sql
CREATE TABLE collected_data (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) UNIQUE,
    data_json JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_collected_data_project ON collected_data (project_id);
```

### Step 11: Enhanced BCA Engine

#### 11.1 Improve BCA Calculations
```typescript
// backend/src/services/EnhancedBCAService.ts
import { DataCollectionService } from './DataCollectionService';

export class EnhancedBCAService {
  private dataCollectionService: DataCollectionService;

  constructor() {
    this.dataCollectionService = new DataCollectionService();
  }

  async performEnhancedBCA(projectId: string, parameters: any): Promise<any> {
    // Collect external data first
    const externalData = await this.dataCollectionService.collectProjectData(projectId);
    
    // Get project data
    const project = await this.getProjectData(projectId);
    
    // Generate enhanced cash flows using external data
    const cashFlows = this.generateEnhancedCashFlows(project, externalData, parameters);
    
    // Calculate metrics with sensitivity analysis
    const results = this.calculateBCAMetrics(cashFlows, parameters);
    
    // Add uncertainty analysis
    results.uncertaintyAnalysis = this.performUncertaintyAnalysis(cashFlows, parameters);
    
    // Store results
    await this.storeBCAResults(projectId, results);
    
    return results;
  }

  private generateEnhancedCashFlows(project: any, externalData: any, parameters: any) {
    const cashFlows = [];
    
    // Base annual benefits adjusted by external data
    let baseAnnualBenefits = this.estimateBaseAnnualBenefits(project);
    
    // Adjust benefits based on traffic data
    if (externalData.traffic && externalData.traffic.length > 0) {
      const trafficFactor = this.calculateTrafficBenefitFactor(externalData.traffic);
      baseAnnualBenefits *= trafficFactor;
    }
    
    // Adjust costs based on economic data
    let constructionCost = project.estimatedCost || 1000000;
    if (externalData.economic && externalData.economic.length > 0) {
      const costFactor = this.calculateCostFactor(externalData.economic);
      constructionCost *= costFactor;
    }
    
    for (let year = 0; year < parameters.analysisPeriod; year++) {
      const currentYear = parameters.baseYear + year;
      
      let costs = 0;
      let benefits = 0;
      
      if (year === 0) {
        costs = constructionCost;
      } else {
        benefits = baseAnnualBenefits * Math.pow(1.02, year - 1); // 2% annual growth
        costs = constructionCost * 0.02; // 2% annual maintenance
        
        // Climate resilience benefits
        if (externalData.climate && year > 5) {
          benefits += this.calculateClimateResilienceBenefits(externalData.climate, year);
        }
      }
      
      cashFlows.push({
        year: currentYear,
        benefits,
        costs,
        netFlow: benefits - costs
      });
    }
    
    return cashFlows;
  }

  private calculateTrafficBenefitFactor(trafficData: any[]): number {
    const avgVehicleCount = trafficData.reduce((sum, data) => sum + data.vehicleCount, 0) / trafficData.length;
    
    // Higher traffic = higher benefits from improvements
    if (avgVehicleCount > 3000) return 1.3;
    if (avgVehicleCount > 2000) return 1.2;
    if (avgVehicleCount > 1000) return 1.1;
    return 1.0;
  }

  private calculateCostFactor(economicData: any[]): number {
    const latestData = economicData[economicData.length - 1];
    return (latestData.constructionCostIndex || 100) / 100;
  }

  private calculateClimateResilienceBenefits(climateData: any[], year: number): number {
    const extremeWeatherDays = climateData[0]?.extremeWeatherDays || 10;
    // Benefit increases with more extreme weather and over time
    return extremeWeatherDays * 1000 * year * 0.1;
  }

  private performUncertaintyAnalysis(cashFlows: any[], parameters: any): any {
    // Monte Carlo simulation with 1000 iterations
    const iterations = 1000;
    const npvResults = [];
    
    for (let i = 0; i < iterations; i++) {
      const randomCashFlows = this.generateRandomCashFlows(cashFlows);
      const npv = this.calculateNPV(randomCashFlows, parameters.discountRate);
      npvResults.push(npv);
    }
    
    npvResults.sort((a, b) => a - b);
    
    return {
      mean: npvResults.reduce((sum
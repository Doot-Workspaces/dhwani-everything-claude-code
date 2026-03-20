---
name: angular-node-patterns
description: Angular + Node.js/Express.js + MongoDB patterns for building and maintaining microservice-based applications. Covers Amgrant v2/Form stack, component design, API patterns, and migration strategies.
origin: DRIS
---

# Angular + Node.js + MongoDB Patterns

## When to Activate

- Working on Amgrant v2 (Angular + Node microservices + MongoDB)
- Working on Amgrant Form (Angular + Express.js + MongoDB)
- Building or modifying Angular components, services, or modules
- Writing Express.js APIs or middleware
- Designing MongoDB schemas or aggregation pipelines
- Migrating features between v2 and v3 (Frappe)

## Amgrant Stack Reference

| Product | Frontend | Backend | Database | Architecture |
|---------|----------|---------|----------|-------------|
| Amgrant v3 | Frappe UI | Frappe (Python) | MySQL | Monolith (Frappe) |
| Amgrant v2 | Angular | Node.js microservices | MongoDB | Microservices |
| Amgrant Form | Angular | Express.js | MongoDB | Monolith |

## Angular Patterns

### Component Architecture

```typescript
// Feature module structure
// src/app/features/beneficiary/
// ├── beneficiary.module.ts
// ├── beneficiary-routing.module.ts
// ├── components/
// │   ├── beneficiary-list/
// │   │   ├── beneficiary-list.component.ts
// │   │   ├── beneficiary-list.component.html
// │   │   ├── beneficiary-list.component.scss
// │   │   └── beneficiary-list.component.spec.ts
// │   ├── beneficiary-form/
// │   └── beneficiary-detail/
// ├── services/
// │   ├── beneficiary.service.ts
// │   └── beneficiary.service.spec.ts
// ├── models/
// │   └── beneficiary.model.ts
// └── guards/
//     └── beneficiary-access.guard.ts
```

### Smart vs Dumb Components

```typescript
// SMART (Container) Component — handles data and business logic
@Component({
  selector: 'app-beneficiary-list',
  template: `
    <app-search-bar (search)="onSearch($event)"></app-search-bar>
    <app-beneficiary-table
      [beneficiaries]="beneficiaries$ | async"
      [loading]="loading"
      (select)="onSelect($event)"
      (delete)="onDelete($event)">
    </app-beneficiary-table>
    <app-pagination
      [total]="total"
      [page]="currentPage"
      (pageChange)="onPageChange($event)">
    </app-pagination>
  `
})
export class BeneficiaryListComponent implements OnInit {
  beneficiaries$ = this.beneficiaryService.beneficiaries$;
  loading = false;
  total = 0;
  currentPage = 1;

  constructor(
    private beneficiaryService: BeneficiaryService,
    private router: Router,
    private authService: AuthService
  ) {}

  ngOnInit() {
    this.loadBeneficiaries();
  }

  onSearch(query: string) {
    this.beneficiaryService.search(query);
  }

  onSelect(id: string) {
    // Check permission before navigation
    if (this.authService.hasPermission('beneficiary', 'read')) {
      this.router.navigate(['/beneficiary', id]);
    }
  }

  onDelete(id: string) {
    // Soft delete only — government data cannot be hard deleted
    if (this.authService.hasPermission('beneficiary', 'delete')) {
      this.beneficiaryService.softDelete(id);
    }
  }

  private loadBeneficiaries() {
    this.loading = true;
    this.beneficiaryService.getAll({ page: this.currentPage }).subscribe({
      next: (res) => {
        this.total = res.total;
        this.loading = false;
      },
      error: () => { this.loading = false; }
    });
  }
}

// DUMB (Presentational) Component — only receives inputs and emits events
@Component({
  selector: 'app-beneficiary-table',
  template: `
    <table *ngIf="!loading; else loadingTpl">
      <thead>
        <tr>
          <th>Name</th>
          <th>District</th>
          <th>Status</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let b of beneficiaries">
          <td>{{ b.name }}</td>
          <td>{{ b.district }}</td>
          <td><span [class]="'badge badge-' + b.status">{{ b.status }}</span></td>
          <td>
            <button (click)="select.emit(b._id)">View</button>
          </td>
        </tr>
      </tbody>
    </table>
    <ng-template #loadingTpl><app-spinner></app-spinner></ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class BeneficiaryTableComponent {
  @Input() beneficiaries: Beneficiary[] = [];
  @Input() loading = false;
  @Output() select = new EventEmitter<string>();
  @Output() delete = new EventEmitter<string>();
}
```

### Service Pattern with Error Handling

```typescript
// services/beneficiary.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse, HttpParams } from '@angular/common/http';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { catchError, map, tap } from 'rxjs/operators';
import { environment } from '../../../environments/environment';

export interface Beneficiary {
  _id: string;
  name: string;
  aadhaarMasked: string;  // Never store full Aadhaar on frontend
  district: string;
  block: string;
  status: 'active' | 'inactive' | 'archived';
  project: string;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}

@Injectable({ providedIn: 'root' })
export class BeneficiaryService {
  private apiUrl = `${environment.apiUrl}/api/beneficiaries`;
  private beneficiariesSubject = new BehaviorSubject<Beneficiary[]>([]);
  beneficiaries$ = this.beneficiariesSubject.asObservable();

  constructor(private http: HttpClient) {}

  getAll(params: { page?: number; limit?: number; district?: string } = {}):
    Observable<PaginatedResponse<Beneficiary>> {
    let httpParams = new HttpParams()
      .set('page', (params.page || 1).toString())
      .set('limit', (params.limit || 20).toString());

    if (params.district) {
      httpParams = httpParams.set('district', params.district);
    }

    return this.http.get<PaginatedResponse<Beneficiary>>(this.apiUrl, { params: httpParams })
      .pipe(
        tap(res => this.beneficiariesSubject.next(res.data)),
        catchError(this.handleError)
      );
  }

  getById(id: string): Observable<Beneficiary> {
    return this.http.get<Beneficiary>(`${this.apiUrl}/${id}`)
      .pipe(catchError(this.handleError));
  }

  create(data: Partial<Beneficiary>): Observable<Beneficiary> {
    return this.http.post<Beneficiary>(this.apiUrl, data)
      .pipe(catchError(this.handleError));
  }

  update(id: string, data: Partial<Beneficiary>): Observable<Beneficiary> {
    return this.http.put<Beneficiary>(`${this.apiUrl}/${id}`, data)
      .pipe(catchError(this.handleError));
  }

  softDelete(id: string): Observable<void> {
    // Soft delete — sets status to 'archived', never removes data
    return this.http.patch<void>(`${this.apiUrl}/${id}/archive`, {})
      .pipe(catchError(this.handleError));
  }

  search(query: string): Observable<Beneficiary[]> {
    const params = new HttpParams().set('q', query);
    return this.http.get<Beneficiary[]>(`${this.apiUrl}/search`, { params })
      .pipe(
        tap(results => this.beneficiariesSubject.next(results)),
        catchError(this.handleError)
      );
  }

  private handleError(error: HttpErrorResponse) {
    let message = 'An error occurred';
    if (error.status === 401) message = 'Session expired. Please login again.';
    else if (error.status === 403) message = 'You do not have permission for this action.';
    else if (error.status === 404) message = 'Record not found.';
    else if (error.status === 422) message = error.error?.message || 'Validation failed.';
    else if (error.status >= 500) message = 'Server error. Please try again later.';
    console.error('API Error:', error);
    return throwError(() => new Error(message));
  }
}
```

### Auth Guard for Route Protection

```typescript
// guards/role.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router } from '@angular/router';
import { AuthService } from '../../core/services/auth.service';

@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles = route.data['roles'] as string[];

    if (!this.auth.isLoggedIn()) {
      this.router.navigate(['/login']);
      return false;
    }

    if (requiredRoles && !requiredRoles.some(role => this.auth.hasRole(role))) {
      this.router.navigate(['/unauthorized']);
      return false;
    }

    return true;
  }
}

// Usage in routing module:
const routes: Routes = [
  {
    path: 'beneficiary',
    component: BeneficiaryListComponent,
    canActivate: [RoleGuard],
    data: { roles: ['admin', 'project_manager', 'data_entry'] }
  },
  {
    path: 'admin/settings',
    component: SettingsComponent,
    canActivate: [RoleGuard],
    data: { roles: ['admin'] }
  }
];
```

### HTTP Interceptors

```typescript
// core/interceptors/auth.interceptor.ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private auth: AuthService, private router: Router) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.auth.getToken();

    let authReq = req;
    if (token) {
      authReq = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
    }

    return next.handle(authReq).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.auth.logout();
          this.router.navigate(['/login']);
        }
        return throwError(() => error);
      })
    );
  }
}

// core/interceptors/audit.interceptor.ts
// Log all write operations for audit trail
@Injectable()
export class AuditInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(req.method)) {
      console.log(`[AUDIT] ${req.method} ${req.url} by ${this.getUser()}`);
    }
    return next.handle(req);
  }

  private getUser(): string {
    return localStorage.getItem('user_email') || 'anonymous';
  }
}
```

## Express.js API Patterns

### Route Structure

```
src/
├── server.ts                 # Entry point
├── config/
│   ├── database.ts          # MongoDB connection
│   ├── environment.ts       # Env variables (validated)
│   └── cors.ts              # CORS config
├── middleware/
│   ├── auth.middleware.ts    # JWT verification
│   ├── role.middleware.ts    # Role-based access
│   ├── validate.middleware.ts # Request validation
│   ├── audit.middleware.ts   # Audit logging
│   └── rate-limit.middleware.ts
├── routes/
│   ├── index.ts             # Route aggregator
│   ├── beneficiary.routes.ts
│   ├── project.routes.ts
│   └── auth.routes.ts
├── controllers/
│   ├── beneficiary.controller.ts
│   └── project.controller.ts
├── services/
│   ├── beneficiary.service.ts
│   └── project.service.ts
├── models/
│   ├── beneficiary.model.ts
│   └── project.model.ts
└── utils/
    ├── logger.ts
    ├── errors.ts
    └── validators.ts
```

### Secure Express API

```typescript
// server.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import mongoSanitize from 'express-mongo-sanitize';
import { connectDB } from './config/database';
import { routes } from './routes';
import { errorHandler } from './middleware/error.middleware';
import { auditMiddleware } from './middleware/audit.middleware';

const app = express();

// Security middleware — apply BEFORE routes
app.use(helmet());                              // Security headers
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') })); // Restricted CORS
app.use(express.json({ limit: '10mb' }));       // Body size limit
app.use(mongoSanitize());                       // Prevent NoSQL injection

// Rate limiting
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                    // 100 requests per window
  message: { error: 'Too many requests, please try again later' }
}));

// Audit all write operations
app.use(auditMiddleware);

// Routes
app.use('/api', routes);

// Health check (no auth required)
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Global error handler (MUST be last)
app.use(errorHandler);

// Start
connectDB().then(() => {
  app.listen(process.env.PORT || 3000, () => {
    console.log(`Server running on port ${process.env.PORT || 3000}`);
  });
});
```

### Controller Pattern

```typescript
// controllers/beneficiary.controller.ts
import { Request, Response, NextFunction } from 'express';
import { BeneficiaryService } from '../services/beneficiary.service';
import { AppError } from '../utils/errors';

export class BeneficiaryController {
  constructor(private service: BeneficiaryService) {}

  getAll = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { page = 1, limit = 20, district, status } = req.query;

      // User can only see their own district's data (unless admin)
      const filters: any = {};
      if (req.user?.role !== 'admin' && req.user?.district) {
        filters.district = req.user.district;
      }
      if (district) filters.district = district;
      if (status) filters.status = status;

      const result = await this.service.findAll(
        filters,
        Number(page),
        Number(limit)
      );

      res.json(result);
    } catch (error) {
      next(error);
    }
  };

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const beneficiary = await this.service.findById(req.params.id);
      if (!beneficiary) {
        throw new AppError('Beneficiary not found', 404);
      }

      // Mask PII based on role
      if (!req.user?.permissions?.includes('view_pii')) {
        beneficiary.aadhaar = this.maskAadhaar(beneficiary.aadhaar);
        beneficiary.phone = this.maskPhone(beneficiary.phone);
      }

      res.json(beneficiary);
    } catch (error) {
      next(error);
    }
  };

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const beneficiary = await this.service.create({
        ...req.body,
        createdBy: req.user?.email,
        district: req.user?.district || req.body.district,
      });
      res.status(201).json(beneficiary);
    } catch (error) {
      next(error);
    }
  };

  // Soft delete only — NEVER hard delete government data
  archive = async (req: Request, res: Response, next: NextFunction) => {
    try {
      await this.service.archive(req.params.id, req.user?.email);
      res.json({ message: 'Beneficiary archived successfully' });
    } catch (error) {
      next(error);
    }
  };

  private maskAadhaar(aadhaar?: string): string {
    if (!aadhaar || aadhaar.length < 4) return '****';
    return 'XXXX-XXXX-' + aadhaar.slice(-4);
  }

  private maskPhone(phone?: string): string {
    if (!phone || phone.length < 4) return '****';
    return 'XXXXX-' + phone.slice(-4);
  }
}
```

### Validation Middleware

```typescript
// middleware/validate.middleware.ts
import { Request, Response, NextFunction } from 'express';
import Joi from 'joi';

export function validate(schema: Joi.ObjectSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error } = schema.validate(req.body, { abortEarly: false });
    if (error) {
      const messages = error.details.map(d => d.message);
      return res.status(422).json({ errors: messages });
    }
    next();
  };
}

// Schemas
export const beneficiarySchema = Joi.object({
  name: Joi.string().required().min(2).max(200),
  district: Joi.string().required(),
  block: Joi.string().required(),
  phone: Joi.string().pattern(/^[6-9]\d{9}$/).message('Invalid Indian phone number'),
  aadhaar: Joi.string().pattern(/^\d{12}$/).message('Aadhaar must be 12 digits'),
  project: Joi.string().required(),
  status: Joi.string().valid('active', 'inactive').default('active'),
});

// Usage in routes:
// router.post('/', authenticate, authorize(['admin', 'data_entry']),
//   validate(beneficiarySchema), controller.create);
```

### Audit Middleware

```typescript
// middleware/audit.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AuditLog } from '../models/audit-log.model';

export function auditMiddleware(req: Request, res: Response, next: NextFunction) {
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(req.method)) {
    const originalJson = res.json.bind(res);

    res.json = function (body: any) {
      // Log after response is determined
      AuditLog.create({
        method: req.method,
        path: req.path,
        user: (req as any).user?.email || 'anonymous',
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        statusCode: res.statusCode,
        timestamp: new Date(),
        body: sanitizeForLog(req.body), // Remove PII before logging
      }).catch(err => console.error('Audit log failed:', err));

      return originalJson(body);
    };
  }
  next();
}

function sanitizeForLog(body: any): any {
  if (!body) return body;
  const sanitized = { ...body };
  // Never log PII
  const sensitiveFields = ['aadhaar', 'password', 'phone', 'pan', 'token'];
  for (const field of sensitiveFields) {
    if (sanitized[field]) sanitized[field] = '[REDACTED]';
  }
  return sanitized;
}
```

## Environment Configuration

```typescript
// config/environment.ts
import Joi from 'joi';

const envSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'staging', 'production').required(),
  PORT: Joi.number().default(3000),
  MONGODB_URI: Joi.string().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string().default('6h'),
  ALLOWED_ORIGINS: Joi.string().required(),
  RATE_LIMIT_MAX: Joi.number().default(100),
  LOG_LEVEL: Joi.string().valid('error', 'warn', 'info', 'debug').default('info'),
}).unknown();

const { error, value } = envSchema.validate(process.env);
if (error) {
  throw new Error(`Environment validation failed: ${error.message}`);
}

export const config = value;
```

## Testing Patterns

### Angular Component Tests

```typescript
// beneficiary-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HttpClientTestingModule } from '@angular/common/http/testing';
import { RouterTestingModule } from '@angular/router/testing';
import { BeneficiaryListComponent } from './beneficiary-list.component';
import { BeneficiaryService } from '../../services/beneficiary.service';
import { of } from 'rxjs';

describe('BeneficiaryListComponent', () => {
  let component: BeneficiaryListComponent;
  let fixture: ComponentFixture<BeneficiaryListComponent>;
  let service: jasmine.SpyObj<BeneficiaryService>;

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('BeneficiaryService', ['getAll', 'search']);
    spy.beneficiaries$ = of([]);
    spy.getAll.and.returnValue(of({ data: [], total: 0, page: 1, limit: 20 }));

    await TestBed.configureTestingModule({
      imports: [HttpClientTestingModule, RouterTestingModule],
      declarations: [BeneficiaryListComponent],
      providers: [{ provide: BeneficiaryService, useValue: spy }]
    }).compileComponents();

    fixture = TestBed.createComponent(BeneficiaryListComponent);
    component = fixture.componentInstance;
    service = TestBed.inject(BeneficiaryService) as jasmine.SpyObj<BeneficiaryService>;
  });

  it('should load beneficiaries on init', () => {
    fixture.detectChanges();
    expect(service.getAll).toHaveBeenCalledWith({ page: 1 });
  });

  it('should search when query changes', () => {
    service.search.and.returnValue(of([]));
    component.onSearch('test');
    expect(service.search).toHaveBeenCalledWith('test');
  });
});
```

### Express API Tests

```typescript
// __tests__/beneficiary.test.ts
import request from 'supertest';
import { app } from '../server';
import { Beneficiary } from '../models/beneficiary.model';

describe('Beneficiary API', () => {
  let authToken: string;

  beforeAll(async () => {
    // Login to get token
    const res = await request(app)
      .post('/api/auth/login')
      .send({ email: 'admin@dris.com', password: process.env.TEST_PASSWORD });
    authToken = res.body.token;
  });

  describe('GET /api/beneficiaries', () => {
    it('returns paginated list', async () => {
      const res = await request(app)
        .get('/api/beneficiaries')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(res.body).toHaveProperty('data');
      expect(res.body).toHaveProperty('total');
      expect(Array.isArray(res.body.data)).toBe(true);
    });

    it('rejects unauthenticated request', async () => {
      await request(app)
        .get('/api/beneficiaries')
        .expect(401);
    });

    it('masks PII for non-privileged users', async () => {
      // Login as non-admin
      const loginRes = await request(app)
        .post('/api/auth/login')
        .send({ email: 'viewer@dris.com', password: process.env.TEST_PASSWORD });

      const res = await request(app)
        .get('/api/beneficiaries/BENF-001')
        .set('Authorization', `Bearer ${loginRes.body.token}`)
        .expect(200);

      expect(res.body.aadhaar).toMatch(/^XXXX-XXXX-\d{4}$/);
      expect(res.body.phone).toMatch(/^XXXXX-\d{4}$/);
    });
  });

  describe('POST /api/beneficiaries', () => {
    it('validates required fields', async () => {
      const res = await request(app)
        .post('/api/beneficiaries')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test' }) // Missing required fields
        .expect(422);

      expect(res.body.errors).toBeDefined();
    });

    it('validates Aadhaar format', async () => {
      const res = await request(app)
        .post('/api/beneficiaries')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          name: 'Test User',
          district: 'Jaipur',
          block: 'Sanganer',
          aadhaar: '123', // Invalid
          project: 'PROJ-001'
        })
        .expect(422);

      expect(res.body.errors).toContainEqual(
        expect.stringContaining('Aadhaar')
      );
    });
  });
});
```

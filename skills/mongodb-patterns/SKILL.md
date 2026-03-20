---
name: mongodb-patterns
description: MongoDB patterns for Node.js/Express applications including schema design, aggregation pipelines, indexing, migration strategies, and security hardening for the Amgrant v2/Form stack.
origin: DRIS
---

# MongoDB Patterns

## When to Activate

- Designing MongoDB schemas for Amgrant v2 or Form
- Writing aggregation pipelines for reports and dashboards
- Optimizing MongoDB queries and indexes
- Migrating data between MongoDB collections or to PostgreSQL/Frappe
- Setting up MongoDB security and backup automation

## Schema Design

### Mongoose Model with Validation

```typescript
// models/beneficiary.model.ts
import mongoose, { Schema, Document } from 'mongoose';

export interface IBeneficiary extends Document {
  name: string;
  aadhaar: string;        // Encrypted at rest
  phone: string;           // Encrypted at rest
  district: string;
  block: string;
  village: string;
  project: mongoose.Types.ObjectId;
  status: 'active' | 'inactive' | 'archived';
  benefits: IBenefit[];
  createdBy: string;
  archivedAt?: Date;
  archivedBy?: string;
}

interface IBenefit {
  scheme: string;
  amount: number;
  disbursedOn: Date;
  verifiedBy: string;
}

const benefitSchema = new Schema<IBenefit>({
  scheme: { type: String, required: true },
  amount: { type: Number, required: true, min: 0 },
  disbursedOn: { type: Date, required: true },
  verifiedBy: { type: String, required: true },
});

const beneficiarySchema = new Schema<IBeneficiary>({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    minlength: 2,
    maxlength: 200,
  },
  aadhaar: {
    type: String,
    required: true,
    select: false,  // Never returned in queries by default
  },
  phone: {
    type: String,
    select: false,  // Never returned in queries by default
  },
  district: {
    type: String,
    required: true,
    index: true,
  },
  block: {
    type: String,
    required: true,
    index: true,
  },
  village: String,
  project: {
    type: Schema.Types.ObjectId,
    ref: 'Project',
    required: true,
    index: true,
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'archived'],
    default: 'active',
    index: true,
  },
  benefits: [benefitSchema],
  createdBy: { type: String, required: true },
  archivedAt: Date,
  archivedBy: String,
}, {
  timestamps: true,
  // Never allow hard delete
  // Use archive pattern instead
});

// Compound indexes for common queries
beneficiarySchema.index({ district: 1, status: 1 });
beneficiarySchema.index({ project: 1, status: 1 });
beneficiarySchema.index({ createdAt: -1 });

// Text index for search
beneficiarySchema.index({ name: 'text', village: 'text' });

// Pre-save: never allow status change from 'archived' back
beneficiarySchema.pre('save', function(next) {
  if (this.isModified('status') && this.status !== 'archived') {
    const original = this.get('status', String, { getters: false });
    if (original === 'archived') {
      return next(new Error('Cannot reactivate archived records'));
    }
  }
  next();
});

// Never actually delete — override remove to archive
beneficiarySchema.pre('deleteOne', function(next) {
  next(new Error('Hard delete is not allowed. Use archive instead.'));
});

export const Beneficiary = mongoose.model<IBeneficiary>('Beneficiary', beneficiarySchema);
```

## Aggregation Pipelines

### Dashboard Report

```typescript
// services/report.service.ts

export async function getProjectDashboard(projectId: string) {
  const result = await Beneficiary.aggregate([
    // Stage 1: Filter by project
    { $match: { project: new mongoose.Types.ObjectId(projectId), status: { $ne: 'archived' } } },

    // Stage 2: Group by district
    {
      $group: {
        _id: '$district',
        totalBeneficiaries: { $sum: 1 },
        activeBeneficiaries: {
          $sum: { $cond: [{ $eq: ['$status', 'active'] }, 1, 0] }
        },
        totalBenefits: { $sum: { $size: '$benefits' } },
        totalAmount: { $sum: { $sum: '$benefits.amount' } },
      }
    },

    // Stage 3: Sort by count
    { $sort: { totalBeneficiaries: -1 } },

    // Stage 4: Reshape for frontend
    {
      $project: {
        district: '$_id',
        _id: 0,
        totalBeneficiaries: 1,
        activeBeneficiaries: 1,
        totalBenefits: 1,
        totalAmount: 1,
        percentActive: {
          $round: [
            { $multiply: [
              { $divide: ['$activeBeneficiaries', { $max: ['$totalBeneficiaries', 1] }] },
              100
            ]},
            1
          ]
        }
      }
    }
  ]);

  return result;
}

// Monthly trend report
export async function getMonthlyTrend(projectId: string, months: number = 12) {
  const startDate = new Date();
  startDate.setMonth(startDate.getMonth() - months);

  return Beneficiary.aggregate([
    {
      $match: {
        project: new mongoose.Types.ObjectId(projectId),
        createdAt: { $gte: startDate },
      }
    },
    {
      $group: {
        _id: {
          year: { $year: '$createdAt' },
          month: { $month: '$createdAt' },
        },
        count: { $sum: 1 },
        amount: { $sum: { $sum: '$benefits.amount' } },
      }
    },
    { $sort: { '_id.year': 1, '_id.month': 1 } },
    {
      $project: {
        _id: 0,
        period: {
          $concat: [
            { $toString: '$_id.year' }, '-',
            { $cond: [{ $lt: ['$_id.month', 10] }, { $concat: ['0', { $toString: '$_id.month' }] }, { $toString: '$_id.month' }] }
          ]
        },
        count: 1,
        amount: 1,
      }
    }
  ]);
}
```

## Security

### NoSQL Injection Prevention

```typescript
// ALWAYS sanitize user input before MongoDB queries

// BAD — vulnerable to NoSQL injection
// If user sends: { "$gt": "" } as the email field
app.get('/api/users', async (req, res) => {
  // const users = await User.find({ email: req.query.email }); // VULNERABLE

  // GOOD — use express-mongo-sanitize middleware (installed globally)
  // AND explicitly cast/validate query params
  const email = String(req.query.email || '');
  const users = await User.find({ email });
  res.json(users);
});

// GOOD — validate ObjectId before query
import mongoose from 'mongoose';

function validateObjectId(id: string): boolean {
  return mongoose.Types.ObjectId.isValid(id);
}

app.get('/api/beneficiary/:id', async (req, res) => {
  if (!validateObjectId(req.params.id)) {
    return res.status(400).json({ error: 'Invalid ID format' });
  }
  const doc = await Beneficiary.findById(req.params.id);
  res.json(doc);
});
```

### Encryption at Rest for PII

```typescript
// utils/encryption.ts
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex'); // 32 bytes

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const tag = cipher.getAuthTag().toString('hex');
  return `${iv.toString('hex')}:${tag}:${encrypted}`;
}

export function decrypt(encrypted: string): string {
  const [ivHex, tagHex, data] = encrypted.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const tag = Buffer.from(tagHex, 'hex');
  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(tag);
  let decrypted = decipher.update(data, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}

// Usage in Mongoose middleware:
beneficiarySchema.pre('save', function(next) {
  if (this.isModified('aadhaar') && this.aadhaar && !this.aadhaar.includes(':')) {
    this.aadhaar = encrypt(this.aadhaar);
  }
  if (this.isModified('phone') && this.phone && !this.phone.includes(':')) {
    this.phone = encrypt(this.phone);
  }
  next();
});
```

## Indexing Strategy

```javascript
// Check index usage
db.beneficiaries.aggregate([{ $indexStats: {} }]);

// Find queries without index (slow query log)
db.setProfilingLevel(1, { slowms: 100 });
db.system.profile.find().sort({ ts: -1 }).limit(10);

// Recommended indexes for Amgrant
db.beneficiaries.createIndex({ district: 1, status: 1 });
db.beneficiaries.createIndex({ project: 1, status: 1 });
db.beneficiaries.createIndex({ createdAt: -1 });
db.beneficiaries.createIndex({ name: "text", village: "text" });

db.projects.createIndex({ status: 1, company: 1 });
db.projects.createIndex({ "milestones.date": 1 });

db.audit_logs.createIndex({ timestamp: -1 });
db.audit_logs.createIndex({ user: 1, timestamp: -1 });
// TTL index: auto-delete audit logs after 3 years
db.audit_logs.createIndex({ timestamp: 1 }, { expireAfterSeconds: 94608000 });
```

## Backup & Migration

### MongoDB Backup Script

```bash
#!/bin/bash
# scripts/backup-mongodb.sh
set -euo pipefail

DB_NAME="${1:-amgrant}"
BACKUP_DIR="/backups/mongodb"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR/$DATE"

# Dump with compression
mongodump --db "$DB_NAME" --gzip --out "$BACKUP_DIR/$DATE/"

# Verify backup
COLLECTIONS=$(mongodump --db "$DB_NAME" --gzip --out /dev/null --dryRun 2>&1 | grep -c "done dumping")
echo "Backed up $COLLECTIONS collections to $BACKUP_DIR/$DATE"

# Clean old backups
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +
```

### MongoDB to Frappe Migration Pattern

```python
# scripts/migrate_mongo_to_frappe.py
"""Migrate beneficiary data from MongoDB (Amgrant v2) to Frappe (Amgrant v3)."""

import pymongo
import frappe
import json

def migrate_beneficiaries():
    """Migrate beneficiaries from MongoDB to Frappe DocType."""
    mongo_client = pymongo.MongoClient(os.environ['MONGODB_URI'])
    mongo_db = mongo_client['amgrant']

    cursor = mongo_db.beneficiaries.find(
        {"status": {"$ne": "archived"}},
        batch_size=100
    )

    migrated = 0
    errors = []

    for doc in cursor:
        try:
            # Map MongoDB fields to Frappe fields
            frappe_doc = frappe.get_doc({
                "doctype": "Beneficiary",
                "beneficiary_name": doc["name"],
                "district": doc["district"],
                "block": doc["block"],
                "village": doc.get("village", ""),
                "project": map_project_id(doc["project"]),
                "status": map_status(doc["status"]),
                "mongo_id": str(doc["_id"]),  # Keep reference for audit
            })
            frappe_doc.insert(ignore_permissions=True)
            migrated += 1

            if migrated % 100 == 0:
                frappe.db.commit()
                print(f"Migrated {migrated} records...")

        except Exception as e:
            errors.append({"mongo_id": str(doc["_id"]), "error": str(e)})

    frappe.db.commit()
    print(f"Migration complete: {migrated} migrated, {len(errors)} errors")

    if errors:
        with open("migration_errors.json", "w") as f:
            json.dump(errors, f, indent=2)
```

## Common Anti-Patterns

1. **Using `$where` with user input** — SQL injection equivalent. Never use `$where`.
2. **Not limiting array growth** — Unbounded arrays in documents cause performance degradation. Use separate collections for large arrays.
3. **Missing `select: false` on PII** — Aadhaar, phone, PAN must have `select: false` so they're never accidentally returned.
4. **Hard deletes** — Use status-based archival for government data.
5. **No validation at model level** — Always use Mongoose validators, not just frontend validation.
6. **Huge aggregation pipelines on unindexed fields** — Always ensure `$match` stages use indexed fields and appear first.

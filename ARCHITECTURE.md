# Architecture Deep Dive

## System Overview

Match My Course is a Next.js application with a Node.js/Express backend API. The system prioritizes SEO performance through server-side rendering while maintaining fast client-side interactions.

## Database Schema (MongoDB)

```
┌─────────────────────┐     ┌─────────────────────┐
│      schools        │     │      courses        │
├─────────────────────┤     ├─────────────────────┤
│ _id                 │     │ _id                 │
│ name                │     │ school_id (ref)     │
│ slug                │     │ name                │
│ city                │     │ type                │
│ country             │     │ duration_weeks      │
│ description         │     │ price               │
│ founded_year        │     │ includes_visa       │
│ rating              │     │ created_at          │
│ images[]            │     └─────────────────────┘
│ logo                │
│ status (active/inactive)  │
│ created_at          │
└─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│       users         │     │    blog_posts       │
├─────────────────────┤     ├─────────────────────┤
│ _id                 │     │ _id                 │
│ name                │     │ title               │
│ email               │     │ slug                │
│ password (hashed)   │     │ content             │
│ role (admin/user)   │     │ author_id (ref)     │
│ created_at          │     │ status              │
└─────────────────────┘     │ created_at          │
                            └─────────────────────┘
```

## Request Flow

### Public Pages (SSR/SSG)

```
User Request
     │
     ▼
┌─────────────┐
│   Vercel    │ ← CDN Edge Cache
│   (Next.js) │
└──────┬──────┘
       │
       ▼ getServerSideProps / getStaticProps
┌─────────────┐
│  Node.js    │
│    API      │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────┐ ┌─────┐
│Redis│ │Mongo│
│Cache│ │ DB  │
└─────┘ └─────┘
```

### Admin Panel (Client-side)

```
Admin Action
     │
     ▼
┌─────────────┐
│   React     │
│   (SPA)     │
└──────┬──────┘
       │ JWT Auth
       ▼
┌─────────────┐
│  Node.js    │
│    API      │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────┐ ┌─────┐
│ S3  │ │Mongo│
│Image│ │ DB  │
└─────┘ └─────┘
```

## API Endpoints

### Public API

```
GET  /api/schools              # List schools (with filters)
GET  /api/schools/:slug        # School detail
GET  /api/courses              # List courses (with filters)
GET  /api/countries            # List available countries
GET  /api/cities/:country      # Cities by country
GET  /api/blog                 # Blog posts
GET  /api/blog/:slug           # Blog post detail
```

### Admin API (Protected)

```
POST   /api/admin/login        # Admin authentication
GET    /api/admin/schools      # List all schools
POST   /api/admin/schools      # Create school
PUT    /api/admin/schools/:id  # Update school
DELETE /api/admin/schools/:id  # Delete school
POST   /api/admin/upload       # Upload image to S3
GET    /api/admin/blog         # List blog posts
POST   /api/admin/blog         # Create blog post
PUT    /api/admin/blog/:id     # Update blog post
```

## Caching Strategy

### Redis Cache Layers

| Data | TTL | Invalidation |
|------|-----|--------------|
| School list | 5 min | On school CRUD |
| School detail | 10 min | On school update |
| Course list | 5 min | On course CRUD |
| City list | 1 hour | Manual |
| Blog posts | 5 min | On post CRUD |

### Cache Key Pattern

```
mmc:schools:list:{country}:{city}:{type}
mmc:schools:detail:{slug}
mmc:courses:list:{filters_hash}
mmc:cities:{country}
mmc:blog:list
mmc:blog:detail:{slug}
```

## SEO Implementation

### Dynamic Meta Tags

```javascript
// pages/schools/[slug].js
export async function getServerSideProps({ params }) {
  const school = await getSchool(params.slug);
  
  return {
    props: {
      school,
      meta: {
        title: `${school.name} - English Courses in ${school.city}`,
        description: school.description.substring(0, 160),
        image: school.images[0],
        url: `https://matchmycourse.com/schools/${school.slug}`
      }
    }
  };
}
```

### Structured Data (JSON-LD)

```javascript
const structuredData = {
  "@context": "https://schema.org",
  "@type": "EducationalOrganization",
  "name": school.name,
  "address": {
    "@type": "PostalAddress",
    "addressLocality": school.city,
    "addressCountry": school.country
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": school.rating
  }
};
```

### XML Sitemap

Generated dynamically at `/sitemap.xml`:
- All school pages
- All course type pages
- All city pages
- Blog posts
- Static pages (home, about, contact)

## Performance Optimizations

### API Response Time (800ms → 200ms)

**Problem:** Initial school list query was slow.

**Solutions implemented:**

1. **MongoDB Indexes**
```javascript
// Compound index for common queries
db.schools.createIndex({ country: 1, city: 1, status: 1 })
db.schools.createIndex({ slug: 1 }, { unique: true })
db.courses.createIndex({ school_id: 1, type: 1 })
```

2. **Redis Caching**
```javascript
async function getSchools(filters) {
  const cacheKey = `mmc:schools:${hash(filters)}`;
  
  // Check cache first
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);
  
  // Query DB
  const schools = await School.find(filters).lean();
  
  // Cache for 5 minutes
  await redis.setex(cacheKey, 300, JSON.stringify(schools));
  
  return schools;
}
```

3. **Selective Projection**
```javascript
// Only fetch needed fields for list view
const schools = await School.find(filters)
  .select('name slug city country rating logo price')
  .lean();
```

### Image Optimization

```javascript
// Next.js Image component with optimization
<Image
  src={school.images[0]}
  alt={school.name}
  width={400}
  height={300}
  placeholder="blur"
  loading="lazy"
/>
```

### Static Generation for High-Traffic Pages

```javascript
// Generate static pages for popular schools
export async function getStaticPaths() {
  const schools = await getTopSchools(50);
  
  return {
    paths: schools.map(s => ({ params: { slug: s.slug } })),
    fallback: 'blocking' // Generate others on-demand
  };
}
```

## Image Upload Flow

```
Admin uploads image
        │
        ▼
┌─────────────────┐
│ Multer (memory) │
│   validates     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Sharp (resize)  │
│ 1200px max      │
│ WebP convert    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   AWS S3        │
│ /schools/{id}/  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CloudFront     │
│  CDN URL        │
└─────────────────┘
```

## Deployment

### Frontend (Vercel)

```
Git push to main
       │
       ▼
Vercel auto-deploy
       │
       ├── Build Next.js
       ├── Generate static pages
       └── Deploy to edge
```

### Backend (AWS EC2)

```
Git push to main
       │
       ▼
GitHub Actions
       │
       ├── Run tests
       ├── Build Docker image
       └── Deploy to EC2
            │
            └── PM2 process manager
```

## Environment Variables

```bash
# Database
MONGODB_URI=mongodb+srv://...
REDIS_URL=redis://...

# AWS
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=matchmycourse-images
AWS_CLOUDFRONT_URL=https://d1234.cloudfront.net

# Auth
JWT_SECRET=...
ADMIN_EMAIL=...

# Next.js
NEXT_PUBLIC_API_URL=https://api.matchmycourse.com
```

## Monitoring

- **Errors:** Sentry integration
- **Analytics:** Google Analytics 4
- **Performance:** Vercel Analytics
- **Uptime:** AWS CloudWatch

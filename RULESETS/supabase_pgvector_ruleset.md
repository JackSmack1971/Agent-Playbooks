---
trigger: [glob, model_decision]
description: Comprehensive technical ruleset for Supabase with PostgreSQL and pgvector covering database setup, vector operations, authentication, performance optimization, and production best practices
globs: ["**/*.sql", "**/*.js", "**/*.ts", "**/*.py", "**/.env*", "**/supabase/**/*", "**/package.json"]
---

# Supabase PostgreSQL + pgvector Technical Ruleset

## Database Setup and Configuration

### Extension Management
- **Enable pgvector extension properly**: Always use `create extension if not exists vector with schema extensions;` to enable pgvector in the extensions schema
- **Verify pgvector version**: Ensure pgvector version 0.6+ for accelerated HNSW build speeds - check in Dashboard → Database → Extensions
- **Enable required extensions for AI workflows**: For automatic embeddings, also enable pgmq, pg_net, pg_cron, and hstore extensions
- **Use correct connection strings**: For pooled connections, use session pooler (port 5432) for long-running operations like index creation; use transaction pooler (port 6543) for short queries

### Vector Table Design
- **Size vector columns appropriately**: Match vector dimensions to your embedding model (e.g., 384 for gte-small, 1536 for OpenAI text-embedding-ada-002)
- **Use halfvec for space efficiency**: For embeddings ≤ 2048 dimensions, consider `halfvec(1536)` instead of `vector(1536)` to reduce storage by ~50%
- **Include proper primary keys**: Vector tables require `id` primary key for compatibility with embedding queue systems
- **Design for metadata filtering**: Include columns for common filter criteria alongside vector columns

```sql
-- Good: Properly structured vector table
create table documents (
  id bigint primary key generated always as identity,
  title text not null,
  content text not null,
  category_id bigint,
  created_at timestamp with time zone default now(),
  embedding vector(384)  -- or halfvec(384) for efficiency
);
```

## Vector Operations and pgvector

### Distance Functions and Operators
- **Choose appropriate distance function**: Use `<=>` (cosine) for normalized embeddings, `<#>` (inner product) for normalized vectors, `<->` (L2/Euclidean) for raw distances
- **Match index type to query operator**: Index operator class must match query operator - use `vector_cosine_ops` with `<=>`, `vector_ip_ops` with `<#>`, `vector_l2_ops` with `<->`
- **Normalize embeddings when using cosine**: Cosine distance requires normalized vectors for accurate similarity scoring

### Vector Indexing
- **Use HNSW over IVFFlat**: HNSW provides better performance and is more robust against changing data
- **Create indexes on populated tables**: For IVFFlat, never create index on empty tables (severely degrades recall); HNSW can be created immediately
- **Use external connections for indexing**: Dashboard has ~2 minute timeout; use psql or direct connection for large table indexing
- **Apply proper HNSW parameters**: Default `m=16` and `ef_construction=64` work for most cases; increase for high-dimensional data or accuracy requirements

```sql
-- Good: HNSW index for cosine similarity
create index on documents using hnsw (embedding vector_cosine_ops);

-- Good: HNSW with custom parameters for high accuracy
create index on documents using hnsw (embedding vector_cosine_ops) 
with (m = 32, ef_construction = 128);
```

### Vector Queries
- **Implement similarity search functions**: Create PL/pgSQL functions for reusable similarity search with proper filtering
- **Handle filtered queries carefully**: Pre-filtering with WHERE clauses may return fewer results than LIMIT expects; use iterative search for exact result counts
- **Use RPC for complex vector queries**: Execute similarity functions via `supabase.rpc()` for better performance and reusability

```sql
-- Good: Similarity search function with filtering
create or replace function match_documents (
  query_embedding vector(384),
  match_threshold float,
  match_count int,
  filter_category_id bigint default null
)
returns table (
  id bigint,
  title text,
  content text,
  similarity float
)
language sql stable
as $$
  select
    documents.id,
    documents.title,
    documents.content,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where 
    (filter_category_id is null or category_id = filter_category_id)
    and 1 - (documents.embedding <=> query_embedding) > match_threshold
  order by (documents.embedding <=> query_embedding) asc
  limit match_count;
$$;
```

## Authentication and Security

### Row Level Security (RLS)
- **Enable RLS on all user-facing tables**: Use `alter table table_name enable row level security;` for tables containing user data
- **Create specific policies for each operation**: Define separate policies for SELECT, INSERT, UPDATE, DELETE operations
- **Use auth.uid() for user-based access**: Filter rows based on authenticated user: `auth.uid() = user_id`
- **Optimize RLS performance**: Wrap JWT functions in SELECT statements: `(select auth.uid()) = user_id` to enable initPlan caching
- **Index RLS filter columns**: Add indexes on columns used in RLS policies for performance

```sql
-- Good: User-specific RLS policy with performance optimization
create policy "Users can view own documents"
  on documents for select
  using ((select auth.uid()) = user_id);

-- Good: Index for RLS performance
create index idx_documents_user_id on documents (user_id);
```

### API Key Management
- **Never expose service_role key on frontend**: Use service_role only in server environments or edge functions
- **Use anon key for public operations**: Client-side code should only use public anon key
- **Rotate keys when compromised**: Regenerate API keys in Supabase dashboard if exposure suspected
- **Implement rate limiting**: Use middleware or edge functions to limit API usage

### Authentication Best Practices
- **Enforce email verification**: Enable email confirmation in Auth settings to prevent fake accounts
- **Use strong password policies**: Configure minimum password requirements in Auth settings
- **Implement MFA where appropriate**: Enable multi-factor authentication for sensitive applications
- **Handle auth state changes**: Subscribe to `onAuthStateChange` for proper session management

## Client Configuration and Usage

### Supabase Client Setup
- **Configure client options properly**: Set appropriate timeouts, custom fetch, and connection pooling options
- **Use environment variables**: Store API keys and URLs in environment variables, never hard-code
- **Enable session persistence**: Use `persistSession: true` for better user experience
- **Configure realtime appropriately**: Only enable realtime for tables that need live updates

```javascript
// Good: Proper client configuration
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  {
    auth: {
      persistSession: true,
      autoRefreshToken: true,
      detectSessionInUrl: true
    },
    realtime: {
      params: {
        eventsPerSecond: 10
      }
    }
  }
);
```

### Database Interactions
- **Use type-safe queries**: Generate TypeScript types with `supabase gen types`
- **Handle errors properly**: Always check for error responses and implement proper error handling
- **Use transactions for related operations**: Wrap related database operations in transactions when consistency is critical
- **Implement proper pagination**: Use `range()` for pagination instead of LIMIT/OFFSET for better performance

### Vector Operations via Client
- **Generate embeddings client-side when appropriate**: Use Transformers.js for browser-based embedding generation
- **Validate embedding dimensions**: Ensure embedding dimensions match your table schema before insertion
- **Use RPC for complex vector queries**: Execute vector similarity searches via stored procedures for better performance

```javascript
// Good: Vector similarity search via RPC
const { data: similarDocs } = await supabase.rpc('match_documents', {
  query_embedding: embedding,
  match_threshold: 0.7,
  match_count: 5,
  filter_category_id: categoryId
});
```

## Performance Optimization

### Database Performance
- **Monitor query performance**: Use `EXPLAIN ANALYZE` to understand query execution plans
- **Create appropriate indexes**: Index columns used in WHERE clauses, JOINs, and ORDER BY statements
- **Use pg_prewarm for vector indexes**: Preload vector indexes into memory: `select pg_prewarm('index_name');`
- **Configure connection pooling**: Use appropriate pooler (session vs transaction) based on query type

### Vector-Specific Performance
- **Build indexes on populated tables**: Ensure sufficient data before creating vector indexes for optimal performance
- **Tune HNSW parameters for workload**: Adjust `m` and `ef_construction` based on your accuracy vs speed requirements
- **Set ef_search for query time**: Configure `set hnsw.ef_search = N;` to balance speed vs accuracy per session
- **Monitor index usage**: Check that vector queries are using indexes via query plans

### Scaling Considerations
- **Use compute add-ons for large datasets**: Upgrade compute for better indexing and query performance
- **Implement query caching**: Cache frequent vector similarity results at application level
- **Consider read replicas**: Use read replicas for high-read workloads with vector operations
- **Monitor database metrics**: Track CPU, memory, and I/O usage during vector operations

## Production Deployment and Monitoring

### Environment Configuration
- **Set appropriate database settings**: Configure `statement_timeout`, `max_connections`, and memory settings for production workloads
- **Enable database backups**: Configure automated backups with appropriate retention periods
- **Use SSL/TLS everywhere**: Ensure all connections use encrypted transport
- **Configure log retention**: Set appropriate log levels and retention for debugging

### Monitoring and Observability
- **Monitor vector index performance**: Track query execution times for vector similarity searches
- **Set up alerting**: Configure alerts for high CPU usage, connection limits, and query timeouts
- **Track embedding quality**: Monitor similarity score distributions to detect embedding drift
- **Log authentication events**: Track sign-ins, failures, and unusual patterns

### Backup and Recovery
- **Test backup restoration**: Regularly verify backup integrity and restoration procedures
- **Document schema changes**: Maintain migration scripts and schema versioning
- **Plan for disaster recovery**: Have procedures for database recovery and failover
- **Backup vector indexes**: Consider index recreation time in disaster recovery planning

## Known Issues and Mitigations

### Common Vector Issues
- **Empty IVFFlat indexes**: Never create IVFFlat indexes on empty tables - build after data insertion
- **Dimension mismatches**: Validate embedding dimensions match table schema to prevent runtime errors
- **Index not used**: Ensure query operator matches index operator class for index utilization
- **Memory issues during indexing**: Use external connections and monitor memory usage during large index creation

### Authentication Pitfalls
- **Session not persisting**: Ensure `persistSession` is enabled and localStorage is available
- **RLS bypassing**: Never rely on client-side filtering - implement proper RLS policies
- **JWT expiration**: Handle token refresh gracefully with `autoRefreshToken: true`
- **Cookie issues**: Configure appropriate cookie settings for your domain and security requirements

### Performance Pitfalls
- **Missing RLS indexes**: Always index columns used in RLS policies for performance
- **Inefficient vector queries**: Use proper similarity functions instead of raw distance calculations
- **Connection pool exhaustion**: Monitor and tune connection limits for your workload
- **Large result sets**: Implement proper pagination for large vector query results

### Production Issues
- **Connection string confusion**: Use correct pooler type (session vs transaction) for operation type
- **Extension version lag**: Keep pgvector extension updated for latest performance improvements
- **Index maintenance**: Monitor and rebuild vector indexes as data grows significantly
- **Security key exposure**: Regularly audit code for accidentally exposed service_role keys

## Migration and Versioning

### Schema Evolution
- **Version vector schemas**: Track embedding model changes and dimension updates
- **Migrate vector data carefully**: Plan for recomputing embeddings when changing models
- **Test migrations thoroughly**: Validate vector index recreation in staging environments
- **Document breaking changes**: Maintain clear migration guides for team members

### Deployment Pipeline
- **Automate migrations**: Use Supabase CLI for repeatable database schema changes
- **Test in staging**: Validate vector operations and performance in production-like environment
- **Monitor post-deployment**: Track performance metrics after vector schema or index changes
- **Rollback procedures**: Have plans for rolling back vector schema changes if issues arise
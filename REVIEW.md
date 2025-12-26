# Lab Dash - Codebase Review & Improvement Suggestions

## Overview

**Lab Dash** is a self-hosted homelab dashboard built with React/TypeScript (frontend) and Express/Node.js (backend). It provides a centralized hub for monitoring and managing various homelab services.

---

## Key Features

| Category | Features |
|----------|----------|
| **Dashboard System** | Drag-and-drop grid layout, separate desktop/mobile layouts, multi-page support, edit mode toggle |
| **Widgets (24+)** | System monitoring, download clients (qBittorrent, Transmission, SABnzbd), media servers (Jellyfin, Sonarr, Radarr), DNS (Pi-hole, AdGuard), notes, app shortcuts |
| **Security** | JWT auth, AES-256-CBC encryption for credentials, rate limiting (7 tiers), role-based access (admin/user) |
| **Data** | JSON-based config (no database), automatic weekly backups, file uploads for custom backgrounds |
| **Deployment** | Docker multi-stage build, ARM support (v7/v8/64), Kubernetes configs included |

---

## Unique/Interesting Implementations

### 1. Dual Widget Container
Combines two widgets side-by-side on one card (e.g., CPU + Disk)

### 2. Group Widget
Collapsible containers with nested items, supporting recursive admin-only filtering

### 3. Sensitive Data Masking Pattern
```typescript
// Backend sends: { _hasApiKey: true } instead of actual key
// Frontend knows credential exists without exposing it
// On update, backend restores from existing config
```

### 4. Bulk Widget Data Loading
Single endpoint (`/api/widgets/bulk-data`) with `Promise.allSettled()` for fault-tolerant parallel loading

### 5. Custom Collision Detection
Enhanced drag-drop with group-aware collision zones and special handling for different item types

### 6. Wrapper Pattern for Sortables
Separates widget logic from drag-drop behavior (50+ sortable variants)

---

## Suggestions for Improvement

### 1. Plugin Architecture (Widget Registry)

**Current Problem**: Adding widgets requires modifying 4+ files (base component, sortable wrapper, config form, type definition, route). This is error-prone and not scalable.

**Suggested Approach**:
```typescript
// widgets/registry.ts
export const widgetRegistry = new Map<string, WidgetDefinition>();

export interface WidgetDefinition {
  type: string;
  displayName: string;
  icon: React.ComponentType;
  component: React.ComponentType<WidgetProps>;
  configForm: React.ComponentType<ConfigFormProps>;
  defaultConfig: Record<string, unknown>;
  apiHandler?: (config: unknown) => Promise<unknown>;
  refreshInterval?: number;
}

// Adding a new widget becomes one file:
// widgets/pihole/index.ts
export default {
  type: 'pihole-widget',
  displayName: 'Pi-hole',
  component: PiholeWidget,
  configForm: PiholeConfigForm,
  defaultConfig: { host: '', apiKey: '' },
  apiHandler: fetchPiholeStats,
  refreshInterval: 30000,
} satisfies WidgetDefinition;
```

**Benefits**:
- Single file per widget
- Auto-discovery via dynamic imports
- Enables community plugin ecosystem
- Easier testing in isolation

---

### 2. State Management Upgrade

**Current Problem**: Large `AppContext` with many concerns mixed together. Re-renders entire tree on any state change.

**Suggested Approach**: Use **Zustand** or **Jotai** for atomic state:
```typescript
// stores/dashboardStore.ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface DashboardStore {
  layout: Layout;
  editMode: boolean;
  moveItem: (from: number, to: number) => void;
  updateWidget: (id: string, config: Partial<WidgetConfig>) => void;
}

export const useDashboardStore = create<DashboardStore>()(
  immer((set) => ({
    layout: initialLayout,
    editMode: false,
    moveItem: (from, to) => set((state) => {
      const [item] = state.layout.splice(from, 1);
      state.layout.splice(to, 0, item);
    }),
  }))
);
```

**Benefits**:
- Selective re-renders (only components using changed data update)
- Built-in devtools
- Simpler testing
- Better TypeScript support

---

### 3. Schema-Driven Configuration

**Current Problem**: Widget configs are loosely typed. Validation happens implicitly.

**Suggested Approach**: Use **Zod** for runtime validation:
```typescript
// schemas/widgets.ts
import { z } from 'zod';

export const piholeConfigSchema = z.object({
  host: z.string().url(),
  apiKey: z.string().min(1),
  showAds: z.boolean().default(true),
  refreshInterval: z.number().min(5000).default(30000),
});

export const widgetConfigSchemas = {
  'pihole-widget': piholeConfigSchema,
  'sonarr-widget': sonarrConfigSchema,
  // ...
} as const;

// Auto-generate TypeScript types
type PiholeConfig = z.infer<typeof piholeConfigSchema>;
```

**Benefits**:
- Runtime validation on save
- Auto-generated TypeScript types
- Self-documenting configs
- Easy migration scripts when schema changes

---

### 4. Event-Driven Widget Updates (WebSockets)

**Current Problem**: Widgets poll at intervals, causing unnecessary API calls and delayed updates.

**Suggested Approach**:
```typescript
// backend: services/websocket.ts
import { Server } from 'socket.io';

io.on('connection', (socket) => {
  socket.on('subscribe', (widgetIds: string[]) => {
    widgetIds.forEach(id => socket.join(`widget:${id}`));
  });
});

// When data changes:
io.to(`widget:${widgetId}`).emit('update', newData);

// frontend: hooks/useWidgetData.ts
export function useWidgetData(widgetId: string) {
  const [data, setData] = useState(null);

  useEffect(() => {
    socket.emit('subscribe', [widgetId]);
    socket.on('update', setData);
    return () => socket.off('update', setData);
  }, [widgetId]);

  return data;
}
```

**Benefits**:
- Real-time updates
- Reduced server load (push vs pull)
- Better UX for monitoring dashboards

---

### 5. Database Backend Option (SQLite/PostgreSQL)

**Current Problem**: JSON files don't scale well, lack transactions, and make queries inefficient.

**Suggested Approach**: Add **Prisma** or **Drizzle** ORM with SQLite as default:
```typescript
// prisma/schema.prisma
model Widget {
  id        String   @id @default(cuid())
  type      String
  config    Json
  position  Int
  pageId    String
  page      Page     @relation(fields: [pageId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Page {
  id       String   @id @default(cuid())
  slug     String   @unique
  title    String
  widgets  Widget[]
  isAdmin  Boolean  @default(false)
}
```

**Benefits**:
- ACID transactions
- Efficient queries
- Better backup/restore
- SQLite requires no external service
- Migration path to PostgreSQL for larger deployments

---

### 6. Component Composition Over Wrapper Proliferation

**Current Problem**: 50+ sortable wrapper files that mostly duplicate the same pattern.

**Suggested Approach**: Use a single higher-order component or render props:
```typescript
// components/SortableWidget.tsx
export function SortableWidget({
  id,
  type,
  config,
  children
}: SortableWidgetProps) {
  const { attributes, listeners, setNodeRef, transform } = useSortable({ id });

  const WidgetComponent = widgetRegistry.get(type)?.component;

  return (
    <div ref={setNodeRef} style={getTransformStyle(transform)} {...attributes}>
      <DragHandle {...listeners} />
      {children ?? <WidgetComponent config={config} />}
    </div>
  );
}

// Usage in grid:
{layout.map(item => (
  <SortableWidget key={item.id} id={item.id} type={item.type} config={item.config} />
))}
```

**Benefits**:
- One file instead of 50+
- Consistent drag behavior
- Easier to modify drag UX globally

---

### 7. API Route Consolidation with tRPC

**Current Problem**: 24 separate route files with inconsistent patterns and manual type synchronization.

**Suggested Approach**:
```typescript
// server/routers/widgets.ts
import { router, protectedProcedure } from '../trpc';
import { z } from 'zod';

export const widgetRouter = router({
  getData: protectedProcedure
    .input(z.object({ type: z.string(), config: z.unknown() }))
    .query(async ({ input }) => {
      const handler = widgetRegistry.get(input.type)?.apiHandler;
      return handler?.(input.config);
    }),

  bulkGetData: protectedProcedure
    .input(z.array(z.object({ id: z.string(), type: z.string(), config: z.unknown() })))
    .query(async ({ input }) => {
      return Promise.allSettled(input.map(w =>
        widgetRegistry.get(w.type)?.apiHandler?.(w.config)
      ));
    }),
});
```

**Benefits**:
- End-to-end type safety
- Auto-generated API client
- No manual OpenAPI/type sync
- Built-in validation

---

### 8. Theme System Enhancement

**Current Problem**: Limited theming (just background image and title).

**Suggested Approach**:
```typescript
// themes/types.ts
interface DashboardTheme {
  name: string;
  colors: {
    primary: string;
    secondary: string;
    background: string;
    surface: string;
    text: string;
    textSecondary: string;
  };
  borderRadius: number;
  spacing: number;
  widgetStyle: 'card' | 'flat' | 'glass';
  typography: {
    fontFamily: string;
    headingWeight: number;
  };
}

// Built-in themes: dark, light, nord, dracula, catppuccin, etc.
// Users can create custom themes via JSON
```

**Benefits**:
- Full visual customization
- Community theme sharing
- Accessibility (high contrast themes)

---

### 9. Testing Infrastructure

**Current Problem**: No test files visible in the codebase.

**Suggested Approach**:
```typescript
// Widget testing with Vitest + React Testing Library
describe('PiholeWidget', () => {
  it('displays blocked percentage correctly', async () => {
    server.use(
      rest.get('/api/pihole/*', (req, res, ctx) =>
        res(ctx.json({ ads_blocked_today: 1500, dns_queries_today: 10000 }))
      )
    );

    render(<PiholeWidget config={mockConfig} />);

    expect(await screen.findByText('15%')).toBeInTheDocument();
  });
});

// E2E with Playwright
test('can add widget via edit mode', async ({ page }) => {
  await page.click('[data-testid="edit-mode-toggle"]');
  await page.click('[data-testid="add-widget"]');
  await page.selectOption('select[name="type"]', 'pihole-widget');
  await page.fill('input[name="host"]', 'http://pihole.local');
  await page.click('button[type="submit"]');

  await expect(page.locator('[data-widget-type="pihole-widget"]')).toBeVisible();
});
```

---

### 10. Improved Mobile Experience

**Suggested Additions**:
- **Swipe gestures**: Swipe between pages
- **Bottom sheet modals**: Native mobile feel
- **Pull-to-refresh**: For manual data refresh
- **Haptic feedback**: On drag operations (via `navigator.vibrate`)
- **PWA enhancements**: Offline mode with service worker caching

---

## Architecture Vision

```
┌─────────────────────────────────────────────────────────────┐
│                     Lab Dash v2.0                            │
├─────────────────────────────────────────────────────────────┤
│  Frontend (React + Zustand)                                  │
│  ├── Widget Registry (dynamic imports)                       │
│  ├── Theme Engine (CSS variables + presets)                  │
│  └── Real-time Updates (WebSocket subscription)              │
├─────────────────────────────────────────────────────────────┤
│  API Layer (tRPC)                                            │
│  ├── Type-safe procedures                                    │
│  ├── Zod validation                                          │
│  └── Automatic client generation                             │
├─────────────────────────────────────────────────────────────┤
│  Backend (Node.js + Fastify)                                 │
│  ├── Plugin system for widgets                               │
│  ├── Event bus for real-time                                 │
│  └── Database abstraction (SQLite default, Postgres option)  │
├─────────────────────────────────────────────────────────────┤
│  Storage Layer (Prisma ORM)                                  │
│  ├── Widgets, Pages, Users, Themes                           │
│  ├── Migrations for schema changes                           │
│  └── Encrypted credential storage                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Priority Recommendations

| Priority | Change | Impact | Effort |
|----------|--------|--------|--------|
| High | Widget Registry Plugin System | Extensibility | Medium |
| High | Testing Infrastructure | Reliability | Medium |
| Medium | Zustand State Management | Performance | Low |
| Medium | Zod Schema Validation | Type Safety | Low |
| Medium | Consolidate Sortable Wrappers | Maintainability | Low |
| Low | tRPC API Layer | Developer Experience | High |
| Low | Database Backend | Scalability | High |
| Low | WebSocket Real-time | UX Polish | Medium |

---

## Conclusion

The codebase is well-structured and production-ready. The most impactful improvements would be the **plugin architecture** (enables community contributions) and **testing infrastructure** (ensures reliability as the project grows).

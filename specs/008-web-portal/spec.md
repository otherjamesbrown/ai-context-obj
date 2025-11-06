# Spec 008: Web Portal (React UI)

## Purpose

Web-based UI for managing organizations, users, API keys, and viewing usage analytics.

## Dependencies

- **005-user-org-service**: Authentication and management APIs
- **007-analytics-service**: Analytics APIs

## Success Criteria

- [ ] Users can login with email/password
- [ ] Dashboard shows token usage charts
- [ ] Users can create/revoke API keys
- [ ] Org Admins can manage users
- [ ] System Admins can manage organizations
- [ ] Responsive design (mobile-friendly)

## Pages

### 1. Login Page
- Email/password form
- "Accept Invite" flow

### 2. Dashboard
- Total tokens used (current month)
- Token usage chart (last 30 days)
- Breakdown by model
- Breakdown by user (Org Admin only)

### 3. API Keys
- List all keys (name, created, last used)
- Create new key modal
- Revoke key action
- Copy key to clipboard (shown once)

### 4. Users (Org Admin only)
- List users in org
- Invite user (generates link)
- Remove user

### 5. Organizations (System Admin only)
- List all orgs
- Create org
- Edit budget

### 6. Profile
- View/edit user profile
- Change password

## Tech Stack

```
Frontend:
  - React 18 + TypeScript
  - Vite (build tool)
  - TailwindCSS (styling)
  - shadcn/ui (component library)
  - React Router (routing)
  - React Query (API state management)
  - Chart.js or Recharts (usage charts)
  
Build:
  - Vite build → static files
  - Served via Nginx container
```

## Project Structure

```
web/
├── src/
│   ├── components/       # Reusable components
│   │   ├── Layout.tsx
│   │   ├── Sidebar.tsx
│   │   ├── KeyCard.tsx
│   │   └── UsageChart.tsx
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── Dashboard.tsx
│   │   ├── APIKeys.tsx
│   │   ├── Users.tsx
│   │   └── Organizations.tsx
│   ├── hooks/
│   │   ├── useAuth.tsx
│   │   ├── useAPIKeys.ts
│   │   └── useUsage.ts
│   ├── api/
│   │   └── client.ts      # Axios/fetch wrapper
│   ├── types/
│   │   └── index.ts       # TypeScript types
│   ├── App.tsx
│   └── main.tsx
├── Dockerfile
├── nginx.conf             # Nginx configuration
├── package.json
└── vite.config.ts
```

## Authentication Flow

```tsx
// Store JWT tokens in localStorage or secure cookies
// Axios interceptor adds Bearer token to requests
// On 401, redirect to login
```

## Contracts

See `contracts/` for:
- `wireframes.md` - Page descriptions and mockups
- `routes.md` - Route structure
- `api-integration.md` - How to call backend APIs

## Deployment

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Runtime stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

## Next Steps

After completion:
- Users can manage the platform via UI
- Complements CLI tool (spec 009)


# OnTheFloor — Frontend Developer API Documentation

**Version:** 1.0.0 · **Base URL:** `https://api.onthefloor.app/api` (prod) · `http://localhost:5000/api` (dev)  
**Swagger UI:** `http://localhost:5000/api/docs`  
**Live health check:** `GET /api/health`

---

## Table of Contents

1. [Authentication](#1-authentication)
2. [Standard Response Format](#2-standard-response-format)
3. [Error Codes Reference](#3-error-codes-reference)
4. [Pagination & Search](#4-pagination--search)
5. [File Uploads](#5-file-uploads)
6. [Auth APIs (`/api/auth`)](#6-auth-apis)
7. [User APIs (`/api/users`)](#7-user-apis)
8. [Workshop Discovery (`/api/workshops`)](#8-workshop-discovery-apis)
9. [Choreographer Panel (`/api/choreographer`)](#9-choreographer-panel-apis)
10. [Studio Panel (`/api/studio-panel`)](#10-studio-panel-apis)
11. [Booking APIs (`/api/bookings`)](#11-booking-apis)
12. [Review APIs (`/api/reviews`)](#12-review-apis)
13. [Notification APIs (`/api/notifications`)](#13-notification-apis)
14. [Role System Quick Reference](#14-role-system-quick-reference)
15. [React / React Native Integration](#15-react--react-native-integration)

---

## 1. Authentication

### How it works

All protected endpoints use **Firebase Authentication**. On every API call, include the Firebase ID token:

```
Authorization: Bearer <firebase_id_token>
```

### Getting the token (Firebase client SDK)

```js
import { getAuth } from 'firebase/auth';

const auth = getAuth();
const token = await auth.currentUser.getIdToken();
// Token expires every 1 hour — Firebase refreshes automatically via getIdToken()
```

### First-time login (user provisioning)

On a user's **very first login**, the backend auto-creates their MongoDB account. Pass `x-user-role` to set their role:

```
x-user-role: choreographer    ← or: studio | user
```

> **Important:** Only send `x-user-role` on the FIRST login. After that, the role is stored in MongoDB and this header is ignored.

### Request example (JavaScript)

```js
const response = await fetch('http://localhost:5000/api/choreographer/profile', {
  headers: {
    Authorization: `Bearer ${firebaseIdToken}`,
    'Content-Type': 'application/json',
    // Only on first login:
    'x-user-role': 'choreographer',
  },
});
```

### `req.user` shape (what the backend attaches)

```json
{
  "id":    "661abc...",   ← MongoDB User._id (string)
  "uid":   "firebase-uid-xyz",  ← Firebase UID
  "role":  "choreographer",     ← "user" | "choreographer" | "studio"
  "email": "user@example.com"
}
```

---

## 2. Standard Response Format

**Every** API response follows this shape:

### ✅ Success
```json
{
  "success": true,
  "message": "Profile updated successfully",
  "data": { ... }
}
```

### ❌ Error
```json
{
  "success": false,
  "message": "Validation failed.",
  "errors": [
    { "field": "price", "message": "Price must be 0 or greater" },
    { "field": "danceStyle", "message": "danceStyle or style is required" }
  ],
  "requestId": "req-1711181876-k4x9z"
}
```

### 📋 List with pagination
```json
{
  "success": true,
  "data": {
    "workshops": [...],
    "pagination": {
      "total":   47,
      "page":    2,
      "limit":   10,
      "pages":   5,
      "hasNext": true,
      "hasPrev": true
    }
  }
}
```

---

## 3. Error Codes Reference

| HTTP Status | Meaning | When it happens |
|-------------|---------|-----------------|
| `200` | OK | Successful read/update |
| `201` | Created | Resource created |
| `400` | Bad Request | Validation failed, missing fields |
| `401` | Unauthorized | Token missing, expired, or invalid |
| `403` | Forbidden | Valid token but wrong role |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate (e.g. email already exists) |
| `429` | Too Many Requests | Rate limit hit |
| `500` | Server Error | Backend bug — report to backend team |
| `503` | Service Unavailable | Firebase Admin not initialized |

### Rate Limits

| Endpoint group | Limit |
|----------------|-------|
| `/api/*` (all endpoints) | 150 req / 15 min per IP |
| `/api/auth/*` | 10 req / 15 min per IP |
| File uploads | 20 req / 1 min per IP |

When rate limited, you receive:
```json
{ "success": false, "message": "Too many requests from this IP. Please try again later." }
```
The `Retry-After` and `RateLimit-Remaining` headers are included in rate-limited responses.

---

## 4. Pagination & Search

All list endpoints accept these standard query params:

| Param | Type | Default | Max | Description |
|-------|------|---------|-----|-------------|
| `page` | integer | `1` | — | Page number |
| `limit` | integer | `10` | `100` | Items per page |

### Workshop-specific query params

| Param | Type | Example | Description |
|-------|------|---------|-------------|
| [search](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#441-475) or `q` | string | `?search=hip hop beginner` | Full-text search (title + style + city + description) |
| `danceStyle` | string | `?danceStyle=Kathak` | Partial match on dance style |
| `city` | string | `?city=Mumbai` | Partial match on city |
| `type` | `Offline\|Online` | `?type=Offline` | Exact match |
| `level` | string | `?level=Beginner` | `Beginner`, `Intermediate`, `Advanced`, `All Level` |
| `status` | string | `?status=Upcoming` | [Upcoming](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#260-273), `Ongoing`, `Completed`, `Confirmed`, `Cancelled` |
| `priceType` | string | `?priceType=Monthly` | `Monthly` or `Session` |
| `minPrice` | number | `?minPrice=500` | Minimum price filter |
| `maxPrice` | number | `?maxPrice=2000` | Maximum price filter |
| `startFrom` | ISO date | `?startFrom=2026-04-01` | Classes starting on or after |
| `startBefore` | ISO date | `?startBefore=2026-06-30` | Classes starting on or before |
| `sortBy` | string | `?sortBy=price_asc` | See sort options below |

**Sort options for `sortBy`:**

| Value | Sorts by |
|-------|----------|
| `newest` (default) | Most recently created first |
| `oldest` | Oldest first |
| `price_asc` | Cheapest first |
| `price_desc` | Most expensive first |
| `rating` | Highest rated first |
| `popular` | Most enrolled first |
| `relevance` | Text search relevance score (auto-selected when [search](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#441-475) is used) |

**Example:**
```
GET /api/workshops?search=contemporary&city=mumbai&level=Beginner&minPrice=500&maxPrice=2000&sortBy=price_asc&page=1&limit=12
```

---

## 5. File Uploads

All uploads use `multipart/form-data`. Files are stored on Cloudinary and a URL is returned.

| Constraint | Value |
|------------|-------|
| Max file size | 10 MB |
| Allowed types | JPEG, PNG, WebP |
| For gallery (multi-upload) | Max 10 files at once |

### Single image upload

```js
const formData = new FormData();
formData.append('avatar', file);  // field name varies per endpoint

const res = await fetch('/api/choreographer/profile/avatar', {
  method: 'POST',
  headers: { Authorization: `Bearer ${token}` },
  // DO NOT set Content-Type — browser sets it automatically with boundary
  body: formData,
});
```

### Multiple image upload (studio gallery)

```js
const formData = new FormData();
files.forEach(file => formData.append('images', file));
formData.append('captions', JSON.stringify(['Main Hall', 'Stage Area', 'Lobby']));

const res = await fetch('/api/studio-panel/profile/images', {
  method: 'POST',
  headers: { Authorization: `Bearer ${token}` },
  body: formData,
});
```

---

## 6. Auth APIs

Base: `/api/auth` · Rate limited: 10 req / 15 min

### `POST /api/auth/register`
Register a new user with email + password (JWT flow).

**Request:**
```json
{
  "name": "Priya Patel",
  "email": "priya@example.com",
  "password": "SecurePass123!",
  "role": "user"
}
```

**Response `201`:**
```json
{
  "success": true,
  "message": "Registration successful",
  "data": {
    "user": {
      "_id": "661...",
      "name": "Priya Patel",
      "email": "priya@example.com",
      "role": "user"
    },
    "accessToken": "eyJhbGci...",
    "refreshToken": "eyJhbGci..."
  }
}
```

---

### `POST /api/auth/login`
Login with email + password (JWT flow).

**Request:**
```json
{
  "email": "priya@example.com",
  "password": "SecurePass123!"
}
```

**Response `200`:** Same shape as register.

---

### `POST /api/auth/firebase-sync`
Sync a Firebase-authenticated user with the backend. Call this on first app open after Firebase sign-in.

**Headers:**
```
Authorization: Bearer <firebase_id_token>
x-user-role: choreographer   ← only needed on first login
```

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "_id": "661...",
    "name": "Ananya Sharma",
    "email": "ananya@example.com",
    "role": "choreographer",
    "isNewUser": true
  }
}
```

---

### `POST /api/auth/refresh`
Refresh JWT access token.

**Request:**
```json
{ "refreshToken": "eyJhbGci..." }
```

**Response `200`:**
```json
{
  "success": true,
  "data": { "accessToken": "eyJhbGci..." }
}
```

---

### `POST /api/auth/logout`
Invalidate refresh token.  
**Auth required:** Yes

---

### `POST /api/auth/forgot-password`
Send password reset email.

**Request:**
```json
{ "email": "priya@example.com" }
```

---

## 7. User APIs

Base: `/api/users` · Auth: required

### `GET /api/users/me`
Get the currently logged-in user's profile.

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "_id": "661...",
    "name": "Priya Patel",
    "email": "priya@example.com",
    "role": "user",
    "profileImage": "https://res.cloudinary.com/...",
    "city": "Mumbai",
    "phone": "+91 9876543210",
    "bio": null,
    "danceStyles": ["Hip Hop", "Contemporary"],
    "savedWorkshops": [...],
    "enrolledWorkshops": [...],
    "following": [...],
    "isEmailVerified": true,
    "createdAt": "2026-03-01T10:00:00.000Z"
  }
}
```

---

### `PUT /api/users/me`
Update own profile.

**Request:**
```json
{
  "name": "Priya Sharma",
  "city": "Pune",
  "phone": "+91 9876543210",
  "bio": "Dance enthusiast",
  "danceStyles": ["Hip Hop", "Kathak"]
}
```

---

### `POST /api/users/me/avatar`
Upload profile picture. Field = `avatar`.

**Response `200`:**
```json
{ "success": true, "data": { "profileImage": "https://res.cloudinary.com/..." } }
```

---

### `GET /api/users/:id`
Get any user's public profile.

---

### `POST /api/users/me/save-workshop/:workshopId`
Save a workshop to favourites.

### `DELETE /api/users/me/save-workshop/:workshopId`
Remove from favourites.

### `GET /api/users/me/saved-workshops`
List saved workshops.

### `POST /api/users/me/follow/:choreographerId`
Follow a choreographer.

### `DELETE /api/users/me/follow/:choreographerId`
Unfollow.

---

## 8. Workshop Discovery APIs

Base: `/api/workshops` · **Auth: NOT required for GET** (public)

### `GET /api/workshops`
Paginated, searchable workshop listing. See [Pagination & Search](#4-pagination--search) for all query params.

**Example response `200`:**
```json
{
  "success": true,
  "data": {
    "workshops": [
      {
        "_id": "661abc...",
        "title": "Hip Hop Beginner Batch",
        "className": "Hip Hop Beginner Batch",
        "danceStyle": "Hip Hop",
        "level": "Beginner",
        "description": "A 4-week intensive for beginners.",
        "price": 1200,
        "priceType": "Monthly",
        "capacity": 20,
        "attendees": 11,
        "isFull": false,
        "startDate": "2026-04-05T00:00:00.000Z",
        "endDate": "2026-05-31T00:00:00.000Z",
        "timings": {
          "startTime": "18:00",
          "endTime": "19:30",
          "days": "Tue, Thu",
          "duration": "90 min"
        },
        "thumbnail": "https://res.cloudinary.com/...",
        "city": "Mumbai",
        "type": "Offline",
        "status": "Upcoming",
        "createdBy": "660xyz...",
        "createdByRole": "choreographer",
        "createdAt": "2026-03-20T10:00:00.000Z"
      }
    ],
    "pagination": {
      "total": 47, "page": 1, "limit": 10,
      "pages": 5, "hasNext": true, "hasPrev": false
    },
    "filters": { "applied": ["search", "city"] }
  }
}
```

---

### `GET /api/workshops/:id`
Get a single workshop with creator + trainer details.

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "_id": "661abc...",
    "title": "Hip Hop Beginner Batch",
    "createdBy": {
      "_id": "660xyz...",
      "name": "Ananya Sharma",
      "email": "ananya@...",
      "profileImage": "https://..."
    },
    "trainerRef": {
      "_id": "661trainer...",
      "name": "Raj Kumar",
      "profileImage": "https://...",
      "danceStyles": ["Hip Hop"],
      "city": "Mumbai",
      "rating": 4.8,
      "experience": "5 years"
    }
  }
}
```

---

### `POST /api/workshops`
Create a workshop. **Auth: choreographer or studio**

**Request body:**
```json
{
  "title": "Contemporary Beginner Workshop",
  "danceStyle": "Contemporary",
  "level": "Beginner",
  "description": "A 4-week intensive for beginners.",
  "startDate": "2026-04-01",
  "endDate": "2026-04-28",
  "timings": {
    "startTime": "10:00",
    "endTime": "11:30",
    "days": "Mon, Wed, Fri",
    "duration": "90 min"
  },
  "price": 1500,
  "priceType": "Monthly",
  "capacity": 15,
  "type": "Offline",
  "city": "Mumbai",
  "ageGroup": "15 to 30",
  "status": "Upcoming"
}
```

---

### `PUT /api/workshops/:id`
Update workshop. **Auth: creator only**

### `DELETE /api/workshops/:id`
Soft-delete (enrollment history preserved). **Auth: creator only**

### `POST /api/workshops/:id/image`
Upload workshop thumbnail. Field = `image`. **Auth: creator only**

### `GET /api/workshops/:id/students`
Get enrolled students. **Auth: owner only**

---

## 9. Choreographer Panel APIs

Base: `/api/choreographer` · **Auth: choreographer role required**

---

### Profile

#### `POST /api/choreographer/profile`
Create profile — call once on first setup. **Idempotent.**

**Request:**
```json
{
  "name": "Ananya Sharma",
  "bio": "Contemporary specialist",
  "city": "Mumbai"
}
```

---

#### `GET /api/choreographer/profile`
Get own full profile.

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "_id": "661...",
    "email": "ananya@example.com",
    "name": "Ananya Sharma",
    "bio": "Contemporary and Bharatanatyam specialist",
    "experience": "6 years",
    "danceStyles": ["Contemporary", "Bharatanatyam"],
    "city": "Mumbai",
    "phone": "+91 9876543210",
    "profileImage": "https://res.cloudinary.com/...",
    "bannerImage": "https://res.cloudinary.com/...",
    "socialLinks": [
      { "platform": "instagram", "url": "https://instagram.com/ananya" }
    ],
    "rating": 4.7,
    "totalRatings": 23,
    "isProfileComplete": true,
    "isProfileCompleteStep1": true,
    "isProfileCompleteStep2": true
  }
}
```

---

#### `PUT /api/choreographer/profile`
Update any profile fields.

**Request (any combination):**
```json
{
  "name": "Ananya Sharma",
  "bio": "Updated bio",
  "experience": "7 years",
  "city": "Pune",
  "phone": "+91 9876543210",
  "danceStyles": ["Contemporary", "Kathak"],
  "socialLinks": [
    { "platform": "instagram", "url": "https://instagram.com/ananya" },
    { "platform": "youtube", "url": "https://youtube.com/@ananya" }
  ]
}
```

---

#### `PUT /api/choreographer/profile/step1`
Onboarding step 1.

```json
{
  "name": "Ananya Sharma",
  "bio": "Contemporary specialist",
  "experience": "6+ years",
  "city": "Mumbai",
  "phone": "+91 9876543210"
}
```

---

#### `PUT /api/choreographer/profile/step2`
Onboarding step 2.

```json
{
  "danceStyles": ["Contemporary", "Bharatanatyam", "Hip Hop"],
  "socialLinks": [
    { "platform": "instagram", "url": "https://instagram.com/ananya" },
    { "platform": "youtube",   "url": "https://youtube.com/@ananya" },
    { "platform": "website",   "url": "https://ananya.dance" }
  ]
}
```

**Allowed platforms:** `instagram` `youtube` `facebook` `twitter` `website` `other`

---

#### `POST /api/choreographer/profile/avatar`
Upload profile image. Form field = `avatar`.

#### `POST /api/choreographer/profile/banner`
Upload banner. Form field = `banner`.

---

### Dashboard

#### `GET /api/choreographer/dashboard`

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "totalWorkshops": 8,
    "totalStudents": 147,
    "totalRevenue": 73500,
    "activeWorkshops": 3,
    "completedWorkshops": 4,
    "rating": 4.7,
    "totalRatings": 23,
    "isProfileComplete": true,
    "upcomingWorkshops": [
      {
        "_id": "661...",
        "title": "Contemporary Beginner Batch",
        "danceStyle": "Contemporary",
        "startDate": "2026-04-01T00:00:00.000Z",
        "price": 1500,
        "capacity": 15,
        "attendees": 8,
        "status": "Upcoming"
      }
    ]
  }
}
```

---

#### `GET /api/choreographer/upcoming`
Quick upcoming list. Query: `?limit=5`

---

### Workshops

#### `GET /api/choreographer/workshops`
Own workshops.

| Query | Description |
|-------|-------------|
| `status` | Filter: [Upcoming](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#260-273), `Ongoing`, `Completed`, `Confirmed`, `Cancelled` |
| `page` / `limit` | Pagination |

#### `POST /api/choreographer/workshops`
Create workshop. Same fields as `/api/workshops` POST.

#### `GET /api/choreographer/workshops/:id`
Single workshop with enrolled students.

#### `PUT /api/choreographer/workshops/:id`
Update workshop.

#### `DELETE /api/choreographer/workshops/:id`
Soft-delete.

#### `POST /api/choreographer/workshops/:id/thumbnail`
Upload thumbnail. Form field = `thumbnail`.  
Also accepts: `POST /api/choreographer/workshops/:id/image` (field = `image`)

---

### Students

#### `GET /api/choreographer/students`
All unique students across all workshops.

| Query | Description |
|-------|-------------|
| [search](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#441-475) | Filter by name, email, or city |

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "_id": "...",
      "name": "Priya Patel",
      "email": "priya@example.com",
      "profileImage": "https://...",
      "city": "Mumbai",
      "phone": "+91 ...",
      "totalEnrolled": 3
    }
  ]
}
```

---

#### `GET /api/choreographer/workshops/:id/students`
Students for a specific workshop.

---

### Reviews

#### `GET /api/choreographer/reviews`

| Query | Default |
|-------|---------|
| `page` | 1 |
| `limit` | 10 |

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "reviews": [
      {
        "_id": "...",
        "rating": 5,
        "comment": "Amazing instructor! Loved the class.",
        "reviewer": { "name": "Priya Patel", "profileImage": "https://..." },
        "workshop": { "title": "Contemporary Workshop", "danceStyle": "Contemporary" },
        "createdAt": "2026-03-20T10:30:00.000Z"
      }
    ],
    "stats": {
      "avgRating": 4.7,
      "totalRatings": 23,
      "distribution": { "5": 15, "4": 6, "3": 2, "2": 0, "1": 0 }
    },
    "pagination": { "total": 23, "page": 1, "limit": 10, "pages": 3 }
  }
}
```

---

### Notifications

#### `GET /api/choreographer/notifications`

| Query | Description |
|-------|-------------|
| `unreadOnly=true` | Only unread notifications |
| `page` / `limit` | Pagination (default limit: 20) |

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "_id": "...",
        "type": "booking_confirmed",
        "title": "New Booking",
        "message": "Priya Patel booked 'Contemporary Workshop'.",
        "isRead": false,
        "resourceType": "Booking",
        "resourceId": "...",
        "sender": { "name": "Priya Patel", "profileImage": "https://..." },
        "createdAt": "2026-03-23T10:30:00.000Z"
      }
    ],
    "unreadCount": 3,
    "pagination": { "total": 12, "page": 1, "limit": 20, "pages": 1 }
  }
}
```

---

#### `PATCH /api/choreographer/notifications/read-all`
Mark all (or specific) notifications as read.

**Request (optional):**
```json
{ "ids": ["661aaa...", "661bbb..."] }
```
Omit `ids` to mark ALL as read.

---

## 10. Studio Panel APIs

Base: `/api/studio-panel` · **Auth: studio role required**

---

### Profile

#### `POST /api/studio-panel/profile`
Create profile (first setup). Idempotent.

```json
{
  "studioName": "Mumbai Dance Studio",
  "city": "Mumbai",
  "contact": { "phone": "+91 9876543210", "email": "info@..." }
}
```

---

#### `GET /api/studio-panel/profile`
Full profile with choreographer roster.

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "_id": "661...",
    "studioName": "Mumbai Dance Studio",
    "description": "Premier dance studio in South Mumbai",
    "address": "123 Marine Drive, Mumbai",
    "city": "Mumbai",
    "facilities": ["AC Hall", "Dressing Room", "Parking", "Sound System"],
    "danceStyles": ["Contemporary", "Bollywood", "Hip Hop"],
    "images": [
      { "_id": "...", "url": "https://...", "caption": "Main Hall" }
    ],
    "contact": {
      "phone": "+91 9876543210",
      "email": "info@...",
      "website": "https://mumbaidance.com",
      "whatsapp": "+91 9876543210"
    },
    "socialLinks": [
      { "platform": "instagram", "url": "https://instagram.com/..." }
    ],
    "choreographers": [
      {
        "_id": "...",
        "name": "Ananya Sharma",
        "profileImage": "https://...",
        "danceStyles": ["Contemporary"],
        "city": "Mumbai",
        "rating": 4.8
      }
    ],
    "profileImage": "https://...",
    "bannerImage": "https://...",
    "rating": 4.6,
    "totalRatings": 31,
    "isProfileComplete": true
  }
}
```

---

#### `PUT /api/studio-panel/profile`
Update any profile fields.

```json
{
  "studioName": "Mumbai Dance Studio",
  "description": "Updated description",
  "facilities": ["AC Hall", "Dressing Room", "Parking"],
  "contact": { "phone": "+91 ...", "email": "new@..." }
}
```

---

#### `PUT /api/studio-panel/profile/step1`
Onboarding step 1.

```json
{
  "studioName": "Mumbai Dance Studio",
  "address": "123 Marine Drive, Mumbai",
  "city": "Mumbai",
  "contact": {
    "phone": "+91 9876543210",
    "email": "info@mumbaidance.com",
    "website": "https://mumbaidance.com",
    "whatsapp": "+91 9876543210"
  }
}
```

---

#### `PUT /api/studio-panel/profile/step2`
Onboarding step 2.

```json
{
  "danceStyles": ["Contemporary", "Bollywood", "Hip Hop", "Kathak"],
  "facilities": ["AC Hall", "Dressing Room", "Parking", "Sound System", "Mirror Wall"],
  "description": "Premier dance studio offering professional classes.",
  "socialLinks": [
    { "platform": "instagram", "url": "https://instagram.com/..." }
  ]
}
```

---

#### `POST /api/studio-panel/profile/logo`
Upload studio logo. Field = `logo`. (Also: `avatar` field accepted for compat)

#### `POST /api/studio-panel/profile/banner`
Upload banner. Field = `banner`.

---

#### `POST /api/studio-panel/profile/images`
Upload gallery images (up to 10).

```js
// Multipart form
formData.append('images', file1);
formData.append('images', file2);
formData.append('captions', JSON.stringify(['Main Hall', 'Stage']));
```

**Response `200`:**
```json
{
  "success": true,
  "message": "2 image(s) added to gallery",
  "data": {
    "addedImages": [
      { "url": "https://...", "caption": "Main Hall" },
      { "url": "https://...", "caption": "Stage" }
    ],
    "totalImages": 5
  }
}
```

---

#### `DELETE /api/studio-panel/profile/images/:imageId`
Remove a gallery image by its MongoDB subdocument `_id`.

---

### Dashboard

#### `GET /api/studio-panel/dashboard`

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "totalClasses": 12,
    "totalStudents": 284,
    "totalRevenue": 142000,
    "activeClasses": 5,
    "completedClasses": 6,
    "totalTrainers": 3,
    "rating": 4.6,
    "totalRatings": 31,
    "isProfileComplete": true,
    "upcomingClasses": [
      {
        "_id": "...",
        "className": "Hip Hop Batch 4",
        "danceStyle": "Hip Hop",
        "startDate": "2026-04-05T00:00:00.000Z",
        "trainerRef": { "name": "Ananya Sharma", "profileImage": "..." },
        "capacity": 20,
        "attendees": 11
      }
    ]
  }
}
```

#### `GET /api/studio-panel/upcoming`
Query: `?limit=5`

---

### Classes (Batches)

#### `GET /api/studio-panel/classes`
List all studio classes.

| Query | Description |
|-------|-------------|
| `status` | Filter by status |
| `trainerId` | Filter by trainer (ChoreographerProfile._id) |
| `page` / `limit` | Pagination |

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "classes": [
      {
        "_id": "...",
        "className": "Hip Hop Batch 4",
        "danceStyle": "Hip Hop",
        "level": "Beginner",
        "startDate": "2026-04-05T00:00:00.000Z",
        "timings": { "startTime": "18:00", "endTime": "19:30", "days": "Tue, Thu", "duration": "90 min" },
        "price": 1200,
        "capacity": 20,
        "attendees": 11,
        "isFull": false,
        "trainerRef": { "name": "Ananya Sharma", "profileImage": "...", "rating": 4.8 },
        "thumbnail": "https://...",
        "status": "Upcoming"
      }
    ],
    "pagination": { "total": 12, "page": 1, "limit": 10, "pages": 2 }
  }
}
```

---

#### `POST /api/studio-panel/classes`
Create a class.

```json
{
  "className": "Hip Hop Batch 4",
  "danceStyle": "Hip Hop",
  "level": "Beginner",
  "description": "Beginner-friendly Hip Hop batch.",
  "trainerRef": "661choreographerProfileId",
  "startDate": "2026-04-05",
  "endDate": "2026-05-31",
  "timings": {
    "startTime": "18:00",
    "endTime": "19:30",
    "days": "Tue, Thu",
    "duration": "90 min"
  },
  "price": 1200,
  "priceType": "Monthly",
  "capacity": 20,
  "type": "Offline",
  "city": "Mumbai",
  "ageGroup": "15 to 35",
  "status": "Upcoming"
}
```

**Field aliases (all accepted):**

| Studio panel field | Mobile app alias |
|-------------------|-----------------|
| `className` | `title` or `name` |
| `danceStyle` | `style` |
| `thumbnail` | `image` |
| `trainerRef` | `trainerId` |

---

#### `GET /api/studio-panel/classes/:id`
Single class with full trainer + enrolled students.

#### `PUT /api/studio-panel/classes/:id`
Update class.

#### `DELETE /api/studio-panel/classes/:id`
Soft-delete.

#### `POST /api/studio-panel/classes/:id/thumbnail`
Upload class thumbnail. Field = `thumbnail` or `image`.

---

### Trainer Assignment

#### `PUT /api/studio-panel/classes/:id/trainer`
Assign a choreographer as trainer for a class.

```json
{ "choreographerProfileId": "661..." }
```

**Response `200`:** Updated class with populated trainer.

---

#### `DELETE /api/studio-panel/classes/:id/trainer`
Remove trainer from a class.

---

#### `GET /api/studio-panel/choreographers`
Studio's own trainer roster.

---

#### `GET /api/studio-panel/choreographers/search`
Search ALL platform choreographers to assign.

| Query | Description |
|-------|-------------|
| [search](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#441-475) | Name or bio search |
| `danceStyle` | Filter by dance style |
| `city` | Filter by city |
| `page` / `limit` | Max 50/page |

**Response `200`:**
```json
{
  "data": {
    "choreographers": [
      {
        "_id": "661...",
        "name": "Ananya Sharma",
        "profileImage": "https://...",
        "danceStyles": ["Contemporary"],
        "city": "Mumbai",
        "experience": "6 years",
        "rating": 4.8,
        "totalRatings": 23
      }
    ],
    "pagination": { "total": 14, "page": 1, "limit": 10, "pages": 2 }
  }
}
```

---

### Students

#### `GET /api/studio-panel/students`
All unique students across **all** studio classes.

| Query | Description |
|-------|-------------|
| [search](file:///c:/Users/royal/OneDrive/Desktop/onthefloorbackend/src/services/studio.service.js#441-475) | Name, email, or city |

#### `GET /api/studio-panel/classes/:id/students`
Students in a specific class.

---

### Reviews

#### `GET /api/studio-panel/reviews`
Same shape as choreographer reviews (see §9).

---

### Notifications

#### `GET /api/studio-panel/notifications`
Same shape as choreographer notifications (see §9).

#### `PATCH /api/studio-panel/notifications/read-all`
Same as choreographer version.

---

### Backward-Compatible Aliases

All `/api/studio-panel/classes` endpoints are also available as `/api/studio-panel/workshops` for mobile apps already using that path.

---

## 11. Booking APIs

Base: `/api/bookings` · **Auth: required**

### `POST /api/bookings`
Book a class. **Role: user**

**Request:**
```json
{
  "workshopId": "661abc...",
  "paymentMethod": "online",
  "notes": "Prefer morning slots if rescheduled"
}
```

**Response `201`:**
```json
{
  "success": true,
  "message": "Booking confirmed successfully",
  "data": {
    "_id": "661booking...",
    "user": "661user...",
    "workshop": "661workshop...",
    "status": "confirmed",
    "paymentStatus": "pending",
    "bookingDate": "2026-03-23T12:00:00.000Z"
  }
}
```

---

### `GET /api/bookings/my`
All my bookings (as a student). Paginated.

**Query:** `?status=confirmed&page=1&limit=10`

---

### `GET /api/bookings/:id`
Get single booking detail.

---

### `PATCH /api/bookings/:id/cancel`
Cancel a booking.

```json
{ "reason": "Schedule conflict" }
```

---

### `GET /api/bookings/workshop/:workshopId`
All bookings for a workshop. **Auth: workshop owner only**

---

## 12. Review APIs

Base: `/api/reviews`

### `POST /api/reviews`
Submit a review. **Auth: user · Requires a confirmed booking**

**Request:**
```json
{
  "workshopId": "661abc...",
  "rating": 5,
  "comment": "Amazing instructor! Loved the class."
}
```

> Reviews auto-update the `rating` and `totalRatings` on the choreographer/studio profile.

---

### `GET /api/reviews/workshop/:workshopId`
All reviews for a specific workshop. **Public**

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "reviews": [...],
    "stats": {
      "avgRating": 4.7,
      "totalRatings": 12,
      "distribution": { "5": 8, "4": 3, "3": 1, "2": 0, "1": 0 }
    },
    "pagination": { ... }
  }
}
```

---

### `GET /api/reviews/choreographer/:choreographerId`
All reviews for a choreographer. **Public**

### `GET /api/reviews/studio/:studioId`
All reviews for a studio. **Public**

### `PUT /api/reviews/:id`
Update own review. **Auth: reviewer only**

### `DELETE /api/reviews/:id`
Delete own review. **Auth: reviewer only**

---

## 13. Notification APIs

Base: `/api/notifications` · **Auth: required**

### `GET /api/notifications`
Current user's notification list.

| Query | Description |
|-------|-------------|
| `unreadOnly=true` | Only unread |
| `page` / `limit` | Pagination (default limit: 20) |

---

### `PATCH /api/notifications/:id/read`
Mark a single notification as read.

### `PATCH /api/notifications/read-all`
Mark all as read. Optional body: `{ "ids": ["..."] }`

### `DELETE /api/notifications/:id`
Delete a single notification.

---

## 14. Role System Quick Reference

| Route prefix | Required role | Who uses it |
|---|---|---|
| `/api/auth/*` | None | Everyone |
| `/api/workshops` (GET) | None | Everyone (public) |
| `/api/workshops` (POST/PUT/DELETE) | `choreographer` or `studio` | Instructors |
| `/api/users/me` | Any authenticated | Logged-in user |
| `/api/choreographer/*` | `choreographer` | Choreographers only |
| `/api/studio-panel/*` | `studio` | Studios only |
| `/api/bookings` | `user` (create) | Students |
| `/api/reviews` | Any authenticated | All roles |
| `/api/notifications` | Any authenticated | All roles |

---

## 15. React / React Native Integration

### Axios base client (recommended)

```js
// src/api/client.js
import axios from 'axios';
import { getAuth } from 'firebase/auth';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000/api',
  timeout: 15000,
});

// Attach Firebase token to every request automatically
api.interceptors.request.use(async (config) => {
  const auth = getAuth();
  const user = auth.currentUser;
  if (user) {
    const token = await user.getIdToken();
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Global error handling
api.interceptors.response.use(
  (res) => res.data,  // Unwrap — returns { success, message, data } directly
  (err) => {
    const message = err.response?.data?.message || 'Something went wrong';
    const status  = err.response?.status;

    if (status === 401) {
      // Token expired or invalid — redirect to login
      window.location.href = '/login';
    }

    return Promise.reject({ message, status, errors: err.response?.data?.errors });
  }
);

export default api;
```

---

### First-time Firebase login (auto-provision)

```js
// src/hooks/useFirebaseLogin.js
import { signInWithPopup, GoogleAuthProvider, getAuth } from 'firebase/auth';
import api from '../api/client';

export const loginWithGoogle = async (role = 'user') => {
  const auth     = getAuth();
  const provider = new GoogleAuthProvider();

  const result = await signInWithPopup(auth, provider);
  const token  = await result.user.getIdToken();

  // Sync with backend — pass x-user-role on first login
  const response = await api.post('/auth/firebase-sync', {}, {
    headers: {
      Authorization: `Bearer ${token}`,
      'x-user-role': role,  // 'choreographer' | 'studio' | 'user'
    },
  });

  return response.data;  // { _id, name, email, role, isNewUser }
};
```

---

### Fetch workshops with search

```js
// src/api/workshops.js
import api from './client';

export const getWorkshops = (params = {}) =>
  api.get('/workshops', { params });

// Usage:
const { data } = await getWorkshops({
  search: 'hip hop',
  city: 'Mumbai',
  level: 'Beginner',
  minPrice: 500,
  maxPrice: 2000,
  sortBy: 'price_asc',
  page: 1,
  limit: 12,
});
// data.workshops, data.pagination
```

---

### File upload helper

```js
// src/api/upload.js
import api from './client';

export const uploadAvatar = (file) => {
  const formData = new FormData();
  formData.append('avatar', file);
  return api.post('/choreographer/profile/avatar', formData);
};

export const uploadGalleryImages = (files, captions = []) => {
  const formData = new FormData();
  files.forEach(f => formData.append('images', f));
  if (captions.length) {
    formData.append('captions', JSON.stringify(captions));
  }
  return api.post('/studio-panel/profile/images', formData);
};
```

---

### Choreographer Panel — complete API module

```js
// src/api/choreographerPanel.js
import api from './client';

const base = '/choreographer';

export const choreoApi = {
  // Profile
  createProfile: (data) => api.post(`${base}/profile`, data),
  getProfile:    ()     => api.get(`${base}/profile`),
  updateProfile: (data) => api.put(`${base}/profile`, data),
  step1:         (data) => api.put(`${base}/profile/step1`, data),
  step2:         (data) => api.put(`${base}/profile/step2`, data),

  // Dashboard
  getDashboard:  ()           => api.get(`${base}/dashboard`),
  getUpcoming:   (limit = 5)  => api.get(`${base}/upcoming`, { params: { limit } }),

  // Workshops
  getWorkshops:  (params)     => api.get(`${base}/workshops`, { params }),
  createWorkshop:(data)       => api.post(`${base}/workshops`, data),
  getWorkshop:   (id)         => api.get(`${base}/workshops/${id}`),
  updateWorkshop:(id, data)   => api.put(`${base}/workshops/${id}`, data),
  deleteWorkshop:(id)         => api.delete(`${base}/workshops/${id}`),

  // Students
  getStudents:       (params) => api.get(`${base}/students`, { params }),
  getWorkshopStudents:(id)    => api.get(`${base}/workshops/${id}/students`),

  // Reviews
  getReviews: (params) => api.get(`${base}/reviews`, { params }),

  // Notifications
  getNotifications: (params) => api.get(`${base}/notifications`, { params }),
  markRead:  (ids)  => api.patch(`${base}/notifications/read-all`, { ids }),
  markAllRead: ()   => api.patch(`${base}/notifications/read-all`),
};
```

---

### Studio Panel — complete API module

```js
// src/api/studioPanel.js
import api from './client';

const base = '/studio-panel';

export const studioApi = {
  // Profile
  createProfile: (data) => api.post(`${base}/profile`, data),
  getProfile:    ()     => api.get(`${base}/profile`),
  updateProfile: (data) => api.put(`${base}/profile`, data),
  step1:         (data) => api.put(`${base}/profile/step1`, data),
  step2:         (data) => api.put(`${base}/profile/step2`, data),
  removeGalleryImage: (imageId) => api.delete(`${base}/profile/images/${imageId}`),

  // Dashboard
  getDashboard: ()          => api.get(`${base}/dashboard`),
  getUpcoming:  (limit = 5) => api.get(`${base}/upcoming`, { params: { limit } }),

  // Classes
  getClasses:  (params)   => api.get(`${base}/classes`, { params }),
  createClass: (data)     => api.post(`${base}/classes`, data),
  getClass:    (id)       => api.get(`${base}/classes/${id}`),
  updateClass: (id, data) => api.put(`${base}/classes/${id}`, data),
  deleteClass: (id)       => api.delete(`${base}/classes/${id}`),

  // Trainer assignment
  assignTrainer:   (classId, choreographerProfileId) =>
    api.put(`${base}/classes/${classId}/trainer`, { choreographerProfileId }),
  unassignTrainer: (classId) =>
    api.delete(`${base}/classes/${classId}/trainer`),
  getChoreographers:       ()       => api.get(`${base}/choreographers`),
  searchChoreographers:    (params) => api.get(`${base}/choreographers/search`, { params }),

  // Students
  getStudents:      (params) => api.get(`${base}/students`, { params }),
  getClassStudents: (id)     => api.get(`${base}/classes/${id}/students`),

  // Reviews
  getReviews: (params) => api.get(`${base}/reviews`, { params }),

  // Notifications
  getNotifications: (params) => api.get(`${base}/notifications`, { params }),
  markAllRead:      ()       => api.patch(`${base}/notifications/read-all`),
};
```

---

### Error handling in components

```jsx
// src/components/CreateWorkshop.jsx
const handleSubmit = async (formData) => {
  try {
    await choreoApi.createWorkshop(formData);
    showToast('Workshop created!', 'success');
  } catch (err) {
    if (err.errors) {
      // Field-level validation errors from Zod
      err.errors.forEach(({ field, message }) => {
        setFieldError(field, message);
      });
    } else {
      showToast(err.message || 'Something went wrong', 'error');
    }
  }
};
```

---

## Appendix: Complete Endpoint Map

```
GET    /api/health                                     → Health check (public)

── AUTH ──────────────────────────────────────────────────────────────
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/firebase-sync
POST   /api/auth/refresh
POST   /api/auth/logout
POST   /api/auth/forgot-password

── USERS ─────────────────────────────────────────────────────────────
GET    /api/users/me
PUT    /api/users/me
POST   /api/users/me/avatar
GET    /api/users/:id
POST   /api/users/me/save-workshop/:workshopId
DELETE /api/users/me/save-workshop/:workshopId
GET    /api/users/me/saved-workshops
POST   /api/users/me/follow/:choreographerId
DELETE /api/users/me/follow/:choreographerId

── WORKSHOPS (PUBLIC) ────────────────────────────────────────────────
GET    /api/workshops                                  → Search + filter + paginate
GET    /api/workshops/:id
POST   /api/workshops                                  → (auth: choreo/studio)
PUT    /api/workshops/:id                              → (auth: owner)
DELETE /api/workshops/:id                              → (auth: owner)
POST   /api/workshops/:id/image                        → (auth: owner)
GET    /api/workshops/:id/students                     → (auth: owner)

── CHOREOGRAPHER PANEL ───────────────────────────────────────────────
POST   /api/choreographer/profile
GET    /api/choreographer/profile
PUT    /api/choreographer/profile
PUT    /api/choreographer/profile/step1
PUT    /api/choreographer/profile/step2
POST   /api/choreographer/profile/avatar
POST   /api/choreographer/profile/banner
GET    /api/choreographer/dashboard
GET    /api/choreographer/upcoming
GET    /api/choreographer/workshops
POST   /api/choreographer/workshops
GET    /api/choreographer/workshops/:id
PUT    /api/choreographer/workshops/:id
DELETE /api/choreographer/workshops/:id
POST   /api/choreographer/workshops/:id/thumbnail
POST   /api/choreographer/workshops/:id/image
GET    /api/choreographer/workshops/:id/students
GET    /api/choreographer/students
GET    /api/choreographer/reviews
GET    /api/choreographer/notifications
PATCH  /api/choreographer/notifications/read-all

── STUDIO PANEL ──────────────────────────────────────────────────────
POST   /api/studio-panel/profile
GET    /api/studio-panel/profile
PUT    /api/studio-panel/profile
PUT    /api/studio-panel/profile/step1
PUT    /api/studio-panel/profile/step2
POST   /api/studio-panel/profile/logo
POST   /api/studio-panel/profile/avatar
POST   /api/studio-panel/profile/banner
POST   /api/studio-panel/profile/images              → Gallery upload (multi)
DELETE /api/studio-panel/profile/images/:imageId
GET    /api/studio-panel/dashboard
GET    /api/studio-panel/upcoming
GET    /api/studio-panel/classes
POST   /api/studio-panel/classes
GET    /api/studio-panel/classes/:id
PUT    /api/studio-panel/classes/:id
DELETE /api/studio-panel/classes/:id
POST   /api/studio-panel/classes/:id/thumbnail
POST   /api/studio-panel/classes/:id/image
GET    /api/studio-panel/classes/:id/students
PUT    /api/studio-panel/classes/:id/trainer
DELETE /api/studio-panel/classes/:id/trainer
GET    /api/studio-panel/choreographers
GET    /api/studio-panel/choreographers/search
GET    /api/studio-panel/students
GET    /api/studio-panel/reviews
GET    /api/studio-panel/notifications
PATCH  /api/studio-panel/notifications/read-all
── (Backward compat aliases: /api/studio-panel/workshops/*) ──────────

── BOOKINGS ──────────────────────────────────────────────────────────
POST   /api/bookings
GET    /api/bookings/my
GET    /api/bookings/:id
PATCH  /api/bookings/:id/cancel
GET    /api/bookings/workshop/:workshopId             → (auth: owner)

── REVIEWS ───────────────────────────────────────────────────────────
POST   /api/reviews
GET    /api/reviews/workshop/:workshopId
GET    /api/reviews/choreographer/:choreographerId
GET    /api/reviews/studio/:studioId
PUT    /api/reviews/:id
DELETE /api/reviews/:id

── NOTIFICATIONS ─────────────────────────────────────────────────────
GET    /api/notifications
PATCH  /api/notifications/:id/read
PATCH  /api/notifications/read-all
DELETE /api/notifications/:id
```

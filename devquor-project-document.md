# Devgram / Devquor Project Documentation

## 1. Introduction

Devgram (also referenced as Devquor in the repository) is a full-stack social platform for developers. It combines social posting, threaded commenting, messaging, groups/clusters, and real-time notification support. The solution is built with a Node.js/Express backend, PostgreSQL database, and a React/Vite frontend with optional Electron packaging.

## 2. Purpose and Use Cases

### Primary Use Cases
- Developer community sharing: users can upload developer posts and thoughts.
- Social interaction: likes, comments, threaded replies, and bookmarks.
- Group collaboration: create and join clusters for group discussions.
- Real-time messaging: one-to-one chat and cluster chat.
- Notification support: push notifications using Firebase tokens.

### Typical Users
- Developers sharing code-related media or ideas.
- Community members interacting around technical content.
- Group creators organizing topic-based clusters.
- Users needing private conversation and collaboration.

## 3. Industry Value

### Why this project matters
- Social collaboration for developers is a strong niche.
- It supports technical community-building and idea exchange.
- Real-time features encourage engagement and frequent usage.
- Cross-platform potential with Electron packaging and a PWA-ready frontend.

### Business benefits
- Better developer networking and team collaboration.
- Potential for monetization through premium features or sponsored groups.
- Useful for technical communities, bootcamps, study groups, and knowledge sharing.

## 4. Roles

### Supported roles
- Registered User
  - Can login, interact with posts, comment, like, bookmark, and message others.
- Author
  - Every user can post uploads, comment, and moderate their own content.
- Cluster Owner / Superadmin
  - The cluster creator becomes a superadmin of that cluster.
- Authenticated User
  - Many APIs require authentication, usually via JWT stored in cookie/header.

## 5. Technology Stack

### Backend
- `Node.js` + `Express`
  - Core runtime and HTTP server framework.
- `PostgreSQL`
  - Primary relational database accessed through `pg`.
- `cloudinary`
  - Image/media asset storage and deletion support.
- `jsonwebtoken`
  - JWT authentication for protected API routes.
- `bcryptjs`
  - Password hashing and verification.
- `firebase-admin` and `firebase`
  - Notification and push token support.
- `socket.io`
  - Realtime socket foundation (backend side initialized in `backend/config/SocketConnection.js`).
- `dotenv`
  - Environment variable configuration.
- `cookie-parser`
  - Cookie support for auth token retrieval.

### Frontend
- `React`
  - UI framework for component-based web app.
- `Vite`
  - Fast build and development tool.
- `Tailwind CSS`
  - Utility-first CSS framework for UI styling.
- `Axios`
  - HTTP client for API calls.
- `React Router`
  - Page navigation and route management.
- `Firebase Messaging`
  - Web push notification token management.
- `Socket.IO Client`
  - Real-time connection support from frontend.
- `Electron`
  - Desktop application packaging support.

### Dev / Build Tools
- `ESLint`
  - JavaScript linting and quality enforcement.
- `electron-builder`
  - Desktop app packaging for Windows, macOS, and Linux.
- `vite-plugin-pwa`
  - Progressive Web App capability.

## 6. Architecture Overview

### Backend architecture
- Entry point: `backend/server.js`
- Route groups:
  - `backend/router/user.router.js`
  - `backend/router/upload.router.js`
  - `backend/router/cluster.route.js`
- Controllers implement business logic in:
  - `backend/controller/user.controller.js`
  - `backend/controller/upload.controller.js`
  - `backend/controller/cluster.controller.js`
- Auth middleware:
  - `backend/middleware/auth.middleware.js`
- Database connection:
  - `backend/config/sqlDb.js`

### Frontend architecture
- React app root: `frontend/src/main.jsx`
- Core app shell: `frontend/src/App.jsx`
- Pages and features located in:
  - `frontend/src/pages/`
  - `frontend/src/components/`
- Notification helpers:
  - `frontend/src/firebase/notificationHelper.js`
- WebSocket wrapper:
  - `frontend/src/Funtions/WebSocket.js`
- Electron support:
  - `frontend/electron/main.cjs`
  - `frontend/electron/preload.cjs`
  - `frontend/electron/utils.cjs`

## 7. Functionalities

### Authentication
- Register user
- Login and set JWT cookie
- Logout and clear cookie
- Email verification code generation
- Password reset flow

### User Management
- Get authenticated user profile
- Update profile info
- List users with pagination
- Get another user’s details

### Uploads and Social Interaction
- Create upload post
- Update / modify upload Detail ( thought )
- Fetch all uploads with pagination
- Fetch author-specific uploads
- Get detailed upload with comments and stats
- Like/unlike upload
- Bookmark upload
- Post comments (“thoughts”) on uploads
- Like/unlike thoughts
- Add sub-thoughts (threaded replies)
- Like sub-thoughts

### Notifications
- Save Firebase notification token
- Fetch notifications for a user

### Cluster / Group Features
- Create a cluster
- Join a cluster
- Fetch clusters the user is part of
- Get cluster details
- Update cluster details
- Delete cluster image in Cloudinary

### Messaging
- Get person-to-person conversation list
- Send direct message to another user
- Fetch direct messages with pagination
- Send and fetch cluster messages

## 8. Route Reference

### Global Health Routes
| Route | Method | Auth | Body / Query | Response |
|---|---|---|---|---|
| `/` | GET | No | none | Returns "Hello from Express!" |
| `/data` | POST | No | JSON body | Echos back received `data` |
| `/ping` | GET | No | none | Returns "Connected!" if DB healthy |
| `/dbinfo` | GET | No | none | Returns database schema info |

### User Routes (`/api/user`)
| Route | Method | Auth | Request | Success Response | Errors |
|---|---|---|---|---|---|
| `/userRegister` | POST | No | `{username, email, password, firstname?, lastname?, bio?, gender?, profile_id?, date_of_birth?}` | `201 { token, message:"User created", userId, user }` | `500` on failure |
| `/userLogin` | POST | No | `{email, password, username}` | `200 { message:"Login successful", token, user }` | `400` invalid credentials, `500` |
| `/userLogout` | POST | No | none | `200 { msg:"Logged out successfully" }` | `400` logout failure |
| `/userVerificationCode/email` | POST | No | `{ email }` | `200 { message:"Verification email sent successfully", success:true }` | `400` email not registered, `500` |
| `/userPasswordReset/:genPassCode` | POST | No | `{ email, emailCode }` | `200 { message:"Code verified", success:true, urlLink }` | `400` invalid/expired code, `500` |
| `/userPasswordChange/:genPassCode` | POST | No | `{ email, newPassword }` | `200 { message:"Password reset successful", success:true }` | `400` user not found, `500` |
| `/saveNotificationToken` | POST | Yes | `{ userId, token }` | `200 { success:true, message:"Token saved successfully"}` | `400` missing fields, `500` |
| `/saveNotificationToken` | GET | Yes | none | `200 { message:"fetched all the notification", success:true, notifications }` | `400` missing token/userId, `500` |
| `/getAuthorDetails` | GET | Yes | none | `200 { message:"user detail fetch !", success:true, User, uploads }` | `400` user not found, `500` |
| `/updateAuthor` | PUT | Yes | `{ firstname?, lastname?, bio?, gender?, date_of_birth?, profile_id?, profile_pic?, oldProfileId? }` | `200 { message:"User updated successfully" }` | `400` no fields, `404` user not found, `500` |
| `/getAllUsers` | GET | Yes | `?page=` | `200 { page, users }` | `500` |
| `/getUserDetail/:userid` | GET | Yes | none | `201 { message:"user found!", success:true, User }` | `400` user not found, `500` |

### Upload Routes (`/api/upload`)
| Route | Method | Auth | Request | Success Response | Errors |
|---|---|---|---|---|---|
| `/createUpload` | POST | Yes | `{ uploads_url, uploads_id, thought }` | `200 { message:"Upload created successfully", uploadId, upload }` | `400` missing fields, user not found, `500` |
| `/deleteUploads/:authorId/:uploadId` | DELETE | Yes | `{ uploadId, cloudinaryId }` | `200 { message:"Upload deleted successfully", deletedUploadId }` | `400` unauthorized, deletion failed, `404` not found, `500` |
| `/getAuthorUploads` | GET | Yes | `?page=` | `200 { message:"author uploads has been fetched!", success:true, uplaods }` | `400` none found, `500` |
| `/getAllUploads` | GET | No | `?page=` | `200 { message:"uploads fetched!", success:true, allUploads }` | `400` no uploads, `500` |
| `/getUploadDetail/:uploadId` | GET | Yes | `?page=&limit=` | `200 { message:"Upload detail found successfully!", success:true, uploadDetail }` | `400` invalid request, `404` not found, `500` |
| `/likeOrDislikeUpload/:uploadId/:token` | POST | Yes | none | `200 { message:"Like added successfully!", success:true, like:true }` or `200 { message:"Like removed successfully!", success:true, like:false }` | `400` invalid request, `404` not found, `500` |
| `/uploadBookmarks/:uploadId` | POST | Yes | none | `200 { message:"Bookmark added successfully!", success:true }` or `200 { message:"Bookmark removed successfully!", success:true }` | `400` invalid upload, `500` |
| `/thoughtOnUploads/:uploadId` | POST | Yes | `{ thought }` | `200 { message:"thoughted on upload successfully!", success:true, thought }` | `400` invalid upload, `500` |
| `/likeOrDislikeThought/:thoughtId` | POST | Yes | none | `200 { message:"Thought liked successfully!", success:true }` or `200 { message:"Like removed from thought successfully!", success:true }` | `500` |
| `/subThoughtOnThought/:thoughtId` | POST | Yes | `{ thought }` | `200 { message:"tought uplaoded !", sucess:true }` | `400` missing thought, `500` |
| `/subThoughtLikeNDislike/:subThoughtId` | POST | Yes | none | `200 { message:"thought liked", success:true }` | `400` not found, `500` |

### Cluster Routes (`/api/cluster`)
| Route | Method | Auth | Request | Success Response | Errors |
|---|---|---|---|---|---|
| `/createCluster` | POST | Yes | `{ groupName, groupDescription }` | `201 { success:true, message:"Cluster created successfully", cluster }` | `400` validation, `404` user not found, `500` |
| `/joinCluster` | POST | Yes | `{ clusterId, c_id }` | `201 { success:true, message:"Joined cluster successfully", member, clusterDetail }` | `400` already joined / cannot join, `404` cluster not found, `500` |
| `/getAuthorMsgedClusters` | GET | Yes | none | `200 { message:"cluster found!", sucess:true, clusters }` | `400` none found, `500` |
| `/getClusterDetail` | POST | Yes | `{ cluster_id, c_id }` plus `?page=&limit=` | `200 { success:true, message:"cluster Detail Found", clusterDetail, getClusterMembers, messages, countResult }` | `400` no cluster/user found, `500` |
| `/editClusterDetail` | PUT | Yes | `{ cluster_id, groupDescription, profile_pic, profile_id, oldPublicId }` | `200 { success:true, message:"Cluster updated successfully!", cluster }` | `400` no changes, `404` not found, `500` |
| `/sendClusterMessage/:cluster_id` | POST | Yes | `{ message, sender_name }` | `201 { success:true, message:"Message sent successfully" }` | `404` not found, `500` |
| `/getClusterMessage/:cluster_id` | GET | Yes | none | `200 { success:true, message:"Cluster messages found!", messages }` | `500` |
| `/getUserInPerson` | GET | Yes | none | `200 { success:true, message:"Messages found!", messages }` | `404` none found, `500` |
| `/sendMessageToPerson` | POST | Yes | `{ receiverId, message }` | `201 { success:true, message:"Message sent successfully", sentMessage, userInPersonMessages }` | `400` invalid user / cannot send, `500` |
| `/getMessageToPerson/:receiverId` | GET | Yes | `?page=&limit=` | `200 { success:true, message:"Messages found!", receiver, messages, getInPersonInfo, page, limit, countResult }` | `404` none found, `500` |
| `/delete-image/:public_id` | DELETE | Yes | none | `200 { success:true, message:"Image deleted successfully", result }` | `404` not found, `500` |
| `/deleteCluster/:cluster_id` | DELETE | Yes | none | `201 { success: true, message: "Cluster deleted successfully", sucess:true }` | `404` not found,`400` credenatil req,`403` athorization, `500` |

## 9. Authentication Flow

### Token usage
- A JWT token is issued during login or registration.
- Stored as cookie `devquor_Token`.
- Protected routes require:
  - Cookie `devquor_Token`, or
  - Header `devquor_Token`
- Middleware:
  - `backend/middleware/auth.middleware.js`
  - Verifies token and sets `req.id` and `req.email`.

### Failure responses
- `401 Unauthorized`
  - No token
  - Invalid token
- `404 Not Found`
  - Token decode error or missing item

## 10. Key Data and Payload Shapes

### User object
- `id`
- `username`
- `email`
- `firstname`
- `lastname`
- `bio`
- `gender`
- `date_of_birth`
- `profile_pic`
- `profile_id`
- `is_premium`

### Upload object
- `AuthUpload_id`
- `uploadUrl`
- `uploadId`
- `author_id`
- `UploadCreated_at`
- `authorThought`
- `author_username`
- `author_profile_pic`
- `total_thoughts`
- `upload_likes`
- `upload_dislikes`
- `upload_likes_array`
- `upload_dislikes_array`
- `thoughts` (with nested comments and likes)

## 11. Frontend Implementation Notes

### UI areas
- Authentication and user account pages
- Home feed with uploads
- Upload creation component
- Upload detail view with comments and like/bookmark actions
- Profile pages and editing page
- Notification UI components
- Messaging and cluster pages

### React + Firebase
- Firebase config is in `frontend/src/firebase.js`
- Notification permission handled in:
  - `frontend/src/requestNotificationPermission.js`
  - `frontend/src/firebase/notificationHelper.js`
- Frontend sends notification tokens to backend endpoint:
  - `/api/user/save-notification-token`

### Electron support
- Desktop packaging with Electron in:
  - `frontend/electron/main.cjs`
  - `frontend/electron/preload.cjs`
  - `frontend/electron/utils.cjs`
- Build targets configured in `frontend/package.json`

## 12. How to Run

### Backend
- Use Node.js environment variables from `.env`
- Start server via `node backend/server.js`
- Ensure PostgreSQL is available and `process.env.DATABASE_URL` / DB configs are set

### Frontend
- Run `npm install` in `frontend`
- Start local dev server with `npm run dev`
- For Electron, use `npm run electron-dev`

## 13. Screenshots

> Add screenshots here manually from the frontend UI:
- Login / Register page
- Home feed / uploads page
- Upload detail view
- Profile page
- Notifications page
- Cluster chat / messaging page

## 14. Notes and Observations

- Some endpoints use overlapping route names and inconsistent path patterns.
- `upload.controller.js` and `cluster.controller.js` use raw SQL with PostgreSQL parameter placeholders.
- The app is designed for both web and Electron desktop use.
- Notification token persistence and cleanup logic exists in `user.controller.js`.

---

### Suggested file
Save this content into:
- `PROJECT_DOCUMENTATION.md`

# Devgram Backend Public Overview

## Purpose

This document is a public-facing backend overview intended for interviewer review and project showcase. It describes the backend architecture, feature set, API surface, and security best practices without exposing secrets or implementation-only details.

## Backend Architecture

- Node.js + Express server in `backend/server.js`
- PostgreSQL database access through `backend/config/sqlDb.js`
- JWT authentication with `jsonwebtoken`
- Password hashing with `bcryptjs`
- File/image delivery via Cloudinary
- Real-time messaging using `socket.io`
- Notification support via Firebase and Resend-style helpers
- CORS configured to allow frontend origins and credentials

## Main API Areas

### Authentication and User Management

- `POST /api/user/userRegister`
  - Creates a new user
  - Hashes the password before saving
  - Issues a JWT token in an HTTP-only cookie

- `POST /api/user/userLogin`
  - Authenticates by email or username
  - Verifies password securely
  - Returns a JWT token in an HTTP-only cookie

- `POST /api/user/userLogout`
  - Clears the auth cookie

- `POST /api/user/userVerificationCode/email`
  - Generates a short-lived verification code
  - Saves it in the database
  - Sends a verification email via an email helper

- `POST /api/user/userPasswordReset/:genPassCode`
  - Validates password reset code and expiry
  - Returns a safe frontend reset link

- `POST /api/user/userPasswordChange/:genPassCode`
  - Updates password after verification
  - Uses password hashing before saving

- `GET /api/user/getAuthorDetails`
  - Returns the authenticated user profile
  - Requires JWT authentication

- `PUT /api/user/updateAuthor`
  - Updates profile fields
  - Optionally deletes old Cloudinary image when profile picture changes

- `GET /api/user/getAllUsers`
  - Returns a paginated list of users
  - Authenticated route

- `GET /api/user/getUserDetail/:userid`
  - Returns profile details for another user
  - Authenticated route

### Upload and Content Interaction

- `POST /api/upload/createUpload`
  - Authenticated upload creation
  - Accepts image URL + thought text

- `GET /api/upload/getAllUploads`
  - Public feed endpoint

- `GET /api/upload/getUploadDetail/:uploadId`
  - Authenticated detail view

- `POST /api/upload/likeOrDislikeUpload/:uploadId/:token`
  - Authenticated like/dislike toggle
  - Designed for quick feed interactions

- `GET /api/upload/getAuthorUploads`
  - Authenticated route for author uploads

- `POST /api/upload/modifyUploads/:uploads_id`
  - Update / modify upload detail

- Additional content routes for thoughts, replies, bookmarks, and comments are also available.

### Chat and Real-time Messaging

- `POST /api/cluster/createCluster`
- `POST /api/cluster/joinCluster`
- `GET /api/cluster/getAuthorMsgedClusters`
- `POST /api/cluster/getClusterDetail`
- `POST /api/cluster/sendClusterMessage/:cluster_id`
- `GET /api/cluster/getClusterMessage/:cluster_id`
- `GET /api/cluster/getUserInPerson`
- `POST /api/cluster/sendMessageToPerson`
- `GET /api/cluster/getMessageToPerson/:receiverId`

These routes support both group cluster chat and direct person-to-person chat.

## Security Practices Implemented

### 1. JWT Authentication

- JWT tokens are signed with a backend secret using `process.env.JWT_SECRET`
- The token is stored in an HTTP-only cookie named `devquor_Token`
- Cookie options include:
  - `httpOnly: true`
  - `sameSite: "none"`
  - `secure: true`

This prevents JavaScript access to the auth token and supports secure cross-site use when served over HTTPS.

### 2. Password Hashing

- All passwords are hashed before storage using a secure hashing service in `backend/services/user.services.js`
- Plaintext passwords are never stored in the database

### 3. Database Query Safety

- PostgreSQL queries use parameterized SQL statements via `$1`, `$2`, etc.
- This protects against SQL injection attacks for nearly all database operations

### 4. Environment Configuration

- Secrets are loaded from environment variables, not hard-coded into source code
- Key values such as database connection strings, JWT secret, Cloudinary keys, and frontend URLs are read from `.env`
- Public showcase should use `.env.example` instead of shipping actual keys

### 5. CORS and Cookie Security

- CORS is configured to restrict allowed origins to known frontend hosts
- The backend allows credentials when the frontend origin is trusted
- `cookie-parser` handles cookie reading securely

### 6. Cloudinary Cleanup

- When users update profile images, the backend deletes the old Cloudinary asset if applicable
- This prevents orphaned media assets from accumulating in the cloud account

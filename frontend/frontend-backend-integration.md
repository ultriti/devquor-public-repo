# Frontend / Backend Integration Guide

## Overview

This document describes how the React frontend in `frontend/src/` connects to the Node/Express backend in `backend/`. It covers: route mapping, data flow, navigation state, localStorage usage, API payloads, and socket/chat interactions.

The frontend uses:

- React Router v6 for app routing
- Axios for REST calls
- `withCredentials: true` on backend requests to include auth cookies
- `localStorage` for persistent user/session data
- `socket.io-client` via `frontend/src/Funtions/WebSocket.js`
- Firebase messaging for notifications

## Main App Routes (frontend/src/App.jsx)

| Route                                   | Component            | Protected | Notes                           |
| --------------------------------------- | -------------------- | --------- | ------------------------------- |
| `/user/Login`                           | `UserLogin`          | No        | login page                      |
| `/user/Register`                        | `UserRegister`       | No        | registration                    |
| `/user/ForgotPassword`                  | `ForgotPassword`     | No        | request email verification code |
| `/user/SendEmailVerification`           | `EmailVerification`  | No        | send verification email         |
| `/user/userPasswordChange/:genPassCode` | `ChangePassword`     | No        | set new password after code     |
| `/`                                     | `HomePage`           | Yes       | protected home feed             |
| `/user/CreateUpload`                    | `UploadsCreate`      | Yes       | create new upload               |
| `/user/userProfile/:userId`             | `UserProfilePage`    | Yes       | view profile                    |
| `/user/editProfile/:userId`             | `EditProfilePage`    | Yes       | edit current user's profile     |
| `/user/notification`                    | `MobileNotification` | Yes       | notifications page              |
| `/user/messageBox`                      | `MessagesPage`       | Yes       | message dashboard               |
| `/user/messageBox/inPerson`             | `Messagesprivate`    | Yes       | personal/direct conversations   |
| `/user/messageBox/global`               | `MessageGlobal`      | Yes       | global group chat overview      |
| `/user/messageBox/inPerson/:roomId`     | `InUserMessage`      | Yes       | direct chat with a user         |
| `/user/messageBox/Message/:roomId`      | `InPersonMessage`    | Yes       | cluster/group chat room         |

## Frontend Auth and Session Storage

The frontend uses these `localStorage` keys:

- `devquor_token` - auth token string obtained after login/register
- `devquor_UserDetail` - JSON object with authenticated user data
- `reset_email` - email address saved during password reset flow
- `genPassCode` - generated reset code used for password change
- `dq_userLst` - cached direct message list from `Messagesprivate`
- `dq_clusterLst` - cached cluster/group chat list from `MessagesPage`

### Where data is saved

- `UserLogin.jsx` and `UserRegister.jsx` save both `devquor_token` and `devquor_UserDetail` after successful login/register.
- `UserProfilePage.jsx` updates `devquor_UserDetail` if the logged-in user views their own profile again.
- `Message` pages persist conversation lists in `dq_clusterLst` and `dq_userLst`.
- Password reset flow stores `reset_email` and `genPassCode` during verification.

### Cookie + token behavior

Axios requests to backend routes usually include:

- `withCredentials: true` so cookies are sent
- In some cases, the token is also passed as a URL parameter, e.g. like/dislike route uses `devquor_token` from localStorage.

This means backend authentication is handled by cookies plus optional route token values.

## API Route Connections and Payloads

Environment variable used by frontend:

- `import.meta.env.VITE_BACKEND_URL`

### Auth endpoints

| Frontend component  | Backend route                          | Method | Payload                                              | Response data saved                   |
| ------------------- | -------------------------------------- | ------ | ---------------------------------------------------- | ------------------------------------- |
| `UserRegister`      | `/api/user/userRegister`               | POST   | `{ username, email, password, firstname, lastname }` | `devquor_token`, `devquor_UserDetail` |
| `UserLogin`         | `/api/user/userLogin`                  | POST   | `{ email, password }`                                | `devquor_token`, `devquor_UserDetail` |
| `EmailVerification` | `/api/user/userVerificationCode/email` | POST   | `{ email }`                                          | saves `reset_email`                   |
| `ForgotPassword`    | `/api/user/userPasswordReset/:code`    | POST   | `{ email }`                                          | saves `genPassCode`, `reset_email`    |
| `ChangePassword`    | `/api/user/userPasswordChange/:code`   | POST   | `{ email, password, confirmPassword }`               | navigates to login                    |

### Home feed and upload interaction

`HomePage.jsx`:

- GET `/api/upload/getAllUploads` to retrieve the feed
- POST `/api/upload/likeOrDislikeUpload/:uploadId/:token` to toggle like state
- `token` is read from `localStorage.getItem('devquor_token')`

`HomePage` renders upload cards using `HomePageUplaodTemplate.jsx`:

- clicking author profile image navigates to `/user/userProfile/${upload.author_id}`
- double-clicking on a thought or like button triggers `uploadLikenDislikeFun(...)`
- clicking the detail button opens `UploadDetailFrame` overlay without route navigation

`UploadDetailFrame.jsx`:

- GET `/api/upload/getUploadDetail/${uploadId}` to load full upload detail
- allows like toggle by calling `uploadLikenDislikeFun(uploadId)`
- no route state is passed; it uses local component state and overlay modal logic.

### Create upload flow

`UploadCreate.jsx`:

- Uploads image to Cloudinary using:
  - `https://api.cloudinary.com/v1_1/dusxzq0ws/image/upload`
  - `upload_preset=devgram`
- After Cloudinary returns `secure_url` and `public_id`, the frontend POSTs to backend:
  - `/api/upload/createUpload`
  - body: `{ uploads_url, uploads_id, thought }`
- On success, it navigates back to `/`

### Profile fetch and update flows

`UserProfilePage.jsx`:

- Reads route `:userId` using `useParams()`
- Determines if user is viewing own profile by comparing `userId` with `devquor_UserDetail.id`
- If own profile:
  - GET `/api/user/getAuthorDetails`
  - updates `devquor_UserDetail` in localStorage
- If other profile:
  - GET `/api/user/getUserDetail/${userId}`
- Response includes profile data and author uploads
- On render, it also uses `navigate('/')` for back action

`EditProfilePage.jsx` (not fully read in this session but code structure implies):

- Uploads a new profile image to Cloudinary
- Then calls `/api/user/updateAuthor` with updated profile fields and `oldProfileId`
- On error, may call `/api/cluster/delete-image/` to remove orphaned Cloudinary image
- Route path is `/user/editProfile/:userId`

### Notification flow

`requestNotificationPermission.js`:

- Requests browser notification permission if needed
- On grant, obtains Firebase token with `getToken(messaging, { vapidKey })`
- POSTs to `/api/user/save-notification-token` with `{ userId, token }`
- Uses `withCredentials: true`

`App.jsx` runs this once in `useEffect()` on app load.

## Chat / Socket Integration

### Message dashboard: `MessagesPage.jsx`

Purpose:

- shows message dashboard and cluster chat list
- allows user to join or create clusters
- allows navigating to direct or group chat pages

Key backend call:

- GET `/api/cluster/getAuthorMsgedClusters`
- stores result in `dq_clusterLst`
- also sets local `getMessagedCluster`

Navigation from cluster list:

- `navigate('/user/messageBox/Message/${cluster_id}', { state: { userId, groupName, groupId, c_id, clusterImage } })`
- This state is consumed by `InPersonMessage.jsx`

### Direct message list: `Messagesprivate.jsx`

Purpose:

- shows personal/direct message channels
- loads cached data from `dq_userLst` and refreshes from backend
- navigates to `/user/messageBox/inPerson/:roomId` for direct chats

Important backend call:

- GET `/api/cluster/getUserInPerson?page=${page}&limit=${limit}`
- response includes `messages`, `receiver`, `countResult`, `totalPages`, etc.

Data persistence:

- response data is saved to localStorage via `setLocalStorage('dq_userLst', response.data)`
- used to render the direct chat list

### Direct chat page: `InUserMessage.jsx`

Route: `/user/messageBox/inPerson/:roomId`

Input data:

- `userId` is extracted from `location.state` (sender is current logged-in user)
- `groupName`, `clusterImage`, `groupId` are also passed in route state
- Logged-in user detail comes from `devquor_UserDetail`

Backend request:

- GET `/api/cluster/getMessageToPerson/${userId}?page=1&limit=${pageLimit}`
- on success it:
  - formats server messages into chat UI objects
  - calls `socket.emit('joinPersonalChat', response.data.getInPersonInfo.id)`
  - sets local `messages`
  - sets `totalMsgCount`

Sending message:

- POST `/api/cluster/sendMessageToPerson`
- body: `{ receiverId: userId, message: trimmed, MsgType: msgType }`
- on success it appends a local message and emits socket event `chatMsg`

Socket behavior:

- listens for `chatMsgBk` from backend with `(groupName, msg, senderUserId)`
- on arrival, appends new message to UI
- also listens for `userJoinedPersonalChat`

### Group/cluster chat page: `InPersonMessage.jsx`

Route: `/user/messageBox/Message/:roomId`

Input data from route state:

- `userId`, `groupName`, `clusterImage`, `groupId`
- `groupId` is the cluster room ID used to join the correct room

Backend request:

- GET `/api/cluster/getClusterDetail?roomId=${groupId}&c_id=${c_id}`
- on success it:
  - formats returned `messages`
  - emits `socket.current.emit('joinCluster', response.data.getClusterDetail.id)`
  - sets local `messages`

Sending cluster message:

- POST `/api/cluster/sendClusterMessage/${cluster_id}`
- body: `{ message: trimmed, MsgType: msgType }`
- on success it emits socket event `clusterMsg` with the new message object

Socket behavior:

- listens for `clusterMsgBk` from backend
- listens for `userJoinedCluster`
- appends incoming messages to UI

### Global chat page: `MessageGlobal.jsx`

Purpose:

- renders the global chat room view
- likely connects to global cluster messages and backend cluster APIs
- no route state is required because it is a global room

## Data flow and navigation patterns

### Component -> route -> backend flow

1. User action triggers navigation or API call.
2. If route navigation is used, React Router pushes a new URL and may pass state with `navigate(path, { state })`.
3. Receiving page reads `useParams()` for URL path variables and `useLocation().state` for transient navigation state.
4. The page sends a backend request using Axios.
5. The backend response is converted into component state and often stored in `localStorage` for later reuse.
6. UI updates immediately if possible; additional socket events may continue incoming updates.

### Example: open a cluster chat from `MessagesPage`

- Click cluster item in `MessagesPage`
- `navigate('/user/messageBox/Message/${cluster_id}', { state: { userId, groupName, groupId, c_id, clusterImage } })`
- `InPersonMessage.jsx` reads `location.state`
- It calls `/api/cluster/getClusterDetail` and loads messages
- It emits `joinCluster` to socket with the cluster room ID
- New chat messages from other users arrive through `clusterMsgBk`

### Example: load and like an upload from home page

- `HomePage` calls GET `/api/upload/getAllUploads`
- Renders each upload with `HomePageUplaodTemplate`
- Like button calls `uploadLikenDislikeFun(uploadId)`
- This sends POST `/api/upload/likeOrDislikeUpload/:uploadId/:token`
- The UI updates like count locally immediately and optionally refreshes data

### Example: edit profile and local data

- `EditProfilePage` updates profile details via backend
- On success, backend may return updated user object
- Frontend should update `devquor_UserDetail` if editing the logged-in user
- `UserProfilePage` already refreshes this localStorage when the user views their own profile

## Key frontend file connections

- `frontend/src/App.jsx` — main route definitions
- `frontend/src/context/UserProtectedWrapper.jsx` — protects authenticated routes
- `frontend/src/context/UserContext.jsx` — global notification state
- `frontend/src/requestNotificationPermission.js` — browser notification permission and token saving
- `frontend/src/Funtions/WebSocket.js` — shared socket connection
- `frontend/src/pages/homepage/HomePage.jsx` — feed, upload list, like/dislike
- `frontend/src/components/uploadComponent/HomePageUplaodTemplate.jsx` — upload card display and navigation to profile/detail
- `frontend/src/components/uploadComponent/UploadDetailFrame.jsx` — full upload modal and detail fetch
- `frontend/src/pages/CreatePage/UploadCreate.jsx` — create upload with Cloudinary + backend
- `frontend/src/pages/ProfilePage/userProfilePage.jsx` — profile fetch and localStorage update
- `frontend/src/pages/chatPages/MessagesPage.jsx` — message dashboard and cluster list
- `frontend/src/pages/chatPages/Messagesprivate.jsx` — personal conversation list and direct chat navigation
- `frontend/src/pages/chatPages/InPersonMessage.jsx` — cluster chat room page
- `frontend/src/pages/chatPages/InUserMessage.jsx` — direct person-to-person chat page

## Practical integration notes

- Most backend calls include `withCredentials: true`. This is essential for auth cookie handling.
- When navigating with state, if the user refreshes the page, `location.state` can be lost. Pages that depend on transient state should also support fallback via backend requests or route params.
- `devquor_UserDetail` is the primary frontend source for current user identity. Many components rely on it for decisions such as `isMine`, message sender name, and profile navigation.
- Chat pages use both REST fetch and socket events. REST loads historic messages; sockets deliver realtime updates.
- Cloudinary file uploads are performed entirely on the frontend, then the backend stores the resulting `uploads_url` and `uploads_id`.

## Recommended documentation next step

Add a short reference section per backend endpoint in `document/backend.md` and cross-link it from this frontend doc. That helps developers follow the exact request/response structure and which frontend pages call each route.

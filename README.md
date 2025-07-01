# Secure API Client (Vue 3 + Vite)

A modern web client for authenticating users and interacting with a secure API, built with Vue 3, Vite, and Axios. This project demonstrates a full authentication flow (register, login, OTP verification, password reset, protected routes, role-based access, and session management) with robust token handling and role management.

---

## Features

- **User Registration & OTP Verification**: Register with a phone number and password, verify via OTP.
- **Login & JWT Authentication**: Secure login with device tracking and JWT-based access tokens.
- **Password Reset**: Request and verify OTP for password reset.
- **Protected Routes**: Access protected data after authentication.
- **Role-Based Access**: User, Moderator, and Admin roles with different access levels.
- **Admin Dashboard**: Assign/remove roles, view admin-only data.
- **Session Management**: Logout from current or all devices.
- **Automatic Token Refresh**: Handles expired tokens and refreshes them seamlessly.
- **Device & Platform Tracking**: Each request includes device and platform headers.

---

## Getting Started

### Prerequisites
- [Node.js](https://nodejs.org/) (v16+ recommended)
- [npm](https://www.npmjs.com/) (comes with Node.js)

### Installation
```bash
npm install
```

### Running the App
```bash
npm run dev
```
Visit [http://localhost:5173](http://localhost:5173) in your browser.

### Build for Production
```bash
npm run build
```

### Preview Production Build
```bash
npm run preview
```

---

## Usage Overview

1. **Register**: Enter your phone number and password. An OTP will be sent for verification.
2. **Verify OTP**: Enter the OTP to activate your account.
3. **Login**: Use your phone number and password to log in. JWT tokens are managed automatically.
4. **Access Protected Content**: After login, access general and role-specific protected data.
5. **Admin Actions**: If logged in as an admin, assign or remove roles for users.
6. **Session Management**: Logout from the current device or all devices.

---

## API Endpoints Used

- `POST /api/register` — Register a new user
- `POST /api/verify-otp` — Verify phone number with OTP
- `POST /api/resend-otp` — Resend OTP for verification
- `POST /api/login` — Login and receive JWT access token
- `POST /api/password-reset-request` — Request OTP for password reset
- `POST /api/password-reset` — Reset password with OTP
- `POST /api/refresh-token` — Refresh JWT access token
- `GET /api/protected` — Get general protected data (requires authentication)
- `GET /api/user/profile` — Get user profile data (role: user)
- `GET /api/content/moderate` — Get moderation data (role: moderator/admin)
- `GET /api/admin/dashboard` — Get admin dashboard data (role: admin)
- `POST /api/admin/assign-role` — Assign a role to a user (admin only)
- `POST /api/admin/remove-role` — Remove a role from a user (admin only)
- `POST /api/logout` — Logout from current device
- `POST /api/logout-all-devices` — Logout from all devices

> **Note:** The API base URL is currently hardcoded as `http://localhost:3000/` in the code. Update this in `src/App.vue` if your backend runs elsewhere.

---

## Project Structure

```
├── public/           # Static assets (e.g., logos)
├── src/
│   ├── App.vue       # Main application logic and UI
│   ├── main.js       # App entry point
│   ├── style.css     # Global styles
│   └── assets/       # Project images/icons
├── index.html        # HTML entry point
├── package.json      # Project metadata and scripts
├── vite.config.js    # Vite configuration
└── README.md         # Project documentation
```

---

## Configuration

- **API URL**: Update the API base URL in `src/App.vue` if needed.
- **Environment Variables**: No `.env` file is used by default. For advanced configuration, consider using Vite's [environment variables](https://vitejs.dev/guide/env-and-mode.html).

---

## Dependencies
- [Vue 3](https://vuejs.org/)
- [Vite](https://vitejs.dev/)
- [Axios](https://axios-http.com/)
- [uuid](https://www.npmjs.com/package/uuid)

---

## Customization & Styling
- Global styles are in `src/style.css`.
- App-specific styles are in `<style>` blocks in `App.vue`.
- The app uses the default Vite and Vue logos.

---

## License

This project is for demonstration and educational purposes. Adapt as needed for your use case.

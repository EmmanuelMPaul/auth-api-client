<template>
  <div id="app">
      <h1>Secure API Client</h1>

      <div v-if="message" :class="messageType" class="message-box">
          {{ message }}
      </div>

      <div v-if="currentView === 'register'" class="auth-section">
          <h2>Register</h2>
          <input v-model="register.phoneNumber" placeholder="Phone Number (e.g., +2547...)" />
          <input v-model="register.password" type="password" placeholder="Password" />
          <button @click="handleRegister">Register</button>
          <p class="switch-link" @click="currentView = 'login'">
              Already have an account? Login here.
          </p>
      </div>

      <div v-if="currentView === 'verifyOtp'" class="auth-section">
          <h2>Verify Phone Number</h2>
          <p>An OTP has been sent to {{ otpVerification.phoneNumber }}.</p>
          <input v-model="otpVerification.otp" placeholder="Enter OTP" />
          <button @click="handleVerifyOtp">Verify OTP</button>
          <button @click="handleResendOtp">Resend OTP</button>
          <p class="switch-link" @click="currentView = 'login'">Go back to Login</p>
      </div>

      <div v-if="currentView === 'login'" class="auth-section">
          <h2>Login</h2>
          <input v-model="login.phoneNumber" placeholder="Phone Number" />
          <input v-model="login.password" type="password" placeholder="Password" />
          <button @click="handleLogin">Login</button>
          <p class="switch-link" @click="currentView = 'register'">
              Don't have an account? Register here.
          </p>
          <p class="switch-link" @click="currentView = 'forgotPassword'">
              Forgot Password?
          </p>
      </div>

      <div v-if="currentView === 'forgotPassword'" class="auth-section">
          <h2>Forgot Password</h2>
          <p v-if="!forgotPassword.otpSent">
              Enter your phone number to receive a password reset OTP.
          </p>
          <p v-else>
              An OTP has been sent to {{ forgotPassword.phoneNumber }}. Enter the OTP
              and your new password.
          </p>

          <input v-model="forgotPassword.phoneNumber" placeholder="Phone Number" />
          <div v-if="forgotPassword.otpSent">
              <input v-model="forgotPassword.otp" placeholder="Enter OTP" />
              <input v-model="forgotPassword.newPassword" type="password" placeholder="New Password" />
          </div>
          <button v-if="!forgotPassword.otpSent" @click="handleForgotPasswordRequest">
              Request Password Reset
          </button>
          <button v-else @click="handlePasswordReset">Reset Password</button>
          <p class="switch-link" @click="currentView = 'login'">Back to Login</p>
      </div>

      <div v-if="currentView === 'protected'" class="protected-section">
          <h2>Protected Content</h2>
          <p v-if="userInfo">Welcome, {{ userInfo.phoneNumber }}!</p>
          <p v-if="userInfo && userInfo.roles">Your roles: {{ userInfo.roles.join(', ') }}</p>
          <p v-else>Loading user information...</p>

          <h3>General Protected Data</h3>
          <button @click="getProtectedData">Get General Protected Data</button>
          <pre v-if="protectedData">{{ protectedData }}</pre>

          <h3>Role-Specific Routes</h3>

          <div v-if="userInfo && userInfo.roles.includes('user')" class="role-section">
              <h4>User Profile</h4>
              <button @click="getUserProfileData">Get User Profile Data</button>
              <pre v-if="userProfileData">{{ userProfileData }}</pre>
          </div>

          <div v-if="userInfo && (userInfo.roles.includes('admin') || userInfo.roles.includes('moderator'))"
              class="role-section">
              <h4>Content Moderation</h4>
              <button @click="getContentModerateData">Get Content Moderation Data</button>
              <pre v-if="contentModerateData">{{ contentModerateData }}</pre>
          </div>

          <div v-if="userInfo && userInfo.roles.includes('admin')" class="role-section">
              <h4>Admin Functionality</h4>
              <button @click="currentView = 'adminDashboard'">Go to Admin Dashboard</button>
          </div>

          <hr />
          <h3>Session Management</h3>
          <button @click="handleLogout">Logout Current Device</button>
          <button @click="handleLogoutAll">Logout All Devices</button>
      </div>

      <div v-if="currentView === 'adminDashboard'" class="admin-dashboard-section">
          <h2>Admin Dashboard</h2>
          <p v-if="userInfo">Welcome, {{ userInfo.phoneNumber }}!</p>
          <p v-if="userInfo && userInfo.roles">Your roles: {{ userInfo.roles.join(', ') }}</p>
          <p v-else>Loading user information...</p>

          <h3>Admin-Only Data</h3>
          <button @click="getAdminDashboardData">Get Admin Dashboard Data</button>
          <pre v-if="adminDashboardData">{{ adminDashboardData }}</pre>

          <h3>Manage User Roles</h3>
          <div class="role-management-form">
              <h4>Assign/Remove Role</h4>
              <input v-model="roleManagement.userId" placeholder="User ID" />
              <input v-model="roleManagement.roleName" placeholder="Role Name (e.g., admin, user, moderator)" />
              <button @click="handleAssignRole">Assign Role</button>
              <button @click="handleRemoveRole">Remove Role</button>
          </div>

          <p class="switch-link" @click="currentView = 'protected'">Back to Protected Content</p>
          <hr />
          <h3>Session Management</h3>
          <button @click="handleLogout">Logout Current Device</button>
          <button @click="handleLogoutAll">Logout All Devices</button>
      </div>
  </div>
</template>

<script>
import axios from "axios";
import { v4 as uuidv4 } from 'uuid';

axios.defaults.withCredentials = true;

// Define a flag to prevent multiple refresh token requests at once
let isRefreshing = false;
// Queue of failed requests that need to be retried after the token is refreshed
let failedQueue = [];

// Function to process the queue of requests
const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
      if (error) {
          prom.reject(error);
      } else {
          prom.resolve(token);
      }
  });
  failedQueue = [];
};

// Add a request interceptor
axios.interceptors.request.use(
  (config) => {
      // Attach accessToken to all requests IF it exists and IF it's not a login/register/forgot-password call
      // (Login/Register/Forgot Password don't need access token)
      const publicRoutes = ['/api/login', '/api/register', '/api/verify-otp', '/api/resend-otp', '/api/password-reset-request', '/api/password-reset', '/api/refresh-token'];
      if (config.url && !publicRoutes.some(route => config.url.includes(route)) && localStorage.getItem("accessToken")) {
          config.headers.Authorization = `Bearer ${localStorage.getItem("accessToken")}`;
      }
      // Always attach device ID and platform
      config.headers['X-Device-Id'] = localStorage.getItem('webDeviceId') || uuidv4();
      config.headers['X-Platform'] = "web";
      return config;
  },
  (error) => {
      return Promise.reject(error);
  }
);


// Add a response interceptor
// Add a response interceptor
axios.interceptors.response.use(
  (response) => response,
  async (error) => { // Keep this `async` because the interceptor itself needs to await
      const originalRequest = error.config;

      // Skip interceptor logic if no response (network error, etc.) or if it's already marked as retried
      if (!error.response || originalRequest._retry) {
          return Promise.reject(error);
      }

      // Handle 401 Unauthorized errors
      if (error.response.status === 401) {
          // Check if the 401 came from the refresh token endpoint itself
          // This means the refresh token is also invalid or missing.
          if (originalRequest.url.endsWith('/api/refresh-token')) {
              console.error("Refresh token itself failed (401). Forcing client-side logout.");
              // Immediately clear tokens and redirect to login
              localStorage.removeItem('accessToken');
              localStorage.removeItem('webDeviceId'); // Clear device ID as well to get a new one on next login
              window.location.href = '/login'; // Or use router.push if you have one
              return Promise.reject(error); // Reject the original failed refresh request
          }

          // Mark the original request to prevent infinite loops
          originalRequest._retry = true;

          // If a refresh is already in progress, queue the current request
          if (isRefreshing) {
              return new Promise((resolve, reject) => {
                  failedQueue.push({ resolve, reject });
              }).then(token => {
                  originalRequest.headers['Authorization'] = 'Bearer ' + token;
                  return axios(originalRequest); // Retry the original request with the new token
              }).catch(err => {
                  return Promise.reject(err);
              });
          }

          // Start the refresh process
          isRefreshing = true;

          // ***** MODIFIED SECTION START *****
          return new Promise((resolve, reject) => { // Removed `async` here
              (async () => { // Use an IIFE (Immediately Invoked Function Expression) if you need `await` inside
                  try {
                      console.log("Access token expired. Attempting to refresh...");
                      // No need to pass headers manually here, interceptor will add them
                      const refreshResponse = await axios.post("http://localhost:3000/api/refresh-token", {});
                      const newAccessToken = refreshResponse.data.accessToken;

                      localStorage.setItem("accessToken", newAccessToken);
                      // Update the Authorization header for the original failed request
                      originalRequest.headers['Authorization'] = `Bearer ${newAccessToken}`;

                      // Process all queued requests with the new token
                      processQueue(null, newAccessToken);

                      // Resolve the promise for the current request with the retried request
                      resolve(axios(originalRequest));

                  } catch (refreshError) {
                      console.error("Refresh token error (during access token refresh):", refreshError.response?.data || refreshError.message);
                      // If refresh fails, process all queued requests with the error
                      processQueue(refreshError);
                      // Immediately clear tokens and redirect to login
                      localStorage.removeItem('accessToken');
                      localStorage.removeItem('webDeviceId');
                      window.location.href = '/login'; // Or router.push
                      reject(refreshError); // Reject the initial request that triggered the refresh
                  } finally {
                      isRefreshing = false;
                  }
              })(); // IIFE ends here
          });
          // ***** MODIFIED SECTION END *****
      }

      // For any other non-401 errors, just re-throw
      return Promise.reject(error);
  }
);

export default {
  name: "App",
  data() {
      return {
          currentView: "login",
          message: "",
          messageType: "",

          // Form data
          register: {
              phoneNumber: "",
              password: "",
          },
          otpVerification: {
              phoneNumber: "",
              otp: "",
          },
          login: {
              phoneNumber: "",
              password: "",
          },
          forgotPassword: {
              phoneNumber: "",
              otp: "",
              newPassword: "",
              otpSent: false,
          },
          roleManagement: {
              userId: "",
              roleName: "",
          },

          // User info and protected data
          userInfo: null,
          protectedData: null,
          userProfileData: null,
          contentModerateData: null,
          adminDashboardData: null,

          // accessToken is now primarily managed by localStorage and interceptors
          // Vue reactivity will still pick up changes if you manually update localStorage
          accessToken: localStorage.getItem("accessToken") || null,
          webDeviceId: localStorage.getItem('webDeviceId') || uuidv4(),
      };
  },
  watch: {
      // Watch for changes in accessToken and automatically switch view
      accessToken(newToken) {
          if (newToken) {
              this.currentView = "protected";
              // When accessToken changes, try to get user info and data
              this.getProtectedData();
          } else {
              this.currentView = "login";
              this.userInfo = null;
              this.protectedData = null;
              this.userProfileData = null;
              this.contentModerateData = null;
              this.adminDashboardData = null;
          }
      },
      // Watch for currentView changes to ensure userInfo is fetched if needed
      currentView(newView) {
          if (newView === 'protected' && !this.userInfo && this.accessToken) {
              this.getProtectedData();
          } else if (newView === 'adminDashboard' && (!this.userInfo || !this.userInfo.roles.includes('admin')) && this.accessToken) {
              // Prevent direct admin dashboard access without admin role
              this.setMessage("Unauthorized: You do not have admin access.", "error");
              this.currentView = 'protected';
          }
      }
  },
  created() {
      // Persist the unique device ID if it's new
      if (!localStorage.getItem('webDeviceId')) {
          localStorage.setItem('webDeviceId', this.webDeviceId);
      }

      // On app creation, if an access token exists, assume authenticated and try to fetch protected data
      if (this.accessToken) {
          this.currentView = "protected";
          this.getProtectedData();
      } else {
          this.currentView = "login";
      }
  },
  methods: {
      async setMessage(msg, type = "info", duration = 3000) {
          this.message = msg;
          this.messageType = type;
          if (duration > 0) {
              setTimeout(() => {
                  this.message = "";
                  this.messageType = "";
              }, duration);
          }
      },

      // This callApi helper is now much simpler as the interceptor handles token adding and refreshing
      async callApi(method, url, data = {}, additionalHeaders = {}) {
          try {
              const config = { headers: additionalHeaders }; // Interceptor adds auth, device, platform
              if (method === 'post' || method === 'put' || method === 'patch') {
                  return await axios[method](url, data, config);
              } else {
                  return await axios[method](url, config);
              }
          } catch (error) {
              // Interceptor will handle 401s (refresh/logout) and re-throw if needed.
              // Just log other errors or display a generic message here if desired.
              console.error(`API Call Error (${url}):`, error.response?.data || error.message);
              if (error.response?.status !== 401) { // If it's not a 401 (handled by interceptor)
                  this.setMessage(
                      error.response?.data?.message || `Failed to ${method} ${url}.`,
                      "error"
                  );
              }
              throw error; // Re-throw so specific handlers can catch if needed
          }
      },

      // --- Authentication Flow (No changes here, as they don't involve access token) ---

      async handleRegister() { /* ... unchanged ... */
          try {
              const response = await axios.post("http://localhost:3000/api/register", this.register);
              this.setMessage(response.data.message, "success");
              this.otpVerification.phoneNumber = this.register.phoneNumber;
              this.currentView = "verifyOtp";
          } catch (error) {
              this.setMessage(error.response?.data?.message || "Registration failed. Please try again.", "error");
              console.error("Registration error:", error.response?.data || error.message);
          }
      },

      async handleVerifyOtp() { /* ... unchanged ... */
          try {
              const response = await axios.post("http://localhost:3000/api/verify-otp", this.otpVerification);
              this.setMessage(response.data.message, "success");
              this.currentView = "login";
              this.otpVerification.otp = "";
          } catch (error) {
              this.setMessage(error.response?.data?.message || "OTP verification failed.", "error");
              console.error("OTP verification error:", error.response?.data || error.message);
          }
      },

      async handleResendOtp() { /* ... unchanged ... */
          try {
              const response = await axios.post("http://localhost:3000/api/resend-otp", { phoneNumber: this.otpVerification.phoneNumber });
              this.setMessage(response.data.message, "info");
          } catch (error) {
              this.setMessage(error.response?.data?.message || "Failed to resend OTP.", "error");
              console.error("Resend OTP error:", error.response?.data || error.message);
          }
      },

      async handleLogin() {
          try {
              const headers = { // These headers are only for login, not other protected calls now
                  "X-Device-Id": this.webDeviceId,
                  "X-Platform": "web",
              };
              const response = await axios.post("http://localhost:3000/api/login", this.login, {
                  headers,
                  withCredentials: true // <-- Add this line!
              });

              this.accessToken = response.data.accessToken;
              localStorage.setItem("accessToken", this.accessToken); // Update local storage
              this.userInfo = response.data.user;
              this.setMessage("Login successful!", "success");
              this.login.password = "";
              // currentView watch will handle the switch to 'protected'
          } catch (error) {
              this.setMessage(error.response?.data?.message || "Login failed. Please check credentials.", "error");
              console.error("Login error:", error.response?.data || error.message);
          }
      },

      async handleForgotPasswordRequest() { /* ... unchanged ... */
          try {
              const response = await axios.post("http://localhost:3000/api/password-reset-request", { phoneNumber: this.forgotPassword.phoneNumber });
              this.setMessage(response.data.message, "info");
              this.forgotPassword.otpSent = true;
          } catch (error) {
              this.setMessage(error.response?.data?.message || "Failed to request password reset.", "error");
              console.error("Password reset request error:", error.response?.data || error.message);
          }
      },

      async handlePasswordReset() { /* ... unchanged ... */
          try {
              const response = await axios.post("http://localhost:3000/api/password-reset", { phoneNumber: this.forgotPassword.phoneNumber, otp: this.forgotPassword.otp, newPassword: this.forgotPassword.newPassword });
              this.setMessage(response.data.message, "success");
              this.currentView = "login";
              this.forgotPassword = { phoneNumber: "", otp: "", newPassword: "", otpSent: false, };
          } catch (error) {
              this.setMessage(error.response?.data?.message || "Failed to reset password.", "error");
              console.error("Password reset error:", error.response?.data || error.message);
          }
      },

      // --- Protected Route and Token Refresh ---

      async getProtectedData() {
          if (!this.accessToken) {
              this.setMessage("No access token found. Please login.", "error");
              this.currentView = "login";
              this.userInfo = null;
              return;
          }
          try {
              // The interceptor will add the token and handle refresh if needed
              const response = await this.callApi("get", "http://localhost:3000/api/protected");
              this.protectedData = JSON.stringify(response.data, null, 2);
              this.userInfo = response.data.user; // Assuming user info comes with protected data
              this.setMessage("General protected data loaded successfully!", "success");
          } catch (error) {
              console.error("Error fetching general protected data:", error.message);
              this.protectedData = null;
              // The interceptor will handle forced logout for 401s on refresh failure
          }
      },

      async getUserProfileData() { /* ... unchanged, uses callApi now ... */
          try {
              const response = await this.callApi("get", "http://localhost:3000/api/user/profile");
              this.userProfileData = JSON.stringify(response.data, null, 2);
              this.setMessage("User profile data loaded successfully!", "success");
          } catch (error) {
              console.error("Error fetching user profile data:", error.message);
              this.userProfileData = null;
          }
      },

      async getContentModerateData() { /* ... unchanged, uses callApi now ... */
          try {
              const response = await this.callApi("get", "http://localhost:3000/api/content/moderate");
              this.contentModerateData = JSON.stringify(response.data, null, 2);
              this.setMessage("Content moderation data loaded successfully!", "success");
          } catch (error) {
              console.error("Error fetching content moderation data:", error.message);
              this.contentModerateData = null;
          }
      },

      async getAdminDashboardData() { /* ... unchanged, uses callApi now ... */
          try {
              const response = await this.callApi("get", "http://localhost:3000/api/admin/dashboard");
              this.adminDashboardData = JSON.stringify(response.data, null, 2);
              this.setMessage("Admin dashboard data loaded successfully!", "success");
          } catch (error) {
              console.error("Error fetching admin dashboard data:", error.message);
              this.adminDashboardData = null;
          }
      },

      // handleRefreshToken is now primarily called by the interceptor
      // You can keep it as a standalone method if you have other explicit refresh scenarios
      async handleRefreshToken() {
          // This method is now simplified because the interceptor will manage the actual refresh logic.
          // If it's explicitly called, it will just try to perform a refresh.
          try {
              // The interceptor will add device headers and handle token
              const response = await axios.post("http://localhost:3000/api/refresh-token", {});
              this.accessToken = response.data.accessToken;
              localStorage.setItem("accessToken", this.accessToken);
              this.setMessage("Tokens refreshed successfully!", "success");
          } catch (error) {
              // The interceptor will have already handled the logout for 401 on refresh.
              // Just log and perhaps set a generic error message if not handled.
              console.error("Manual refresh token error:", error.response?.data || error.message);
              if (error.response?.status !== 401) {
                  this.setMessage(error.response?.data?.message || "Failed to refresh token manually.", "error");
              }
              // No explicit handleLogoutClientSide() call here because the interceptor does it.
              throw error; // Re-throw to propagate
          }
      },

      // --- Logout Functions ---

      handleLogoutClientSide() {
          this.accessToken = null;
          localStorage.removeItem("accessToken");
          localStorage.removeItem('webDeviceId'); // Clear device ID on logout
          this.userInfo = null;
          this.protectedData = null;
          this.userProfileData = null;
          this.contentModerateData = null;
          this.adminDashboardData = null;
          this.currentView = "login";
          this.setMessage("You have been logged out.", "info");
      },

      async handleLogout() {
          try {
              // callApi will handle headers and eventual 401s (though logout shouldn't give 401 if token is present)
              await this.callApi("post", "http://localhost:3000/api/logout");
              this.setMessage("Logged out successfully from current session.", "success");
          } catch (error) {
              console.error("Logout error:", error.message);
          } finally {
              this.handleLogoutClientSide(); // Always perform client-side cleanup
          }
      },

      async handleLogoutAll() {
          try {
              await this.callApi("post", "http://localhost:3000/api/logout-all-devices");
              this.setMessage("Logged out successfully from all devices.", "success");
          } catch (error) {
              console.error("Logout all error:", error.message);
          } finally {
              this.handleLogoutClientSide(); // Always perform client-side cleanup
          }
      },

      // New: Admin Role Management Functions
      async handleAssignRole() { /* ... unchanged ... */
          try {
              if (!this.roleManagement.userId || !this.roleManagement.roleName) {
                  this.setMessage("Please enter both User ID and Role Name.", "error");
                  return;
              }
              const response = await this.callApi("post", "http://localhost:3000/api/admin/assign-role", {
                  userId: this.roleManagement.userId,
                  roleName: this.roleManagement.roleName,
              });
              this.setMessage(response.data.message, "success");
          } catch (error) {
              console.error("Error assigning role:", error.message);
          }
      },

      async handleRemoveRole() { /* ... unchanged ... */
          try {
              if (!this.roleManagement.userId || !this.roleManagement.roleName) {
                  this.setMessage("Please enter both User ID and Role Name.", "error");
                  return;
              }
              const response = await this.callApi("post", "http://localhost:3000/api/admin/remove-role", {
                  userId: this.roleManagement.userId,
                  roleName: this.roleManagement.roleName,
              });
              this.setMessage(response.data.message, "success");
          } catch (error) {
              console.error("Error removing role:", error.message);
          }
      },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  max-width: 600px;
  margin-left: auto;
  margin-right: auto;
  padding: 20px;
  border: 1px solid #eee;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

h1,
h2,
h3,
h4 {
  color: #34495e;
}

.auth-section,
.protected-section,
.admin-dashboard-section {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 5px;
  background-color: #f9f9f9;
}

.role-section {
  margin-top: 20px;
  padding: 15px;
  border: 1px dashed #b9d8c7;
  border-radius: 5px;
  background-color: #e6f5ed;
}

.admin-dashboard-section {
  border-color: #a3d9b8;
  background-color: #eaf7f0;
}

.role-management-form {
  margin-top: 20px;
  padding: 15px;
  border: 1px solid #cceeff;
  border-radius: 5px;
  background-color: #f0f8ff;
}

input {
  display: block;
  width: calc(100% - 20px);
  padding: 10px;
  margin: 10px auto;
  border: 1px solid #ccc;
  border-radius: 4px;
}

button {
  background-color: #42b983;
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin: 5px;
  font-size: 16px;
}

button:hover {
  background-color: #369c70;
}

hr {
  border: none;
  border-top: 1px solid #ccc;
  margin: 20px 0;
}

.switch-link {
  color: #42b983;
  cursor: pointer;
  margin-top: 15px;
  font-size: 0.9em;
}

.switch-link:hover {
  text-decoration: underline;
}

.message-box {
  padding: 10px;
  margin-top: 20px;
  border-radius: 5px;
  font-weight: bold;
}

.message-box.success {
  background-color: #d4edda;
  color: #155724;
  border-color: #c3e6cb;
}

.message-box.error {
  background-color: #f8d7da;
  color: #721c24;
  border-color: #f5c6cb;
}

.message-box.info {
  background-color: #d1ecf1;
  color: #0c5460;
  border-color: #bee5eb;
}
</style>
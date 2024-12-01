Lately, I've noticed that many newcomers to development struggle with the concept of handling JWT tokens. This is a common challenge because JWT tokens often have a short lifespan, while users prefer staying signed in for longer periods to avoid frequent login hassles.

First, let's begin by examining a setup that employs Axios as an HTTP client for making API requests using JWT tokens. To get started, we'll establish a base URL, which will serve as our API domain. Let's take a closer look at the following code snippet:

```javascript
const PROD_BASE_URL = "http://localhost:5000";

const BASE_URL = import.meta.env.VITE_BASE_URL ?? PROD_BASE_URL;
```

Here, we've declared a domain as production server, which will be utilized in the production build. Furthermore, we're importing a domain from the environment as development server. This approach offers a straightforward way to switch backend domains without the need to modify the core code.

Now that our backend server URL is set up, we can proceed to create an instance of Axios, which will use this URL by default for any invocations of the instance.

```javascript
const apiClient = axios.create({ baseURL: BASE_URL });
```

With this 'apiClient', we can now perform standard CRUD operations, such as 'apiClient("/api/test-get")' or 'apiClient.get("/api/test-get")'. This will concatenate the endpoint with the base URL and execute the HTTP request to the backend server.

Now, returning to our main topic, for APIs that require authentication, we must supply a token. We can obtain this token by calling endpoints that provide it when valid credentials are supplied. Suppose we have stored the tokens in the local storage of the web browser. To complete the authentication, we need to send the token in the HTTP request header. This can be achieved through the following method:

```javascript
apiClient("/api/test-get", {
  headers: {
    Authorization: "Bearer ---jwt token---",
  },
})
```

Imagine having to do this for every API request across your entire application! It would not be an ideal solution and would certainly be challenging to maintain. Axios provides us with a handy feature known as interceptors. Let's take a look at the following code snippet, which accomplishes the same task but with the assistance of an interceptor:

```javascript
apiClient.interceptors.request.use((config) => {
  config.headers.Authorization = "Bearer " + getAccessToken();
  return config;
});
```

Notice that we've added the callback interceptor to the request. This interceptor will be executed before every API request made using the apiClient instance. Another point to observe is that we didn't use template strings to provide the token; instead, we provided a callback function to retrieve the JWT token, perhaps from local storage, and concatenated it with "Bearer ". The reason for this approach is to ensure that the current JWT token from storage is attached in case it gets refreshed or cleared.

With this setup, we are well-prepared to access authorized API endpoints. However, consider a scenario where the token has expired. This configuration will persist in attaching the bearer token to requests, but it won't be authorized. It should trigger an unauthorized error with a 401 status code from the backend. In such cases, we need to leverage this error response to refresh our token on the fly.

Let's examine the code below, which serves as our response interceptor. It's designed to detect 401 error responses and take actions as specified. I'll provide an explanation to help you understand the concept:

```javascript
// Setting up variables and functions
let isRefreshingToken = false; // A flag to prevent concurrent token refresh
let requestQueue = []; // Store requests that need a refreshed token

const addRequestToQueue = (callback) => {
  requestQueue.push(callback);
};

const processRequestQueue = (accessToken) => {
  while (requestQueue.length) {
    const currentRequest = requestQueue.shift();
    currentRequest(accessToken);
  }
};

// Axios response interceptor
apiClient.interceptors.response.use(undefined, (error) => {
  const originalRequest = error?.config;
  const refreshToken = getRefreshToken(); // Retrieve the refresh token

  if (
    error?.response?.status === 401 &&
    !originalRequest?._retry &&
    !!refreshToken
  ) {
    originalRequest._retry = true; // Prevent retrying the same request

    // Start the token refresh process if it's not already in progress
    if (!isRefreshingToken) {
      isRefreshingToken = true;

      // Make a request to refresh the token
      axios
        .post(`${BASE_URL}/api/refresh-token`, {
          refreshToken,
        })
        .then(({ data = {} }) => {
          setAccessToken(data?.jwtToken); // Update the access token
          processRequestQueue(data?.jwtToken); // Process pending requests
        })
        .catch(() => {
          removeTokens(); // Remove tokens if the refresh fails
          processRequestQueue(false); // Process pending requests with a failure flag
        })
        .finally(() => {
          isRefreshingToken = false;
        });
    }

    // Queue the request for later retry
    return new Promise((resolve, reject) => {
      addRequestToQueue((accessToken) => {
        if (accessToken) {
          originalRequest.headers["Authorization"] = "Bearer " + accessToken;
          resolve(axios(originalRequest));
        }
        reject(originalRequest);
      });
    });
  }

  return Promise.reject(error);
});
```

The provided code is a token refreshing mechanism within an Axios-based API client. It's designed to manage token expiration during API requests and ensure uninterrupted access to protected resources.

At its core, the code uses interceptors to handle HTTP responses. When the API returns a 401 status code (indicating an expired token), it triggers the token refresh process. This process involves checking for a valid refresh token and, if present, making a request to refresh the access token.

To prevent multiple simultaneous token refreshes, it employs a boolean flag, isRefreshingToken, which ensures that only one token refresh process occurs at a time. Additionally, it maintains a requestQueue to store failed requests due to token expiration or those that arrived during token refresh.

When a token refresh is initiated, the code checks for an ongoing refresh process. If no process is active, it sets the isRefreshingToken flag and sends a request to refresh the token. Upon success, it updates the access token and reprocesses any pending requests in the requestQueue using the new token. In case of failure during token refresh, it removes tokens and processes queued requests with a failure flag.

The mechanism also handles request queuing by storing failed requests in the requestQueue and resolving/rejecting them based on the availability of a new access token. If a new token is obtained, it updates the request's authorization header and retries the request; otherwise, it rejects the request.

Overall, this setup ensures a seamless experience by automatically refreshing tokens when needed, preventing disruptions due to token expiration during API interactions.


Please feel free to reach out if you have any questions, concerns, or suggestions for enhancing this solution. Your feedback is greatly appreciated and can contribute to the continuous improvement of our implementation.
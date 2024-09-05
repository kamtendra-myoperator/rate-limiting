# Implementing Different Rate Limits per Client Using API Keys with NGINX

This guide provides a full and detailed implementation of how to apply **different rate limits per client** using unique API keys with NGINX.

## 1. Setting up NGINX to Use Rate Limiting per API Key

We'll use NGINX's `limit_req_zone` directive, and combine it with the `map` directive to apply different rate limits based on API keys.

### Key Configuration Components:
- **`limit_req_zone`**: Defines a zone in shared memory for rate limiting. Each client's API key is used as a key to track request counts.
- **`map`**: Maps API keys to client identifiers (or rate limits directly).
- **`limit_req`**: Applies the rate-limiting policy in the specific NGINX location.

## 2. Step-by-Step Configuration

### Step 1: Define the `limit_req_zone`
In the `http` block of your NGINX configuration file, define a `limit_req_zone` where the rate-limiting states for each client will be stored:

```nginx
http {
    # Shared memory zone for rate-limiting based on client API keys
    limit_req_zone $client_id zone=client_zone:10m rate=5r/s;
}
```
- **`$client_id`**: A variable that we'll later map to the actual API key from the HTTP request.
- **`client_zone`**: A shared memory zone where the request counts are stored.
- **`rate=5r/s`**: Sets a default limit of 5 requests per second.

### Step 2: Map API Keys to Client Identifiers
The `map` directive is used to associate specific API keys with client IDs, which will be used for rate limiting.

```nginx
map $http_api_key $client_id {
    default "unknown";          # Default if the API key is not recognized
    "API_KEY_123" "client_1";   # Assigns client 1's API key
    "API_KEY_456" "client_2";   # Assigns client 2's API key
    "API_KEY_789" "client_3";   # Assigns client 3's API key
}
```
- **`$http_api_key`**: This captures the API key passed in the HTTP request header.
- Each API key (e.g., `API_KEY_123`) is mapped to a corresponding client ID (e.g., `client_1`).

### Step 3: Apply Rate Limits in the `server` or `location` Block
Once the `map` and `limit_req_zone` are defined, you can apply the rate limits in the appropriate `server` or `location` block.

```nginx
server {
    listen 80;
    server_name myapi.com;

    location /api/ {
        # Apply rate limiting per client based on their API key
        limit_req zone=client_zone burst=10 nodelay;

        # Pass request to the upstream application
        proxy_pass http://backend_server;
    }
}
```
- **`limit_req`**: This applies the rate limiting using the zone defined earlier (`client_zone`). It allows a burst of 10 requests before limiting and does not delay requests (`nodelay`).
- **`proxy_pass`**: This forwards requests to the backend application (your API).

## 3. Explanation of the Configuration
- **Rate Limit Customization**: The `limit_req_zone` directive defines the rate limit (5 requests per second by default). You can adjust this based on your needs.
- **Per Client Logic**: The `map` directive assigns API keys to specific clients. If you want to set different rate limits for each client, you would create separate zones for different clients.
- **Burst**: The `burst=10` parameter allows handling bursts of traffic. If more than 5 requests/second come in, the server can handle an extra 10 requests in a short burst before enforcing strict limits.

## 4. Implementing Different Rate Limits for Different Clients
If you want each client to have a unique rate limit, you can modify the `map` and `limit_req_zone` configuration to assign different limits.

### Example of Different Rate Limits for Clients:

```nginx
http {
    # Define a shared memory zone for each client with different rate limits
    limit_req_zone $client_id zone=client1_zone:10m rate=10r/s;
    limit_req_zone $client_id zone=client2_zone:10m rate=5r/s;
    limit_req_zone $client_id zone=client3_zone:10m rate=20r/s;

    map $http_api_key $client_id {
        default "unknown";
        "API_KEY_123" "client1";
        "API_KEY_456" "client2";
        "API_KEY_789" "client3";
    }
}

server {
    listen 80;
    server_name myapi.com;

    location /api/ {
        # Apply rate limit based on the mapped client
        limit_req zone=client1_zone burst=10 nodelay;
        limit_req zone=client2_zone burst=5 nodelay;
        limit_req zone=client3_zone burst=15 nodelay;

        # Backend service
        proxy_pass http://backend_server;
    }
}
```
- **Client 1**: Can make 10 requests per second with a burst of 10.
- **Client 2**: Limited to 5 requests per second with a burst of 5.
- **Client 3**: Can make 20 requests per second with a burst of 15.

## 5. Considerations
- **Storage Usage**: The shared memory zone (`client_zone`) is configured with 10 MB of memory, which is sufficient to track thousands of requests. You may need to adjust this based on traffic.
- **Custom Logic for API Keys**: If API keys are stored in a database and frequently change, you can implement dynamic key loading using NGINX's Lua module or integrate with a separate API management service.

## 6. Logging and Debugging
You can add logs to track rate-limiting actions:

```nginx
limit_req_log_level notice;
limit_req_status 429;  # HTTP status code for rate limit exceeded
```

This ensures that any requests exceeding the limit are logged and return a `429 Too Many Requests` error.

## Conclusion:
This configuration allows you to implement client-specific rate limiting using API keys with NGINX. By leveraging the `map` directive and `limit_req_zone`, you can easily apply different rate limits based on API keys. This method is efficient for scaling and managing traffic from multiple clients with distinct rate-limiting needs.

For further customization or real-time dynamic rate limits, consider integrating this setup with NGINX Plus or an external rate-limiting service.

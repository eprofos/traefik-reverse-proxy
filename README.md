# Secure Traefik v3.2 Reverse Proxy Configuration

A production-ready Traefik v3.2 reverse proxy setup with automatic SSL certificate management, enhanced security features, and Docker integration. This configuration provides a secure gateway for your containerized applications with features like automatic HTTPS, rate limiting, and security headers.

## Features

- 🔒 Automatic SSL/TLS certificate management via Let's Encrypt
- 🛡️ Enhanced security headers and middleware
- 🔄 Automatic HTTP to HTTPS redirection
- 🚦 Rate limiting protection
- 🗜️ Gzip compression
- 📊 Secure dashboard access with authentication
- 🐳 Docker provider integration
- 📝 Structured JSON logging
- ⚡ Modern TLS 1.3 with strong cipher suites

## Architecture

```
├── acme/
│   └── acme.json         # Let's Encrypt certificates storage
├── config/
│   └── config.yml        # Main Traefik configuration
├── certs/                # Directory for manual certificates (if needed)
├── .env                  # Environment variables
└── compose.yaml          # Docker Compose configuration
```

## Requirements

- Docker Engine + Docker Compose
- A registered domain pointed to your server's IP
- Open ports: 80 (HTTP) and 443 (HTTPS)

## Installation

1. Clone or download this configuration to your server

2. Create required Docker network and directories:
```bash
docker network create proxy
mkdir -p ./config ./certs ./acme
touch ./acme/acme.json
```

Then set the proper permissions for acme.json:

For Unix-based systems (Linux, macOS):
```bash
chmod 600 ./acme/acme.json
```

For Windows systems:
```bash
icacls ./acme/acme.json /reset
icacls ./acme/acme.json /grant:r "%USERNAME%":(F)
icacls ./acme/acme.json /inheritance:r
```

3. Configure environment variables:
```bash
cp .env.example .env
```
Edit `.env` file with your specific settings:
- Set `TRAEFIK_DASHBOARD_DOMAIN`
- Configure `ACME_EMAIL` for Let's Encrypt notifications
- Set rate limiting values
- Configure TLS settings

4. Generate dashboard authentication password:
```bash
docker run --rm httpd:2.4-alpine htpasswd -nbB admin "your-desired-password"
```
- Copy the output (format: admin:$2y$05$....)
- In `.env`, update:
  - `ADMIN_AUTH` 

5. Start Traefik:
```bash
docker compose up -d
```

6. Verify installation by accessing your dashboard:
```
https://traefik.yourdomain.com
```

## Usage Example: Adding a Service

Here's an example of how to add a service to be handled by Traefik. We'll use the `whoami` service as an example.

Create a new file called `whoami-service.yml`:

```yaml
version: '3'

services:
  whoami:
    image: traefik/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      # Router configuration
      - "traefik.http.routers.whoami.rule=Host(`whoami.yourdomain.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
      # Service configuration
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
      # Middleware chain
      - "traefik.http.routers.whoami.middlewares=secure-headers"
      # Security headers middleware
      - "traefik.http.middlewares.secure-headers.headers.sslRedirect=true"
      - "traefik.http.middlewares.secure-headers.headers.stsSeconds=31536000"
      - "traefik.http.middlewares.secure-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.secure-headers.headers.stsPreload=true"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

To deploy the service:

1. Save the above configuration in `whoami-service.yml`
2. Make sure your domain (whoami.yourdomain.com) points to your server
3. Deploy the service:
```bash
docker compose -f whoami-service.yml up -d
```

The service will be available at `https://whoami.yourdomain.com` with:
- Automatic HTTPS
- Let's Encrypt SSL certificate
- Security headers
- Integration with Traefik's monitoring

Key points in the configuration:
- `traefik.enable=true`: Tells Traefik to handle this container
- `Host` rule: Defines which domain will route to this service
- `certresolver=letsencrypt`: Uses Let's Encrypt for SSL certificates
- `middlewares`: Applies security headers
- `networks`: Connects to the same network as Traefik

## Security Features

### TLS Configuration
- TLS 1.3 enforced
- Strong cipher suites:
  - TLS_AES_256_GCM_SHA384
  - TLS_CHACHA20_POLY1305_SHA256
  - TLS_AES_128_GCM_SHA256
- Strict SNI checking
- ECDSA curves: P521, P384

### Security Headers
- Browser XSS Filter
- Content Type Nosniff
- Frame Deny
- SSL Redirect
- STS Preload
- Custom security response headers

### Rate Limiting
- Configurable average and burst rates
- Protection against DDoS attacks
- Customizable through environment variables

### Authentication
- Basic authentication for dashboard access
- Secure password hashing
- Environment variable configuration

## Monitoring

### Dashboard Access
- Protected by authentication
- Provides real-time metrics
- Shows router and service status
- Available at: https://traefik.yourdomain.com

### Logging
- JSON formatted logs
- Configurable log levels
- Access logs with detailed request information
- Filtered sensitive information

## Troubleshooting

1. Certificate Issues:
   - Ensure ports 80 and 443 are open
   - Verify domain DNS points to your server
   - Check acme.json permissions (600 on Unix, proper ACLs on Windows)
   - Verify acme.json is writable by the Traefik container

2. Dashboard Access:
   - Verify correct password in .env file
   - Ensure proper dollar sign escaping
   - Check dashboard domain resolution

3. Container Connectivity:
   - Verify 'proxy' network exists
   - Check container network attachment
   - Validate service labels

## Best Practices

1. Security:
   - Regularly update Traefik version
   - Keep passwords secure and complex
   - Monitor access logs for suspicious activity
   - Use rate limiting appropriate for your traffic
   - Ensure proper acme.json permissions are maintained

2. Maintenance:
   - Backup acme.json regularly
   - Monitor certificate expiration
   - Keep environment variables secure
   - Review logs periodically

3. Performance:
   - Enable compression for appropriate content types
   - Configure proper rate limiting values
   - Monitor resource usage

## Important Notes

- Never commit .env file with sensitive data
- Keep acme.json backed up securely and maintain its permissions
- Monitor dashboard access attempts
- Review security headers periodically
- Keep Traefik version updated
- Test configuration changes in staging

## Support

For issues and feature requests, please:
1. Check the [Traefik documentation](https://doc.traefik.io/traefik/)
2. Review common issues in troubleshooting section
3. Ensure all requirements are met
4. Verify configuration syntax

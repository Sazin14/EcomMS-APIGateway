# API Gateway

Entry point for the e-commerce microservices.

```
Client → nginx (TLS, rate limits, headers) → Kong (routing only) → services
```

- **nginx**: HTTPS, rate limiting, body size limits, security headers, request IDs, strips any client-sent `X-Internal-Key`.
- **Kong** (DB-less): route management only. No plugins.
- **Services**: authentication and authorization are done inside each service.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Runs nginx + Kong for local development |
| `kong.yml` | Kong routes (declarative config) |
| `nginx.conf` | Dev nginx config (HTTP only) |
| `nginx.prod.conf` | Production nginx config (HTTPS) |
| `kong.prod.yml` | Kong routes with production upstream addresses |
| `fpm-sidecar/nginx.conf` | HTTP front for the php-fpm-only images (Auth, Product, Order) |

## Services

| Service | Address (host) | Public paths |
|---|---|---|
| AuthService | `:8001` | `/api/auth/*`, `/api/staff/*`, `/api/roles`, `/api/permissions` |
| ProductService | `:8002` | `/api/categories`, `/api/products`, `/api/admin/{products,categories,attributes,attribute-values,brands,variants,images}` |
| InventoryService | `:8003` | `/api/availability`, `/api/admin/{inventory,warehouses,stock,movements,alerts,reservations,reports,suppliers,purchase-orders}` |
| OrderService | `:8004` | `/api/{cart,checkout,orders,preorders,returns}`, `/api/payments/webhook`, `/api/admin/{orders,preorders,payments,fulfillments,returns,refunds}` |
| NotificationService | `:8005` | `/api/notifications`, `/api/notification-preferences`, `/api/admin/{notifications,notification-templates,notification-deliveries}` |

`/api/internal/*` (service-to-service, `X-Internal-Key`) is never exposed through the gateway.
`/api/admin/inventory/*` (Inventory pre-order dashboard and stock-return log) is not the same as
`/api/admin/preorders` and `/api/admin/returns`, which belong to OrderService.

Browser apps (storefront, admin) must call the gateway (`http://localhost:8080/api` in development),
not the services directly. Each service only accepts the origins in its `CORS_ALLOWED_ORIGINS`.

## Run locally

1. Start each service listening on all interfaces, for example:
   ```bash
   php artisan serve --host=0.0.0.0 --port=8002
   ```
2. Start the gateway:
   ```bash
   docker compose up -d
   ```
3. Test:
   ```bash
   curl -i http://localhost:8080/api/products
   ```

The gateway is available at `http://localhost:8080`.

## Common tasks

```bash
# After editing kong.yml
docker compose restart kong

# After editing nginx.conf
docker compose exec nginx nginx -s reload

# Validate kong.yml without starting anything
docker run --rm -v $PWD/kong.yml:/kong.yml -e KONG_DATABASE=off kong:3.8 kong config parse /kong.yml

# Logs
docker compose logs -f nginx kong
```

## Add a service or route

Edit `kong.yml`: add a service (or a path under an existing route), then restart Kong. Only paths listed in `kong.yml` are reachable, everything else returns 404.

## Production

1. Replace `api.example.com` in `nginx.prod.conf` with your domain.
2. Issue a certificate (for example with certbot). nginx expects it at `/etc/letsencrypt/live/<domain>/`.
3. In the compose file used on the server: mount `nginx.prod.conf` instead of `nginx.conf`, publish ports `80` and `443`, and mount `/etc/letsencrypt` (read-only) and `./certbot-www`.
4. Change the service URLs in `kong.yml` if the services no longer run on the same host (`host.docker.internal` only works for services on the same machine).
5. Set `APP_DEBUG=false` in every service and restrict CORS to your real domains in the services' CORS config.
6. In each Laravel service, configure `TrustProxies` so client IPs and `https` URLs are correct behind nginx and Kong.

## Notes

- Rate limits are set in nginx (10/min on login, register and refresh; 20 req/s per IP elsewhere).
- Body limit: 5 MB (the API takes JSON only; product images are sent as URLs).
- Production upstreams are in `kong.prod.yml`; php-fpm images (Auth, Product, Order) need `fpm-sidecar/nginx.conf` in front.
- Background workers every service needs: `../Infrastructure/DEPLOYMENT.md`.
- Never commit certificates, private keys or `.env` files.

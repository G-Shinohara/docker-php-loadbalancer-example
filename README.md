# Docker PHP load balancer example

A small teaching setup from a [Digital Innovation One](https://www.dio.me/) live coding session (Denilson Bonatti). It shows an nginx container balancing HTTP traffic across several PHP servers that all write to the same MySQL database. Each insert records the hostname of the server that handled the request, so you can see the load spread across backends.

## How it fits together

```
client :4500
    |
    v
nginx (this image)
    |  round-robin across the upstream
    +--> PHP server 172.31.0.37:80
    +--> PHP server 172.31.0.151:80
    +--> PHP server 172.31.0.149:80
              |
              v
         MySQL (meubanco.dados)
```

1. nginx listens on port **4500** and proxies `/` to an upstream named `all`.
2. With no extra directives, nginx uses its default **round-robin** policy, so successive requests land on different PHP servers.
3. `index.php` on each PHP server opens a connection to MySQL, inserts one row into `dados`, and stores `gethostname()` in the `Host` column.
4. Querying `dados` shows which backend served each request.

The PHP servers and the database are **not** defined in this repository. They were separate machines (private IPs from the live session). This repo only contains the load balancer image, the PHP page those servers are expected to serve, and the table those pages write to.

## Files

| File | Role |
| --- | --- |
| `dockerfile` | Image based on `nginx`. Copies `nginx.conf` over the default config. |
| `nginx.conf` | Upstream list and the proxy server on port 4500. |
| `index.php` | Page deployed on each PHP backend. Prints the PHP version and inserts a row tagged with the host name. |
| `banco.sql` | `CREATE TABLE` for `dados` (`AlunoID`, `Nome`, `Sobrenome`, `Endereco`, `Cidade`, `Host`). |

## Run the load balancer

Point the upstream at servers you actually have, then build and start nginx:

```bash
docker build -f dockerfile -t php-lb .
docker run --rm -p 4500:4500 php-lb
```

Open `http://localhost:4500`. Each successful request prints the PHP version and `New record created successfully`.

## Database

Create the database and table on the MySQL host the PHP servers use:

```sql
CREATE DATABASE meubanco;
USE meubanco;
```

Then run `banco.sql`. After a few requests:

```sql
SELECT Host, COUNT(*) FROM dados GROUP BY Host;
```

The counts show how nginx distributed traffic.

## Before you use it

`nginx.conf` and `index.php` still contain the addresses and credentials from the original live session. Replace them with your own:

- **Upstream servers** in `nginx.conf` (`upstream all`). These must be reachable from the nginx container.
- **MySQL host, user, password, and database** in `index.php`.

`index.php` builds the `INSERT` by concatenating values into the SQL string. That is fine for a classroom demo and unsafe for anything exposed to real users. Use prepared statements if you extend this beyond the example.

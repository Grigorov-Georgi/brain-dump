# curl cheatsheet

Most-used flags with short examples. Full reference: [curl man page](https://curl.se/docs/manpage.html).

## Request method and body

- **`-X METHOD`, `--request METHOD`** — HTTP method: GET, POST, PUT, DELETE, PATCH, etc.
- **`-d DATA`, `--data DATA`** — Send body (implies POST). Multiple `-d` are joined with `&`.
- **`-G`, `--get`** — Use GET and put `-d` data in the URL as query string.
- **`--json DATA`** — Send JSON body (sets `Content-Type: application/json` and `Accept: application/json`).

```bash
curl -X POST -d "name=alice&age=30" https://example.com/api/users
curl -G -d "q=search" -d "limit=10" https://example.com/search
curl --json '{"key":"value"}' https://example.com/api
```

## Headers

- **`-H "Header: value"`, `--header`** — Add or override a request header. Use multiple times for several headers.

```bash
curl -H "Authorization: Bearer TOKEN" -H "X-Custom: foo" https://example.com/api
```

## Output

- **`-o FILE`, `--output FILE`** — Write response body to file. `-o -` = stdout.
- **`-O`, `--remote-name`** — Save to a file named like the URL path (last segment).
- **`-i`, `--include`** — Include response headers in the output.
- **`-I`, `--head`** — Request headers only (HEAD), no body.
- **`-s`, `--silent`** — No progress meter or error text (still prints body unless redirected).
- **`-w FORMAT`, `--write-out`** — Print info after the transfer (e.g. status code, time).

```bash
curl -o page.html https://example.com
curl -O https://example.com/files/doc.pdf
curl -i https://example.com/api/health
curl -I https://example.com
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

## Redirects and errors

- **`-L`, `--location`** — Follow 3xx redirects.
- **`-f`, `--fail`** — Treat HTTP 4xx/5xx as failure (exit 22, no body).

```bash
curl -L https://example.com/redirect
curl -f https://example.com/api/thing
```

## Auth and cookies

- **`-u USER:PASS`, `--user`** — Basic auth. `-u user` prompts for password.
- **`-b FILE|STRING`, `--cookie`** — Send cookies (file or `name=value; ...`).
- **`-c FILE`, `--cookie-jar FILE`** — Save received cookies to file.

```bash
curl -u alice:secret https://example.com/protected
curl -b "session=abc123" https://example.com/dashboard
curl -c cookies.txt -b cookies.txt -L https://example.com/login
```

## Timeouts and TLS

- **`--connect-timeout SECS`** — Max time to establish connection.
- **`-m SECS`, `--max-time SECS`** — Max time for the whole transfer.
- **`-k`, `--insecure`** — Skip TLS certificate verification (insecure; use only for testing).

```bash
curl --connect-timeout 5 -m 30 https://example.com
curl -k https://self-signed.example.com
```

## Proxy

- **`-x URL`, `--proxy URL`** — Use proxy. `http://host:port`, `socks5://host:port`, etc.

```bash
curl -x http://proxy.local:8080 https://example.com
```

## One-liner reference

| Goal              | Example |
|-------------------|--------|
| GET with query    | `curl -G -d "a=1" -d "b=2" https://example.com` |
| POST form         | `curl -d "name=alice" -d "age=30" https://example.com` |
| POST JSON         | `curl --json '{"x":1}' https://example.com` |
| Custom method     | `curl -X DELETE https://example.com/resource/1` |
| Headers + auth    | `curl -H "Authorization: Bearer X" -u user:pass https://example.com` |
| Follow + save     | `curl -L -o out.html https://example.com` |
| Status code only  | `curl -s -o /dev/null -w "%{http_code}" https://example.com` |

```
#!/usr/bin/env python3
"""Run: python3 app.py. Open http://localhost:8080/tell-dwp-about-change."""

from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit

PATH = "/tell-dwp-about-change"


def page(title, content):
    return f"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{title}</title>
  <style>
    body {{ font: 20px/1.5 system-ui, sans-serif; color: #17212b;
            max-width: 760px; margin: 64px auto; padding: 0 24px; }}
    h1 {{ line-height: 1.2; }}
    button {{ background: #00703c; color: white; border: 0;
              padding: 12px 24px; font: inherit; cursor: pointer; }}
    code {{ background: #eee; padding: 2px 6px; }}
    a {{ color: #005ea5; }}
  </style>
</head>
<body><main>{content}</main></body>
</html>""".encode("utf-8")


class Handler(BaseHTTPRequestHandler):
    def respond(self, status, title, content):
        body = page(title, content)
        self.send_response(status)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        if urlsplit(self.path).path != PATH:
            self.respond(404, "Not found", "<h1>Page not found</h1>")
            return
        self.respond(200, "Tell DWP about a change", f"""
          <h1>Tell DWP about a change</h1>
          <p>This page returned HTTP <strong>200</strong>.</p>
          <p>Click the button to render an error at the same URL.</p>
          <form method="post" action="{PATH}">
            <button type="submit">Show 500 error</button>
          </form>
        """)

    def do_POST(self):
        if urlsplit(self.path).path != PATH:
            self.respond(404, "Not found", "<h1>Page not found</h1>")
            return
        self.respond(500, "Sorry, there is a problem with the service", f"""
          <h1>Sorry, there is a problem with the service</h1>
          <p>This page returned HTTP <strong>500</strong>.</p>
          <p>The browser URL is still <code>{PATH}</code>.</p>
          <p>This is a separate error template rendered by the same handler.</p>
          <p><a href="{PATH}">Return to the normal page</a></p>
        """)


if __name__ == "__main__":
    server = ThreadingHTTPServer(("127.0.0.1", 8080), Handler)
    print(f"Open http://localhost:8080{PATH}", flush=True)
    print("Press Ctrl+C to stop.", flush=True)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()

```


```
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit

PATH = "/tell-dwp-about-change"


def page(title, content):
    return f"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{title}</title>
  <style>
    body {{ font: 20px/1.5 system-ui, sans-serif; color: #17212b;
            max-width: 760px; margin: 64px auto; padding: 0 24px; }}
    h1 {{ line-height: 1.2; }}
    button {{ background: #00703c; color: white; border: 0;
              padding: 12px 24px; font: inherit; cursor: pointer; }}
    code {{ background: #eee; padding: 2px 6px; }}
    a {{ color: #005ea5; }}
  </style>
</head>
<body><main>{content}</main></body>
</html>""".encode("utf-8")


class Handler(BaseHTTPRequestHandler):
    def respond(self, status, title, content):
        body = page(title, content)
        self.send_response(status)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        if urlsplit(self.path).path == "/500":
            self.respond(500, "Sorry, there is a problem with the service", f"""
              <h1>Sorry, there is a problem with the service</h1>
              <p>This page returned HTTP <strong>500</strong>.</p>
              <p>The browser URL has changed to <code>/500</code>.</p>
              <p>Refresh repeats GET /500 and stays on this error page.</p>
              <p><a href="{PATH}">Return to the normal page</a></p>
            """)
            return
        if urlsplit(self.path).path != PATH:
            self.respond(404, "Not found", "<h1>Page not found</h1>")
            return
        self.respond(200, "Tell DWP about a change", f"""
          <h1>Tell DWP about a change</h1>
          <p>This page returned HTTP <strong>200</strong>.</p>
          <p>Click the button to redirect to a separate error URL.</p>
          <form method="post" action="{PATH}">
            <button type="submit">Redirect to 500 error</button>
          </form>
        """)

    def do_POST(self):
        if urlsplit(self.path).path != PATH:
            self.respond(404, "Not found", "<h1>Page not found</h1>")
            return
        # 303 makes the browser follow the redirect with GET /500.
        self.send_response(303)
        self.send_header("Location", "/500")
        self.send_header("Content-Length", "0")
        self.send_header("Cache-Control", "no-store")
        self.end_headers()



if __name__ == "__main__":
    server = ThreadingHTTPServer(("127.0.0.1", 8081), Handler)
    print(f"Open http://localhost:8081{PATH}", flush=True)
    print("Press Ctrl+C to stop.", flush=True)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()

```

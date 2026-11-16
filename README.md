```
import http from 'node:http';

const host = '127.0.0.1';
const port = 8085;

const server = http.createServer((request, response) => {
  if (request.method === 'GET' && request.url === '/health') {
    response.writeHead(200, { 'content-type': 'text/plain' });
    response.end('Healthy');
    return;
  }

  if (request.method === 'GET' && request.url === '/live') {
    response.writeHead(200, { 'content-type': 'text/plain' });
    response.end('Live');
    return;
  }

  response.writeHead(404, { 'content-type': 'text/plain' });
  response.end('Not Found');
});

server.listen(port, host, () => {
  console.log(`AA Data Publisher mock listening on http://${host}:${port}`);
});

```

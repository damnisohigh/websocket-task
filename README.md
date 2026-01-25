# Real-Time WebSocket Notifications

This project exposes a FastAPI application that provides a `/ws` WebSocket
endpoint capable of broadcasting real-time notifications to every connected
client. It also includes a graceful shutdown routine that defers termination
until all clients disconnect or 30 minutes elapse after receiving a SIGINT /
SIGTERM.

## Prerequisites

- Python 3.13+
- Redis (local or via Docker)

## Local Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Running the API

From the repository root:

```bash
cd app
python3 main.py
```

To run with multiple workers (graceful shutdown works per worker):

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 2
```

Or start (FastAPI) via Docker:

```bash
docker compose up --build
```

## Testing the WebSocket Endpoint

1. Connect any WebSocket client to `ws://localhost:8000/ws`.
2. You'll receive a welcome payload with your `connection_id`.
3. Text messages you send are echoed back with timestamps.
4. Every 10 seconds, while at least one client is connected, the server
   sends a `"test-notification"` broadcast to all clients.

Example with [`websocat`](https://github.com/vi/websocat):

```bash
websocat ws://localhost:8000/ws
```

## Broadcasting on Demand

Use the REST helper endpoint to push a message to all connected sockets:

```bash
curl -X POST http://localhost:8000/ws/broadcast \
  -H "Content-Type: application/json" \
  -d '{"message":"Deployment tonight", "sender":"ops"}'
```

The response lists how many clients received the payload and their IDs.

## Graceful Shutdown Strategy

- `ConnectionManager` keeps track of all local WebSocket connections per worker
  process (works even when `uvicorn` runs multiple workers).
- `NotificationBroadcaster` handles the periodic (10s) `test-notification`
  messages while clients are present.
- When a SIGINT or SIGTERM is delivered, `GracefulShutdownGuard`:
  - logs the connection count and remaining wait time every few seconds,
  - waits until every client disconnects,
  - or force-closes any remaining sockets after 30 minutes.

This guarantees the service only stops when it is safe (no active clients) or
after the configured timeout, satisfying the graceful shutdown requirement.


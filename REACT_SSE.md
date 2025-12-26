# React SSE Client Guide

This guide shows how to connect a React app to the SSE API at `/api/sse`.

## Prerequisites

- A React app (Vite, CRA, Next.js client component, etc).
- Your Laravel app running with the SSE route enabled.

## Basic Usage (hook + component)

```jsx
import { useEffect, useRef, useState } from 'react';

export function useSse(url) {
  const [messages, setMessages] = useState([]);
  const [status, setStatus] = useState('connecting');
  const sourceRef = useRef(null);

  useEffect(() => {
    const source = new EventSource(url);
    sourceRef.current = source;

    source.onopen = () => setStatus('open');

    source.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        setMessages((prev) => [data, ...prev].slice(0, 50));
      } catch {
        setMessages((prev) => [event.data, ...prev].slice(0, 50));
      }
    };

    source.onerror = () => {
      setStatus('error');
      // EventSource reconnects automatically by default.
    };

    return () => {
      source.close();
      setStatus('closed');
    };
  }, [url]);

  return { messages, status };
}

export default function SseViewer() {
  const { messages, status } = useSse('/api/sse');

  return (
    <div>
      <div>SSE status: {status}</div>
      <pre>{JSON.stringify(messages, null, 2)}</pre>
    </div>
  );
}
```

## Using Query Parameters

The SSE endpoint accepts:

- `channel` (string, default: `default`)
- `interval` (ms, minimum: 200)
- `retry` (ms, minimum: 1000)

Example:

```jsx
const { messages, status } = useSse('/api/sse?channel=orders&interval=1000&retry=3000');
```

## Cross-Origin Notes

If your React app runs on a different origin, use the full URL:

```jsx
const { messages, status } = useSse('http://your-host.test/api/sse');
```

Ensure your Laravel CORS settings allow the React app origin for `GET` requests.

## Event Payload Format

Each message is JSON:

```json
{
  "channel": "default",
  "id": 1,
  "timestamp": "2025-01-01T12:00:00Z",
  "uptime_ms": 1234
}
```

## Troubleshooting

- If the connection closes immediately, check your server logs and reverse proxy buffering.
- If no messages arrive, confirm the endpoint is reachable in the browser.
- If the browser blocks the request, verify CORS settings.

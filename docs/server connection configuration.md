# Archipelago Server Connection Configuration

This document describes the network connection handling and configuration options for the Archipelago MultiServer.

## Overview

The Archipelago MultiServer uses WebSocket connections to communicate with clients. The server is built on top of the Python `websockets` library and uses `asyncio` for asynchronous connection handling.

## Maximum Parallel Connections

### Current Architecture

The Archipelago MultiServer **does not impose a hard limit** on the maximum number of parallel network connections. Instead, it relies on:

1. **Operating System Limits**: The underlying OS limits on file descriptors and socket connections
2. **Available Memory**: Each connection consumes memory for state tracking
3. **Asyncio Event Loop**: Python's asyncio handles concurrent connections efficiently using cooperative multitasking

### Connection Tracking

The server tracks all active connections through:
- `Context.endpoints` (list): All connected client endpoints (list of Client objects)
- `Context.clients` (dict): Clients organized by team and slot numbers

### WebSocket Library Parameters

The server uses `websockets.serve()` with the following default parameters relevant to connection handling:

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| `max_queue` | 16 | Maximum length of the queue of incoming connections. If the server is processing the maximum number of connections and `max_queue` additional connections are pending, the server will refuse further connections. |
| `max_size` | 1048576 (1 MB) | Maximum size of a single incoming message in bytes. |
| `ping_interval` | 20 seconds | Interval at which the server sends ping frames to clients. |
| `ping_timeout` | 20 seconds | Timeout for receiving a pong frame from clients. |
| `open_timeout` | 10 seconds | Timeout for the opening handshake. |
| `close_timeout` | 10 seconds | Timeout for the closing handshake. |

**Key Point**: The `max_queue` parameter (default 16) represents the maximum number of connections that can be waiting in the queue while the server is busy handling other connections. This is NOT a limit on total concurrent connections.

## Practical Limits

In practice, the number of concurrent connections is limited by:

1. **System Resources**:
   - Available RAM (each client requires memory for state tracking, items, location checks, etc.)
   - CPU capacity for processing client messages
   - Network bandwidth

2. **Operating System**:
   - Maximum file descriptors (typically 1024 or more on Linux)
   - Socket buffer sizes
   - Network stack capacity

3. **Python Runtime**:
   - Asyncio event loop performance
   - Garbage collection overhead
   - Memory management

### Typical Use Cases

Based on the Archipelago architecture:

- **Small Games (1-10 players)**: No performance concerns
- **Medium Games (11-50 players)**: Server runs smoothly on modest hardware
- **Large Games (51-100+ players)**: May require dedicated server hardware with adequate RAM and CPU

### Known Behaviors

- **Client Reconnections**: Clients can reconnect multiple times; the server maintains state per team/slot
- **Multiple Connections per Slot**: Multiple clients can connect to the same slot (useful for co-op play or multiple devices)
- **Idle Connections**: Kept alive via WebSocket ping/pong mechanism (20-second intervals)

## Configuring Connection Limits

### Modifying WebSocket Parameters

To modify the default WebSocket parameters, you can adjust the `websockets.serve()` call in `MultiServer.py`:

```python
ctx.server = websockets.serve(
    functools.partial(server, ctx=ctx),
    host=ctx.host,
    port=ctx.port,
    ssl=ssl_context,
    extensions=[server_per_message_deflate_factory],
    max_queue=32,  # Increase queue size for high-traffic servers
    max_size=2*1024*1024,  # Increase max message size if needed (2 MB)
)
```

### Operating System Tuning

For hosting large games, consider adjusting OS-level settings:

**Linux (increase file descriptor limit):**
```bash
# Temporary (current session)
ulimit -n 4096

# Permanent (add to /etc/security/limits.conf)
* soft nofile 4096
* hard nofile 8192
```

**Increasing Socket Buffer Sizes:**
```bash
# Increase socket buffer sizes (Linux)
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
```

## Monitoring Connections

You can monitor active connections by:

1. **Server Logs**: Enable `--log_network` flag to see all connection events
2. **In-Game Commands**: Use `!players` or `/players` to see connected players
3. **Server Admin**: Use `/status` to see detailed connection information per team

## Performance Considerations

### Memory Usage

Each connected client consumes memory for:
- WebSocket connection overhead: ~10-50 KB
- Client state (items, locations, hints): Variable, typically 100-500 KB
- Message buffers: Configured by `max_size` parameter

**Estimated Memory per Client**: 200 KB - 1 MB depending on game complexity

### CPU Usage

- Message processing is the primary CPU consumer
- Location check broadcasts scale with player count: O(n²) in worst case
- Hint processing and item distribution create CPU spikes

### Network Bandwidth

- Typical per-client bandwidth: 1-10 KB/s during active play
- Broadcast messages (ItemSend, PrintJSON) multiply with player count
- Message compression is enabled by default (PerMessageDeflate)

## Troubleshooting

### "Too many open files" Error

**Symptom**: Server crashes with "OSError: [Errno 24] Too many open files"

**Solution**: Increase OS file descriptor limit (see Operating System Tuning above)

### Slow Server Response with Many Clients

**Symptom**: Clients experience lag, delayed item receipts

**Possible Solutions**:
- Upgrade server hardware (more RAM, faster CPU)
- Optimize game generation settings (fewer locations/items)
- Consider using dedicated hosting infrastructure

### Connection Refused During Peak Load

**Symptom**: Clients cannot connect when many others are connecting simultaneously

**Solution**: Increase `max_queue` parameter in `websockets.serve()` call:

```python
# In MultiServer.py, around line 2720
ctx.server = websockets.serve(
    functools.partial(server, ctx=ctx),
    host=ctx.host,
    port=ctx.port,
    ssl=ssl_context,
    extensions=[server_per_message_deflate_factory],
    max_queue=32,  # Increase from default 16
)
```

## Recommendations

For hosting Archipelago games:

1. **Small Games (1-10 players)**: Default settings are sufficient
2. **Medium Games (11-50 players)**: Monitor server resources, consider increasing `max_queue` to 32
3. **Large Games (51-100+ players)**: 
   - Use dedicated server with 4+ GB RAM
   - Increase `max_queue` to 64 or higher
   - Monitor and adjust OS limits
   - Consider load balancing for very large events (100+ players)

## Future Enhancements

Potential improvements for connection handling:
- Configurable connection limits via command-line arguments
- Connection rate limiting to prevent abuse
- Connection pooling and load balancing support
- Enhanced monitoring and metrics
- Automatic scaling based on load

## References

- [WebSockets Library Documentation](https://websockets.readthedocs.io/)
- [Python Asyncio Documentation](https://docs.python.org/3/library/asyncio.html)
- [Archipelago Network Protocol](network%20protocol.md)

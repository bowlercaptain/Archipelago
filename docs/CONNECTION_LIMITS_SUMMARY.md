# Archipelago Server Connection Limits - Quick Reference

## TL;DR

**The Archipelago MultiServer does not have a hard-coded maximum connection limit.**

The number of parallel connections is limited only by:
- Available system resources (RAM, CPU, file descriptors)
- WebSocket library's accept queue: **max_queue=16** by default
- Operating system limits

## Key Facts

| Aspect | Value/Description |
|--------|------------------|
| **Hard Connection Limit** | None - dynamically limited by system resources |
| **Accept Queue Size** | 16 pending connections (configurable) |
| **Max Message Size** | 1 MB (1,048,576 bytes) |
| **Ping Interval** | 20 seconds |
| **Connection Timeout** | 10 seconds for handshake |
| **Typical Memory per Client** | 200 KB - 1 MB |

## Practical Capacity

Based on the architecture:

- **1-10 players**: No concerns, works on any hardware
- **11-50 players**: Runs smoothly on modest hardware (2GB+ RAM)
- **51-100+ players**: Requires dedicated server (4GB+ RAM, multi-core CPU)

## How to Increase Limits

If you need to support more connections:

1. **Increase Accept Queue** (in MultiServer.py):
   ```python
   ctx.server = websockets.serve(
       ...,
       max_queue=32,  # Increase from default 16
   )
   ```

2. **Increase OS File Descriptor Limit** (Linux):
   ```bash
   ulimit -n 4096
   ```

3. **Increase Server Resources**:
   - More RAM (primary bottleneck)
   - Faster CPU (for message processing)
   - Better network connection

## Monitoring

Check active connections:
- Server logs: `--log_network` flag
- In-game: `!players` or `/players` command
- Admin: `/status` command

## More Information

For detailed documentation, see: [server connection configuration.md](server%20connection%20configuration.md)

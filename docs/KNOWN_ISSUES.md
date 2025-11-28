# Known Issue: Ingestion MCP Response Delay

## Symptom

When running `ingest_codebase` via Claude Code, the prompt does not regain control even though ingestion completes successfully.

## Evidence

**From logs** (`~/.mcp-servers/logs/pixeltable-memory.log`):
```
16:00:07.026 [DEBUG] ingest_codebase called
16:00:26,084 [INFO] Ingested 530 files in 19.1s  
16:00:26,086 [DEBUG] Response sent  ← Server completes and sends response!
```

**User experience**: Must wait ~48 seconds total before hitting ESC.

**Result**: Data successfully ingested (verified via `check-status.sh`).

## Root Cause

29-second delay AFTER our server sends response but BEFORE Claude Code processes it.

Our code is correct - this is a **Claude Code/MCP framework issue**.

## Workaround

After starting ingestion, wait 25-30 seconds, then press **ESC**. The data will be fully ingested.

Verify with:
```bash
./scripts/check-status.sh
```

## Details

- ✅ Ingestion completes in ~19 seconds
- ✅ MCP server sends response immediately
- ✅ No errors in our code
- ❌ Claude Code doesn't process response for 29+ more seconds

Other tools work because they complete in < 1 second (delay not noticeable).

## References

- Commit fixing event loop blocking: `0fba74b`
- Debug logs: `~/.mcp-servers/logs/pixeltable-memory.log`

# CLR MMF protocol

See `crates/axon-clr/src/protocol.rs` for constants.

## Mapping

- Name: `Local\AXON_CLR_{pid}` (configurable global)
- Size: 4 MiB default
- Header: 128 bytes seqlock (`ClrShmHeader`)
- Command slot: offset `0x80` (512 B)
- Result slot: offset `0x280` (512 B)
- Ring: offset `0x480` … end

## Ring record

```
u16 record_len | u8 op | u8 flags | u64 ts_us | payload[record_len-12]
```

Opcodes: `0x01` field_changed, `0x02` register_snapshot, `0x03` heap_chunk,
`0x04` type_catalog, `0x05` heartbeat, `0x06` overflow.

## Host read

Seqlock: spin while `seq` odd; read; verify `seq` unchanged.

# Enhanced S5.js Design Documentation

This repository contains the architectural design and technical specifications for the Enhanced S5.js project, funded by the Sia Foundation.

## Contents

- **Enhanced S5_js - Revised Code Design.md** - Core architecture for path-based APIs, DAG-CBOR serialisation, and HAMT sharding
- **Enhanced S5_js - Revised Code Design - part II.md** - Media processing pipeline with WASM integration
- **[Coming Soon] Rust Implementation Design** - Cross-implementation compatibility specifications

## About

This project adds advanced directory handling and media support capabilities to the official s5.js library, including:
- Simple path-based storage operations
- Efficient handling of directories with millions of entries
- Client-side thumbnail generation and media processing
- Full compatibility with S5 v1 specification

## Grant Information

Sia Foundation Standard Grant - Enhanced s5.js  
Grant Period: June 2025 - Jan 2026

## Implementation Status

The Enhanced S5.js design is currently being implemented as part of a Sia Foundation grant.

**Active Development Fork**: [github.com/julesl23/s5.js](https://github.com/julesl23/s5.js)

- **Status**: Phase 1 (Core Infrastructure) complete
- **Current Phase**: Phase 2 (Path-Based API) 
- **Expected PR**: End of Milestone 2 (~2 weeks)

Progress is being tracked in the fork's [IMPLEMENTATION.md](https://github.com/julesl23/s5.js/blob/main/docs/IMPLEMENTATION.md).

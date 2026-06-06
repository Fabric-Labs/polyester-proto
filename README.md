# Polyester Public API Artifacts

Public protobuf contracts and generated API artifacts for the Polyester API.

This repository contains the canonical public contract bundle used to generate
Polyester language SDKs and API documentation.

- `proto/common`: shared public-safe protobuf dependencies
- `proto/public`: public Polyester API protobuf contracts
- `.prototools`: local Buf tool shim configuration
- `buf.yaml` and `buf.lock`: Buf workspace config for generation and checks
- `descriptor/public.pb`: public FileDescriptorSet
- `openapi`: public OpenAPI outputs
- `docs`: generated public developer-portal artifacts

## Validation

```bash
buf lint
```

## Maintenance

This repository is maintained by Fabric Labs and updated as the public API
evolves.

# Basalt on Coolify - RTX PRO 4000 Blackwell (24 GB VRAM, 64 GB RAM)

> Before deploy: copy `basalt_config.json` from this repo to `/opt/basalt-config/basalt_config.json` on the Coolify host. The container reads that host copy; `allowed-host` is already set to the public domain Coolify's proxy sends as the `Host` header. Re-copy after any edit and redeploy.

One model at a time. Stop PolyStrata first if it is still installed on this machine.

## Before the first start (on the host)

```sh
mkdir -p /opt/basalt-models /opt/basalt-config
cp /path/to/cloned/basalt/basalt_config.json /opt/basalt-config/basalt_config.json
pip install -U huggingface_hub
hf download jesdga/Qwen3.8-Flash-Next-Basalt Qwen3.8-FN-NVFP4-DQ6_K.basalt \
   --local-dir /opt/basalt-models
```

The config bind mount is an absolute host path, not `./basalt_config.json`, so it does not depend on the directory Coolify runs Compose from. If that path did not exist at an earlier deploy, Docker created a *directory* there; remove it with `rmdir /opt/basalt-config/basalt_config.json` before copying the file.

Fallback pack: `Qwen3.8-FN-UD-IQ3_XXS.basalt` (87.6 GB), downloaded the same way.
To use it instead, set `"model"` in `/opt/basalt-config/basalt_config.json` to `/models/Qwen3.8-FN-UD-IQ3_XXS.basalt` and redeploy.

## GO / NO-GO gate

Before pointing traffic at the pack, run it through the engine on this server:

```sh
basalt-bench --community --model /opt/basalt-models/Qwen3.8-FN-NVFP4-DQ6_K.basalt --submitter NAME
```

`basalt-bench --community` refuses to write a row if any expert would be read from the SSD. On 64 GB RAM the NVFP4 pack (63.3 GiB of experts) is the one at risk; UD-IQ3_XXS (40 GiB of frames) is the safe pick. If the gate fails, free RAM or switch to UD-IQ3_XXS; do not deploy a pack that spills to disk.

## Coolify

1. Create the app as **Docker Compose** from this repository.
2. Add the environment variable `BASALT_API_KEY` with a long random secret. No key is stored in any file here.
3. Deploy, then verify:

```sh
curl -s -H "Authorization: Bearer $KEY" https://sa5ofjawgf1p1ncibwts5qji.62.238.92.182.sslip.io/v1/models
```

`KEY` is the value of `BASALT_API_KEY`.

## Exposure note

`--api-key` protects only `/v1/*`. `/health`, `/metrics`, `/slots`, `/api/logs` and the stats page are open behind the proxy. Add basic auth or a Coolify proxy rule if that matters.

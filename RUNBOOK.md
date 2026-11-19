# Basalt on Coolify - RTX PRO 4000 Blackwell (24 GB VRAM, 64 GB RAM)

> The container reads `basalt-data/basalt_config.json` from this repo, baked into the image at `/data/basalt_config.json`. `allowed-host` is already set to the public domain Coolify's proxy sends as the `Host` header. Edit the config in the repo, commit and redeploy so the image is rebuilt; no host-side config file is needed.

One model at a time. Stop PolyStrata first if it is still installed on this machine.

## Before the first start (on the host)

```sh
mkdir -p /opt/basalt-models
pip install -U huggingface_hub
hf download jesdga/Qwen3.8-Flash-Next-Basalt Qwen3.8-FN-NVFP4-DQ6_K.basalt \
   --local-dir /opt/basalt-models
```

The config is copied into the image by the Dockerfile, so no config bind mount is involved. The Docker host still needs to see `/opt/basalt-models` for the model mount; if it cannot, point that mount at a host path Coolify can see.

Fallback pack: `Qwen3.8-FN-UD-IQ3_XXS.basalt` (87.6 GB), downloaded the same way.
To use it instead, set `"model"` in `basalt-data/basalt_config.json`, commit and redeploy.

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

# Various docker images I use

## pytorch

based on official pytorch/pytorch image, with some additions based on jupyter
stacks images.

```
DOCKER_BUILDKIT=1 docker build -t ejsmit/pytorch:latest pytorch
docker tag ejsmit/pytorch:latest ejsmit/pytorch:1.4-cuda10.1-cudnn7-runtime
docker tag ejsmit/pytorch:latest ejsmit/pytorch:1.4
```

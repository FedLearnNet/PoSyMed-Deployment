# PoSyMed is now FL-Net

PoSyMed has been merged into FL-Net. All PoSyMed features are available in
FL-Net, which also adds federated learning. This repository no longer deploys
PoSyMed. It only serves a static notice page for old PoSyMed links, for example
links from the PoSyMed publication.

To deploy the platform, use the FL-Net Platform deployment:

- [FL-Net Platform deployment repository](https://github.com/FedLearnNet/FL-Net-Platform-Deployment)
- [FL-Net self-deployment guide](https://federated-learning.net/documentation/docs/deployment/self-deployment-guide)

The original PoSyMed compose stack is kept for reference at the tag
[`posymed-final`](https://github.com/FedLearnNet/PoSyMed-Deployment/tree/posymed-final).
It is no longer maintained, and its `env/` and Keycloak realm files were removed.

## Notice page

```sh
docker compose up -d
```

The page is served on port `8291`. Every other path redirects to the notice.

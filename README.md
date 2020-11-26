# Platforming Blockchain: Ethereum


<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/Blockchain_private_Ethereum_network.png" width="800">
<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/Blockchain-Ethereum-stats-private-Ethereum-nodes-reporting.png" width="800">





## Technology Reference
- [Ethereum]
- [Geth]
- [Ethstats]

## Listings

- [Listing : Bootnode Service](cluster-davar-eth/100-eth/10-bootnode/10-service.yml)
- [Listing : Bootnode Deployment](cluster-davar-eth/100-eth/10-bootnode/30-deployment.yml)
- [Listing : Bootnode Registrar Service](cluster-davar-eth/100-eth/20-bootnode-reg/10-service.yml)
- [Listing : Bootnode Deployment](cluster-davar-eth/100-eth/20-bootnode-reg/30-deployment.yml)
- [Listing : Ethstats Service](cluster-davar-eth/100-eth/30-ethstats/10-service.yml)
- [Listing : Ethstats Secret](cluster-davar-eth/100-eth/30-ethstats/15-secret.yml)
- [Listing : Ethstats Deployment](cluster-davar-eth/100-eth/30-ethstats/30-deployment.yml)
- [Listing : Ethstats Ingress](cluster-davar-eth/100-eth/30-ethstats/50-ingress.yml)
- [Listing : Geth Secret](cluster-davar-eth/100-eth/40-miner/15-secret.yml)
- [Listing : Geth ConfigMap](cluster-davar-eth/100-eth/40-miner/20-configmap.yml)
- [Listing : Geth Deployment](cluster-davar-eth/100-eth/40-miner/30-deployment.yml)
- [Listing : Geth transaction node Service](cluster-davar-eth/100-eth/50-tx/10-service.yml)
- [Listing : Geth transaction node Deployment](cluster-davar-eth/100-eth/50-tx/30-deployment.yml)
- [Listing : Function for returning details on the last block in the Blockchain](/utils/function/last-block/handler.py)<!-- @IGNORE PREVIOUS: link -->

[Ethstats]: https://github.com/cubedro/eth-netstats
[Geth]: https://geth.ethereum.org/
[Ethereum]: https://ethereum.org/en/

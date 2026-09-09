# Chapitre 1 — Presentation de Frax Ether

Frax Ether est le liquid staking token d ETH de l ecosysteme Frax. Contrairement a des protocoles comme Lido (stETH, rebasing) ou Rocket Pool (rETH, share-based), Frax Ether separe deliberement le token de depot du token de rendement en deux contrats distincts : frxETH et sfrxETH.

frxETH.sol est un token ERC-20 simple, non-rebasant, frappe 1 pour 1 contre de l ETH depose. Il ne genere aucun rendement par lui-meme : detenir du frxETH brut equivaut a detenir de l ETH sans le staker. Pour toucher le rendement du staking, l utilisateur doit deposer son frxETH dans le second contrat, sfrxETH.sol, un coffre ERC-4626 qui applique un taux de change croissant entre frxETH et sfrxETH au fil du temps.

Cette separation en deux tokens (mint token + vault token) plutot qu un unique token rebasant est le choix architectural central du protocole. Elle permet a frxETH de rester parfaitement compatible avec tout systeme DeFi qui ne gere pas bien les tokens rebasants (AMM, protocoles de pret), tout en offrant via sfrxETH un vehicule de rendement conforme au standard ERC-4626 largement supporte.

Le depot ne contient que 5 contrats non-triviaux (365 lignes de code au total selon le README du contest d audit) : ERC20PermitPermissionedMint.sol (parent de frxETH), frxETH.sol, frxETHMinter.sol, OperatorRegistry.sol et sfrxETH.sol. C est une base de code volontairement minimaliste, focalisee sur une seule fonction : convertir de l ETH en frxETH, puis optionnellement en sfrxETH.

Ce parcours s appuie sur le depot clone a la date d ecriture (FraxFinance/frxETH-public, branche master).

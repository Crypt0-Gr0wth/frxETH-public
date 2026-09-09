# Chapitre 10 — Comparaison avec les autres liquid staking tokens

Le corpus de ce parcours couvre deja Lido (stETH) et Rocket Pool (rETH), ce qui permet de situer precisement le choix architectural de Frax.

Lido : stETH est un token rebasant — le solde de chaque detenteur augmente automatiquement a chaque rebase, ce qui pose des problemes d integration avec des protocoles DeFi qui ne gerent pas les changements de solde spontanes (d ou l existence de wstETH, un wrapper non-rebasant construit par-dessus).

Rocket Pool : rETH suit deja un modele a taux de change croissant (share-based), proche dans l esprit de sfrxETH, mais integre au sein d un seul token — pas de separation entre un token de depot et un token de rendement.

Frax : separe explicitement les deux roles des le depart. frxETH reste un ancrage de valeur simple et stable (1:1 avec l ETH deposant), pendant que sfrxETH, un coffre ERC-4626 distinct, porte seul la complexite du taux de change croissant. Cette separation permet a frxETH de servir de brique de base neutre (par exemple comme collateral dans un protocole de pret qui prefererait ne pas gerer un taux de change variable), tout en offrant sfrxETH comme vehicule de rendement optionnel et composable nativement avec l ecosysteme ERC-4626.

Le prix a payer est une etape supplementaire pour l utilisateur qui veut du rendement : deposer son frxETH dans sfrxETH, plutot que de detenir directement un token porteur de rendement des la conversion de l ETH.

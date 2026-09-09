# Chapitre 6 — sfrxETH : le coffre ERC-4626 de staking

sfrxETH.sol est le contrat qui transforme du frxETH statique en une position generatrice de rendement. Il herite de xERC4626 (une extension du standard ERC-4626 de Solmate), qui lui-meme herite de ERC4626.

Le constructeur fixe le nom (Staked Frax Ether), le symbole (sfrxETH), et un parametre cle : rewardsCycleLength, la duree en secondes d un cycle de distribution de recompenses (immutable, fixee au deploiement).

sfrxETH surcharge les quatre fonctions standards d un vault ERC-4626 — deposit, mint, withdraw, redeem — pour y ajouter le modifier andSync, qui declenche automatiquement syncRewards() si le cycle de recompenses en cours est termine (block.timestamp >= rewardsCycleEnd) avant d executer l operation demandee. Cela evite qu un utilisateur doive attendre qu un tiers appelle syncRewards() manuellement pour beneficier d un taux de change a jour.

pricePerShare() est un raccourci pratique qui retourne convertToAssets(1e18), c est-a-dire combien de frxETH vaut une unite de sfrxETH — la mesure directe du rendement accumule depuis le depot initial. depositWithSignature() ajoute un chemin de depot en une seule transaction via un permit EIP-2612 sur l asset (frxETH), evitant une transaction d approve separee.

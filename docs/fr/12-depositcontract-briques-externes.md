# Chapitre 12 — Le token DepositContract et les briques externes

DepositContract.sol, present dans le depot, n est pas un contrat proprietaire de Frax : c est une copie du contrat officiel de depot ETH 2.0, deploye par la Fondation Ethereum, integre ici uniquement pour permettre de tester le flux complet localement (fork de mainnet dans les tests Foundry). Il n a pas ete audite dans le cadre de ce projet car il s agit d une infrastructure Ethereum core, deja largement verifiee independamment.

Owned.sol (cree par Synthetix.io) et une partie des contrats ERC20 heritent de bibliotheques externes deja largement auditees ailleurs — OpenZeppelin pour ERC20Permit et ERC20Burnable, Solmate pour le socle ERC4626 dont herite xERC4626. Le README classe explicitement ces briques dans les contrats inclus mais hors du perimetre d audit specifique au projet.

SigUtils.sol n est utilise que dans les tests (generation de signatures EIP-712 pour valider les flux de permit) et n a aucune presence en production.

Cette organisation — code proprietaire minimal, briques externes standard non reecrites — explique pourquoi la surface auditee de Frax Ether tient en seulement 365 lignes malgre un systeme complet de mint, staking et gestion de validateurs.

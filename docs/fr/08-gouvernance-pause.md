# Chapitre 8 — Gouvernance et pause : le modele a deux cles

Un motif se repete a l identique dans ERC20PermitPermissionedMint.sol, OperatorRegistry.sol et frxETHMinter.sol (via son heritage d OperatorRegistry) : le modifier onlyByOwnGov, qui autorise une action si msg.sender est soit owner (via Owned.sol de Synthetix) soit timelock_address, une adresse distincte destinee a heberger un contrat de timelock on-chain.

Ce modele a deux cles separe une gouvernance rapide (owner, souvent un multisig d equipe) d une gouvernance plus lente et transparente (timelock, qui impose un delai visible avant l execution d une action sensible). Les deux peuvent agir independamment sur les memes fonctions protegees, ce qui est un choix pragmatique plutot qu un vrai systeme de vote on-chain — Frax garde la gouvernance fine (parametres du protocole) hors du perimetre de ce depot, qui se concentre sur le mint/stake d ETH.

frxETHMinter expose deux flags de pause independants, submitPaused et depositEtherPaused, bascules via togglePauseSubmits et togglePauseDepositEther, tous deux proteges par onlyByOwnGov. Cette granularite permet par exemple de bloquer l acceptation de nouveaux depots ETH tout en continuant a permettre le staking de l ETH deja accumule, ou l inverse.

recoverEther et recoverERC20 sont des fonctions d urgence classiques pour recuperer des fonds envoyes par erreur au contrat, egalement proteges par onlyByOwnGov.

# Chapitre 4 — depositEther et OperatorRegistry : la pile de validateurs

Une fois que suffisamment d ETH s est accumule dans frxETHMinter, quelqu un (generalement un bot, sans permission particuliere requise) appelle depositEther(max_deposits) pour effectivement staker cet ETH via le contrat officiel ETH 2.0 DepositContract.

Le calcul est simple : numDeposits = (balance du contrat - currentWithheldETH) / DEPOSIT_SIZE, avec DEPOSIT_SIZE fixe a 32 ether. Le parametre max_deposits permet de plafonner le nombre de depots traites en une seule transaction, pour eviter un depassement de gas si un tres gros montant s est accumule d un coup.

Chaque iteration de la boucle appelle getNextValidator(), une fonction interne d OperatorRegistry qui depile (pop) le dernier validateur du tableau validators — une structure LIFO (last-in, first-out), pas une file FIFO. Chaque Validator contient pubKey, signature et depositDataRoot ; le withdrawalCredential courant (curr_withdrawal_pubkey) est commun a tous les validateurs et stocke separement.

Le mapping activeValidators trackee par pubKey empeche de deposer une seconde fois 32 ETH sur un validateur deja actif, ce qui bloquerait ces fonds jusqu a l activation des retraits. OperatorRegistry expose aussi des fonctions de maintenance reservees a la gouvernance (addValidator, addValidators, removeValidator, swapValidator, popValidators, clearValidatorArray) — le commentaire du code precise explicitement que la validite des cles et signatures n est PAS verifiee on-chain pour economiser du gas ; c est une verification qui doit etre faite off-chain avant l ajout, le DepositContract officiel d Ethereum servant de derniere ligne de defense en cas d erreur.
